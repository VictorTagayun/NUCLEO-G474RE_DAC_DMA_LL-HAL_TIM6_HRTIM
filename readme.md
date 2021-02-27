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

	### DAC  
	### DAC  
	### DAC  

	
## Waveform of Sine12bit  
		
		const uint16_t Sine12bit[32] = { 511,  611,  707,  796,  873,  937,  984, 1013,
                                1023, 1013,  984,  937,  873,  796,  707,  611,
                                 511,  412,  315,  227,  149,   86,   38,    9,
                                   0,    9,   38,   86,  149,  227,  315,  412};
		
		![]()