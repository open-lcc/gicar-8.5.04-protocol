# General Architecture

## Architecture
In the Lelit Bianca and in the Lelit Elizabeth, the Gicar 8.5.04 is set up as an IO Expander. The LCC drives the display and buttons, and acts as a PID controller. The Gicar 8.5.04 control board does power management, controls relays, and ADC. These two boards communicate via a bus, which is the main focus of this project.

The bus is 6 wires with the following purpose:

* Pinout (red wire is pin 6):
  * Pin 1: 12V for OLED (Measured 12.6V)
  * Pin 2: TX (to Control board, 3V3)
  * Pin 3: RX (from Control board, 5V)
  * Pin 4: GND
  * Pin 5: 3V3 for OLED (Measured 3.1V)
  * Pin 6: 3V3 for MCU(Measured 3.1V)

The TX and RX pins is a UART running at 9600 bps 8N1 with inverted signalling. The LCC acts as the bus master. It sends a message to the control board, and the control board replies.

## LCC to Control Board messages

```
80:11:17:00:28
80 ii jj bb zz

ii = Shift register 2
jj = Shift register 1
bb = Front buttons, 0x08 if the minus button is pressed, 0x04 if the plus button is pressed, and 0x0B if both.
zz = Checksum, CheckSum8 Modulo 128, skipping the first byte
```

### The shift registers

In the control board there are two shift registers, which are unlabled on the PCB, but which I call Shift Register 1 and Shift Register 2. They are both STPIC6C595's, and SER_OUT of SR1 is connected to SER_IN of SR2. SR1 gets its data from the control board MCU (a STM8S003F3P6).

Here's bitmask values for what the different bits in the shift registers do:

```
typedef enum : uint8_t {
    GICAR_SHIFT_REGISTER_2_CN5  = 1 << 0,
    GICAR_SHIFT_REGISTER_2_FA10 = 1 << 4,
    GICAR_SHIFT_REGISTER_2_FA9  = 1 << 5,
} GicarShiftRegister2Flags;

typedef enum : uint8_t {
    GICAR_SHIFT_REGISTER_1_CN6_1 = 1 << 0,
    GICAR_SHIFT_REGISTER_1_CN6_3 = 1 << 1,
    GICAR_SHIFT_REGISTER_1_CN6_5 = 1 << 2,
    GICAR_SHIFT_REGISTER_1_CN9   = 1 << 3,
    GICAR_SHIFT_REGISTER_1_FA7   = 1 << 4,
    GICAR_SHIFT_REGISTER_1_FA8   = 1 << 5,
    GICAR_SHIFT_REGISTER_1_CN10_DISABLE_OLED_12V = 1 << 6, // Maybe. They are connected to some kind of transistor that would presumably cut power to 12V on CN10 if used, but this has not been tested yet
    GICAR_SHIFT_REGISTER_1_CN10_DISABLE_OLED_3V3 = 1 << 7, // Maybe. They are connected to some kind of transistor that would presumably cut power to 3V3 OLED on CN10 if used, but this has not been tested yet
} GicarShiftRegister1Flags;
```

From what I can tell, drains 1-3 and drains 6-7 on SR2 are unconnected. When the control board starts up, it starts up in a safe state, i.e. all relays off and all solenoids closed. However, **it does not reset to a safe state if it stops receiving messages from the LCC**. If the LCC is about to stop sending messages, it is therefore *highly* advisable to first send LCC packets to set the control board to a safe state.

#### Exceptions

In some machines, the MCU firmware also looks at the values of the shift register bytes for some of its outputs.

In the Lelit Elizabeth firmware, GicarShiftRegister2Flags is extended with the following:

```
    GICAR_SHIFT_REGISTER_2_CN7  = 1 << 1, 
```

CN7 is not connected to a shift register, but rather to the MCU. It's also not an open drain, but a GPIO with a 5V pull up resistor. In the Elizabeth it is configured as an a 5V output, and in the Lelit Bianca, as an input. 

## Control board to LCC messages

```
81:00:00:5D:7F:00:79:7F:02:5D:7F:03:2B:00:02:05:7F:67
81 lu cc cc cc ss ss ss CC CC CC SS SS SS tt tt tt zz

l = CN2 input, 0 = open (water tank not empty), 4 = closed (water tank empty)
u = CN7 input, 0 = open, 2 = closed
c = CN4 ADC value, low gain
C = CN4 ADC value, high gain
s = CN3 ADC value, low gain
S = CN3 ADC value, high gain
t = CN1 ADC Capacitance value (service boiler water level probe, full value is around 128, once the level drops it goes to around 600 pretty quick)
z = Checksum, CheckSum8 Modulo 128, first byte skipped
```

### Weird number conversion

The ADC values have a with a weird encoding. The high/low gain thing are a speculation on my part, but it seems likely. The code for converting a 3 byte number to an uint16 is as follows:

```c
if (weirdNum[2] == 0x7F) {
    return (weirdNum[1] | 0x80) + (weirdNum[0] << 8);
} else if (weirdNum[2] == 0x00) {
    return weirdNum[1] + (weirdNum[0] << 8);
}
```

### ADC value to temperatures

The Lelit Bianca and Lelit Elizabeth uses 50k NTCs as their temperature probes. The temperature probe ADC values can first be converted to ohms, using this function for the high gain value:

```c
uint32_t high_gain_adc_to_ohm(float floatValue) {
	// Experimentally found using a resistance decade. Presumably not strictly correct, but it produces correct enough numbers in our span.
    return 7181.23*pow(-(floatValue-1018.15)/(floatValue-1.56789),(8000.f/8043.f));
}
```

The resistance is then converted to degrees celsius using the Steinhart-Hart equation:

```c
float ntc_ohm_to_celsius(uint32_t ohm, uint32_t r25, uint32_t b) {
    return (1.f/(log(((float)ohm)/((float)r25))/((float)b)+(1.f/298.15)))-273.15;
}
```

The R25 value of the NTCs is 50k, and the β value seems to be 4018, though further experimentation could yield more insights here.