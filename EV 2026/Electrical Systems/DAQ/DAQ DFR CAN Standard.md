
## Overview
The DFR CAN standard is a standardized layout for CAN(FD) frames. Its most important to have a unified consistent format for the Identifier, as the data field is just whatever data goes along with the ID.

## Identifier Format
CANFD allows for 11-bit or 29-bit (standard and extended id's respectively) CAN Frame identifiers. We will be using 29 bits as we are allowed much more flexibility this way.
Our 29 bits are divided like this:

\[3 bits priority]\[5 bit target id]\[16 bit command]\[5 bit source id]

As you can see, every device on the bus must have a 5 bit id. The first 3 bits are used to determine priority, so that higher priority messages win arbitration. The 16 bit command is self explanatory: you have a table of commands and assign each one a meaning for your device/host to agree about what the data field contains (if the data field is not empty that is). For example, a raspi wants to turn an onboard LED on a daq node on or off via CAN. Heres what that message may look like:

ID: \[011]\[00101]\[0000000000000111]\[00001]
ID:\[Prio: mid]\[Target: Daq Node]\[Command: toggle led]\[Source: Raspi]

In order for each device to know whwat the commands and other device IDs are, we need common command tables and device registries on each device.

```c
typedef enum {

CMD_ID_PING = 0x001,

CMD_ID_REQ_DATA = 0x050,

CMD_ID_SENDING_DATA = 0x051,

CMD_ID_RESET_NODE = 0x099,

CMD_ID_SET_LED = 0x100, // Data[0]: 0=Off, 1=On

CMD_ID_SET_FREQ = 0x101,

CMD_ID_RESET_SIM = 0x102, 

CMD_ID_SET_OFFSET = 0x103

} CommandID_t;
```


Building a command table is really up to the developer. I could think of a trillion ways to toggle an led. You could have a general Update System command with each bit in the data field turning something on or off, or have individual "turn led on" or "turn led off" commands with an empty data field. I would stray away from the latter implementation, and instead but actual configuration data in the data field so we aren't running out of commands and can keep commands unique but to each their own.

```c
typedef enum {

NODE_ID_ALL_NODES = 0x01, // 00001 (Broadcast)

NODE_ID_FRONT_LEFT = 0x02, // 00001

NODE_ID_FRONT_RIGHT = 0x03, // 00010

NODE_ID_REAR_LEFT = 0x04, // 00011

NODE_ID_REAR_RIGHT = 0x05, // 00100

  

NODE_ID_NUCLEO_1 = 0x06, // 01010

NODE_ID_NUCLEO_2 = 0x07, // 01011

NODE_ID_RASPI = 0x1E, // 11110

NODE_ID_DASH = 0x1F, // 11111 (Node 31)

NODE_ID_UNKNOWN = 0x00

} NodeHardwareID_t;
```

To map devices to CAN ID's, each device should have a table like above. This makes your code readable by being able to see which devices you are sending messages to without having to go look up the table and map the id every time.

I think thats all we need to make our CAN messages now! Heres a macro I use to properly create CAN ID's while taking care of C bit shifts and proper endian-ness:
```c

#define BUILD_CAN_ID(priority, target, cmd, source) \

((((uint32_t)(priority) & 0x07) << 26) | \

(((uint32_t)(target) & 0x1F) << 21) | \

(((uint32_t)(cmd) & 0xFFFF) << 5) | \

(((uint32_t)(source) & 0x1F)))

```