# Setup DAC (with OpAmp follower for higher speed) DMA by HAL and LL using TIM6

## DAC  

	### Parameter setting  
	use Dac3 or 4 and output to OpAmp
	Trigger > Tim6 Trigger Out Event

	![]()

	### DMA setting  
	Direction > Mem to Periph
	Circular > Memory 
	Data width > Periph = Word (will not work if other data with or less)  
	
	![]()
	
## OpAmp  

	### Mode and Parameter setting  
	Follower of DAC non-inverting output 
	Power Mode > High Speed
	
	![]()
	
## TIM6

	### Parameter setting  
	Prescaler & ARR > set accorningly, shown below is 1Mhz
	Trigger Event > Update event (other selection will not work)  
	
	![]()
		
## Code by HAL  

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

## Code by LL

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
		
		![]()
		
## Waveform of Sawtooth12bit = 4096  
		
		for(uint16_t cntr = 0; cntr < (Sawtooth12bit_SIZE); cntr++)
			Sawtooth12bit[cntr] = cntr;
		
		![]()
		
## Waveform of Sawtooth12bit - 4000  
		
		for(uint16_t cntr = 0; cntr < (Sawtooth12bit_SIZE - 4000); cntr++)
			Sawtooth12bit[cntr] = cntr;
		
		![]()
		
## Waveform of Sine1k_15k  
		
		const uint16_t Sine1k_15k[] = { 1861, 2597, 1744, 2099, 3102, 2379, 2300, 3419,
										2936, 2435, 3499, 3329, 2482, 3329, 3499, 2435,
										2936, 3419, 2300, 2379, 3102, 2099, 1744, 2597,
										1861, 1126, 1979, 1624,  620, 1343, 1423,  304,
										 787, 1288,  224,  394, 1241,  394,  224, 1288,
										 787,  304, 1423, 1343,  620, 1624, 1979, 1126};
		
		![]()
		
## Suggestions and Observations  
	
	It is still ok to use HAL instead of LL because the codes will be used only durign init.
	
### Online sine generator