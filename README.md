# COBO Sensors
We interfaced with the following sensors from COBO in this project:
- FTI2 Incline/IMU.
- FTM2 Speed, direction, distance and angle radar sensor.
- FTU1 Ultrasonic sensor for distance measurements.
- FTP2 IMU

These sensors work on [CANopen](https://en.wikipedia.org/wiki/CANopen), a high-level protocol based on the CAN bus standard, designed for modular and real-time control in industrial automation systems.\
\
For interfacing with these sensors, we used a [Lawicel CANUSB adapter](https://www.canusb.com/products/canusb/).\
\
The setup for a CANUSB seems quite specific to the USB and the system. We originally used [this guide](https://pascal-walter.blogspot.com/2015/08/installing-lawicel-canusb-on-linux.html) to set up. On one Jetson this went perfectly, when attempting to replicate the set up on another Jetson we encountered the problem that the Serial CAN module (slcan) could not be found when running `sudo lsmod | grep slcan` and `sudo modprobe slcan`. This seemed to be a problem with the linux installation and was not solveable without significant effort.


## ROS 2 Integration
For integration with ROS2, we used a GitHub repository from ROS4SPACE, [ros2can_bridge](https://github.com/ROS4SPACE/ros2can_bridge.git). Some changes had to be made to get the code working, mostly renaming attributes which did not match what they were supposed to. A more detailed documentation of what has changed can be found in the commit logs.
1. Clone this repository to your src folder
  ```bash
  cd ~/dev_ws/src/ && git clone https://github.com/luca-rn/ros2can_bridge.git
  ```
2. Install necessary dependencies and build the workspace
  ```bash
  cd ~/dev_ws && sudo rosdep install --frompaths src --ignore-src –r -y
  sudo apt install ros-humble-can-msgs
  colcon build --packages-select ros2socketcan_bridge can_msgs
  ```

## Using the CANBus with ROS 2
Assuming you have been able to follow the instruction from [Pascal Walter's guide](https://pascal-walter.blogspot.com/2015/08/installing-lawicel-canusb-on-linux.html), using the CANBus with ROS 2 should be relatively simple.

1. First, check than the CANBus is working after plugging the the USB:\
  Note: you do not need to bring up the slcan interface again if you succesfully made it permanent earlier. Also, if ttyUSBX is different then it needs to be changed, it can be checked by running "ls /dev" in terminal
  ```bash
  sudo slcand -o -c -f -s4 /dev/ttyUSB0 slcan0
  sudo ifconfig slcan0 up
  candump slcan0
  ```
  The COBO sensors must be given a start command, this can be done by sending the following with another terminal:
  ```bash
  cansend slcan0 000#0100
  ```
  Confirm that the candump is publishing messages. A stop command can then be sent with
  ```bash
  cansend slcan0 000#0200
  ```
  See the [CANopen wikipedia page](https://en.wikipedia.org/wiki/CANopen) to learn more about the various CANopen protocol command messages.
2. Test the sensors in ROS 2
  To start the ROS 2 node, use the command:
  ```bash
  ros2 run ros2socketcan_bridge ros2socketcan slcan0
  ```
  You can check what topics are published using
  ```bash
  ros2 topic list
  ```
  The topics `CAN/slcan0/transmit` and `CAN/slcan0/receive` should appear.\
  \
  The topic names are structured in 2 field names and the transmit and receive topic. The first field name “CAN” identifies the topic within ROS2.0 as a CAN Topic. The ‘CAN_bus_name’ (slcan0) identifies the CAN bus within an building block, because multiple CAN buses can be connected. A ROS 2.0 to CAN Bridge node is always coupled to one BUS bus. The name of the CAN Bus can be adjusted with a command line argument, e.g. “ros2can_bridge slcan1”.\
  \
  Again, to get the sensors to begin transmitting you must send a start command. Use the following command to test this.
  ```bash
  cd ~/dev_ws
  ros2 topic pub /CAN/slcan0/transmit can_msgs/msg/Frame "{id: 0x000, dlc: 2, data: [0x01, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00], err: 0, rtr: 0, eff:0}" --once
  ```
  You can then check if data is being sent using:
  ```bash
  ros2 topic echo CAN/slcan0/receive
  ```
  To stop transmitting, use the command
  ```bash
  ros2 topic pub /CAN/slcan0/transmit can_msgs/msg/Frame "{id: 0x000, dlc: 2, data: [0x02, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00], err: 0, rtr: 0, eff:0}" --once
  ```

## ROS2 Message Type
The message definitions are done in the can_msgs package. It provides the "frame" message type for topics. 

### Frame:
```
std_msgs/Header header
uint32 id
uint8 dlc
uint8[8] data
uint8 err
uint8 rtr
uint8 eff
```

# Resources Used
[Guide to Setting up a Lawicel CANUsb](https://pascal-walter.blogspot.com/2015/08/installing-lawicel-canusb-on-linux.html)
