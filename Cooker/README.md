Cooking has never been so complicated

![Prototype video.](Prototype.mp4 "Prototype video.")

Food is important. Scientific consesus suggests that cooking was an evolutionary step which enabled the growth of brains since it made food safer and more nutritious to eat.
Since I am a proud decendant of those same humans that made our brains bigger by cooking, I decided to pay an homage to my ancestors by creating a uniquely designed cooker.


This project consists of two STM32 Black Pill microcontrollers boards which communicate via HC05 Bluetooth modules. One is equiped with and LCD screen and the other with temperature sensor.
in the final version of this project the microcontroller with temp. sensor will also be connected to an electric circuit of an induction heater through a relay switch: this enables the microcontroller to regulate the temperature of the heater by measuring it an turning off the circuit if it is too high.

Assembling of the cooker can be divided into four segments:
* Bluetooth communication using UART protocol
* Displaying paramaters with LCD using I2C protocol
* Measuring temperature and sending it to the LCD screen
* Assembling the induction heater circuit and integration with the rest of the project 

To give an initial impression of the project, the video at the beginning of this readme shows an early prototype. It consists of two Black Pill boards communicating with UART protocol.
The workflow goes something like this:
* The first Black pill, outfitted with and LCD screen and three buttons, prompts the user to insert the desired time interval
* User presses the buttons to input the desired time interval. The nearest button increases the interval by 10 seconds, and the middle buttons confirms the value. The farthest button decreases the value of interval by 10 seconds.
* Countdown begins. Temperature detected by the sensor from the other Black Pill is also displayed.
* After the countdown screen displays a message signaling that the process is finished.

In this video I hold the sensor in my hand to showcase the project's ability to use temp. readings and change certain values accordingly. In this example, the blue LED lights up if the detected temp. is above 30 degrees celsius.

This project is in its earliest stages of development and is under construction. Updates soon to follow.
