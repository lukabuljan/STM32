# Grand Finale

This is where the magic happens.

## Button interface

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

## Timer

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
