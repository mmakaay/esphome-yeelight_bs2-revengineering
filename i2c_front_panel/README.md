# Yeelight Bedside Lamp 2 Front Panel I2C protocol

The front panel of the device contains a two touch buttons (power on/off and
color selection) and a touch slider (for setting the brightness level).
This panel is lit when the device is turned on. The light behind the slider
will represent the actual brightness setting of the device.

<img src="i2c_front_panel/images/front_panel.jpg" width="150">


## Communication

The front panel is a fully separated component, with its own control chip.
Communication between the ESP chip and the front panel is done using:

- An I2C bus
  - the front panel is the I2C slave, the ESP is the I2C master
   (pardon the standard terminology, I am aware of the controversy)
  - the front panel device ID is 0x2C
  - SDA is connected to ESP pin GPIO21
  - SCL is connected to ESP pin GPIO19
- An interrupt data line to signal the ESP about new events
  - this line is connected to ESP pin GPIO16
  - the default state is HIGH
  - line is pulled LOW for at least 6 ms when a new event is avilable

Commands can be written to and data can be read from the front panel
component using I2C. The I2C protocol is fairly simple. All read and write
operations uses 6 bytes of data. No register selection is done before
reading or writing.

The interrupt data line is used by the front panel, to signal the ESP that
a new button or slider event is available. Further details on this can be
found below.

The reverse engineered 6 byte sequences that can be read and written,
can be found in the file [i2c_front_panel/i2c_commands.txt](i2c_front_panel/i2c_commands.txt)


## Connection to the main board

The front panel is connected to the main board using a flat cable.
The picture below shows the connector on the main board, including the
functions of the cable pins:

<img src="i2c_front_panel/images/front_panel_flat_cable_connection.jpg" width="400">


## Writing commands to the front panel

Commands can be written to the front panel at any time.

The available commands can be summarized as:

- Turn on the front panel light
- Turn off the front panel light
- Set the light behind the slider to a certain level

The exact byte sequences that can be written for these commands can
be found in the abovementioned I2C commands file.


## Reading events from the front panel

The types of events that can occur can be summarized as:

- Touch or release the power button
- Touch or release the color selection button
- Touch or release the slider at a certain level

Because the front panel is an I2C slave device, it cannot contact the ESP
via I2C. Only an I2C master device can initiate communication.
Thefore, when the front panel has a new event available, it will pull down
the interrupt line for a short period of time, to signal the ESP about this
new event.

*Note that the ESP needs to poll the interrupt line at least at 667 Hz to be
able to trustworthy detect the 6 ms signal. Unfortunately, the interrupt line
does not wait for the ESP to respond to its signalling.*

After detecting this signal, the ESP must first write the "READY FOR EVENT"
command (01 00 00 00 00 00 01) to the front panel.

After the front panel has ACK'ed this command, the ESP can read 6 bytes,
which will represent the event that occurred.

The mapping from a received byte sequence into the type of event that
occurred can be found in the abovementioned I2C commands file.

