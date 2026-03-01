---
title: DAQ CAN standard
---

## Overview
The DFR CAN standard is a standardized layout for CAN(FD) frames. Its most important to have a unified consistent format for the Identifier, as the data field is just whatever data goes along with the ID.

## Identifier Format
CANFD allows for 11-bit or 29-bit (standard and extended id's respectively) CAN Frame identifiers. We will be using 29 bits as we are allowed much more flexibility this way.
Our 29 bits are divided like this:

\[3 bits priority]\[5 bit target id]\[16 bit command]\[5 bit source id]

As you can see, every device on the bus must have a 5 bit id. The first 3 bits are used to determine priority, so that higher priority messages win arbitration. The 16 bit command is self explanatory: you have a table of commands and assign each one a meaning for your device/host to agree about what the data field contains (if the data field is not empty that is). For example, a raspi wants to turn an onboard LED on a daq node on or off via CAN. Heres what that message may look like:

ID: \[011]\[00101]\[0000000000000111]\[00001]
ID:\[Prio: mid]\[Target: Daq Node]\[Command: toggle led]\[Source: Raspi]

In order for each device to know what the commands and other device IDs are, we need common command tables and device registries on each device.

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

And heres how its called:
```c
HAL_StatusTypeDef CAN_Transmit(uint8_t priority, uint8_t target, uint32_t cmd_type, uint8_t* pData, uint32_t dlc_bytes) {

	FDCAN_TxHeaderTypeDef txHeader;
	
	// Use the internal helper to set up fixed FD/Extended settings
	
	CAN_InitHeader(&txHeader);
	
	// Build the 29-bit ID: [Priority][Target][Command][Source]
	
	// self_node_id is the 'Source' established during App_Hardware_Init
	
	txHeader.Identifier = BUILD_CAN_ID(priority, target, cmd_type, self_node_id);
	
	// Set the data length (must be an FDCAN_DLC_BYTES_x macro)
	
	txHeader.DataLength = dlc_bytes;
	
	  
	
	return HAL_FDCAN_AddMessageToTxFifoQ(&hfdcan2, &txHeader, pData);

}
```
Super simple!

## STM32 Self Identifying
Imagine you want to run multiple stm32's on the car that use the same firmware. We want to use the same firmware for devices that are able in order to not have to maintain multiple versions of the same code. Since devices respond to commands with the proper response, they can share a lot of the same code. Its mainly up to the controller to determine how it uses each device by requesting the corresponding data from it. Anyways, to reuse firmware, you need some way to have each node get a unique device id (the 5 bit one mentioned above), so that multiple devices arent sharing a device id on your can line. To do this, we can make a table matching STM32 UIDs to CAN device IDs, and have each device identify itself by looking up its CAN id in the fleet table.
```c
static const UID_Mapping_t Fleet_Table[] = {

	{{0x12345678, 0xABCDEF01, 0x55667788}, NODE_ID_FRONT_LEFT},
	
	{{0x87654321, 0x10FEDCBA, 0x99887766}, NODE_ID_FRONT_RIGHT},
	
	{{0x11223344, 0x55667788, 0x99AABBCC}, NODE_ID_REAR_LEFT},
	
	{{0x44332211, 0x88776655, 0xCCBBAA99}, NODE_ID_REAR_RIGHT},
	
	{{0x001E005F, 0x33335101, 0x32313831}, NODE_ID_NUCLEO_1},
	
	{{0x4D3C2B1A, 0x80706050, 0x40302010}, NODE_ID_NUCLEO_2},
	
	{{0xDEADBEEF, 0xCAFEBABE, 0xFEEDFACE}, NODE_ID_DASH}

};

  

NodeHardwareID_t self_node_id = NODE_ID_UNKNOWN;

  

void Identify_Self(void) {

	uint32_t current_uid[3];
	
	current_uid[0] = HAL_GetUIDw0();
	
	current_uid[1] = HAL_GetUIDw1();
	
	current_uid[2] = HAL_GetUIDw2();
	
	  
	
	for (int i = 0; i < sizeof(Fleet_Table)/sizeof(UID_Mapping_t); i++) {
	
		if (current_uid[0] == Fleet_Table[i].uid[0] && 
		current_uid[1] == Fleet_Table[i].uid[1] &&
		current_uid[2] == Fleet_Table[i].uid[2]) {
		
			self_node_id = Fleet_Table[i].nodeType;
			return;
		}
	}
	
	self_node_id = NODE_ID_UNKNOWN; // Node not recognized

}
```

Now you can see each device UID mapped to a CAN ID, and how each stm32 can get its unique CAN ID on startup even though they have identical firmware! Awesome

## CAN ID Filtering
Since we now have device IDs established, we want to set up a filter so that we only read messages meant for each device. That means, we want the target field of the identifier to be our device id, so we aren't reading messages meant for other devices. This is important, since our CAN is interrupt based, we would be triggering an interrupt for every single message on the bus without a filter, and thats just wasted compute. In order to achieve this filtering, we will use a bit mask filter. It works like so:
### Bit mask filtering
This works by ...
