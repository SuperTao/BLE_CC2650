GAP初始化是创建了一个task，相当于一个线程。

初始化完成之后，在循环中一直读取从BLE stack发过来的event。

main.c
```
int main()
{
	GAPRole_createTask();
}
```

PROFILES/peripheral.c

创建线程

```
void GAPRole_createTask(void)
{
  Task_Params taskParams;

  // Configure task
  // Task参数初始化
  Task_Params_init(&taskParams);
  // 栈位置
  taskParams.stack = gapRoleTaskStack;
  // 栈大小
  taskParams.stackSize = GAPROLE_TASK_STACK_SIZE;
  // 优先级
  taskParams.priority = GAPROLE_TASK_PRIORITY;
  // 创建Task，相当于创建了一个线程，线程的主函数,主循环gapRole_taskFxn.
  Task_construct(&gapRoleTask, gapRole_taskFxn, &taskParams, NULL);
}
```

线程的回调函数

```
static void gapRole_taskFxn(UArg a0, UArg a1)
{
  // Initialize profile
  gapRole_init();	// 初始化

  // Profile main loop
  for (;;)
  {
#ifdef ICALL_EVENTS
    uint32_t events;

    // Wait for an event to be posted
    events = Event_pend(syncEvent, Event_Id_NONE, GAPROLE_ALL_EVENTS,
                        ICALL_TIMEOUT_FOREVER);

    if (events)
#else //!ICALL_EVENTS
    // Waits for a signal to the semaphore associated with the calling thread.
    // Note that the semaphore associated with a thread is signaled when a
    // message is queued to the message receive queue of the thread or when
    // ICall_signal() function is called onto the semaphore.
        // 一直阻塞，等待事件到来，BLE从底层发上来的消息
    ICall_Errno errno = ICall_wait(ICALL_TIMEOUT_FOREVER);

    if (errno == ICALL_ERRNO_SUCCESS)
#endif //ICALL_EVENTS
    {
      ICall_EntityID dest;
      ICall_ServiceEnum src;
      ICall_HciExtEvt *pMsg = NULL;

      if (ICall_fetchServiceMsg(&src, &dest,
                                (void **)&pMsg) == ICALL_ERRNO_SUCCESS)
      {
        if ((src == ICALL_SERVICE_CLASS_BLE) && (dest == selfEntity))
        {
          ICall_Stack_Event *pEvt = (ICall_Stack_Event *)pMsg;

          // Check for BLE stack events first
          if (pEvt->signature == 0xffff)
          {
            if (pEvt->event_flag & GAP_EVENT_SIGN_COUNTER_CHANGED)
            {
                // 写NV
              // Sign counter changed, save it to NV
              VOID osal_snv_write(BLE_NVID_SIGNCOUNTER, sizeof(uint32_t),
                                  &gapRole_signCounter);
            }
          }
          else
          {
              // 处理接收的事件
            // Process inter-task message
            gapRole_processStackMsg((ICall_Hdr *)pMsg);
          }
        }

        if (pMsg)
        {
          ICall_freeMsg(pMsg);
        }
      }
    }
    // 广播事件
    if (events & START_ADVERTISING_EVT)
    {
#ifndef ICALL_EVENTS
      events &= ~START_ADVERTISING_EVT;
#endif //ICALL_EVENTS
      // 广播使能，或者发出不可连接的广播
      if (gapRole_AdvEnabled || gapRole_AdvNonConnEnabled)
      {
        gapAdvertisingParams_t params;

        // Setup advertisement parameters
        if (gapRole_AdvNonConnEnabled)
        {
            // 广播类型，不可连接
          // Only advertise non-connectable undirected.
          params.eventType = GAP_ADTYPE_ADV_NONCONN_IND;
        }
        else
        {
            // 其他类型的广播
          params.eventType = gapRole_AdvEventType;
          //
          params.initiatorAddrType = gapRole_AdvDirectType;
          VOID memcpy(params.initiatorAddr, gapRole_AdvDirectAddr, B_ADDR_LEN);
        }
        // 广播通道
        params.channelMap = gapRole_AdvChanMap;
        // 广播过滤策略
        params.filterPolicy = gapRole_AdvFilterPolicy;

        if (GAP_MakeDiscoverable(selfEntity, &params) != SUCCESS)
        {
          gapRole_state = GAPROLE_ERROR;
          // 状态改变，协议栈发给GAPRole的消息
          // Notify the application with the new state change
          if (pGapRoles_AppCGs && pGapRoles_AppCGs->pfnStateChange)
          {
            pGapRoles_AppCGs->pfnStateChange(gapRole_state);
          }
        }
      }
    }

    if (events & START_CONN_UPDATE_EVT)
    {
#ifndef ICALL_EVENTS
      events &= ~START_CONN_UPDATE_EVT;
#endif //ICALL_EVENTS
      // 启动连接更新参数
      // Start connection update procedure
      gapRole_startConnUpdate(GAPROLE_NO_ACTION, &gapRole_updateConnParams);
    }
    // 连接参数超时
    if (events & CONN_PARAM_TIMEOUT_EVT)
    {
#ifndef ICALL_EVENTS
      events &= ~CONN_PARAM_TIMEOUT_EVT;
#endif //ICALL_EVENTS
      // 连接参数失败处理
      // Unsuccessful in updating connection parameters
      gapRole_HandleParamUpdateNoSuccess();
    }
  } // for
}
```

gapRole初始化，主要是连接相关的属性设置。

```
static void gapRole_init(void)
{
  // Register the current thread as an ICall dispatcher application
  // so that the application can send and receive messages.
#ifdef ICALL_EVENTS
  ICall_registerApp(&selfEntity, &syncEvent);
#else //!ICALL_EVENTS
  // 初始化信号量
  ICall_registerApp(&selfEntity, &sem);
#endif //ICALL_EVENTS

  gapRole_state = GAPROLE_INIT;
  gapRole_ConnectionHandle = INVALID_CONNHANDLE;

  // Get link DB maximum number of connections
  linkDBNumConns = linkDB_NumConns();

  // Setup timers as one-shot timers
  // 定时器，开始广播
  /*
   * startAdvClock:时钟结构体
   * gapRold_clockHandler：处理函数
   * 第三个参数：时间的长度
   * 第四个参数：为0表示只会触发一次
   * false:定时器不会马上触发，调用了start函数才会触发
   * START_ADVERTISING_EVT: 传给回调函数的参数
   */
  Util_constructClock(&startAdvClock, gapRole_clockHandler,
                      0, 0, false, START_ADVERTISING_EVT);
  // 定时器，开始连接更新
  Util_constructClock(&startUpdateClock, gapRole_clockHandler,
                      0, 0, false, START_CONN_UPDATE_EVT);
  // 定时器，更新参数超时
  Util_constructClock(&updateTimeoutClock, gapRole_clockHandler,
                      0, 0, false, CONN_PARAM_TIMEOUT_EVT);

  // Initialize the Profile Advertising and Connection Parameters
  // GAP的角色：BLE外围设备。另外几个角色，广播者，观察者，集中器
  gapRole_profileRole = GAP_PROFILE_PERIPHERAL;
  VOID memset(gapRole_IRK, 0, KEYLEN);
  VOID memset(gapRole_SRK, 0, KEYLEN);
  gapRole_signCounter = 0;
  gapRole_AdvEventType = GAP_ADTYPE_ADV_IND;    // 广播类型，可连接的，非定向广播
  gapRole_AdvDirectType = ADDRMODE_PUBLIC;
  gapRole_AdvChanMap = GAP_ADVCHAN_ALL;     // 广播信道，37，38， 39
  gapRole_AdvFilterPolicy = GAP_FILTER_POLICY_ALL;  // 过滤规则，允许scan请求和连接请求

  // Restore Items from NV
  // 保存一些选项到NV里面，这些参数不会丢失，一些密钥相关的参数
  VOID osal_snv_read(BLE_NVID_IRK, KEYLEN, gapRole_IRK);
  VOID osal_snv_read(BLE_NVID_CSRK, KEYLEN, gapRole_SRK);
  VOID osal_snv_read(BLE_NVID_SIGNCOUNTER, sizeof(uint32_t),
                     &gapRole_signCounter);
}
```