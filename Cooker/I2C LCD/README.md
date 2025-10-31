# 1602 LCD via I2C

This LCD screen is a very popular choice for prototyping as it can act as an interface for displaying all kinds of messages.
The version I am using in this project is outfitted with PCF8574, which drastically reduces wiring.

It is crucial to understand how the two modules are connected. Below is the sketch visualizing the connections.

It is worth mentioning that the user can only input the higher data bits (D4-D7), and instead of the lower half can manipulate the RS and RW pins to determine the type and direction of data: 
* RS: 0 = sending command, 1 = sending data
* RW: 0 = write, 1 = read. In this project, RW pin will always be 0
In all of the examples below, the sequence of user-defined data will be as shown on the graph:

The device itself has a unique protocol and requires a detailed look at the datasheet, but for simplicity's sake only the most important segments will be explained.

## Finding the address

First step is to determine the address of the device itself - since the microcontroller is actually connected to PCF8574, this module determines the address. Its value depends on multiple factors:

* the purpose of device (read/write)
* state of pins A2, A1 and A0

Since the device is used to write (send) data and pins are not soldered to the common ground by default, the address of the device is 0x4E.

(graf PCF8574 adrese)

## Wake up call

The LCD1602 datasheet provides a detailed explanation of displaying characters on the screen. Before sending any data, the screen must be initialized following these steps:

* send 0x30 three times: wait x miliseconds between 1st and 2nd time, and more than 100 microseconds between 2nd and 3rd time
* set the device to 4-bit mode, enable both rows for displaying characters and adjust the font to 5x8 by sending 0x28
* turn off the LCD by sending 0x08
* clear the display with 0x01
* set the entry mode parameters to incrementing cursor and no shift by sending 0x06
* turn on the LCD by sending 0x08
