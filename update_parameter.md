
```
在操作过程中遇到有些android设备，连接BLE设备之后，过了十几秒就自动断开。

原因：连接之后会自动更新一些连接的参数，例如连接时间间隔等。

uint8_t enableUpdateRequest = DEFAULT_ENABLE_UPDATE_REQUEST;

默认选择的是两个设备都可以向对方发送连接参数更新请求，选择合适的更新时间，这样子会频繁的更新。导致断开。如下。
#define DEFAULT_ENABLE_UPDATE_REQUEST         GAPROLE_LINK_PARAM_UPDATE_INITIATE_BOTH_PARAMS

#define GAPROLE_LINK_PARAM_UPDATE_WAIT_REMOTE_PARAMS   0 // Wait for parameter update request, respond with remote's requested parameters.
#define GAPROLE_LINK_PARAM_UPDATE_INITIATE_BOTH_PARAMS 1 // Initiate parameter update request, respond with best combination of local and remote parameters.
#define GAPROLE_LINK_PARAM_UPDATE_INITIATE_APP_PARAMS  2 // Initiate parameter update request, respond with local requested parameters only.
#define GAPROLE_LINK_PARAM_UPDATE_WAIT_APP_PARAMS      3 // Wait for parameter update request, respond with local requested parameters only.
#define GAPROLE_LINK_PARAM_UPDATE_WAIT_BOTH_PARAMS     4 // Wait for parameter update request, respond with best combination of local and remote parameters.
#define GAPROLE_LINK_PARAM_UPDATE_REJECT_REQUEST       5 // Reject all parameter update requests.
#define GAPROLE_LINK_PARAM_UPDATE_NUM_OPTIONS          6 // Used for parameter checking.

选择第一个GAPROLE_LINK_PARAM_UPDATE_WAIT_REMOTE_PARAMS就不会出现这种情况。

选择之后，BLE这边就不会主动发送更新参数请求，只有android那边可以发送参数更新请求。

当然这个也和机器有关，包括RF，天线等因素。

```
