there are a large number of sensors within the Battery Management System
1. ADBMS6830 Cell monitors
2. 10k NTC Thermistors
3. Current sensor
4. IMD
in addition to these sensors, the BMS also has other external hardware attached to its CAN port:
5. the charger
6. the laptop (for the gui)
the bms is responsible for forwarding internal TB CAN traffic to the rest of the vehicle on the VCAN line, but there are some caveats:
- while the charger is on the 