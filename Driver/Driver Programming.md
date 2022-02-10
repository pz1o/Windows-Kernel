记录我经常用到的东西

# 0x01 struct

## 1. _DRIVER_OBJECT

```
kd> dt nt!_DRIVER_OBJECT
   +0x000 Type             : Int2B
   +0x002 Size             : Int2B
   +0x008 DeviceObject     : Ptr64 _DEVICE_OBJECT
   +0x010 Flags            : Uint4B
   +0x018 DriverStart      : Ptr64 Void
   +0x020 DriverSize       : Uint4B
   +0x028 DriverSection    : Ptr64 Void
   +0x030 DriverExtension  : Ptr64 _DRIVER_EXTENSION
   +0x038 DriverName       : _UNICODE_STRING
   +0x048 HardwareDatabase : Ptr64 _UNICODE_STRING
   +0x050 FastIoDispatch   : Ptr64 _FAST_IO_DISPATCH
   +0x058 DriverInit       : Ptr64     long 
   +0x060 DriverStartIo    : Ptr64     void 
   +0x068 DriverUnload     : Ptr64     void 
   +0x070 MajorFunction    : [28] Ptr64     long 
```



## 2. _LDR_DATA_TABLE_ENTRY

```
kd> dt _LDR_DATA_TABLE_ENTRY
nt!_LDR_DATA_TABLE_ENTRY
   +0x000 InLoadOrderLinks : _LIST_ENTRY
   +0x010 InMemoryOrderLinks : _LIST_ENTRY
   +0x020 InInitializationOrderLinks : _LIST_ENTRY
   +0x030 DllBase          : Ptr64 Void
   +0x038 EntryPoint       : Ptr64 Void
   +0x040 SizeOfImage      : Uint4B
   +0x048 FullDllName      : _UNICODE_STRING
   +0x058 BaseDllName      : _UNICODE_STRING
   +0x068 FlagGroup        : [4] UChar
   +0x068 Flags            : Uint4B
   +0x068 PackagedBinary   : Pos 0, 1 Bit
   +0x068 MarkedForRemoval : Pos 1, 1 Bit
   +0x068 ImageDll         : Pos 2, 1 Bit
   +0x068 LoadNotificationsSent : Pos 3, 1 Bit
   +0x068 TelemetryEntryProcessed : Pos 4, 1 Bit
   +0x068 ProcessStaticImport : Pos 5, 1 Bit
   +0x068 InLegacyLists    : Pos 6, 1 Bit
   +0x068 InIndexes        : Pos 7, 1 Bit
   +0x068 ShimDll          : Pos 8, 1 Bit
   +0x068 InExceptionTable : Pos 9, 1 Bit
   +0x068 ReservedFlags1   : Pos 10, 2 Bits
   +0x068 LoadInProgress   : Pos 12, 1 Bit
   +0x068 LoadConfigProcessed : Pos 13, 1 Bit
   +0x068 EntryProcessed   : Pos 14, 1 Bit
   +0x068 ProtectDelayLoad : Pos 15, 1 Bit
   +0x068 ReservedFlags3   : Pos 16, 2 Bits
   +0x068 DontCallForThreads : Pos 18, 1 Bit
   +0x068 ProcessAttachCalled : Pos 19, 1 Bit
   +0x068 ProcessAttachFailed : Pos 20, 1 Bit
   +0x068 CorDeferredValidate : Pos 21, 1 Bit
   +0x068 CorImage         : Pos 22, 1 Bit
   +0x068 DontRelocate     : Pos 23, 1 Bit
   +0x068 CorILOnly        : Pos 24, 1 Bit
   +0x068 ChpeImage        : Pos 25, 1 Bit
   +0x068 ReservedFlags5   : Pos 26, 2 Bits
   +0x068 Redirected       : Pos 28, 1 Bit
   +0x068 ReservedFlags6   : Pos 29, 2 Bits
   +0x068 CompatDatabaseProcessed : Pos 31, 1 Bit
   +0x06c ObsoleteLoadCount : Uint2B
   +0x06e TlsIndex         : Uint2B
   +0x070 HashLinks        : _LIST_ENTRY
   +0x080 TimeDateStamp    : Uint4B
   +0x088 EntryPointActivationContext : Ptr64 _ACTIVATION_CONTEXT
   +0x090 Lock             : Ptr64 Void
   +0x098 DdagNode         : Ptr64 _LDR_DDAG_NODE
   +0x0a0 NodeModuleLink   : _LIST_ENTRY
   +0x0b0 LoadContext      : Ptr64 _LDRP_LOAD_CONTEXT
   +0x0b8 ParentDllBase    : Ptr64 Void
   +0x0c0 SwitchBackContext : Ptr64 Void
   +0x0c8 BaseAddressIndexNode : _RTL_BALANCED_NODE
   +0x0e0 MappingInfoIndexNode : _RTL_BALANCED_NODE
   +0x0f8 OriginalBase     : Uint8B
   +0x100 LoadTime         : _LARGE_INTEGER
   +0x108 BaseNameHashValue : Uint4B
   +0x10c LoadReason       : _LDR_DLL_LOAD_REASON
   +0x110 ImplicitPathOptions : Uint4B
   +0x114 ReferenceCount   : Uint4B
   +0x118 DependentLoadFlags : Uint4B
   +0x11c SigningLevel     : UChar

```

DLLbase可以获得模块路径

# 0x02 function

## 1. 遍历内核文件中所有内核模块

### DRIVER_SECTION

```c
#include "ntddk.h"
#include <WinDef.h>
#define USHORT_MAX 1024
typedef enum
{
    MmTagTypeDS = 'M1KE'             //PrintAllLoadedMoudleByDriverSection
}MmTagType;


typedef struct _LDR_DATA_TABLE_ENTRY {
    LIST_ENTRY InLoadOrderLinks;
    LIST_ENTRY InMemoryOrderLinks;
    LIST_ENTRY InInitializationOrderLinks;
    PVOID DllBase;
    PVOID EntryPoint;
    ULONG SizeOfImage;
    UNICODE_STRING FullDllName;
    UNICODE_STRING BaseDllName;
    union {
        ULONG FlagGroup;
        ULONG Flags;
    };
}LDR_DATA_TABLE_ENTRY, * PLDR_DATA_TABLE_ENTRY;

NTSTATUS PrintAllLoadedMoudleByDriverSection(PDRIVER_OBJECT pDriverOjbect)
{
    NTSTATUS ntStatus = STATUS_SUCCESS;
    if ((pDriverOjbect) && (pDriverOjbect->DriverSection))
    {
        ANSI_STRING asBaseDllName = { 0 };
        ANSI_STRING asFullDllName = { 0 };
        PCHAR pBaseDllNameBufer = NULL;
        PCHAR pFullDllNameBufer = NULL;
        do
        {
            pBaseDllNameBufer = (PCHAR)ExAllocatePoolWithTag(PagedPool, USHORT_MAX, MmTagTypeDS);
            if (pBaseDllNameBufer == NULL)
            {
                DbgPrint("【PrintLoadedModule】 pBaseDllNameBufer Allocate Memory Failed!\r\n");
                ntStatus = STATUS_INSUFFICIENT_RESOURCES;
                break;
            }
            RtlZeroMemory(pBaseDllNameBufer, USHORT_MAX);
            pFullDllNameBufer = (PCHAR)ExAllocatePoolWithTag(PagedPool, USHORT_MAX, MmTagTypeDS);
            if (pFullDllNameBufer == NULL)
            {
                DbgPrint("【PrintLoadedModule】 pFullDllNameBufer Allocate Memory Failed!\r\n");
                ntStatus = STATUS_INSUFFICIENT_RESOURCES;
                break;
            }
            RtlZeroMemory(pFullDllNameBufer, USHORT_MAX);
            RtlInitEmptyAnsiString(&asBaseDllName, pBaseDllNameBufer, USHORT_MAX);
            RtlInitEmptyAnsiString(&asFullDllName, pFullDllNameBufer, USHORT_MAX);
            PLDR_DATA_TABLE_ENTRY pldte = (PLDR_DATA_TABLE_ENTRY)pDriverOjbect->DriverSection;
            if (pldte != NULL)
            {
                const PLIST_ENTRY pListHeaderInLoadOrder = pldte->InLoadOrderLinks.Flink;
                if (pListHeaderInLoadOrder != NULL)
                {
                    ULONG ulCount = 0;
                    PLIST_ENTRY pListTemp = pListHeaderInLoadOrder;
                    do
                    {
                        PLDR_DATA_TABLE_ENTRY pldteTemp = (PLDR_DATA_TABLE_ENTRY)CONTAINING_RECORD(pListTemp, LDR_DATA_TABLE_ENTRY,InLoadOrderLinks);
                        if ((pldteTemp != NULL) &&
                            (pldteTemp->BaseDllName.Buffer != NULL) &&
                            (pldteTemp->FullDllName.Buffer != NULL))
                        {
                            //1.直接通过DbgPrint(Ex的 %wZ参数打印 BaseDllName和FullDllName时在XP系统下当时文件路径
                            // 中有中文的时候会截断，故先转化为ANSI类型再打印,如不需要打印则可将相关ANSI类型的定义
                            // 初始化以及转换去掉
                            //2.节点遍历过程中有一个空节点，其中字符串的Buffer为空,通过BaseDllName.Buffer以及
                            //FullDllName.Buffer为空来跳过
                            //DbgPrint("【PrintLoadedModule】","Name:%-30wZ Path:%wZ\r\n",
                            //                                                              &pldteTemp->BaseDllName, &pldteTemp->FullDllName);
                            RtlUnicodeStringToAnsiString(&asBaseDllName, &pldteTemp->BaseDllName, FALSE);
                            RtlUnicodeStringToAnsiString(&asFullDllName, &pldteTemp->FullDllName, FALSE);
                            DbgPrint("【PrintLoadedModule】 Name:%-30Z Path:%-130Z Base:0x%p\r\n",
                                &asBaseDllName, &asFullDllName, pldteTemp->DllBase);

                            ulCount++;
                        }
                        pListTemp = pListTemp->Flink;

                    } while (pListTemp != pListHeaderInLoadOrder);

                    DbgPrint("【PrintLoadedModule】 共计%ud个内核模块!\r\n", ulCount);
                }
            }
        } while (FALSE);
        if (pBaseDllNameBufer)
        {
            ExFreePoolWithTag(pBaseDllNameBufer, MmTagTypeDS);
            pBaseDllNameBufer = NULL;
        }
        if (pFullDllNameBufer)
        {
            ExFreePoolWithTag(pFullDllNameBufer, MmTagTypeDS);
            pFullDllNameBufer = NULL;
        }
    }
    return ntStatus;
}


EXTERN_C NTSTATUS  DriverEntry(PDRIVER_OBJECT pDriverObject,
    PUNICODE_STRING pRegistryPath)
{
    PrintAllLoadedMoudleByDriverSection(pDriverObject);
    return STATUS_SUCCESS;
}
```

## ZwQuerySytemInformation

```c
#include "ntddk.h"
#include <WinDef.h>
#define USHORT_MAX 1024
typedef enum
{
    MmTagTypeQS = 'M1KE'             //PrintAllLoadedMoudleByDriverSection
}MmTagType;

typedef enum _SYSTEM_INFORMATION_CLASS
{
    SystemModuleInformation = 11
} SYSTEM_INFORMATION_CLASS;

typedef struct _SYSTEM_MODULE_INFORMATION_ENTRY64 {
    ULONG Reserved[4];
    PVOID Base;
    ULONG Size;
    ULONG Flags;
    USHORT Index;
    USHORT Unknown;
    USHORT LoadCount;
    USHORT ModuleNameOffset;
    CHAR ImageName[256];
} SYSTEM_MODULE_INFORMATION_ENTRY64, * PSYSTEM_MODULE_INFORMATION_ENTRY64;

extern NTSTATUS  ZwQuerySystemInformation(
    IN SYSTEM_INFORMATION_CLASS SystemInformationClass,
    OUT PVOID SystemInformation,
    IN ULONG SystemInformationLength,
    OUT PULONG ReturnLength);
typedef struct _SYSTEM_MODULE_INFORMATION
{
    ULONG Count;//内核中以加载的模块的个数
#ifdef _AMD64_
    SYSTEM_MODULE_INFORMATION_ENTRY64 Module[1];
#else
    SYSTEM_MODULE_INFORMATION_ENTRY32 Module[1];
#endif

} SYSTEM_MODULE_INFORMATION, * PSYSTEM_MODULE_INFORMATION;
NTSTATUS PrintAllLoadedMoudleByZwQuerySystemInformation()
{
    ULONG ulInfoLength = 0;
    PVOID pBuffer = NULL;
    NTSTATUS ntStatus = STATUS_UNSUCCESSFUL;
    DbgPrint("[+] Enter.....\r\n");
    do
    {
        ntStatus = ZwQuerySystemInformation(SystemModuleInformation,
            NULL,
            NULL,
            &ulInfoLength);
        if ((ntStatus == STATUS_INFO_LENGTH_MISMATCH))
        {
            pBuffer = ExAllocatePoolWithTag(PagedPool, ulInfoLength, MmTagTypeQS);
            if (pBuffer == NULL)
            {
                DbgPrint("[+] Allocate Memory Failed\r\n");
                break;
            }
            ntStatus = ZwQuerySystemInformation(SystemModuleInformation,
                pBuffer,
                ulInfoLength,
                &ulInfoLength);
            if (!NT_SUCCESS(ntStatus))
            {
                DbgPrint("[+] ZwQuerySystemInformation Failed\r\n");
                break;
            }

            PSYSTEM_MODULE_INFORMATION pModuleInformation = (PSYSTEM_MODULE_INFORMATION)pBuffer;
            if (pModuleInformation)
            {
                for (ULONG i = 0; i < pModuleInformation->Count; i++)
                {
                    DbgPrint("[+] Image:%-50s Base:0x%p\r\n",
                        pModuleInformation->Module[i].ImageName, pModuleInformation->Module[i].Base);
                }
                DbgPrint("[+] 共计%ud个内核模块!\r\n", pModuleInformation->Count);
            }

            ntStatus = STATUS_SUCCESS;
        }
    } while (FALSE);

    if (pBuffer)
    {
        ExFreePoolWithTag(pBuffer, MmTagTypeQS);
    }

    return ntStatus;
}
EXTERN_C NTSTATUS  DriverEntry(PDRIVER_OBJECT pDriverObject,
    PUNICODE_STRING pRegistryPath)
{
    PrintAllLoadedMoudleByZwQuerySystemInformation();
    return STATUS_SUCCESS;
}
```

