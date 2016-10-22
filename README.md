# unlimited-axes-lua
This repo has code you can put on your NodeMCU Lua-based ESP8266 devices so they can act as an add-on axis on a CNC machine to effectively give you unlimited axes.

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

A typical device like a dispenser would support Gcode-like commands such as:
- M7 (use coolant on cmd to turn on air pump)
- G0 Z-75 (lowers linear slide down by 75mm) (main CNC controller should wait)
- M3 S100 (starts rotating auger at 100rpm) (no need for wait here, rather main CNC controller should start moves immediately)
- M5 (stops auger from rotating) (no need for wait)
- G0 Z0 (raises linear slide) (main CNC controller should wait)
- M9 (turn off air pump)

In the main Gcode file it is necessary to uniquely identify each sub-device's commands and not confuse them with the main CNC controller's commands:
- M7 (cnc - coolant on)
- M3 S3000 (cnc - spindle on at 3,000 rpm)
- M5 (cnc - spindle off)
- M9 (cnc -coolant off)
- G0 Z10 (cnc - move spindle to clearance height)
- Dispenser.M7 (dispenser - air pump on)
- G0 Dispenser.Z-75 (dispenser - lower linear slide)
- Dispenser.M3 S100 (dispenser - turn on auger)
- G1 X100 (cnc - move 100mm on x axis while dispensing fluid)
- A.M5 (dispenser - turn off auger)
- G0 A.Z0 (dispenser - raise linear slide)
- A.M9 (dispenser - turn off air pump)

Alternatively, taking the lead from TinyG's Active Comments, here is an approach using those so we stick with standard Gcode.
- G0 Z10 (move spindle to clearance height)
- G0 X0 Y0 (move to home)
- (DISPENSER. Make 10mm long paste line along x axis.)
- ({Cayenn:{Dev:DispenserPaste,Cmd:LinearSlide,Axis:A}}) (This macro will insert motor config for linear slide to use A axis)
- (Configure A axis to linear slide settings. 18 deg per step = 20 steps per rotate. 0.5mm per rotation )
- {"4":{"sa":18,"tr":0.5,"mi":1,"po":0,"pm":3,"pl":0}}
- (Axis mode is std like a linear axis. Max G0 fr 500mm/min. Travel max 72mm)
- {"a":{"am":1,"vm":500,"fr":500,"tn":0,"tm":72}}
- M7 ({Cayenn:{Dev:DispenserPaste,Cmd:AirOn}}) (Turn on Air pump)
- G0 A-70 (Move linear slide down to -70mm)
- M9 ({Cayenn:{Dev:DispenserPaste,Cmd:AugerOn,Speed:10}}) (Turn on auger at 10mm/min output)
- (Move 10mm along x axis)
- G1 X10 F100 (100mm/min)
- M7 ({Cayenn:{Dev:DispenserPaste,Cmd:AugerOff}}) (Turn off auger)
- M9 ({Cayenn:{Dev:DispenserPaste,Cmd:AirOff}}) (Turn off Air pump)
- G0 A0 (Move linear slide back to 0)

This Gcode works. It needs a G4 pause after each coolant command to give ESP8266 100ms to trigger the next callback on coolant pin. Also had to shorten comments to not crap out TinyG.
G21
G0 Z10 (move spindle to clearance height)
G0 X0 Y0 (move to home)
M7 G4 P0.1 ({Cayenn:{Dev:"DispenserPaste",Cmd:"LinearSlide",Axis:"A"}})
(Configure A axis to linear slide settings. )
{"4":{"sa":18,"tr":0.5,"mi":1,"po":0,"pm":3,"pl":0}}
(Axis mode is std like a linear axis.)
{"a":{"am":1,"vm":500,"fr":500,"tn":0,"tm":72}}
M9 G4 P0.1 ({Cayenn:{Dev:DispenserPaste,Cmd:AirOn}}) (Turn on Air pump)
G0 A-70 (Move linear slide down to -70mm)
M7 G4 P0.1 ({Cayenn:{Dev:DispenserPaste,Cmd:AugerOn,Speed:10}})
(Move 10mm along x axis)
G1 X10 F100 (100mm/min)
M9 G4 P0.1 ({Cayenn:{Dev:DispenserPaste,Cmd:AugerOff}})
M7 G4 P0.1 ({Cayenn:{Dev:DispenserPaste,Cmd:AirOff}})
G0 A0 (Move linear slide back to 0)
M9 G4 P0.1
