根据差分方式，采用内部偏置还是外部偏置的不同，需要在代码中配置RF偏置。

ICallBLE/ble_user_config.h

```
#elif defined( CC2650EM_5IS )

	#define RF_FE_MODE_AND_BIAS           ( RF_FE_SINGLE_ENDED_RFP |             \
		                                            RF_FE_INT_BIAS )


// RF Front End Settings
// Note: The use of these values completely depends on how the PCB is laid out.
//       Please see Device Package and Evaluation Module (EM) Board below.
#define RF_FE_DIFFERENTIAL              0
#define RF_FE_SINGLE_ENDED_RFP          1
#define RF_FE_SINGLE_ENDED_RFN          2
#define RF_FE_ANT_DIVERSITY_RFP_FIRST   3
#define RF_FE_ANT_DIVERSITY_RFN_FIRST   4
#define RF_FE_SINGLE_ENDED_RFP_EXT_PINS 5
#define RF_FE_SINGLE_ENDED_RFN_EXT_PINS 6
//
#define RF_FE_INT_BIAS                  (0<<3)
#define RF_FE_EXT_BIAS                  (1<<3)
```
