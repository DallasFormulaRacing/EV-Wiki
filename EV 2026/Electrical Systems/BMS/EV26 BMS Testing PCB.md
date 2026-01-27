## Overview
We will use this PCB to simulate battery cells, that can be connected to the BMS segment boards, so we can test cell readings & communications between segments.

The Segment PCB has 2 Molex FFC (Flat Flexible Cable) connectors, these two connectors route voltage taps and thermistors to the segment.
- [39-53-2204](https://www.digikey.com/en/products/detail/molex/0039532204/2421516) - 20 Pin FFC 
- [39-53-2084](https://www.digikey.com/en/products/detail/molex/0039532084/2405221) - 8 Pin FFC
	- Cable used between segment and testing pcb: [1.25mm Pitch Premo-Flex FFC Jumper](https://www.molex.com/en-us/products/part-detail/151680255), Same Side Contacts, cable length TBD

## Block Diagram
![[EV26 BMS Testing PCB-1769487137590.png]]
![[Pasted image 20260126220456.png]]

## Connector Wiring
- I designed the top and bottom PCBs to be interchangeable, but unfortunately, the wiring to both the connectors is different for both boards.
- The Testing PCB will have 2 8-pin connectors and 2 20-pin connectors 
	- For the Bottom configuration, just connect the Green and Purple Labels, ignore the Red. 
	- For the Top configuration, just connect the Red and Purple Labels, ignore the Green. 
![[Pasted image 20260126220519.png]]

## Testing PCB Requirements
- [ ] Make a 2 or 4 layer PCB. Small PCB isn't that important, since its only for testing, keep it under 8"x8" 
- [ ] 24 Cell simulator:
	- [ ] Use a resistor divider to divide the input supply. 
	- [ ] 24 x 4.2V = 100.8V, We don't have a power supply this high, but we can still test using our 30V power supply (I'm working on getting a higher voltage supply)
- [ ] 20 Thermistors
	- [ ] Use NTC Thermistors: [NCU15XH103F6SRC](https://www.digikey.com/en/products/detail/murata-electronics/NCU15XH103F6SRC/10702829)
- [ ] Add a way to connect to the 30V power supply, maybe something like the power supply cables used in labs
- [ ] Add a enable/disable switch for each cell connection, so we can test Open-Wire, I was thinking a DIP switch like this:![[Pasted image 20260126220536.png|372x372]]
- [ ] Add plastic standoff screws to the PCB, we don't want the PCB to be touching the table we put it on

