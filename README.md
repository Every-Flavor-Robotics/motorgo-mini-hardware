# motorgo-mini-hardware
## This repo is the home of the hardware revisions for the MotorGo-mini!
### This is a condensed and specialized version of the MotorGo-Mini, featuring:
- Esp32s3 based system, compatible with ESPIDF or Arduino
- Wifi + BT
- Single channel of motor control
- Integrated MT6701 magenetic encoder on back of board
- DRV8316 TI motor controller with low-side current sense
- 3A continous 8A peak FOC motor controls
- RGB LED indicator
- USB PD x2 with power ORing to protect power sources
- 6P JST PH input: 5V-20V
- CAN bus over USB-C (experimental use of sideband signals in USB 3.2+)

### Its still in development, particularly USBPD seems to be malfunctioning.  But checkout the schematics, renders, use example, and photos of the real board!
### Full board Render:
<img width="1920" height="1080" alt="Gimbusv1r2_straight_on" src="https://github.com/user-attachments/assets/03f2b88f-b317-427f-a4dc-35fae66f907a" />

### Example board + motor + rendering:
<img width="1920" height="1080" alt="Gimbusv1r2_exoploded_viewpng" src="https://github.com/user-attachments/assets/b383c6a5-df42-402b-8c88-80b8fc85c27a" />

### Top Level Schematic (full color):
<img width="1355" height="932" alt="image" src="https://github.com/user-attachments/assets/aa09e942-9a86-4f7f-be79-dd4052c4e0ec" />

### USB Module Sub-Schematic:
<img width="1353" height="931" alt="image" src="https://github.com/user-attachments/assets/c74ad26a-c476-4541-a65a-f12fe209ff12" />

### DRV8316 Sub-Schematic:
<img width="1354" height="930" alt="image" src="https://github.com/user-attachments/assets/622b72df-e74d-4205-8dce-01470cef7635" />

### RGB indicator:
<img width="1352" height="931" alt="image" src="https://github.com/user-attachments/assets/7ce5b652-d832-4f73-9bc7-ef70c0b74dab" />
