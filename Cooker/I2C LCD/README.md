# 1602 LCD via I2C

This LCD screen is a very popular choice for prototyping as it can act as an interface for displaying all kinds of messages.
The version I am using in this project is outfitted with , which drastically reduces wiring.

The device itself has a unique protocol and requires a detailed look at the datasheet, but for simplicity's sake only the most important segments will be explained.

First step is to determine the address of the device itself. It can vary depending on the bit A0, A1 and A2, and by default the adress is 4E.
