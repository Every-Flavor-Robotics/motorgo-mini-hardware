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

- **USB Power Delivery: STUSB4500 (x2)**
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
 

```
Atopile Project Generation Prompt
"Hello Cursor, please generate an atopile project for a brushless motor controller named 'gimbus'. Use the following specification to create the necessary files and connections.

First, please add these components to the project using their LCSC part numbers:

Component	Part Number	LCSC Part #
MCU	ESP32-S3-WROOM-1-N8R8	C2913217
Motor Driver	DRV8316CRHBR	C2965823
Buck Regulator	XL1509-3.3E1	C84852
USB-PD IC	STUSB4500QTR	C495471
CAN Transceiver	TJA1051T/3/1J	C2999411
Encoder	MT6701CT-STD	C2868822
USB-C Connector	TYPE-C-31-M-12	C164639
USB MUX	FSUSB42MUX	C127926
Schottky Diode	SS34	C8676
Now, create the following file structure and populate the files with the specified components and connections."

gimbus/gimbus.ato (Main Project File)
Python

# gimbus.ato
import mcu
import power
import motor_driver
import feedback
import ui
import connectors

component GimbusController:
    # Instantiate all the modules
    mcu = new mcu.MCUBlock
    power = new power.PowerBlock
    motor_driver = new motor_driver.MotorDriverBlock
    feedback = new feedback.FeedbackBlock
    ui = new ui.UIBlock
    connectors = new connectors.ConnectorBlock

    # --- Top Level Connections ---
    
    # Power Rails
    power.v_bus ~ connectors.v_bus
    power.v_in_pd ~ connectors.v_in_pd
    signal gnd
    signal vcc_3v3
    power.v_out_3v3 ~ vcc_3v3
    mcu.power_3v3 ~ vcc_3v3
    motor_driver.power_3v3 ~ vcc_3v3
    feedback.power_3v3 ~ vcc_3v3
    ui.power_3v3 ~ vcc_3v3
    connectors.power_3v3 ~ vcc_3v3
    
    # Communication Buses
    mcu.spi ~ motor_driver.spi
    mcu.i2c ~ feedback.i2c
    mcu.can ~ connectors.can_bus
    mcu.usb ~ connectors.usb_lines
    
    # Control Signals
    mcu.motor_pwm ~ motor_driver.pwm_in
    mcu.mux_select ~ connectors.mux_select_ctrl
    ui.status_led_ctrl ~ mcu.led_pin

gimbus/mcu.ato
Python

# mcu.ato
from "esp32s3_wroom_1.ato" import ESP32S3WROOM1N8R8

module MCUBlock:
    mcu = new ESP32S3WROOM1N8R8

    # --- Power ---
    power_3v3 = mcu.p3v3
    mcu.gnd -> gnd

    # --- Communication Signals ---
    # USB OTG for programming
    usb = new Bundle {
        d_p = mcu.gpio20;
        d_n = mcu.gpio19;
    }

    # SPI for motor driver
    spi = new Bundle {
        sck = mcu.gpio12;
        mosi = mcu.gpio11;
        miso = mcu.gpio13;
        cs = mcu.gpio10;
    }

    # I2C for magnetic encoder
    i2c = new Bundle {
        scl = mcu.gpio9;
        sda = mcu.gpio8;
    }

    # CAN bus to transceiver
    can = new Bundle {
        tx = mcu.gpio5;
        rx = mcu.gpio4;
    }

    # --- Control Signals ---
    # PWM for motor driver
    motor_pwm = new Bundle {
        a = mcu.gpio1;
        b = mcu.gpio2;
        c = mcu.gpio3;
    }
    
    # Control signal for the USB MUX
    mux_select = mcu.gpio18
    
    # Status LED
    led_pin = mcu.gpio35
gimbus/power.ato
Python

# power.ato
# Note to AI: The XL1509-3.3E1 and SS34 will need to be modeled.
# The STUSB4500 reference design should be used, including ideal diodes.

module PowerBlock:
    # --- Main 3.3V Buck Converter ---
    # Takes high voltage V_BUS and steps down to 3.3V
    v_bus = new Power
    v_out_3v3 = new Power

    buck = new Block {
        ic = new XL1509_33E1
        diode = new SS34 # Required Schottky diode
        # Add inductor and capacitors as per datasheet
        # ...
    }
    buck.vin ~ v_bus
    buck.vout ~ v_out_3v3
    
    # --- USB-PD Section ---
    v_in_pd = new Power # This will be the main input before diode OR-ing
    
    pd_a = new STUSB4500_Circuit
    pd_b = new STUSB4500_Circuit
    
    # Ideal diodes should be placed on the output of each PD circuit
    # before they combine into the v_bus signal.
    pd_a.vout ~ ideal_diode1.anode
    pd_b.vout ~ ideal_diode2.anode
    ideal_diode1.cathode -> v_bus
    ideal_diode2.cathode -> v_bus
gimbus/motor_driver.ato
Python

# motor_driver.ato
# Note to AI: Model DRV8316CR and a 3-pin screw terminal or JST connector.

module MotorDriverBlock:
    driver = new DRV8316CR
    motor_connector = new JST_XH_3
    
    power_3v3 = driver.vcc
    driver.vm ~ v_bus # Connects to the high-voltage rail
    driver.gnd -> gnd

    # SPI connection
    spi = driver.spi

    # PWM inputs
    pwm_in = new Bundle {
        a = driver.inha;
        b = driver.inhb;
        c = driver.inhc;
    }
    
    # Motor phase outputs
    driver.out_a ~ motor_connector.p1
    driver.out_b ~ motor_connector.p2
    driver.out_c ~ motor_connector.p3
gimbus/feedback.ato
Python

# feedback.ato
# Note to AI: Model Magntek MT6701

module FeedbackBlock:
    encoder = new MT6701
    
    power_3v3 = encoder.vdd
    encoder.gnd -> gnd
    
    # I2C connection
    i2c = encoder.i2c
    
    # Configuration pins
    encoder.mode -> power_3v3 # Set to I2C mode
    encoder.cs -> power_3v3   # Tie CS high as specified
gimbus/ui.ato
Python

# ui.ato
# A simple block for a status LED.

module UIBlock:
    status_led = new LED
    resistor = new Resistor(value=330ohm)
    
    power_3v3 ~ resistor.p1
    resistor.p2 ~ status_led.anode
    status_led.cathode -> gnd
    
    status_led_ctrl = new Digital # This signal is currently unused in this basic setup
gimbus/connectors.ato
Python

# connectors.ato
# This module contains the physical connectors and routing logic.

module ConnectorBlock:
    # --- Instantiate Connectors and ICs ---
    port_a = new USB_C_Connector # LEFT
    port_b = new USB_C_Connector # RIGHT
    
    usb_mux = new FSUSB42MUX
    can_transceiver = new TJA1051T
    term_resistor = new Resistor(value=120ohm)
    term_jumper = new Jumper_2Pin
    
    # --- Power Connections ---
    v_bus = new Power # Combined VBUS rail
    v_in_pd = new Power # Power to the PD circuits
    port_a.vbus ~ v_in_pd
    port_b.vbus ~ v_in_pd
    power_3v3 = can_transceiver.vcc
    
    # --- USB OTG Routing ---
    # Route D+/D- from ports to the MUX
    port_a.dp ~ usb_mux.d_p_in_a
    port_a.dn ~ usb_mux.d_n_in_a
    port_b.dp ~ usb_mux.d_p_in_b
    port_b.dn ~ usb_mux.d_n_in_b
    
    # Route MUX output to the MCU
    usb_lines = new Bundle {
        d_p = usb_mux.d_p_out;
        d_n = usb_mux.d_n_out;
    }
    
    # MUX Control Logic
    # Simple priority: If Port A is connected, select it. Otherwise, default to B.
    # A voltage divider from port_a.vbus should create the control signal.
    mux_select_ctrl = usb_mux.select
    
    # --- CAN Bus Pass-Through Topology ---
    # Create the main bus by directly connecting the ports
    signal can_h
    signal can_l
    port_a.sstx_p ~ can_h
    port_b.sstx_p ~ can_h
    port_a.sstx_n ~ can_l
    port_b.sstx_n ~ can_l
    
    # Tap the transceiver into the bus
    can_transceiver.canh ~ can_h
    can_transceiver.canl ~ can_l
    
    # Connect transceiver to MCU signals
    can_bus = new Bundle {
        tx = can_transceiver.txd;
        rx = can_transceiver.rxd;
    }

    # Add the switchable termination resistor
    term_jumper.p1 ~ can_h
    term_jumper.p2 ~ term_resistor.p1
    term_resistor.p2 ~ can_l

```