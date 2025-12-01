# DS18B20

This sensor stands out from other sensors and modules with absence of conventional communication protocols such as UART or I2C. Instead, it uses a uniquely designed, one-wire, half-duplex communication to send and receive data.
The sensor is outfitted with three wires: red for VCC, black for GND and yellow which serves to send/receive data. It reliably detects temperatures up to 125 degrees Celsius and stores temp. data in a 16 bit sign-extended two's complement register.

![DS18B20](DS18B20.jpg "DS18B20 Sensor")

![Schematic](STM32_Cooker2.png "Schematic")

Datasheet of the DS18B20 sensor demonstrates how to receive the measured temperature:
1. send reset pulse
2. receive presence pulse from the sensor
3. Master sends skip ROM Command (0xCC, only in case of one sensor)
4. Master sends Convert T Command
5. repeat steps 1., 2. and 3.
6. send Read Scratchpad Command (0xBE)
7. read data bytes from the sensor

Since most of the terms describe events applicable only to this sensor and its datasheet, each step will be described in its own chapter.
Moreover, since all events happen in a span of microseconds, it is neccessary to create a special function to manipulate data on this scale, since the usual HAL_Delay() doesn't offer sufficient speed for these tasks.
A simple function could be made based on one of the timers of the microcontroller. I chose TIM10 as it's the most basic and offers parameter tweaking needed to create a microsecond delay.
Here is how to create a microsecond delay:
* Create a project in STM32 Cube MX or IDE
* Under Clock Configuration -> APB1 Clock choose any frequency above 1 MHz
* Under Conectivity -> USART1 -> Asynchronous choose Parameter settings
* Change the Prescaler value to the number of MHz in APB1 Clock Field and subtract 1 from it, ex. APB1 Clock = 50 Mhz -> Prescaler = 50-1 = 49
* set the register to maximum value (65535 or 0xffff)
* create a function similar to the one displayed below:
```C
void delay(uint8_t time)
{
	HAL_TIM_SET_COUNTER(&htim10, 0);
	while(HAL_TIM_GET_COUNTER(&htim10) < time);
}
```


## Reset Pulse

As shown in the datasheet, the reset pulse consists of pulling the data pin LOW for at least 480 microseconds, a brief pulse which raises the pin to HIGH, and immediate return to LOW forced by the sensor itself.

![Reset Pulse](reset_pulse.png "Reset Pulse")

Quote from the datasheet:

**During the initialization sequence the bus master trans-
mits (TX) the reset pulse by pulling the 1-Wire bus low
for a minimum of 480μs. The bus master then releases
the bus and goes into receive mode (RX). When the bus
is released, the 5kΩ pullup resistor pulls the 1-Wire bus
high. When the DS18B20 detects this rising edge, it waits
15μs to 60μs and then transmits a presence pulse by pull-
ing the 1-Wire bus low for 60μs to 240μs.**

The code should set the pin to LOW, wait 480 us, then wait additional 80 us to enter the timeframe in which the sensor pulls the pin to Low again. After that, another delay is needed to complete the timeframe of 480 us, in this case it means additional 400 us of delay:

```C
uint8_t DS18B20_Start(void)
{
	uint8_t response = 0;
	set_pin_output(DS18B20_PORT, DS18B20_PIN);
	HAL_GPIO_WritePin(DS18B20_PORT, DS18B20_PIN, 0); //pull the pin low
	delay(480); // wait 480 us
	
	set_pin_input(DS18B20_PORT, DS18B20_PIN); // changing pin to input mode pulls the data pin up
	delay(80);
	
	if(!(HAL_GPIO_ReadPin(DS18B20_PORT, DS18B20_PIN)) response = 1;
	else response = -1;
	
	delay(400);
	
	return response;
}
```

## Write Data

Writing data is done according to the datasheet:
"A write time slot is initiated when the host pulls the data line from a high logic level to a low logic level. There are two types of write time slots: Write 1 time slots and Write 0 time slots.  All write time slots must be a minimum of 60 µs in duration with a minimum of a 1-µs recovery time between individual write cycles."

```C
void DS18B20_Write (uint8_t data)
{
	Set_Pin_Output(DS18B20_PORT, DS18B20_PIN);  // set as output

	for (int i=0; i<8; i++)
	{
		if ((data & (1<<i))!=0)  // if the bit is high
		{
			// write 1
			Set_Pin_Output(DS18B20_PORT, DS18B20_PIN);  // set as output
			HAL_GPIO_WritePin (DS18B20_PORT, DS18B20_PIN, 0);  // pull the pin LOW
			delay (1);  // wait for 1 us

			Set_Pin_Input(DS18B20_PORT, DS18B20_PIN);  // set as input
			delay (50);  // wait for 60 us
		}

		else  // if the bit is low
		{
			// write 0
			Set_Pin_Output(DS18B20_PORT, DS18B20_PIN);
			HAL_GPIO_WritePin (DS18B20_PORT, DS18B20_PIN, 0);  // pull the pin LOW
			delay (50);  // wait for 60 us

			Set_Pin_Input(DS18B20_PORT, DS18B20_PIN);
		}
	}
}
```

## Read Data

"The host generates read time slots when data is to be read from the DS18B20.  A read time slot is initiated when the host pulls the data line from a logic high level to logic low level.  The data line must remain at a low logic level for a minimum of 1 µs; output data from the DS18B20 is valid for 15 µs after the falling edge of the read time slot.  The host therefore must stop driving the DQ pin low in order to read its state 15 µs from the start of the read slot (see Figure 12).  By the end of the read time slot, the DQ pin will pull back high via the external pullup resistor.  All read time slots must be a minimum of 60 µs in duration with a minimum of a 1-µs recovery time between individual read slots."

```C
uint8_t DS18B20_Read (void)
{
	uint8_t value=0;
	Set_Pin_Input(DS18B20_PORT, DS18B20_PIN);

	for (int i=0;i<8;i++)
	{
		Set_Pin_Output(DS18B20_PORT, DS18B20_PIN);   // set as output
		HAL_GPIO_WritePin (GPIOA, GPIO_PIN_1, 0);  // pull the data pin LOW
		delay (2);  // wait for timeslot longer than 1 us
		Set_Pin_Input(DS18B20_PORT, DS18B20_PIN);  // set as input
		if (HAL_GPIO_ReadPin (GPIOA, GPIO_PIN_1))  // if the pin is HIGH
		{
			value |= 1<<i;  // read = 1
		}
		delay (60);  // wait for 60 us
	}
	return value;
}
```

## Main algorithm

The "Transaction Sequence" chapter of the datasheet states that data exchange is always done in three steps: initialization, ROM Command and DS18B20 function.
Initialization is done by calling the DS18B20_Start() function while the ROM Command and Ds18B20 function are realized with proper arguments in DS18B20_Write(function).
Datasheet also provides an example of using a single sensor with all the neccessary steps:

* Issue a reset pulse (initialize)
* Send Skip ROM Command (0xCC)
* Send Convert T Command (0x44)
* Issue another reset pulse (initialize)
* Send Skip ROM Command (0xCC)
* Send Read Scratch-Pad Command (0xBE)

After these steps the sensor sends a 9 data bytes, of which only the first two are stored into variables Temp_byte1 and Temp_byte2.
Received bytes must be scaled with the resolution of the sensor, which is 0.0625 degrees per bit, requiring to scale the received value with 16.

```C
while (1)
  {

	Presence = DS18B20_Start ();
	DS18B20_Write (0xCC);  // skip ROM in case of single sensor
	DS18B20_Write (0x44);  // convert t

	Presence = DS18B20_Start ();
	DS18B20_Write (0xCC);  // skip ROM
	DS18B20_Write (0xBE);  // Read Scratch-pad

	Temp_byte1 = DS18B20_Read();
	Temp_byte2 = DS18B20_Read();
	TEMP = ((Temp_byte2<<8))|Temp_byte1;
	temperature = (float)TEMP/16.0;  // resolution is 0.0625

	HAL_Delay(1000);
	Display_Temp(temperature);
    /* USER CODE END WHILE */

    /* USER CODE BEGIN 3 */
  }
  /* USER CODE END 3 */
}
```
