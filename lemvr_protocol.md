# LEMVR Protocol (Version 0.1)
This protocol uses an aligned-buffer concept. Packets are passed back and forth as large buffers of a prenegotiated size. Individual data are stored in these buffers aligned to their size. Libraries should be provided with simple functions like getInt or putLong that write directly to the buffer. Application code should not write directly to the buffer.

Each block of data starts with a 4-byte integer indicating the number of blocks following it that form a chain. Each frame, one chain should be sent containing multiple packets. This allows for 1 read call per frame (or 2 if the block size is too small) and 1 write call per frame.

Packets start with a 1-byte enum indicating their type. The length of the packet is not explicitly provided in the data. The amount of data to be written or read depends on the protocol version which is agreed on at the beginning of the connection.

**Unless otherwise specified**, the following assumptions can be made in this protocol:
- Data types are little-endian
- Data types are unsigned
- Numbers are decimal unless prefixed with "0x" in which case they are hexadecimal
- Floating point types are IEEE-754 and single precision
- Matrices are row-major
- Time is monotonic with an unspecified epoch
- If a long, time is in nanoseconds, if a double, time is in seconds
- Data types are specified in Java-like nomenclature

# Handshake Stage

## Version Handshake
The connection starts with the client sending two magic bytes followed by 2 bytes indicating the protocol version (major then minor). The server then sends two different magic bytes and then a 2-byte boolean indicating if it supports that version.
```
  Client:
    0x4C 0x4D 0x00 0x01
  Server:
    0x55 0x52 0x01 0x00 (accepted) OR
    0x55 0x52 0x00 0x00 (rejected)
```

Note that the magic bytes sent are ascii 'LM' (client) and 'VR' (server)  
This means that the name of this protocol is 'LMVR' and if either side gets an unexpected value it should disconnect.

## Buffer Size Agreement
The client sends magic bytes along with a 16-bit suggested buffer size.  
The server returns with different magic bytes and a 16-bit buffer size. The size returned by the server will be used for this connection.  
```
  Client:
    0x42 0x46 0x00 0x04 (1024)
  Server:
    0x46 0x52 0x00 0x08 (2048)
```

## Time Synchronization
Timer offset is determined with the median latency from N trials

- Client sends int number of trials (N)
- Repeat N times:
  - Client sends client time
  - Server responds with server time
  - Client takes time between send and response, divides it by 2, this is assumed latency
- Client takes the median of all collected latencies, and subtracts it from the most recent receipt time
- The difference between the last received server time and this time is the offset.
- Client sends offset as a double. Offset is server time - client time.

# Metadata Stage
From now on, the protocol is defined in terms of chains. See the first section for more info.

This phase consists of one chain sent by the client requesting metadata, and one chain sent by the server responding to the requests. The packets to do so are defined below

These packets can also be used in the render stage, and should be responded to in the next inbound chain

## Render Size per Eye (client-to-server)
Packet enum: 32
- *no extra data*

## Render Size per Eye (server-to-client)
Packet enum: 33
- (short) width
- (short) height

## Available Framerates (client-to-server)
Packet enum: 34
- *no extra data*

## Available Framerates (server-to-client)
Packet enum: 35
- (byte) numOptions
- (float\[8\]) framerates


# Render Stage
During this stage, the connection follows a loop corresponding to the application frame loop. Per frame, the server sends one chain, and then the client sends one chain. The server sends information like tracking data and controller input and control signals, and the client responds with haptic data and frame submit metadata.

## Frame metadata (server-to-client)
Packet enum: 36
- (double) predicted display time
- (float\[16\]) left view matrix
- (float\[16\]) right view matrix
- (float\[16\]) left projection matrix
- (float\[16\]) right projection matrix

## Poses (server-to-client)
Sends one pose for a tracked object. Which object the pose is for is designated by an enum  
Packet enum: 37
- (byte) Tracked Object Id (0 = headset, 1 = left hand, 2 = right hand)
- (double) Time of pose
- (float\[3\]) Position in stage space
- (float\[4\]) Orientation quaternion
- (float\[3\]) Linear velocity in stage space
- (float\[3\]) angular velocity

## Input (server-to-client)
Controller input. The client should remember values received from this packet; until it receives an overriding input, it should assume the input hasn't changed  
Packet enum: 38
- (byte) hand (0 = left, 1 = right)
- (byte) input id
  - 1: BUTTON_PRIMARY (BOOLEAN)
  - 2: BUTTON_SECONDARY (BOOLEAN)
  - 3: TRIGGER (ANALOG)
  - 4: GRIP (ANALOG)
  - 5: JOYSTICK (ANALOG_2D) (raw values, no deadzone)
  - 6: JOYSTICK_BUTTON (BOOLEAN)
  - 7: MENU (BOOLEAN)
- data, type depends on input id
  - BOOLEAN: 1 byte, 0 = false, anything else = true)
  - ANALOG: float between 0 and 1
  - ANALOG_2D: 2 floats between -1 and 1, first is x, second is y

## Haptics (client-to-server)
Sent once per frame. In the future this should be reworked to pass buffers with specified time  
Packet enum: 39
- (float) leftIntensity
- (float) rightIntensity

## Submit Frame (client-to-server)
No layer definition should come after this packet
Packet enum: 40
- (double) frame display time

## Define Projection Layer (client-to-server)
Packet enum: 41
- (long) Texture ID left
- (long) Texture ID right
- Image Rect (left)
  - (int) x offset
  - (int) y offset
  - (int) height
  - (int) width
- Image Rect (right)
  - (int) x offset
  - (int) y offset
  - (int) height
  - (int) width
- (float\[3\]) position
- (float\[4\]) orientation quaternion
- (float\[16\]) texture coordinates from tan angles (left)
- (float\[16\]) texture coordinates from tan angles (right)

**TODO:** Support other layer types

## End of Chain (both ways)
Packet enum: 99
- *no extra data*

## End Session (server-to-client)
Packet enum: 42
- *no extra data*

## End Session (client-to-server)
Packet enum: 43
- *no extra data*
