# Advanced-Microprocessor-System-Project
## Group 4: Arm Based Implementation of Speech Controlled Wheelchair
### Introduction
Nowadays, the increasing number of disabled and elderly people, leading to more invention in medical support devices. Especially the evolution of wheelchairs from the conventional mechanical wheelchair to nowadays electric wheelchair. However, some users were unable to freely control the wheelchair due to lack of arms or energy or having involuntary action.

Thus, this project is carried out to develop an user friendly speech controlled wheelchair to elderly and disabled groups. We implement Keyword Spotting (KWS) into the wheelchair. Hence, users can control the wheelchair’s movement by voice out simple commands. Any forces not even needed for the wheelchair operation.

In this project, the KWS is implemented on Nucleo-F446RE due to its low power consumption and reliability. Besides, it provides an affordable and flexible alternative for users to build up our prototype with low cost. This project is built on GitHub (Reference: https://github.com/ARM-software/ML-KWS-for-MCU). As the result of the deployment, our prototype can be controlled to move in 4 situations which are “Go”, “Stop”, “Left” and “Right”.

### Requirements
Below are the minimum hardware and software requiremnets for this project.
#### Hardware
1 x Nucleo F446RE Board

2 x Micro 360 Degree Continuous Rotation Servo (FS90R)

2 x Wheel

1 x INMP441 MEMS Omnidirectional Microphones

#### Software
STM32CubeMX IDE

Audacity

### Data Collection
To train the simple command through CNN using DSP, we need to collect at least "50" sampels for each command. By using audacity, we can record and edit the sample to make sure the audio sample meets the requirements which are less than 1s, "mono". 

### Integration of I2S MEMS Microphone with STM32


### Integration CNN
- Import library based on the "reference"

### Coding
For loop is used to keep receiving data from I2S MEMS microphone. Error message `Error Rx` will be printed out if STM32 board failed to receive data from the microphone, otherwise an audio buffer will be set as varible for the input data.

```c
  for (int i=0; i<16000; i+=2){	
		  volatile HAL_StatusTypeDef result = HAL_I2S_Receive(&hi2s2, data_in, 2, 100);	
		  if (result != HAL_OK){
			  strcpy((char*)buf, "Error Rx\r\n");
		  } else {
			  audio_buffer[i] = (int16_t)data_in[0];
			  audio_buffer[i+1] = (int16_t)data_in[1];
		  }
	  }

```

To recognise the received data, the received data will be send to kws file and processed by CNN and DSP library.

```c
char output_class[12][8] = {"Silence", "Unknown", "yes", "no", "up", "down", "left", "right", "on", "off", "stop", "go"};
  KWS_DS_CNN kws(audio_buffer);

  kws.extract_features();
  kws.classify();

````

Data is printed in PuTTY through USART of STM32 board in order to verified the result of the word recognition.

```c
int max_ind = kws.get_top_class(kws.output);

  char buffer [70];
  int buffer_output = 0;
  buffer_output = sprintf(buffer, "Detected %s (%d%%)\r\n", output_class[max_id], ((int)kws.output[max_ind]*100/128));
  HAL_UART_Transmit(&huart2, (unit8_t *)buffer, buffer_output, 100);
```

### Integration of FS90R (Servo Motor) with STM32
FS90R is a 360 degree continuous rotation servo from FEETECH. Since its default rest point is 1.5 ms, it could be configured to rotate counterclockwise by setting the pulse width above the reset point, otherwise it will result in clockwise.  

In this project, we are using two FS90R to demonstrate the wheelchair’s movements which are go, stop, left and right by configuring the two FS90R simultaneously. Table below shows the configuration of the FS90Rs corresponding to each scenario.

First, before coding the servo motor, we need to do some configuration on STM32. In this project, TIM2 was used. Since TIM2 is connected to APB1 bus, thus, the maximum frequency for the timer clock is 90MHz. However, the maximum value we could obtain is only 65535 due to the prescalar register which is only 16 bit. Then, divide the clock using prescalar and ARR register in order to obatin 900 kHz (45MHz/50Hz = 900 kHz).Thus, to get 50 Hz, prescalar should set to 900 whereas ARR set to 1000. Since the ARR, it will be act as 1000% pulse width and it would easier for us to modified setting the "active duration" by just modified with x% to CRR1 register.

After setting up the STM32, let's us insight to the coding used to control the movement of servo motors based on the 4 situation we set.

Coding below shows the two servo motors are coded simultaneosly in order make it move based on the situation that we set. For example, both servo motors will rotating clockwise at the same time when "GO" command is detected. Besides, when "Left" command is detected, the servo motor on the left will be stop while the right side servo motor will rotate in clockwise. This configuration will allow the prototype turns left.

```c
int GO_command(){
	 htim2.Instance->CCR1 = 25;  // duty cycle is .5 ms
	 htim2.Instance->CCR2 = 25;  // duty cycle is .5 ms
	 HAL_Delay(2000);
	 return 0;
}

int STOP_command(){
	 htim2.Instance->CCR1 = 0;  // duty cycle is .5 ms
	 htim2.Instance->CCR2 = 0;  // duty cycle is .5 ms
	 HAL_Delay(2000);
}

int LEFT_command(){
	 htim2.Instance->CCR1 = 25;  // duty cycle is .5 ms
	 htim2.Instance->CCR2 = 0;  // duty cycle is .5 ms
	 HAL_Delay(2000);
}

int RIGHT_command(){
	 htim2.Instance->CCR1 = 0;  // duty cycle is .5 ms
	 htim2.Instance->CCR2 = 25;  // duty cycle is .5 ms
	 HAL_Delay(2000);
}
```

In `main`, we use a while loop to keep looping and wait for the command. When command is detected and recognised as same as one of the data base， the corresponding function will be executed. For example, when "Left" is detected, the `Left_command()` subroutine will be executed.


```c
while (1)
  {

	  // Use default size
	  char *command_array[SIZE_OF_CMD_ARRAY] = {};

	  // COMMAND Strings
	  char *command_go = "go";
	  char *command_stop = "stop";
	  char *command_left = "left";
	  char *command_right = "right";


	  for (i=0; i<SIZE_OF_CMD_ARRAY; i++)
	  {
		  if (command_array[i] == command_left)
			  LEFT_command();

		  else if (command_array[i] == command_right)
			  RIGHT_command();

		  else if (command_array[i] == command_go)
			  GO_command();

		  else if (command_array[i] == command_stop)
			  STOP_command();

		  else
			  STOP_command();

	  }
````
