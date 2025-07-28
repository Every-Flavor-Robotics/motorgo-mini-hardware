## This project is to make a small, bussed servo controller compatible with the motorgo ecosystem.
---
### Objectives and context:
1. Many of our previous boards are designed for maximum versatility - but this board is specifically for a small selection of motors.  **Gimbal motors** around the 45mm diameter. 
2. There is an explicit intent to design this board for use in the KineThreads project that is a part of Vivian Shen's research in HRI.  The KineThread project is an exosuit that enables VR and AR haptics by applying forces to the exosuit's joints via tensioned threads.  Each motor on the suit controls a spool of thread and the tension is currently roughly approximated with voltage to each motor.  This project should be able to provide **current control** - increasing the resolution and consistency of tension on the suit.
3. We have not been able to get **USB-PD** reliably working on our boards before and this project will be a test for us in adding that to our ecosystem.
4. Creating this project in atopile will be a new workflow and this relatively simple project should be a good testing ground for this new tool!

### Requirements:
- SimpleFOC compatibility
- Esp32 based for compatibility to the motorgo ecosystem
- Motor controls with voltage range of up to 24V
- Current control on the motor phases, low side or inline is acceptable
- board size must fit within 45mm diameter circle and its maximum height is 10mm
- CAN bus and two physical ports so that a chain of actuators can be made
- USB-PD 5a 20v
- 4 layer board stackup with complete ground plane
- 1 power indicator led in red, 1 motor enabled indicator led in blue, and one neopixel rgb led for  status indication
- two tactile switches for boot and reset 
- magnetic encoder on center of board, opposite side of esp32
- 3 phase motor wire output

### Component Selection
- **Microcontroller: ESP32-S3-WROOM-N8**
	**Reasoning:** A powerful and cost-effective module with integrated Wi-Fi and Bluetooth. It has all the necessary hardware peripherals for this project, including a native CAN controller (TWAI), USB OTG for programming, SPI, and I2C.

- **Motor Driver: DRV8316CR**
	-**Reasoning:** A highly integrated 3-phase motor driver that simplifies the design by including FETs and current sense amplifiers. It supports up to 35V and communicates via SPI for configuration and fault reporting. Its internal buck regulator will not be used to power the ESP32.

- **Buck Regulator: XL1509-3.3E1**
	- **Reasoning:** This switching regulator was chosen to provide a stable 3.3V rail for the entire board. It accepts an input voltage of up to 40V, providing a safe margin, and can deliver up to 2A, which is more than sufficient for the ESP32's peak current draw during radio use.
	- **Implementation Note:** As a non-synchronous converter, this part requires an external Schottky diode for proper operation.

- **USB Power Delivery: HUSB238A-BB001-QN16R (x2)**
	- **Reasoning:** A dedicated USB-PD controller that simplifies power negotiation over USB-C. The design uses two of these ICs, one for each port, to allow for up to 20V @ 5A power delivery. The outputs will be fed into a common power rail via ideal diodes to prevent back-powering.

- **CAN Transceiver: TJA1051T**
	- **Reasoning:** A standard, reliable, 3.3V-compatible CAN transceiver.
	- **Implementation Note:** Each board will feature a 120Ω termination resistor selectable via a jumper or DIP switch. This allows any board to be configured as a termination point if it is at the physical end of the daisy-chain.

- **Encoder: Magntek MT6701**
	- **Reasoning:** A high-resolution magnetic encoder (16,384 counts/rev) compatible with existing software. It communicates via I2C. The MODE pin will be tied to VDD to select I2C operation.

- **USB-C Connectors: XKB_Connectivity_U262_241N_4BV64 (x2)**
	- **Reasoning:** Standard USB-C connectors chosen for their high density and the availability of robust, full-featured cables.

## Circuit & Logic Design
**The board will feature two identical USB-C ports (LEFT and RIGHT) to allow for daisy-chaining of power and data.**

- **Power Delivery:** The CC and VBUS lines from each port are routed to their own dedicated STUSB4500 PD controller.
- **USB Programming MUX:** The D+/D- lines from both ports are routed to a FSUSB42 MUX to allow programming from either port.
- **Switching Logic:** The VBUS from each port will be used to automatically control the MUX's select pin. To prevent a short circuit, a logic gate (e.g., an OR or NOR gate) will be used to combine the signals, creating a priority system where one port takes precedence if both are connected.
- **CAN Bus Topology:** The board will be wired as a pass-through node.
	- The SSTX+/- lines from the LEFT port will be routed directly to the SSTX+/- lines of the RIGHT port.
	- The CAN transceiver's CANH and CANL pins will "tap into" this main bus line with the shortest possible traces.
	- This design maintains a single, continuous bus line through the device, minimizing stubs and ensuring high signal integrity.

### Scratch notes and dev todos:
==**Links to stuff:**==
 > usbc cables: https://www.amazon.com/dp/B0CS64PLTC/?
 > adafruit can dev: https://www.adafruit.com/product/5708?
	 TJA1051T/3/1J
	 > ![[Pasted image 20250721132601.png]]
 > usbc dev: https://www.digikey.com/en/products/detail/stmicroelectronics/EVAL-SCS001V1/10231579?
 > ESP32s3 Tech Ref Manual: https://www.espressif.com/sites/default/files/documentation/esp32-s3_technical_reference_manual_en.pdf
 > ESP32s3 module datasheet: https://www.espressif.com/sites/default/files/documentation/esp32-s3-wroom-1_wroom-1u_datasheet_en.pdf
 > DRV8316 block diagram application:
 > ![[Pasted image 20250722132813.png]]
 ![[Pasted image 20250722134537.png]]
 > ![[Pasted image 20250722134249.png]]

Impedance control for jlc stackup: 90 ohm and 120 ohm lines (CAN and USB)
![[Pasted image 20250723125023.png]]
- CAN TX needs 5v supply - so i had to add a boost conv off of the 3v3 rail, luckily i had designed a good headroom on the 3v3 - here's what I'm adding based on the input current calc:
	- **Power Out:** Pout​=5V×70mA=350mW
	- **Power In:** Pin​=ηPout​​=0.85350mW​≈412mW
	- **Input Current:** Iin​=Vin​Pin​​=3.3V412mW​≈125mA
	- =) C490380, the DIODES PAM2401YPADJ for simplicity even though its overkill

For usb pd protection / power ORing chip is the lm73100 
- VIN(UV) = 1.22V * (270kΩ + 100kΩ) / 100kΩ = 4.514V
- VIN(OV) = 1.22V * (160kΩ + 10kΩ) / 10kΩ = 20.74V
- isense from ideal diode to get board current:
	- R_imon = V_imon /G* I_imon 
		- R_imon = 1.65 Kohm resistor
		- set ESP32 to ADC_ATTEN_DB_6 (0mv -1750mv)
	- zener diode choose for 1.8v to clamp spikes
