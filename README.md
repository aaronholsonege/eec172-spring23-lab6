
# Multiplayer Pong Game Demo

## Introduction

This project demonstrates an online Pong game that allows players to compete against each other using a custom-built joystick controller and a web client. The game is powered by a CC3200 board and facilitated by a NodeJS server running on an AWS EC2 instance.

## Demo Video
[![Demo](https://i.ytimg.com/vi/7d6nEgtiEZs/maxresdefault.jpg)](https://youtu.be/7d6nEgtiEZs "Demo")
![Schematic](EEC-172-Lab-6-Schematic2.png)
## Methods


*Figure:  Overall architecture (node server + CC3200 + web client)*

### Gameplay

![Gameplay Flow](game-flow.drawio.png)

*Figure 2: Gameplay flow*

### Joystick Sensor

The joystick module had two data pins that were useful to us; the SW and VRx pins. The VRx pin would output an analog signal that ranged in voltage from 0-5v. In order to read the positional data of the x-axis on the joystick we first needed to convert the analog signals to digital. To do so we used the MCP3001 ADC supplying 5v to VREF and VDD to set the ADC’s range to 0-5v. We then connected the VRx pin on the board to the IN+ pin, the IN- and VSS to ground, the DOUT pin to PIN_06 on the board, and the CS to PIN_04 on the board. When reading data from the ADC, we are reading 2 bytes for the 10-bit value. Based on the datasheet for the ADC, the first 3 bits and the last 3 bits of the 2-byte signal are irrelevant (the 10-bit value is center aligned in the 2-bytes. Also, we are reading the payload into a buffer in sequential bytes, meaning they are written to our buffer out of order (in big-endian). As seen in Figure 3, we need to perform a mask operation to remove the “don’t care” bits from the signal and then perform 2 binary shift operations to put the value in the right order and alignment.

![Diagram of 2-byte binary conversion of ADC signal](binary-diagram.drawio.png)

*Figure 3: Diagram of 2-byte binary conversion of ADC signal*

Once we have the correct data format we get a value that ranges from 0-1024 when 0 corresponds to down and 1024 represents up  as seen in Figure 4. Since the center position of the joystick is 512 we subtract the value from the ADC by 512 in order for the center position to be 0. Changing the positional values to be VRx < 0  is down  and and VRx > 0 is up. We also make sure to divide the VRx values by 190 to slowdown the paddle speed. For the Joystick SW we use a GPIO pin read configured as a pull-resisor GPIO  to read when the button is pressed when unpressed = 1 and pressed = 0. 

![Diagram of Joystick Module Values](https://components101.com/sites/default/files/inline-images/Joystick-Module-Analog-Output.png)

*Figure 4: Diagram of Joystick Module Values [3]*

#### Joystick Case

A 3D model for the joystick module was found and printed using a 3D printer. The case was then assembled with screws.

### Node Server

To facilitate online gameplay, we first thought that we would have the server run the game and provide data on the position of the ball and paddles to each board. This was quickly abandoned as it would mean the need for a lot of data to be transferred as well and a high potential for less-than-ideal FPS gameplay.

Instead, we decided (in large part because of the simplicity of pong), to run the game independently on each board and sync data related to user-driver input between the boards via a NodeJS server (written in Typescript for static typing) running on an AWS EC2 instance. This meant each board was running the game autonomously, with the ability to inject game state as it was received.

One reason we chose NodeJS over other languages was due to the stateful nature of NodeJS via its volatile state, so there was no need for long-term storage in order to persist the game state from request to request. Variables defined on the server and set from one request would be available in subsequent requests.

The main duties of the server include passing gameplay data between the two players (Figure 5), and managing the game state.

![](server-client-flow-1.png)

*Figure 5: Data flow when P1 syncs positional data to P2*

The server will peek at data being transferred between boards and emits game state changes based on specific events in that data (Figure 6).

![](server-client-flow-2.png)

*Figure 6: Data flow when P1 syncs positional data to P2 that also triggers the server to update game state data & positional data*

The other important duty of the server is to facilitate the round countdown, which it manages and sends to each board (Figure 7).

![](server-client-flow-3.png)

*Figure 7: Data flow when server syncs state & (reset) positional data to P1 & P2*

To reduce the amount of data being sent to each board, the server buffers payloads so it can be combined and deduped. We only buffer for 1 tick of the processor before releasing the payload. This allows the server to more freely update the game state multiple times based on subsets of gameplay data without needing to know what events and data have already or are going to be sent. When the buffer is flushed, then everything is merged, deduped, and sent as one message.

### WebSocket

It wouldn’t be much of an online game without the use of a network protocol to sync gameplay between the two players. The first decision we made was to use a protocol that supports bi-directional communication over a single TCP connection. It was imperative that the data be able to be pushed to a board rather than rely on the board to request new data. This ruled out the REST endpoints used for our previous lab for our AWS IoT Thing.

Next, we needed a protocol had had a very minimal header. Since we will likely be sending data frequently, we did not want to incur the extra cost of an excessive header size (another reason REST was out).

Finally, we wanted something that would be easy to implement on our server (which admittedly is not a very high bar as most network protocols are supported in NodeJS).

With this list of requirements, we ended up with the WebSocket protocol. This gave us the bi-direction communication we desired (with the ability to push updates to clients), and a very small 2-type header (Figure 8). It also helped that creating a websocket server in NodeJS required only a few lines of code to get up and running.

Now that we had a protocol, we needed to implement it in C on our board (since the protocol itself is not implemented).

To create a websocket connection, you initially send a GET REST request. The response from the server is an upgrade request, and the TCP connection is left open for messages to be passed back and forth.

As messages are received, we parse the first 2 bytes to determine what type and length the message is.

![WebSocket 2-byte header format (excluding mask key)](websocket.drawio.png)

*Figure 8: WebSocket 2-byte header format (excluding mask key)*

For the simplicity of this project (mainly due to timeline constraints) we opted to omit many of the security features of the WebSocket spec, such as:

- Handshake validation on the initial GET request
Message masking
- Given more time, these features would have been implemented.

Too, our WebSocket implementation for the CC3200 board only supports the subset of message operations needed: string and close  messages. We also knew the maximum payload for our string message would be 65 bytes (see Section 3.5), so we did not need to explicitly support message fragmentation (since the max payload of a WebSocket message is 127 bytes).

Our WebSocket payload was a rather straightforward string message (see Figure 9) consisting of a specific syntax for key-value pairs and a separator character.. We created a spec for a significantly reduced binary format (going from a max of 65 bytes to a max of 12 bytes - over 80% reduction), but decided not to implement it as it would have made the observability of messages in real time while debugging the project much more challenging.

```string
st:0|c:1|                           // game state
bf:13|bx:22|by:48|bdy:-1|bdx:43|    // ball
pf:34|py:34|pdy:0                   // paddle
```

*Figure 9: Sample payload*

### Seeding Data between Players

There are key events during the gameplay that facilitated the need for data to be synchronized across the boards. These types of synchronization fall in two two categories:

- Board-to-board synchronization (Figure 5)
- Server-to-board(s) synchronization (Figure 7)

For board-to-board synchronization, this was gameplay data at specific key points:

- When a player changes the direction of their paddle.
- When a player blocks the ball.
- When a player misses the ball.

When a player changes the direction of their paddle, we send both the direction and y position of the paddle (x position never changes) to the server to sync with the other board.

When a player blocks the ball with their paddle, we send the position and (new) directional data of the ball to the other board (through the server).

And finally, when the player misses the ball, we send the other player’s score to the server. Because a players score is determined by the action of the other player’s paddle, it is the responsibility of the other player to report whether they other player scored or not.

For server-to-board synchronization, this was gameplay state data at specific keys points:

If one player syncs a score, server to start the next round and reset the ball.
If on player sync a score that wins the game, the server sets the game state to finished.
During round countdown, the server to send the current countdown value as it changes on the server.


### Synchronization

When one board sends the position/directional data for their paddle or ball, the data will most likely be received by the other board as stale data, having already moved on to rendering subsequent frames.

This latency (and need for real time data) causes a few problems. First, applying the received positional data “as-is” caused the ball/paddle to jump around on the screen.

![](sync-1.png)

*Figure 11: Ball location synchronization without projection*

The larger issue caused by this stale data is that as each player repositions their ball and paddle based on past information as the round plays, the boards become more and more out of sync.

To mitigate this, we needed a way to identify a payload with a specific point in game time. Since the boards do not have an accurate time applied to them (and there wasn’t a simple sure-first way to ensure they truly agreed on the current time), we couldn’t rely on timestamps. Instead, since both boards are operating at the same FPS, we can count frames at the start of every round and provide the frame number with the ball or paddle data. When a board receives positional data, they will also receive the frame number that positional data applies to. The board then compares it to its own current frame number, and projects the provided positional data back into the current frame.

![](sync-2.png)

*Figure 11: Ball location synchronization with projection*

In a further effort to ensure smooth gameplay, each board assumes the opponent player blocks the ball unless it receives a payload that either confirms this, or a new round is started. This means most of the time when we receive stale position data and project it into the current frame, the location of the ball is at the same location.


### Cross-Platform Gameplay

A web client was created to connect to the same server, allowing players to compete against a CC3200 board.

### OLED Rendering

Rendering on the OLED is a rather slow process, so we implemented some tracking to mitigate the rendering speed.

First, every time we actually render an object (ball, paddle, score, display text, etc) we capture the position or value of what was rendering. On subsequent updates, we check against these values and only when those values have changed do we re-rendering.

Re-rendering is also a process in itself. Using the previous location data, we use that to erase the previous rendering before rending the object at the new location.

For the paddle, it was actually fairly slow to completely erase the entire paddle and redraw it (see Figure 12).


![Optimized paddle rendering by erasing/drawing delta](draw-paddle-1.png)

*Figure 12: Paddle rendering*

Instead, we calculate the deltas - the area the paddle is no longer in is erased and the new area the paddle is in is drawn resulting is a much faster re-render (see Figure 13).


![Optimized paddle rendering by erasing/drawing delta](draw-paddle-2.png)

*Figure 13: Optimized paddle rendering by erasing/drawing delta*


### Game States

The game progresses through various states, such as countdown, gameplay, and game over.

## References

[1] Texas Instruments, “CC3200 Peripheral Driver Library User's Guide,” CC3200 peripheral driver library user's guide: GPIO_GENERAL_PURPOSE_INPUTOUTPUT_API, 18-Feb-2016. [Online]. Available: https://software-dl.ti.com/ecs/cc31xx/APIs/public/cc32xx_peripherals/latest/html/group___g_p_i_o___general___purpose___input_output__api.html#ga97225829b669d27fb53bf01e5771fb5b. [Accessed: 03-Apr-2023].
