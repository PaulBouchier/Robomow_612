# Robomow_612
Code and other content related to the Robomow 612 autonomous mower project

## Project Definition

The project definition broadly is as follows:

- uses Robomow 612p mechanics, motors, sensors
- Mostly does not use existing electronics (TBD, battery management and seek-to-dock are possible exceptions). I.e. the design should be applicable to a range of mowers of that class given suitable modifications.
- Electronics based on Raspberry Pi, ESP32, off the shelf motor controller, ublox RTK gps, cellular Internet access, custom wiring and interface to 612p motors and sensors including bump sensor 
- other sensors as required, e.g. IMU, encoders
- ROS software based on linorobot2
- does not include me doing a bunch of work to enable person A's favorite component - generally, you should be competent to design your own modifications to the design.

## Current assessment

I'm confident of getting it able to navigate between GPS waypoints. I'm less confident of being able to make it navigate in arbitrary yards avoiding obstacles and mowing stripes within a geo fenced boundary - that is the main goal but is not assured. IOW you shouldn't spend a bunch of money assuming any particular outcome - this is an advanced hobby project, not a product. 
