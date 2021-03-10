# Setup DAC (with OpAmp follower for higher speed) DMA by HAL and LL using TIM6 and HRTIM
How to setup DAC with OpAmp for DMA  

### Optional tests:  

	- 2 freq/tone sinewaves  
	- 2 freq/tone sinewaves with every halfwave changing  
	- 2 freq/tone sinewaves with every fullwave changing  
	- 2 freq/tone sinewaves with every *few* fullwave changing  
	- Dac Speed Test using sinewave
	- Added DAC trigger by HRTIM external pins still [troubleshooting internal trigger](https://github.com/VictorTagayun/HRTIM_DAC_Trigger_Troubleshooting)

-----------

## DAC Setup 

### Parameter setting  
use Dac3 or 4 and output to OpAmp  
Trigger > Tim6 Trigger Out Event  

![]()

### DMA setting  
Direction > Mem to Periph  
Circular > Memory  
Data width > Periph = Word (will not work if other data with or less)  
	
![]()
	
## OpAmp Setup  

### Mode and Parameter setting  
Follower of DAC non-inverting output  
Power Mode > High Speed  

![]()
	
## TIM6 Setup

### Parameter setting  
Prescaler & ARR > set accorningly, shown below is 1Mhz
Trigger Event > Update event (other selection will not work)  

![]()
		
## Init Code by HAL  

in Main.c

	/*##- Enable DAC Channel and associated DMA ##############################*/
	if(HAL_OK != HAL_DAC_Start_DMA(&hdac3, DAC_CHANNEL_1,
					   (uint32_t*)Sine12bit, Sine12bit_SIZE, DAC_ALIGN_12B_R))
	{
		/* Start DMA Error */
		Error_Handler();
	}

	/*##- Enable TIM peripheral counter ######################################*/
	if(HAL_OK != HAL_TIM_Base_Start(&htim6))
	{
		Error_Handler();
	}
	/*##- Start OPAMP    #####################################################*/
	/* Enable OPAMP */
	if(HAL_OK != HAL_OPAMP_Start(&hopamp6))
	{
		Error_Handler();
	}

## Init Code by LL  

### Enable DAC  

	void LL_Activate_DAC3(void)
	{
		__IO uint32_t wait_loop_index = 0;

		/* Enable DAC channel */
		LL_DAC_Enable(DAC3, LL_DAC_CHANNEL_1);

		/* Delay for DAC channel voltage settling time from DAC channel startup.    */
		/* Compute number of CPU cycles to wait for, from delay in us.              */
		/* Note: Variable divided by 2 to compensate partially                      */
		/*       CPU processing cycles (depends on compilation optimization).       */
		/* Note: If system core clock frequency is below 200kHz, wait time          */
		/*       is only a few CPU processing cycles.                               */
		wait_loop_index = ((LL_DAC_DELAY_STARTUP_VOLTAGE_SETTLING_US * (SystemCoreClock / (100000 * 2))) / 10);
		while(wait_loop_index != 0)
		{
			wait_loop_index--;
		}

		/* Enable DAC channel trigger */
		/* Note: DAC channel conversion can start from trigger enable:              */
		/*       - if DAC channel trigger source is set to SW:                      */
		/*         DAC channel conversion will start after trig order               */
		/*         using function "LL_DAC_TrigSWConversion()".                      */
		/*       - if DAC channel trigger source is set to external trigger         */
		/*         (timer, ...):                                                    */
		/*         DAC channel conversion can start immediately                     */
		/*         (after next trig order from external trigger)                    */
		LL_DAC_EnableTrigger(DAC3, LL_DAC_CHANNEL_1);
	}

### Enable DMA  

	void LL_EnableDMA_DAC3(void)
	{
		  /* Set DMA transfer addresses of source and destination */
		  LL_DMA_ConfigAddresses(DMA1,
								 LL_DMA_CHANNEL_1,
								 (uint32_t)&Sine12bit,
								 LL_DAC_DMA_GetRegAddr(DAC3, LL_DAC_CHANNEL_1, LL_DAC_DMA_REG_DATA_12BITS_RIGHT_ALIGNED),
								 LL_DMA_DIRECTION_MEMORY_TO_PERIPH);

		  /* Set DMA transfer size */
		  LL_DMA_SetDataLength(DMA1,
							   LL_DMA_CHANNEL_1,
							   Sine12bit_SIZE);

		  /* Enable DMA transfer interruption: transfer error */
	//	  LL_DMA_EnableIT_TE(DMA1,
	//	                     LL_DMA_CHANNEL_1);

		  /* Note: In this example, the only DMA interruption activated is            */
		  /*       transfer error.                                                     */
		  /*       If needed, DMA interruptions of half of transfer                   */
		  /*       and transfer complete can be activated.                            */
		  /*       Refer to DMA examples.                                             */

		  /* Activation of DMA */
		  /* Enable the DMA transfer */
		  LL_DMA_EnableChannel(DMA1,
							   LL_DMA_CHANNEL_1);

		  /* Set DAC mode sample-and-hold timings */
		  // LL_DAC_SetSampleAndHoldSampleTime (DAC1, LL_DAC_CHANNEL_1, 0x3FF);
		  // LL_DAC_SetSampleAndHoldHoldTime   (DAC1, LL_DAC_CHANNEL_1, 0x3FF);
		  // LL_DAC_SetSampleAndHoldRefreshTime(DAC1, LL_DAC_CHANNEL_1, 0xFF);

		  /* Set the mode for the selected DAC channel */
		  // LL_DAC_SetMode(DAC1, LL_DAC_CHANNEL_1, LL_DAC_MODE_NORMAL_OPERATION);

		  /* Enable DAC channel DMA request */
		  LL_DAC_EnableDMAReq(DAC3, LL_DAC_CHANNEL_1);

		  /* Enable interruption DAC channel1 under-run */
	//	  LL_DAC_EnableIT_DMAUDR1(DAC1);
	}
	
### Enable OpAmp  

	void LL_Activate_OPAMP6(void)
	{
		__IO uint32_t wait_loop_index = 0;

		/* Enable OPAMP */
		LL_OPAMP_Enable(OPAMP6);

		/* Delay for OPAMP startup time.                                            */
		/* Compute number of CPU cycles to wait for, from delay in us.              */
		/* Note: Variable divided by 2 to compensate partially                      */
		/*       CPU processing cycles (depends on compilation optimization).       */
		/* Note: If system core clock frequency is below 200kHz, wait time          */
		/*       is only a few CPU processing cycles.                               */
		wait_loop_index = ((LL_OPAMP_DELAY_STARTUP_US * (SystemCoreClock / (100000 * 2))) / 10);
		while(wait_loop_index != 0)
		{
			wait_loop_index--;
		}
	}

in Main.c  

	LL_EnableDMA_DAC3();
	LL_Activate_OPAMP6();
	LL_Activate_DAC3();
	LL_TIM_EnableCounter(TIM6);

	
## Waveform of Sine12bit  
		
	const uint16_t Sine12bit[32] = { 511,  611,  707,  796,  873,  937,  984, 1013,
					1023, 1013,  984,  937,  873,  796,  707,  611,
					 511,  412,  315,  227,  149,   86,   38,    9,
					   0,    9,   38,   86,  149,  227,  315,  412};
	
![Waveform of Sine12bit](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint82.jpg)
		
## Waveform of Sawtooth12bit = 4096  
		
	for(uint16_t cntr = 0; cntr < (Sawtooth12bit_SIZE); cntr++)
		Sawtooth12bit[cntr] = cntr;
	
![Sawtooth12bit](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint83.jpg)
		
## Waveform of Sawtooth12bit - 4000  
		
	for(uint16_t cntr = 0; cntr < (Sawtooth12bit_SIZE - 4000); cntr++)
		Sawtooth12bit[cntr] = cntr;
	
![](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint83.jpg)
		
## Waveform of Sine1k_15k  
		
	const uint16_t Sine1k_15k[] = { 1861, 2597, 1744, 2099, 3102, 2379, 2300, 3419,
					2936, 2435, 3499, 3329, 2482, 3329, 3499, 2435,
					2936, 3419, 2300, 2379, 3102, 2099, 1744, 2597,
					1861, 1126, 1979, 1624,  620, 1343, 1423,  304,
					 787, 1288,  224,  394, 1241,  394,  224, 1288,
					 787,  304, 1423, 1343,  620, 1624, 1979, 1126};

In Excel Using Line graph  
![Sine1k_15k_line](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/Sine48_1k_15k_line.png)
	
In Excel Using Bar graph  
![Sine1k_15k_line](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/Sine48_1k_15k_bar.png)

Actual DAC waveform  
![Sine1k_15k_DAC](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint86.jpg)

As can be seen, Excel waveform is better than the DAC because Excel is connecting the dots, while the DAC is holding the voltage similar to bar graph

## Waveform of Sine1k_10k  
		
The actual waveform of *Sine1k_15k* is not of very good resolution. So I generated another array using Online sine generator
		
Sinewave 1kHz, DMA is 2MSPS (500nS)  

	uint32_t MySine2000[] =
	{
		683,685,687,689,691,693,695,698,700,702,704,706,708,710,713,715,717,719,721,723,
		725,727,730,732,734,736,738,740,742,745,747,749,751,753,755,757,760,762,764,766,
		768,770,772,774,777,779,781,783,785,787,789,791,793,796,798,800,802,804,806,808,
		810,812,815,817,819,821,823,825,827,829,831,833,836,838,840,842,844,846,848,850,
		852,854,856,858,861,863,865,867,869,871,873,875,877,879,881,883,885,887,889,891,
		893,895,897,900,902,904,906,908,910,912,914,916,918,920,922,924,926,928,930,932,
		934,936,938,940,942,944,946,948,950,952,954,956,957,959,961,963,965,967,969,971,
		973,975,977,979,981,983,985,987,989,990,992,994,996,998,1000,1002,1004,1006,1008,1009,
		1011,1013,1015,1017,1019,1021,1023,1024,1026,1028,1030,1032,1034,1035,1037,1039,1041,1043,1045,1046,
		1048,1050,1052,1054,1055,1057,1059,1061,1063,1064,1066,1068,1070,1071,1073,1075,1077,1078,1080,1082,
		1084,1085,1087,1089,1091,1092,1094,1096,1097,1099,1101,1103,1104,1106,1108,1109,1111,1113,1114,1116,
		1118,1119,1121,1122,1124,1126,1127,1129,1131,1132,1134,1135,1137,1139,1140,1142,1143,1145,1147,1148,
		1150,1151,1153,1154,1156,1157,1159,1161,1162,1164,1165,1167,1168,1170,1171,1173,1174,1176,1177,1179,
		1180,1181,1183,1184,1186,1187,1189,1190,1192,1193,1194,1196,1197,1199,1200,1201,1203,1204,1206,1207,
		1208,1210,1211,1212,1214,1215,1216,1218,1219,1220,1222,1223,1224,1226,1227,1228,1230,1231,1232,1233,
		1235,1236,1237,1238,1240,1241,1242,1243,1245,1246,1247,1248,1249,1251,1252,1253,1254,1255,1256,1258,
		1259,1260,1261,1262,1263,1264,1266,1267,1268,1269,1270,1271,1272,1273,1274,1275,1276,1277,1279,1280,
		1281,1282,1283,1284,1285,1286,1287,1288,1289,1290,1291,1292,1293,1294,1294,1295,1296,1297,1298,1299,
		1300,1301,1302,1303,1304,1305,1305,1306,1307,1308,1309,1310,1311,1311,1312,1313,1314,1315,1315,1316,
		1317,1318,1319,1319,1320,1321,1322,1322,1323,1324,1325,1325,1326,1327,1328,1328,1329,1330,1330,1331,
		1332,1332,1333,1334,1334,1335,1335,1336,1337,1337,1338,1338,1339,1340,1340,1341,1341,1342,1342,1343,
		1344,1344,1345,1345,1346,1346,1347,1347,1348,1348,1349,1349,1349,1350,1350,1351,1351,1352,1352,1353,
		1353,1353,1354,1354,1354,1355,1355,1356,1356,1356,1357,1357,1357,1358,1358,1358,1358,1359,1359,1359,
		1360,1360,1360,1360,1361,1361,1361,1361,1362,1362,1362,1362,1362,1363,1363,1363,1363,1363,1363,1364,
		1364,1364,1364,1364,1364,1364,1364,1364,1365,1365,1365,1365,1365,1365,1365,1365,1365,1365,1365,1365,
		1365,1365,1365,1365,1365,1365,1365,1365,1365,1365,1365,1365,1365,1364,1364,1364,1364,1364,1364,1364,
		1364,1364,1363,1363,1363,1363,1363,1363,1362,1362,1362,1362,1362,1361,1361,1361,1361,1360,1360,1360,
		1360,1359,1359,1359,1358,1358,1358,1358,1357,1357,1357,1356,1356,1356,1355,1355,1354,1354,1354,1353,
		1353,1353,1352,1352,1351,1351,1350,1350,1349,1349,1349,1348,1348,1347,1347,1346,1346,1345,1345,1344,
		1344,1343,1342,1342,1341,1341,1340,1340,1339,1338,1338,1337,1337,1336,1335,1335,1334,1334,1333,1332,
		1332,1331,1330,1330,1329,1328,1328,1327,1326,1325,1325,1324,1323,1322,1322,1321,1320,1319,1319,1318,
		1317,1316,1315,1315,1314,1313,1312,1311,1311,1310,1309,1308,1307,1306,1305,1305,1304,1303,1302,1301,
		1300,1299,1298,1297,1296,1295,1294,1294,1293,1292,1291,1290,1289,1288,1287,1286,1285,1284,1283,1282,
		1281,1280,1279,1277,1276,1275,1274,1273,1272,1271,1270,1269,1268,1267,1266,1264,1263,1262,1261,1260,
		1259,1258,1256,1255,1254,1253,1252,1251,1249,1248,1247,1246,1245,1243,1242,1241,1240,1238,1237,1236,
		1235,1233,1232,1231,1230,1228,1227,1226,1224,1223,1222,1220,1219,1218,1216,1215,1214,1212,1211,1210,
		1208,1207,1206,1204,1203,1201,1200,1199,1197,1196,1194,1193,1192,1190,1189,1187,1186,1184,1183,1181,
		1180,1179,1177,1176,1174,1173,1171,1170,1168,1167,1165,1164,1162,1161,1159,1157,1156,1154,1153,1151,
		1150,1148,1147,1145,1143,1142,1140,1139,1137,1135,1134,1132,1131,1129,1127,1126,1124,1122,1121,1119,
		1118,1116,1114,1113,1111,1109,1108,1106,1104,1103,1101,1099,1097,1096,1094,1092,1091,1089,1087,1085,
		1084,1082,1080,1078,1077,1075,1073,1071,1070,1068,1066,1064,1063,1061,1059,1057,1055,1054,1052,1050,
		1048,1046,1045,1043,1041,1039,1037,1035,1034,1032,1030,1028,1026,1024,1023,1021,1019,1017,1015,1013,
		1011,1009,1008,1006,1004,1002,1000,998,996,994,992,990,989,987,985,983,981,979,977,975,
		973,971,969,967,965,963,961,959,957,956,954,952,950,948,946,944,942,940,938,936,
		934,932,930,928,926,924,922,920,918,916,914,912,910,908,906,904,902,900,897,895,
		893,891,889,887,885,883,881,879,877,875,873,871,869,867,865,863,861,858,856,854,
		852,850,848,846,844,842,840,838,836,833,831,829,827,825,823,821,819,817,815,812,
		810,808,806,804,802,800,798,796,793,791,789,787,785,783,781,779,777,774,772,770,
		768,766,764,762,760,757,755,753,751,749,747,745,742,740,738,736,734,732,730,727,
		725,723,721,719,717,715,713,710,708,706,704,702,700,698,695,693,691,689,687,685,
		683,680,678,676,674,672,670,667,665,663,661,659,657,655,652,650,648,646,644,642,
		640,638,635,633,631,629,627,625,623,620,618,616,614,612,610,608,605,603,601,599,
		597,595,593,591,588,586,584,582,580,578,576,574,572,569,567,565,563,561,559,557,
		555,553,550,548,546,544,542,540,538,536,534,532,529,527,525,523,521,519,517,515,
		513,511,509,507,504,502,500,498,496,494,492,490,488,486,484,482,480,478,476,474,
		472,470,468,465,463,461,459,457,455,453,451,449,447,445,443,441,439,437,435,433,
		431,429,427,425,423,421,419,417,415,413,411,409,408,406,404,402,400,398,396,394,
		392,390,388,386,384,382,380,378,376,375,373,371,369,367,365,363,361,359,357,356,
		354,352,350,348,346,344,342,341,339,337,335,333,331,330,328,326,324,322,320,319,
		317,315,313,311,310,308,306,304,302,301,299,297,295,294,292,290,288,287,285,283,
		281,280,278,276,274,273,271,269,268,266,264,262,261,259,257,256,254,252,251,249,
		247,246,244,243,241,239,238,236,234,233,231,230,228,226,225,223,222,220,218,217,
		215,214,212,211,209,208,206,204,203,201,200,198,197,195,194,192,191,189,188,186,
		185,184,182,181,179,178,176,175,173,172,171,169,168,166,165,164,162,161,159,158,
		157,155,154,153,151,150,149,147,146,145,143,142,141,139,138,137,135,134,133,132,
		130,129,128,127,125,124,123,122,120,119,118,117,116,114,113,112,111,110,109,107,
		106,105,104,103,102,101,99,98,97,96,95,94,93,92,91,90,89,88,86,85,
		84,83,82,81,80,79,78,77,76,75,74,73,72,71,71,70,69,68,67,66,
		65,64,63,62,61,60,60,59,58,57,56,55,54,54,53,52,51,50,50,49,
		48,47,46,46,45,44,43,43,42,41,40,40,39,38,37,37,36,35,35,34,
		33,33,32,31,31,30,30,29,28,28,27,27,26,25,25,24,24,23,23,22,
		21,21,20,20,19,19,18,18,17,17,16,16,16,15,15,14,14,13,13,12,
		12,12,11,11,11,10,10,9,9,9,8,8,8,7,7,7,7,6,6,6,
		5,5,5,5,4,4,4,4,3,3,3,3,3,2,2,2,2,2,2,1,
		1,1,1,1,1,1,1,1,0,0,0,0,0,0,0,0,0,0,0,0,
		0,0,0,0,0,0,0,0,0,0,0,0,0,1,1,1,1,1,1,1,
		1,1,2,2,2,2,2,2,3,3,3,3,3,4,4,4,4,5,5,5,
		5,6,6,6,7,7,7,7,8,8,8,9,9,9,10,10,11,11,11,12,
		12,12,13,13,14,14,15,15,16,16,16,17,17,18,18,19,19,20,20,21,
		21,22,23,23,24,24,25,25,26,27,27,28,28,29,30,30,31,31,32,33,
		33,34,35,35,36,37,37,38,39,40,40,41,42,43,43,44,45,46,46,47,
		48,49,50,50,51,52,53,54,54,55,56,57,58,59,60,60,61,62,63,64,
		65,66,67,68,69,70,71,71,72,73,74,75,76,77,78,79,80,81,82,83,
		84,85,86,88,89,90,91,92,93,94,95,96,97,98,99,101,102,103,104,105,
		106,107,109,110,111,112,113,114,116,117,118,119,120,122,123,124,125,127,128,129,
		130,132,133,134,135,137,138,139,141,142,143,145,146,147,149,150,151,153,154,155,
		157,158,159,161,162,164,165,166,168,169,171,172,173,175,176,178,179,181,182,184,
		185,186,188,189,191,192,194,195,197,198,200,201,203,204,206,208,209,211,212,214,
		215,217,218,220,222,223,225,226,228,230,231,233,234,236,238,239,241,243,244,246,
		247,249,251,252,254,256,257,259,261,262,264,266,268,269,271,273,274,276,278,280,
		281,283,285,287,288,290,292,294,295,297,299,301,302,304,306,308,310,311,313,315,
		317,319,320,322,324,326,328,330,331,333,335,337,339,341,342,344,346,348,350,352,
		354,356,357,359,361,363,365,367,369,371,373,375,376,378,380,382,384,386,388,390,
		392,394,396,398,400,402,404,406,408,409,411,413,415,417,419,421,423,425,427,429,
		431,433,435,437,439,441,443,445,447,449,451,453,455,457,459,461,463,465,468,470,
		472,474,476,478,480,482,484,486,488,490,492,494,496,498,500,502,504,507,509,511,
		513,515,517,519,521,523,525,527,529,532,534,536,538,540,542,544,546,548,550,553,
		555,557,559,561,563,565,567,569,572,574,576,578,580,582,584,586,588,591,593,595,
		597,599,601,603,605,608,610,612,614,616,618,620,623,625,627,629,631,633,635,638,
		640,642,644,646,648,650,652,655,657,659,661,663,665,667,670,672,674,676,678,680
	};
	
Actual waveforms  
![MySine2000](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint91.jpg)

![DS1Z_QuickPrint110.jpg]()

Add offset  

	MySine2000[cntr] = MySine2000[cntr] + 682;  
	
![Add offset](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint92.jpg)

Sine 10k, DMA is 2MSPS (500nS)   

	const uint32_t MySine200[] =
	{
		683,704,725,747,768,789,810,831,852,873,893,914,934,954,973,992,1011,1030,1048,1066,
		1084,1101,1118,1134,1150,1165,1180,1194,1208,1222,1235,1247,1259,1270,1281,1291,1300,1309,1317,1325,
		1332,1338,1344,1349,1353,1357,1360,1362,1364,1365,1365,1365,1364,1362,1360,1357,1353,1349,1344,1338,
		1332,1325,1317,1309,1300,1291,1281,1270,1259,1247,1235,1222,1208,1194,1180,1165,1150,1134,1118,1101,
		1084,1066,1048,1030,1011,992,973,954,934,914,893,873,852,831,810,789,768,747,725,704,
		683,661,640,618,597,576,555,534,513,492,472,451,431,411,392,373,354,335,317,299,
		281,264,247,231,215,200,185,171,157,143,130,118,106,95,84,74,65,56,48,40,
		33,27,21,16,12,8,5,3,1,0,0,0,1,3,5,8,12,16,21,27,
		33,40,48,56,65,74,84,95,106,118,130,143,157,171,185,200,215,231,247,264,
		281,299,317,335,354,373,392,411,431,451,472,492,513,534,555,576,597,618,640,661
	};

Actual waveform
![Sine 10k](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint93.jpg)

Add together

	MySine2000[cntr] = MySine2000[cntr] + MySine200[cntr % MySine200_SIZE];
	
![Add together](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint94.jpg)
		
## Observations and Suggestions  
	
It is still ok to use HAL instead of LL because the codes will be used only during init. Problem is when you want to change the waveform on the **fly**

## Timing, to check how we can change the waveforms on the **fly**  

### Timing of DMA Interrupt

	void DMA1_Channel1_IRQHandler(void)
	{
	  /* USER CODE BEGIN DMA1_Channel1_IRQn 0 */

	  GPIOC->BSRR = (1<<(8));

	  /* USER CODE END DMA1_Channel1_IRQn 0 */
	  HAL_DMA_IRQHandler(&hdma_dac3_ch1);
	  /* USER CODE BEGIN DMA1_Channel1_IRQn 1 */

	  GPIOC->BSRR = (1<<(8+16));

	  /* USER CODE END DMA1_Channel1_IRQn 1 */
	}
	
![Timing](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint95.jpg)

1.13uS Duration  
![1.13uS Duration](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint98.jpg)

970nS Duration  
![970nS Duration](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint97.jpg)

### Checking of DMA Callback  

	void HAL_DAC_ConvCpltCallbackCh1(DAC_HandleTypeDef *hdac)
	{
	  /* Prevent unused argument(s) compilation warning */
	  UNUSED(hdac);

	  /* NOTE : This function should not be modified, when the callback is needed,
				the HAL_DAC_ConvCpltCallbackCh1 could be implemented in the user file
	   */
	  GPIOC->BSRR = (1<<(8));
	}

	void HAL_DAC_ConvHalfCpltCallbackCh1(DAC_HandleTypeDef *hdac)
	{
	  /* Prevent unused argument(s) compilation warning */
	  UNUSED(hdac);

	  /* NOTE : This function should not be modified, when the callback is needed,
				the HAL_DAC_ConvHalfCpltCallbackCh1 could be implemented in the user file
	   */
	  GPIOC->BSRR = (1<<(8+16));
	}
	
![Callback](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint99.jpg)

### Timing of HAL_DAC_Start_DMA (1.25mS)  

	  GPIOC->BSRR = (1<<(8));
	  /*##- Enable DAC Channel and associated DMA ##############################*/
	  if(HAL_OK != HAL_DAC_Start_DMA(&hdac3, DAC_CHANNEL_1,
					   (uint32_t*)MySine2000, MySine2000_SIZE, DAC_ALIGN_12B_R))
	  {
		/* Start DMA Error */
		Error_Handler();
	  }
	  GPIOC->BSRR = (1<<(8+16));
	  
![HAL_DAC_Start_DMA](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint100.jpg)

As shown, we cannot use **HAL_DAC_Start_DMA** every time we need to change the waveform on the fly due to very long duration of 1.25ms.
As our DAC speed is 2MHz! or 500nS!  

### Find where/what code to update/change the new data to feed to DMA, in _stm32g4xx_hal_dac.c_
HAL_DAC_Start_DMA (stm32g4xx_hal_dac.c) > HAL_DMA_Start_IT (stm32g4xx_hal_dma.c) > DMA_SetConfig

#### Testing *DMA_SetConfig* but got error in the code below and did not proceed  
	
	DMA_SetConfig(DMA_HandleTypeDef *hdma, uint32_t SrcAddress, uint32_t DstAddress, uint32_t DataLength)
	where:  
		*hdma = hdac3->DMA_Handle1
		SrcAddress = MySine2000 / MySine4000
		DstAddress = (uint32_t)&hdac->Instance->DHR12R2 (DAC_ALIGN_12B_R/DHR12R2)
		DataLength = MySine2000/MySine4000_SIZE

#### Use Register level instead  
inside *DMA_SetConfig* is  

    /* Configure DMA Channel source address */
    hdma->Instance->CPAR = SrcAddress;
	
in my case  

	hdma_dac3_ch1.Instance->CMAR = MySine4000; // or MySine2000

#### Callbacks

	void HAL_DAC_ConvCpltCallbackCh1(DAC_HandleTypeDef *hdac)
	{
		/* Prevent unused argument(s) compilation warning */
		UNUSED(hdac);

		hdma_dac3_ch1.Instance->CMAR = MySine4000;
	}
	
	void HAL_DAC_ConvHalfCpltCallbackCh1(DAC_HandleTypeDef *hdac)
	{
		/* Prevent unused argument(s) compilation warning */
		UNUSED(hdac);

		hdma_dac3_ch1.Instance->CMAR = MySine2000;
	}

#### Every half wave change waveform  

![](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint101.jpg)

![](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint102.jpg)

#### Every full wave change waveform  

![](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint104.jpg)

#### Every *few* (3) full wave change waveform  

![](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint105.jpg)

#### Duration of the *Callback* for full wave change waveform  

![](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint106.jpg)

136nS Duration  
![](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint107.jpg)

276nS Duration  
![](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint108.jpg)

### YES!!  


### DAC Speed using sinewave

From [previous project](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_SpeedTest) where we checked how fast the DAC can achieve from minimum to maximum transition, we had around 4.74Mhz! Of course that is worst case, but for this test, we will be using sinewave and the transition will not be as worse.

![](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_SpeedTest/blob/main/waveforms/DS1Z_QuickPrint81.jpg)

Now we use [Sine12bit](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/51186868f263d2b9add39d1ad09b27fbb80560f4/NUCLEO-G474RE_2tonegenerator/Core/Inc/waveforms.h#L10) for a more coarse sine wave to observe the DAC steps better.

Sine12bit running at 2Mhz/2MSPS

![DS1Z_QuickPrint111.jpg](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint111.jpg)

![DS1Z_QuickPrint110.jpg](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint110.jpg)

We can try til 14.7Mhz/MSPS

![DS1Z_QuickPrint112.jpg](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint112.jpg)

But is it now acting erratic

![DS1Z_QuickPrint115.jpg](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint115.jpg)

![DS1Z_QuickPrint117.jpg](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint117.jpg)

It is now due to DMA interrupt, so we disable it and run til around 17Mhz/MSPS

![DS1Z_QuickPrint120.jpg](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint120.jpg)

With 32sample points and TIM6 running @ 170Mhz/(5 X 2), we have a freq of the Sine12bit as 532kHz as shown

![DS1Z_QuickPrint121.jpg](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint121.jpg)

Some abnormal reaction if we increase further 17Mhz/MSPS, like 4.2Mhz/MSPS for TIM6 = 170Mhz/(2 X 2), the 32sample of the sine have different freq

![DS1Z_QuickPrint122.jpg](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_DMA_LL-HAL_TIM6/blob/main/photos-waveforms%26generator/DS1Z_QuickPrint122.jpg)

### TO do

Investigate when TIM6 or DAC is more than 17Mhz/MSPS

### Other Reference :

[Online sine generator](https://www.daycounter.com/Calculators/Sine-Generator-Calculator.phtml)

[DAC Speed testing](https://github.com/VictorTagayun/NUCLEO-G474RE_DAC_SpeedTest)

[Troubleshooting HRTIM internal trigger](https://github.com/VictorTagayun/HRTIM_DAC_Trigger_Troubleshooting)

*Disclaimer:*

*The projects posted here are for learning and educational purposes only.*
*The purpose of a certain project may be for testing a module and may be just a part of a whole project.*
*It should not be used in a production or commercial environment.*
*Any cause of injury and/or death is the sole responsibility of the user.*
