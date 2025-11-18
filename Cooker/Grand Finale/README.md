# Grand Finale

This is where the magic happens.

## Button interface

The three buttons connected to the microcontroller are used to change values of two variables: sek_br stores the desired countdown time, while OK is used as a trigger to start the countdown.

Mechanical buttons introduce the problem of bouncing. It is a rapid and unwanted switching between high and low voltage values and can be remedied programatically or electronically by introducing a parallel capacitor.
In this project I solved the problem programatically by introducing an extra condition which checks if the current voltage level lasts longer than 10 milliseconds.

```C
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
	current_time = HAL_GetTick();

	if(GPIO_Pin == GPIO_PIN_5 && ((current_time - previous_time1) >= 10))
	{
		previous_time1 = current_time;
		sek_br -= 10;
	}

	if(GPIO_Pin == GPIO_PIN_6 && ((current_time - previous_time2) >= 10))
	{
		previous_time2 = current_time;
		OK = 1;
	}

	if(GPIO_Pin == GPIO_PIN_7 && ((current_time - previous_time3) >= 10))
	{
		previous_time3 = current_time;
		sek_br += 10;
	}

}
```

## Receiver

These are the neccessary includes. Special headers for interrupts and interfacing with LCD, as well as stdio.h for the sprintf function are added.

```C
/* Includes ------------------------------------------------------------------*/
#include "main.h"
#include "i2c.h"
#include "tim.h"
#include "usart.h"
#include "gpio.h"

/* Private includes ----------------------------------------------------------*/
/* USER CODE BEGIN Includes */
#include "stm32f4xx_it.h"
#include "i2c-lcd.h"
#include "stdio.h"
```

(ovaj dio u obliku natuknica)
Some variables used later in the code are initialized. An array tekst is used to store strings containing timer values in MM:SS format, min and sek are used to store individual values for minutes and seconds, rxData stores the integer and decimal part of the measured temperature and req_t which used as a trigger for the other microcontroller to start sending temperature data.

```C
/* USER CODE BEGIN PV */
char tekst[6] = {0};

uint16_t min = 0;
uint16_t sek = 0;

uint8_t rxData[2] = {0};

uint8_t req_t = 1;
/* USER CODE END PV */
```

The code starts properly after initialization of all peripherals. In this part, the LCD prompts the user to input the desired time using the buttons connected to the microcontroller.
The OK variable is used to register whether the middle button was pushed or not. Before the user confirms the desired value, the code checks the value of sek_br which is located in stmf4xx_it.c program whose value is changed by using the left and right buttons. The sek_br value is also used to calculate the number of minutes and seconds equivalent to its value and sent to the LCD using the sprintf command. LCD is updated every 200 milliseconds.

```C
while(!OK)
  {
	min = sek_br/60;
	sek = sek_br%60;
	if(sek_br < 10)sprintf(tekst, "0%d:0%d", min, sek);
	else{sprintf(tekst, "0%d:%d", min, sek);}
	lcd_put_cur(1,0);
	lcd_send_string(tekst);
	HAL_Delay(200);
  }
```

When the user confirms the timer value, request for temperature readings is sent first via UART to HC05 BT module. Next, an interrupt handler is called and activates when the microcontroller successfully receives data from the other microcontroller equipped with the temp. sensor. Finally, TIM10 is initialized and the countdown begins LCD displaying the remaining time along with the measured temperature.


## Timer

After confirming the countdown value, TIM10 starts periodically decreasing the sek_br value as well as converting it to MM:SS format. When sek_br reaches zero, it sends a zero to the other microcontroller to terminate temperature transmission.
All that is left is to display a string informing the user that the countdown completed successfully, as well as stopping the timer. 

```C
void HAL_TIM_PeriodElapsedCallback(TIM_HandleTypeDef *htim)
{
	if(sek_br > 0)
	{
		sek_br--;
		min_t = sek_br/60;
		sek_t = sek_br%60;
		if(sek_t < 10){sprintf(tekst_t, "0%d:0%d", min_t, sek_t);}
		else{sprintf(tekst_t, "0%d:%d", min_t, sek_t);}
		lcd_put_cur(1,0);
		lcd_send_string(tekst_t);
	}
	else
	{
		req_t = 0;
		while(HAL_UART_Transmit(&huart1, &req_t, sizeof(req_t), 30) != HAL_OK);

		lcd_clear();
		lcd_put_cur(0, 0);
		lcd_send_string("FINISHED!");

		HAL_TIM_Base_Stop_IT(&htim10);
	}
}
```

## Sender

This part is a straightforward implementation of the DS18B20 functions. The only difference from the original temperature displaying code is the separation of the float data into an integer and decimal part which are then stored in txData array and send via UART to HC05.

```C
 while (1)
  {

	Presence = DS18B20_Start ();
	DS18B20_Write (0xCC);  // skip ROM
	DS18B20_Write (0x44);  // convert t

	Presence = DS18B20_Start ();
	DS18B20_Write (0xCC);  // skip ROM
	DS18B20_Write (0xBE);  // Read Scratch-pad

	Temp_byte1 = DS18B20_Read();
	Temp_byte2 = DS18B20_Read();
	TEMP = ((Temp_byte2<<8))|Temp_byte1;
	temperature = (float)TEMP/16.0;  // resolution is 0.0625
	t_int = (uint8_t)temperature;
	t_dec = (temperature - t_int)*100;
	t_dec_int = (uint8_t)t_dec;
	txData[0] = t_int;
	txData[1] = t_dec_int;

	HAL_Delay(2000);

	if(!rec_req){HAL_UART_Receive_IT(&huart1, &req_t, sizeof(req_t)); rec_req = 1;}
	if(req_t){HAL_UART_Transmit(&huart1, &txData[0], sizeof(txData), 30);}

	HAL_GPIO_WritePin(GPIOC, GPIO_PIN_13, temperature > 30.0f ? 0 : 1);

    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
  }
  /* USER CODE END 3 */
}
```

To save energy I implemented a condition which checks the value of rec_req value: initially its value is 0, but changes to whatever value is stored and sent with the req_t variable of the receiving microcontroller. The last line of the while loop is added as a sort of test for the correct sensor reading: the LED on the board lights up if the sensor measures 30 degrees Celius or above and turns off for the lower values.
