# unlimited-axes-lua
This repo has code you can put on your NodeMCU Lua-based ESP8266 devices so they can act as an add-on axis on a CNC machine.

Each new device acting as an unlimited axis does the following things:
- Announces its existence on the network (via UDP to anyone listening)
- ChiliPeppr (or other end-client) should show all announced devices in a list
- Allow each axis to get an axis ID (currently AA, AB, AC, AD, etc.)
- Allow for a planner upload of commands ahead of the Gcode job starting. Each command has a unique counter number.
- Allow for a planner wipe at any point where device wipes its list of commands
- Allow for a hardware sync signal that the device counts to know when to execute associated unique counter number commands.
- Allow for a command saying what sync signal counter to start counting from (typically zero)

Here is a typical setup:
- A NodeMCU device is controlling a dispenser. The dispenser has 3 sub-devices including 1) a linear slide powered by a stepper motor, 2) a rotating auger powered by a stepper motor, and 3) an air pressure valve to provide backing pressure to the dispenser fluid/paste.

A typical device would support Gcode-like commands such as:
- G0 Z-75 (lowers linear slide down by 75mm) (main CNC controller should wait)
- M3 S100 (starts rotating auger at 100rpm) (no need for wait here, rather main CNC controller should start moves immediately)
- M5 (stops auger from rotating) (no need for wait)
- G0 Z0 (raises linear slide) (main CNC controller should wait)

