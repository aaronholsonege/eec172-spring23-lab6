
# Multiplayer Pong Game Demo

## Introduction

This project demonstrates an online Pong game that allows players to compete against each other using a custom-built joystick controller and a web client. The game is powered by a CC3200 board and facilitated by a NodeJS server running on an AWS EC2 instance.

## Demo Video
[![Demo](https://i.ytimg.com/vi/7d6nEgtiEZs/maxresdefault.jpg)](https://youtu.be/7d6nEgtiEZs "Demo")
![Schematic](EEC-172-Lab-6-Schematic.png)
## Methods


*Figure:  Overall architecture (node server + CC3200 + web client)*

### Gameplay

![Gameplay Flow](game-flow.drawio.png)

*Figure 2: Gameplay flow*

### Joystick Sensor

The joystick module has two data pins, SW and VRx. The VRx pin outputs an analog signal ranging from 0-5v, which is converted to digital using the MCP3001 ADC. The joystick's position is then determined based on the converted signal.

![Diagram of 2-byte binary conversion of ADC signal](binary-diagram.drawio.png)

*Figure 3: Diagram of 2-byte binary conversion of ADC signal*

![Diagram of Joystick Module Values](https://components101.com/sites/default/files/inline-images/Joystick-Module-Analog-Output.png)

*Figure 4: Diagram of Joystick Module Values [3]*

#### Joystick Case

A 3D model for the joystick module was found and printed using a 3D printer. The case was then assembled with screws.

### Node Server

The NodeJS server running on an AWS EC2 instance manages the game state and passes gameplay data between the two players.


*Figure 5: Data flow when P1 syncs positional data to P2*



*Figure 6: Data flow when P1 syncs positional data to P2 that also triggers the server to update game state data & positional data*



*Figure 7: Data flow when server syncs state & (reset) positional data to P1 & P2*

### WebSocket

The WebSocket protocol was chosen for its bi-directional communication capabilities and minimal header size. The WebSocket implementation for the CC3200 board supports string and close messages.

![WebSocket 2-byte header format (excluding mask key)](websocket.drawio.png)

*Figure 8: WebSocket 2-byte header format (excluding mask key)*

### Seeding Data between Players

Data is synchronized between the boards and the server during key gameplay events, such as when a player changes the direction of their paddle, blocks the ball, or misses the ball.

### Synchronization

To ensure smooth gameplay, each board assumes the opponent player blocks the ball unless it receives a payload that confirms this or a new round is started. Frame numbers are used to project the provided positional data back into the current frame.



*Figure 11: Ball location synchronization with projection*

### Cross-Platform Gameplay

A web client was created to connect to the same server, allowing players to compete against a CC3200 board.

### OLED Rendering

Rendering on the OLED is optimized by tracking the position or value of rendered objects and only re-rendering when those values change.

![Optimized paddle rendering by erasing/drawing delta](paddle-draw.drawio.png)

*Figure 13: Optimized paddle rendering by erasing/drawing delta*

### Game States

The game progresses through various states, such as countdown, gameplay, and game over.

## References

[1] Texas Instruments, “CC3200 Peripheral Driver Library User's Guide,” CC3200 peripheral driver library user's guide: GPIO_GENERAL_PURPOSE_INPUTOUTPUT_API, 18-Feb-2016. [Online]. Available: https://software-dl.ti.com/ecs/cc31xx/APIs/public/cc32xx_peripherals/latest/html/group___g_p_i_o___general___purpose___input_output__api.html#ga97225829b669d27fb53bf01e5771fb5b. [Accessed: 03-Apr-2023].
