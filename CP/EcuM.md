



```mermaid
stateDiagram
  [*] --> ECUM_STATE_STARTUP_ONE : EcuM_Init
  ECUM_STATE_STARTUP_ONE --> ECUM_STATE_STARTUP_TWO : EcuM_StartupTwo
  ECUM_STATE_STARTUP_TWO --> ECUM_STATE_RUN
  ECUM_STATE_RUN --> ECUM_STATE_OFF : EcuM_GoDown
  ECUM_STATE_RUN --> ECUM_STATE_GO_SLEEP : EcuM_GoSleep
  ECUM_STATE_GO_SLEEP --> ECUM_STATE_SLEEP
  ECUM_STATE_SLEEP --> ECUM_STATE_WAKEUP_VALIDATION : EcuM_SetWakeupEvent
  ECUM_STATE_WAKEUP_VALIDATION --> ECUM_STATE_RUN : [EcuM_ValidateWakeupEvent]
```





| WakeupSource                 |        |
| ---------------------------- | ------ |
| ECUM_WKSOURCE_POWER          | Fix    |
| ECUM_WKSOURCE_RESET          | Fix    |
| ECUM_WKSOURCE_INTERNAL_RESET | Fix    |
| ECUM_WKSOURCE_INTERNAL_WDG   | Fix    |
| ECUM_WKSOURCE_EXTERNAL_WDG   | Fix    |
| ECUM_WKSOURCE_XXX            | Config |



| WakeupStatus            |                                      |
| ----------------------- | ------------------------------------ |
| ECUM_WKSTATUS_NONE      | 没有检测到唤醒事件或者唤醒事件被清除 |
| ECUM_WKSTATUS_PENDING   | 检测到唤醒事件，但尚未验证           |
| ECUM_WKSTATUS_VALIDATED | 检测到唤醒事件，并成功验证           |
| ECUM_WKSTATUS_EXPIRED   | 检测到唤醒事件，但验证失败           |
| ECUM_WKSTATUS_DISABLED  | 唤醒源被禁止                         |
| ECUM_WKSTATUS_ENABLED   | 唤醒源被使能                         |



```mermaid
stateDiagram
  ECUM_WKSTATUS_ENABLED --> ECUM_WKSTATUS_VALIDATED : EcuM_SetWakeupEvent
  ECUM_WKSTATUS_ENABLED --> ECUM_WKSTATUS_PENDING : EcuM_SetWakeupEvent
  ECUM_WKSTATUS_DISABLED --> ECUM_WKSTATUS_ENABLED : GoSleep
    
  state ECUM_WKSTATUS_DISABLED {
    state Validation  {
      ECUM_WKSTATUS_PENDING --> ECUM_WKSTATUS_VALIDATED : EcuM_ValidateWakeupEvent
      ECUM_WKSTATUS_PENDING --> ECUM_WKSTATUS_EXPIRED : timeout
    }

    Validation --> ECUM_WKSTATUS_NONE : EcuM_ClearWakeupEvent
  }
 
  [*] --> ECUM_WKSTATUS_NONE
```



Startup：

```mermaid
sequenceDiagram

participant startup
participant main
participant Os
participant Rte
participant BswM
participant EcuM
participant EcuM_Callouts
participant EcuM_Cfg
participant Bsw
participant Mcal

startup->>main:main
  main->>EcuM:EcuM_Init()
    Note over EcuM:State = STARTUP_ONE
    EcuM->>EcuM_Callouts:EcuM_AL_DriverInitZero()
      EcuM_Callouts->>EcuM_Cfg:EcuM_DefaultInitListZero()
        EcuM_Cfg->>Bsw:Det_Init()
        EcuM_Cfg->>Bsw:Dem_PreInit()
    EcuM->>EcuM_Callouts:EcuM_AL_SetProgrammableInterrupts()
    EcuM->>EcuM_Callouts:EcuM_AL_DriverInitOne()
      EcuM_Callouts->>EcuM_Cfg:EcuM_DefaultInitListOne()
        EcuM_Cfg->>Mcal:Mcu_Init()
        EcuM_Cfg->>Mcal:Port_Init()
        EcuM_Cfg->>Mcal:Gpt_Init()
        EcuM_Cfg->>Mcal:Wdg_Init()
        EcuM_Cfg->>Mcal:Adc_Init()
        EcuM_Cfg->>Mcal:Icu_Init()
        EcuM_Cfg->>Mcal:Pwm_Init()
        EcuM_Cfg->>Mcal:Ocu_Init()
    EcuM->>Mcal:Mcu_GetResetReason()
    EcuM->>EcuM:EcuM_SetWakeupEvent()
    EcuM->>Os:StartOS()
      Os->>main:StartupHook()
      Os->>Os:ActivateTask(Init)

Note over Os:Init_Task
Os->>EcuM:EcuM_StartupTwo()
  Note over EcuM:State = STARTUP_TWO
  EcuM->>Rte:SchM_Init()
    Rte->>Os:ActivateTask(Bsw)
  EcuM->>BswM:BswM_Init()
  Note over EcuM:State = RUN
  EcuM->>BswM:BswM_EcuM_CurrentState(RUN)

Note over Os:Bsw_Task
Os->>BswM:BswM_MainFunction()
  BswM->>Bsw:Det_Start()
  BswM->>Bsw:WdgM_Init()
  BswM->>Mcal:Fls_Init()
  BswM->>Bsw:Fee_Init()
  BswM->>Bsw:NvM_Init()
  BswM->>Mcal:Can_Init()
  BswM->>Bsw:CanIf_Init()
  BswM->>Bsw:CanTp_Init()
  BswM->>Bsw:PduR_Init()
  BswM->>Bsw:CanSM_Init()
  BswM->>Bsw:Com_Init()
  BswM->>Bsw:Dcm_Init()
  BswM->>Bsw:NvM_ReadAll()

  BswM->>Bsw:ComM_Init()
  BswM->>Bsw:Dem_Init()
  BswM->>Bsw:Dem_SetOperationCycleState(START)
  BswM->>Bsw:FiM_Init()
  BswM->>Bsw:Crypto_Init()
  BswM->>Bsw:CryIf_Init()
  BswM->>Bsw:Csm_Init()

  BswM->>Bsw:ComM_CommunicationAllowed(TRUE)
  BswM->>Bsw:ComM_RequestComMode(FULL_COMMUNICATION)
  BswM->>Bsw:Dio_WriteChannel(CAN_STB, LOW)
  BswM->>Rte:Rte_Start()
    Rte->>Os:ActivateTask(Cdd/Swc)
```



Shutdown：

```mermaid
sequenceDiagram

participant main
participant Os
participant Rte
participant BswM
participant EcuM
participant EcuM_Callouts
participant Bsw
participant Mcal

BswM->>EcuM:EcuM_SelectShutdownTarget(OFF/RESET)
BswM->>Bsw:ComM_RequestComMode(SILENT)
BswM->>Rte:Rte_Stop()
  Rte->>Os:SetEvent(Rte_OSShutdownEvent)
BswM->>Bsw:ComM_DeInit()
BswM->>Bsw:Dem_SetOperationCycleState(END)
BswM->>Bsw:Dem_Shutdown()
BswM->>Bsw:NvM_WriteAll()
BswM->>EcuM:EcuM_GoDown()
  EcuM->>EcuM_Callouts:EcuM_OnGoOffOne()
  EcuM->>BswM:BswM_Deinit()
  EcuM->>Rte:SchM_Deinit()
    Rte->>Os:SetEvent(SchM_OSShutdownEvent)
  Note over EcuM:State = OFF
  EcuM->>Os:ShutdownOS()
    Os->>main:ShutdownHook()
      main->>EcuM:EcuM_Shutdown()
        EcuM->>EcuM_Callouts:EcuM_OnGoOffTwo()
        alt OFF
          EcuM->>EcuM_Callouts:EcuM_AL_SwitchOff()
        else RESET
          EcuM->>EcuM_Callouts:EcuM_AL_Reset()
          alt MCU
            EcuM_Callouts->>Mcal:Mcu_PerformReset()
          else WDG
            EcuM_Callouts->>Bsw:WdgM_PerformReset()
          end
        end
```



Sleep(Halt)

```mermaid
sequenceDiagram

participant Os
participant BswM
participant EcuM
participant EcuM_Callouts
participant EcuM_Cfg
participant Mcu
participant WakeupSource

BswM->>EcuM:EcuM_SelectShutdownTarget(SLEEP)
BswM->>EcuM:EcuM_GoHalt()
  Note over EcuM:State = GO_SLEEP
  EcuM->>BswM:BswM_EcuM_CurrentState(GO_SLEEP)
  EcuM->>EcuM_Callouts:EcuM_OnGoSleep()
  Note over EcuM:WakeupStatus = NONE
  EcuM->>BswM:BswM_EcuM_CurrentWakeup(NONE)
  EcuM->>EcuM_Callouts:EcuM_EnableWakeupSources()
    EcuM_Callouts->>WakeupSource:XXX_EnableWakeup()
  Note over EcuM:WakeupStatus = ENABLED
  Note over EcuM:State = SLEEP
  EcuM->>BswM:BswM_EcuM_CurrentState(SLEEP)

  EcuM->>EcuM_Callouts:EcuM_PreHalt()
  EcuM->>EcuM_Callouts:EcuM_GenerateRamHash()
  EcuM->>Mcu:Mcu_SetMode(HALT)

  Note over Os,Mcu:HALT

    WakeupSource->>EcuM_Callouts:EcuM_CheckWakeup(Source)
      EcuM_Callouts->>WakeupSource:XXX_CheckWakeup(Source)  
        WakeupSource->>EcuM:EcuM_SetWakeupEvent(Source)  
          alt With Validation
            Note over EcuM:WakeupStatus = PENDING
          else Without Validation
            Note over EcuM:WakeupStatus = VALIDATED
          end
  
  Note over Os,Mcu:WAKEUP
  
  EcuM->>EcuM_Callouts:EcuM_PostHalt()
  EcuM->>EcuM_Callouts:EcuM_CheckRamHash()
  EcuM->>Mcu:Mcu_SetMode(NORMAL)
  EcuM->>EcuM_Callouts:EcuM_DisableWakeupSources()
    EcuM_Callouts->>WakeupSource:XXX_DisableWakeup()
  Note over EcuM:WakeupStatus = DISABLED
  EcuM->>EcuM_Callouts:EcuM_AL_DriverRestart()
    EcuM_Callouts->>EcuM_Cfg:EcuM_DefaultRestartList()
  Note over EcuM:State = WAKEUP_VALIDATION
  EcuM->>BswM:BswM_EcuM_CurrentState(WAKEUP_VALIDATION)

opt WakeupStatus == PENDING
Os->>EcuM:EcuM_MainFunction()
  EcuM->>BswM:BswM_EcuM_CurrentWakeup(PENDING)
  EcuM->>BswM:BswM_EcuM_CurrentWakeup(DISABLED)
  EcuM->>EcuM_Callouts:EcuM_StartWakeupSources(Source)  
  EcuM->>EcuM_Callouts:EcuM_CheckValidation(Source)
      EcuM_Callouts->>WakeupSource:XXX_CheckValidation(Source)  
        WakeupSource->>EcuM:EcuM_ValidateWakeupEvent(Source)
          Note over EcuM:WakeupStatus = VALIDATED
end

opt Validation timeout
Os->>EcuM:EcuM_MainFunction()
  Note over EcuM:WakeupStatus = EXPIRED
  EcuM->>EcuM_Callouts:EcuM_StopWakeupSources(wakeupSource)  
  EcuM->>BswM:BswM_EcuM_CurrentWakeup(EXPIRED)
end

opt WakeupStatus == VALIDATED
Os->>EcuM:EcuM_MainFunction()
  Note over EcuM:State = RUN
  EcuM->>BswM:BswM_EcuM_CurrentState(RUN)
  EcuM->>BswM:BswM_EcuM_CurrentWakeup(VALIDATED)
end
```



Sleep(Poll)

```mermaid
sequenceDiagram

participant Os
participant BswM
participant EcuM
participant EcuM_Callouts
participant EcuM_Cfg
participant Mcu
participant WakeupSource

BswM->>EcuM:EcuM_SelectShutdownTarget(SLEEP)
BswM->>EcuM:EcuM_GoPoll()
  Note over EcuM:State = GO_SLEEP
  EcuM->>BswM:BswM_EcuM_CurrentState(GO_SLEEP)
  EcuM->>EcuM_Callouts:EcuM_OnGoSleep()
  Note over EcuM:WakeupStatus = NONE
  EcuM->>BswM:BswM_EcuM_CurrentWakeup(NONE)
  EcuM->>EcuM_Callouts:EcuM_EnableWakeupSources()
    EcuM_Callouts->>WakeupSource:XXX_EnableWakeup()
  Note over EcuM:WakeupStatus = ENABLED
  Note over EcuM:State = SLEEP
  EcuM->>BswM:BswM_EcuM_CurrentState(SLEEP)

  EcuM->>Mcu:Mcu_SetMode(SLEEP)
  loop Pending==0 && Validated==0
    EcuM->>EcuM_Callouts:EcuM_SleepActivity()
    EcuM->>EcuM_Callouts:EcuM_CheckWakeup()
      EcuM_Callouts->>WakeupSource:XXX_CheckWakeup(Source)  
        WakeupSource->>EcuM:EcuM_SetWakeupEvent(Source)
          alt need Validation
            Note over EcuM:WakeupStatus = PENDING
          else no need Validation
            Note over EcuM:WakeupStatus = VALIDATED
          end
  end

  EcuM->>Mcu:Mcu_SetMode(NORMAL)
  EcuM->>EcuM_Callouts:EcuM_DisableWakeupSources()
    EcuM_Callouts->>WakeupSource:XXX_DisableWakeup()
  Note over EcuM:WakeupStatus = DISABLED
  EcuM->>EcuM_Callouts:EcuM_AL_DriverRestart()
    EcuM_Callouts->>EcuM_Cfg:EcuM_DefaultRestartList()
  Note over EcuM:State = WAKEUP_VALIDATION
  EcuM->>BswM:BswM_EcuM_CurrentState(WAKEUP_VALIDATION)

opt WakeupStatus == ECUM_WKSTATUS_PENDING
Os->>EcuM:EcuM_MainFunction()
  EcuM->>BswM:BswM_EcuM_CurrentWakeup(PENDING)
  EcuM->>BswM:BswM_EcuM_CurrentWakeup(DISABLED)
  EcuM->>EcuM_Callouts:EcuM_StartWakeupSources(Source)  
  EcuM->>EcuM_Callouts:EcuM_CheckValidation(Source)
      EcuM_Callouts->>WakeupSource:XXX_CheckValidation(Source)  
        WakeupSource->>EcuM:EcuM_ValidateWakeupEvent(Source)
          Note over EcuM:WakeupStatus = VALIDATED
end

opt validation timeout
Os->>EcuM:EcuM_MainFunction()
  Note over EcuM:WakeupStatus = EXPIRED
  EcuM->>EcuM_Callouts:EcuM_StopWakeupSources(Source)  
  EcuM->>BswM:BswM_EcuM_CurrentWakeup(EXPIRED)
end

opt WakeupStatus == VALIDATED
Os->>EcuM:EcuM_MainFunction()
  Note over EcuM:State = RUN
  EcuM->>BswM:BswM_EcuM_CurrentState(RUN)
  EcuM->>BswM:BswM_EcuM_CurrentWakeup(VALIDATED)
end
```



MutilCore Start

```mermaid
flowchart
  M_Start --> M_EcuM_Init --> M_EcuM_StartPreOS --> M_OS_StartCore --> M_StartOS --> M_StartInitTask
  M_OS_StartCore --> S_Start --> S_EcuM_Init --> S_StartOS --> S_StartInitTask
 
  M_StartInitTask --> M_EcuM_StartupTwo --> M_SchM_Init --> M_BswM_Init --> M_StartBswTask
  S_StartInitTask --> S_EcuM_StartupTwo --> S_SchM_Init --> S_BswM_Init --> S_StartBswTask

  M_StartBswTask --> M_ModuleInit --> M_Rte_Start --> M_StartSwctask
  S_StartBswTask --> S_ModuleInit --> S_Rte_Start --> S_StartSwctask
```



MutilCore Stop

```mermaid
flowchart
  S_Rte_Stop --> S_ModuleDeInit --> S_EcuM_GoDown --> S_BswM_DeInit --set--> SlaveCoreReadyPort --> S_SchM_DeInit
  M_Rte_Stop --> M_ModuleDeInit --> M_EcuM_GoDown --wait-->SlaveCoreReadyPort --> M_BswM_DeInit--> M_SchM_DeInit--> M_ShutdownAllCores
  M_ShutdownAllCores --> S_ShutdownHook --> S_EcuMShutdown
  M_ShutdownAllCores --> M_ShutdownHook --> M_EcuMShutdown --> OFF/Reset
```



MutilCore Sleep(Halt)

```mermaid
flowchart
S_EcuM_GoHalt --> S_EcuM_GoSleep --set--> SlaveCoreReadyPort--> S_Mcu_SetModeSleep --> halt --wait--> MasterCoreReadyPort --> S_EcuM_WakeupRestart
M_EcuM_GoHalt --> M_EcuM_GoSleep --wait-->SlaveCoreReadyPort --> M_EcuM_GenerateRamHash --> M_Mcu_SetModeSleep --> halt --> M_EcuM_CheckRamHash --set--> MasterCoreReadyPort --> M_EcuM_Wakeup --> M_EcuM_AL_DriverRestart
```



MutilCore Sleep(Poll)

```mermaid
flowchart
S_EcuM_GoPoll --> S_EcuM_GoSleep --> S_Mcu_SetModeSleep --> S_EcuM_CheckWakeup --wait--> MasterCoreReadyPort --> S_EcuM_WakeupRestart --> S_Mcu_SetModeNormal
M_EcuM_GoPoll --> M_EcuM_GoSleep --> M_Mcu_SetModeSleep --> M_EcuM_CheckWakeup--set--> MasterCoreReadyPort --> M_EcuM_WakeupRestart --> M_Mcu_SetModeNormal --> M_EcuM_AL_DriverRestart
```

