Project for [Robotics Dojo 2025](https://roboticsdojo.github.io/competition2025.html). A few mods to Josh Newans' articubot_one project documented on [github](https://github.com/joshnewans/articubot_one) and [youtube](https://www.youtube.com/playlist?list=PLunhqkrRNRhYAffV8JDiFOatQXuU-NnxT). 

# Quick Links
* [Sources](#sources)
* [Hardware Configuration](#hardware-configuration)
* [Software Dependencies](#software-dependencies)
* [Modified Parameters](#modified-parameters)
    * [Note](#note)
    * [Note on Localization](#note-on-localization)
* [Usage](#usage)
    * [Prerequisites](#preliminary)
    * [Simulation](#simulation)
    * [Real Robot](#real-robot)
* [Simulation Demo](#simulation-demo)
    
* [Misc.](#misc)
* [Future](#future)

# Sources
- Simulation world obtained from the stl file sent by Lenny Ng'ang'a by converting it to a COLLADA file and importing it in Gazebo Fortress then saving it with the robot spawned in it. `src/jkl/worlds/GF.sdf`
- Map evaluation script obtained from [Dr. Shohei Aoki's repo](https://github.com/shohei/Map_Evaluation)
- diffDriveArduino from [RedstoneGithub's fork](https://github.com/RedstoneGithub/diffdrive_arduino) of Josh's diffdrive arduino
- [sllidar](https://github.com/Slamtec/sllidar_ros2) and [rplidar](https://docs.ros.org/en/ros2_packages/humble/api/rplidar_ros/) from those links
- [joy tester](https://github.com/joshnewans/joy_tester) is from Josh
- [serial](https://github.com/joshnewans/serial) is from Josh
- [serial_motor_demo](https://github.com/joshnewans/serial_motor_demo) is from Josh
- [rosArduinoBridge](https://github.com/joshnewans/ros_arduino_bridge) is Josh's fork of [hrobotics work](https://github.com/hbrobotics/ros_arduino_bridge)
  
Our [technical design paper](Technical%20paper%20Team_1%20Knights%20RDJ%202025.pdf), [technical presentation slides](Robotics%20Dojo%202025%20Knights%20Presentation.pdf), and [project poster](knights_poster%20RDJ%202025.pdf) may have a few more details on this project.



# Hardware Configuration
- Raspberry Pi running Ubuntu 22.04, dev machine running Ubuntu 22.04
- RPLidar A1 M8 connected to Pi via USB
- STM32F401 running [ROSArduinoBridge.ino](arduino-code/ROSArduinoBridge/ROSArduinoBridge.ino), connected to Pi USB
- 4 200 RPM motors with built-in encoders, with the encoders connected as shown in [encoder_driver.h](arduino-code/ROSArduinoBridge/encoder_driver.h)
- DRV8833 motor driver, with motor controls connected as shown in [motor_driver.h](arduino-code/ROSArduinoBridge/motor_driver.h)
- MPU6050 over I2C. The arduino sketch enables the DMP for great IMU performance. Also, a calibration sketch is provided by the [electronic cats library](https://github.com/ElectronicCats/mpu6050/wiki).

Here's the bot [here](1L9A4192.JPG), [here](IMG_6601.jpeg)


# Software Dependencies
- Ubuntu 22.04 - ROS2 Humble pair was used here. Docker on windows is not recommended by us because of network woes. The WSL network is isolated from the system network, so ROS on Pi does not communicate with ROS on Docker by default. We were not able to get it working in time. Apart from that, native Ubuntu has way higher performance.
- twist-mux, ros-humble-teleop-twist-keyboard, ros-humble-teleop-twist-joy, slam-toolbox, ros-humble-navigation2, ros-humble-nav2-bringup, ros-humble-gazebo-ros-pkgs, ros-humble-xacro, ros-humble-ros2-control, ros-humble-rplidar-ros, ros-humble-robot-localization
<!-- - ros-humble-gazebo on pc -->
- tmux is not recommended, it was laggy on the Pi over SSH over wifi


# Modified Parameters
## src/jkl/description
### robot.urdf.xacro
The URDF was built using simple shapes. URDF using meshes was attempted [see knight.xacro](pi-code/src/jkl/description/urdf/knight.xacro). The benefit was the mesh was derived from the actual CAD of the robot. However, it was too laggy to update pose, etc. It was suspected that computing collisions using meshes was the cause, hence the creation of the [current URDF](pi-code/src/jkl/description/urdf/simple.xacro). 

Important to note is that the IMU on the actual robot was mounted such that X and Y axes did not match the X and Y axes of the robot. Also please note that for this manufacturer the axes drawn on the module are actually -X and -Y. Verify with your IMU's raw data if you plan to replicate this

![full circuit with axes](full-circuit-with-axes-ms.png)

So in the URDF, the IMU was rotated about the Z axis to match. This also affects the [EKF configuration file](#ekfyaml): we fuse the Y acceleration instead of X because Y is our axis of interest i.e. robot-forward axis. A justifiable assumption is that the imu_link->...->base_link TF's would account for this behind the scenes with the robot_localization package but no it does not at the time of writing. 
Had this been foreseen, the IMU would have been mounted x on X and y on Y. 

The URDF features an offset value for the centre of rotation. When the robot spins in place, it rotates about `base_link` (because the odom TF is odom->base_link). Therefore, it is important to match the centre of rotation of the real robot (or gazebo simulated robot) with the centre of rotation in Rviz. That's the role of `centre_of_rot_offset_x`. Getting this value right greatly improves odometry: when you set the fixed frame to odom in rviz, the laser frame will not drift. 

### gazebo_control.xacro (although not used)
```xml
<wheel_separation>0.207</wheel_separation>
<wheel_radius>0.0425</wheel_radius>
```

### ros2_control.xacro
```xml
<!-- stm32 -->
<param name="device">/dev/serial/by-id/usb-STMicroelectronics_BLACKPILL_F401CC_CDC_in_FS_Mode_206734803156-if00</param> 
<param name="baud_rate">57600</param>
<param name="enc_counts_per_rev">988</param>
```


## src/jkl/config/
### mapper_params_online_async.yaml
```yaml
loop_search_maximum_distance: 6.0 # (default is 3.0). The default value causes the robot to "teleport" during mapping, ruining a good map
```

### nav2_params.yaml
- initial_pose (not defined by default). Helps save time when using localization. A new pose can be given using rviz. Use it by having the odom reset (relaunching launch_robot) then launching online_async.launch.py. Then set the initial pose as 0 0 0
- controller_server: [mppi controller server](https://docs.nav2.org/configuration/packages/configuring-mppic.html) was used (dwb is default). rpp, teb and grace might have worked better, but mppi worked best out of the box
- controller server loop rate 15 Hz to minimize console spamming "control loop missed its 30Hz update rate"
- local_costmap & global_costmap: resolution: 0.02 (0.05 is default). Seems to get robot get stuck less frequently
- local_costmap: footprint used instead of radius, coz the robot is a polygon. Sized just over the robot's size
- global_costmap: radius retained, perhaps less compute intensive than footprint?
- amcl laser_max_range and min_range 12.0 and 0.3 respectively (default is )
- velocity smoother feedback: "CLOSED_LOOP" (default is OPEN_LOOP)
- velocity smoother velocities and accelerations: [0.35, 0.0, -0] and [12.5, 0.0, 8.2] (default are something else) The accelerations are very large. Something better should be identified and used
- [The docs](https://docs.nav2.org/tuning/index.html#inflation-potential-fields) recommends inflation radii that create **smooth** potential fields in the global and local costmaps. However, in testing, it was observed that having overlapping potentials (such as on corridors and parking spots) would cause the robot to fail passing as expected, even though a path had been planned. So the inflation radii were reduced until there was a little empty space in both costmaps at the narrowest corridor of the gamefield. 

We tried using the [theta-star planner](https://docs.nav2.org/configuration/packages/configuring-thetastar.html) instead of the default navfn. The robot plans are shorter and straighter, fixing the weaving issue we had last year. However, the plans hug the inflation radius too closely, causing navigation around simple corners to fail in a manner similar to that observed when the potential fields overlap. With time, perhaps that can be tuned out?

### my_controllers.yaml
For diff_cont
```yaml
wheel_separation: 0.207
wheel_radius: 0.0425
wheels_per_side: 2
odom_frame_id: odom
enable_odom_tf: false # crucial for EKF
```

These fields were added to provide hardware interfaces to the servo and IMU and 2 more wheels
```yaml
controller_manager:
  ros__parameters:
    # existing diff_cont

    # joints doubled for joint_broad
    joint_broad:
      type: joint_state_broadcaster/JointStateBroadcaster
      joints: ['front_left_joint', 'front_right_joint', 'back_left_joint', 'back_right_joint']
    
    imu_broadcaster:
      type: imu_sensor_broadcaster/IMUSensorBroadcaster # https://control.ros.org/humble/doc/ros2_controllers/imu_sensor_broadcaster/doc/userdoc.html

    servo_controller:
      type: position_controllers/JointGroupPositionController # https://control.ros.org/humble/doc/ros2_controllers/position_controllers/doc/userdoc.html


imu_broadcaster:
  ros__parameters:
    sensor_name: imu_sensor # specified in HW interface
    frame_id: imu_link


servo_controller:
  ros__parameters:
    joints:
      - servo # specified in HW interface
```

It is critical that enable_odom_tf in diff_cont should be false because ros2_control should not publish the odom->base_link transform. That role should be done by the EKF node. 

### ekf.yaml
[This config file](pi-code/src/jkl/config/ekf.yaml) is for sensor fusion using the [robot_localization](https://docs.ros.org/en/noetic/api/robot_localization/html/configuring_robot_localization.html) package. The configuration was largely based on the DOCS with one difference: The IMU on the robot was rotated 90 degrees about X axis wrt to the robot axes. So in order to fuse the acceleration along X axis, we actually fuse the reported acceleration of Y axis. 
#### Notes
* `publish_tf` is true. This is because ros2_control is no longer publishing the tf. 
* `imu0_relative` is true. We are only interested in the change in angular rotation about Z, not absolute change. Otherwise the orientation in RVIZ will be inconsistent per-run
* `imu0_remove_gravitational_acceleration` is true. The IMU reports absolute accelerations without subtracting gravity

## src/jkl/launch/
### rplidar.launch.py
The rplidar_ros launch file for A1 was copied and these fields were modified
```python
serial_port = LaunchConfiguration('serial_port', default='/dev/serial/by-id/usb-Silicon_Labs_CP2102_USB_to_UART_Bridge_Controller_0001-if00-port0') # was '/dev/ttyUSB0'
serial_baudrate = LaunchConfiguration('serial_baudrate', default='115200')
frame_id = LaunchConfiguration('frame_id', default='laser_frame') # was 'laser'
```

## [diffdrive_arduino](pi-code/src/diffdrive_arduino/)
Hardware interfaces for 2 additional wheels, servo and IMU were added. The serial commands for the latter two were added both in this package and in ROSArduinoBridge. Check out [arduino comms](/pi-code/src/diffdrive_arduino/src/arduino_comms.cpp), [arduino_comms.cpp](/pi-code/src/diffdrive_arduino/src/arduino_comms.cpp), [diffdrive_arduino.cpp](/pi-code/src/diffdrive_arduino/src/diffdrive_arduino.cpp), and their .h files for more info. 

Note that the state interface for the servo is not implemnted based on the reported servo angle. Instead, it is reported using the previously commanded angle. Although the command is provided in the arduino sketch, at the time, it was more important to prevent frequent checks on the servo position rather than prioritizing fetching the wheel encoder data. So the command interface only sent commands to the MCU if a different angle command was sent. 

This choice created some buggy behaviour with the servo. If launch_robot was executed while the servo was in the open position, publishing [170.0] to the `servo_controller/commands` topic would not close it. So you would have to "open it" using [90.0] first then it would close. There was also a bug where publishing an angle command to the topic only once would have no effect. As of yet, unclear if this is also related. 

A future implementation would definitely update state variable as it should be. 

Check out [diffdrive_arduino.cpp](pi-code/src/diffdrive_arduino/src/diffdrive_arduino.cpp) and its .h file

## Note
- Josh goes over how to obtain wheel diameter, wheel separation and encoder counts per revolution in [this video](https://www.youtube.com/watch?v=4VVrTCnxvSw&list=PLunhqkrRNRhYAffV8JDiFOatQXuU-NnxT)
- **online_async_launch.py** **localization_launch.py**, **navigation_launch.py**, **mapper_params_online_async.yaml**, **nav2_params.yaml** were copied from the default slam-toolbox and nav2_bringup directories (/opt/ros/humble/share/nav2-or slam-toolbox...) then slightly modified (for example in localization_launch.py instead of `get_package_share_directory('nav2_bringup')` use `get_package_share_directory('jkl')`) This is the recommended way to do it because, Josh's articubot_one repo uses ROS foxy. Apart from fewer parameters in nav2 and mapper yaml, some problems might come up like failing to launch navigation. Some elements were renamed in ROS Humble, like recoveries->behaviors
- all instances of serial ports use `dev/serial/by-id` path. This was preferred over `dev/serial/by-path` and `dev/tty*` because it depends on the device ID rather than the port used or connection order
- Electronic cats provides a [calibration sketch](https://github.com/ElectronicCats/mpu6050/wiki/4.-Examples#zero) for the IMU. This improved accuracy greatly. 

## Note on Localization
In testing, it was noted that localization using AMCL (the default for Nav2) was unsatisfactory. The map and laser scan would go out of alignment very quickly, causing navigation to fail. A few attempts were made to tune the [parameters](/pi-code/src/jkl/config/nav2_params.yaml) using [this guide](https://automaticaddison.com/ros-2-navigation-tuning-guide-nav2/) but the performance was still sub-par. A little research showed **many** others shared our discontent and so provided promising alternatives to AMCL:
1. jie_ware youtube [here](https://youtu.be/sZ5_NEt1vI4?si=0gPA8b-hRr3kHton) and github [here](https://github.com/6-robot/jie_ware) ROS1 :(
2. als_ros youtube [here](https://youtu.be/wsoXvUgJvWk?si=MrWsakbt202Rw9hv) and github [here](https://github.com/NaokiAkai/als_ros) ROS1 :(
3. WLOC youtube [here](https://youtu.be/ik6-cSJZV1k?si=ls9vA8XfLilKkfMt) no source could be found :(
4. rtabmap github [here](https://github.com/introlab/rtabmap/tree/humble-devel) huge and reportedly high compute requirements :( However, it supports mapping with stereo RGB-D cameras, 3D lidar, and [a lot more](https://github.com/introlab/rtabmap/wiki/Tutorials)
5. GMCL youtube [here](https://youtu.be/J9ZcCon6k-g?si=GTzK6mh_Yn7lSSTg) and github here
6. neo_localization github [here](https://github.com/neobotix/neo_localization2/tree/humble) docs [here](https://neobotix-docs.de/ros/packages/neo_localization.html)

We only tested neo_localization and jie_ware. Neo_localization was just a drop-in replacement for AMCL, but was indistinguishable from it. Perhaps with a little more tuning and consideration, it could be ironed out. 

Jie_ware was written for ROS1. After a little effort to port it to ROS2, we found it to be very aggressive, exactly what we wanted. We could have tested the others but they either required a lot more effort to integrate. Jie_ware only had 3 source files


# Usage
## 
- Wifi (or ethernet) has to be connected, on both pi and PC whenever you're running ROS, especially if using cyclone DDS as the RMW_IMPLEMENTATION
## Preliminary
- [install ros-humble](https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debs.html)
- add `source /opt/ros/humble/setup.bash` to ~/.bashrc so you don't have to in every new terminal
- install [nav2, slam-toolbox](https://roboticsbackend.com/ros2-nav2-generate-a-map-with-slam_toolbox/), colcon, and optionally [change RMW_IMPLEMENTATION to cyclone_dds](https://roboticsbackend.com/ros2-nav2-generate-a-map-with-slam_toolbox/)
- install controllers, plugins, xacro and twist mux 
```bash
sudo apt install ros-humble-ros2-control ros-humble-ros2-controllers && \
# sudo apt install ros-humble-gazebo-ros2-control && \ # Gazebo Classic went EOL so this won't work, make migration
# sudo apt install ros-humble-gazebo-ros-pkgs && \ 
sudo apt install ros-humble-xacro && \
sudo apt install ros-humble-twist-mux && \
sudo apt install ros-humble-slam-toolbox && \
sudo apt install ros-humble-navigation2 ros-humble-nav2-bringup && \
sudo apt install ros-humble-robot-localization && \
sudo apt install ros-humble-rplidar-ros && \
```
- disable brltty service. This service always takes serial port devices to be braille device, preventing you from accessing your microcontroller. Disable the service
```bash 
systemctl stop brltty-udev.service && \
sudo systemctl mask brltty-udev.service && \
systemctl stop brltty.service && \
systemctl disable brltty.service
```
- enable read/write to serial port, you can do this by adding yourself to the dialout group `sudo usermod -a -G dialout $USER` **then reboot**
- clone the repository in both Pi and dev machine
- cd to the location of src directory in both Pi and dev machine e.g. `cd ~/robotics-dojo-2024/pi-code`
- edit pi-code/src/jkl/description/ros2_control.xacro line 11 to the appropriate for your microcontroller. You can check which is appropriate by connecting the microcontroller to your machine then running `ls /dev/serial/by-id`. You can then add it as appropriate
- run `colcon build --symlink-install` in both Pi and dev machine. It may throw a lot of errors and/or warnings on first build, just ignore and run `colcon build --symlink-install` again. This should be fixed in future versions of this project.
- run `source install/setup.bash` on each terminal (or add `source yourworkspace/install/setup.bash` to ~/.bashrc) in both Pi and dev machine

## Simulation
Everything is run on pc. Instead of running `ros2 launch jkl launch_robot.launch.py` **and** `ros2 launch jkl rplidar.launch.py`, use `ros2 launch jkl launch_sim.launch.py use_sim_time:=true world:=./src/jkl/worlds/dojo2024` 
Change every instance of `use_sim_time:=false` to `use_sim_time:=true`
All else is as below

## Real Robot
- cd to the location of src directory in both Pi and dev machine e.g. `cd ~/robotics-dojo-2024/pi-code`
- On PC run `rviz2 -d ./src/comp.rviz`
- On Pi, run `ros2 launch jkl launch_robot.launch.py`
- On Pi in another terminal run `ros2 launch jkl rplidar.launch.py`. You may also use `ros2 launch rplidar_ros rplidar_a1_launch.py` or `ros2 launch sllidar_ros2 sllidar_a1_launch.py`. In a few tests, we noticed that the latter two run at 8kHz, instead of the former's 2kHz. Also, these launch files show the lidar is in `Sensitivity` mode rather than `Standard` mode. We're not sure if there are benefits in using these two over Josh's. 

### For mapping phase
- On PC (Pi is also fine but during competition we found PC was better, did not have the "teleportation" issue) run `ros2 launch jkl online_async_launch.py slam_params_file:=./src/jkl/config/mapper_params_online_async.yaml use_sim_time:=false`
- On PC or Pi run `ros2 run teleop_twist_keyboard teleop_twist_keyboard`
- you may also use gamepad instead of keyboard, but you'll have to configure the joystick yaml appropriately. Josh demonstrates the process well [here](https://www.youtube.com/watch?v=F5XlNiCKbrY&list=PLunhqkrRNRhYAffV8JDiFOatQXuU-NnxT&index=17). But in short run `ros2 run joy joy_node` in one terminal then `ros2 run joy_tester test_joy` in another. Press a key, identify its id using the GUI and use that id in the jkl/config/joystick.yaml. Having configured the yaml correctly, you can use `ros2 launch jkl joystick.launch.py` in place of or in conjuction with teleop_twist_keyboard. joystick **does not** require its terminal window to be in focus to work
- With the teleop_twist_keyboard terminal in focus use `u i o j k l m , .` to drive the bot around until a satisfactory map is shown in rviz
- Open slam toolbox panel in rviz and fill the input boxes with a desired name. Click save map and serialize map. Four map files will be saved in current directory

### For navigation phase
- (on pc or pi, but pc is more powerful) You may use slam_toolbox or localization_launch for localization. slam_toolbox is not recommended since it (currently) overwrites (or updates) the existing map with new scans. If you wish to proceed anyway, this is how you do it: change `mode: mapping` to `mode: localization` and uncomment `map_file_name:` followed by path to the **serialized** map file (i.e. the file that has .data or .posegraph extension), without extension, in src/jkl/config/mapper_params_online_async.yaml. Then run same command as when mapping `ros2 launch jkl online_async_launch.py slam_params_file:=./src/jkl/config/mapper_params_online_async.yaml`.
- To use localization_launch (AMCL localization), kill online_async if it was running then run  `ros2 launch jkl localization_launch.py map:=path_to_map.yaml use_sim_time:=false`. To set/change pose estimate, you can use rviz
- To use jie_ware localization package: [jie_ware](pi-code/src/jie_ware/)

The **jie_ware** package provides advanced **LIDAR-based localization** capabilities. It allows the robot to automatically localize itself within its operational area using **only lidar data** and the map, making it highly suitable for autonomous navigation and mapping tasks. That means the odometry need not be precise, or even remotely accurate as [some](https://www.reddit.com/r/ROS/comments/1iwslhb/amcl_alternatives_found_a_lightweight_lidaronly/) have used it with a **static!** odom->base_link TF without trouble. 

This package was originally developed for **ROS 1** and is available here:  
 [https://github.com/6-robot/jie_ware](https://github.com/6-robot/jie_ware)  
It has been successfully **ported to ROS 2** for integration with this project.

---

###  Running the Localization Node
To use **jie_ware** localization, run the following command:

```bash
ros2 launch jie_ware lidar_loc.launch.py map:=<map_name>.yaml use_sim_time:=false
```

Replace `<map_name>.yaml` with the desired map file that you have saved.

When running, the robot will automatically localize itself if it is within its known mapped area.

---

###  Demo Video
A demonstration of the **jie_ware localization** package can be found here:

 [Watch on YouTube](https://youtu.be/sZ5_NEt1vI4?si=CmsiGj_6Rxt5UtrA)

---

###  Key Features
- Automatic localization within known mapped environments
- LIDAR-based positioning for accuracy and robustness
- Compatible with ROS 2 navigation stacks
- Fully ported from ROS 1 for better performance and integration
### Running navigaton

* **(On PC or Raspberry Pi — though PC is recommended due to the Pi’s computational limitations)** run the following command to launch navigation:

  ```bash
  ros2 launch jkl navigation_launch.py map_subscribe_transient_local:=true params_file:=./src/jkl/config/nav2_params.yaml use_sim_time:=false
  ```

* Instead of using the RViz navigation plugin to set waypoints, **PyTrees** was adopted, following Dr. Aoki’s tutorial: [PyTrees Navigation Tutorial](https://youtu.be/EqSgKu8-xEU?si=lQmXRzx3Bd9OWw90).

* The script `pi-code/src/nav_to_poses/nav_to_poses/nav_to_poses.py` was used to execute the navigation tasks defined in the YAML configuration file:

  ```
  pi-code/src/nav_to_poses/nav_to_poses/config/nav_goals.yaml
  ```

* After launching navigation, the script was executed using:

  ```bash
  ros2 run nav_to_poses nav_to_poses
  ```

* Tasks requiring external script execution from within `nav_to_poses.py` were handled using Python’s `subprocess.run()` method, as described in the [official documentation](https://docs.python.org/3/library/subprocess.html).

* For full autonomy, the **Raspberry Pi Camera 2 (picamera2)** stream was continuously active on the Raspberry Pi. Frames were captured on-demand for specific tasks, including **potato leaf disease detection** and **colour detection** of the white and blue cubes.


## Computer Vision

## [rdj2025_potato_disease_detection](pi-code/src/rdj2025_potato_disease_detection/)

###   Project Overview
This package is part of a computer vision system for detecting **potato leaf diseases** using **deep learning** on a **ROS 2** framework. It supports both **RTSP-based live camera feeds** from a Raspberry Pi Camera Module 2 and **locally saved test images**. The model can classify potato leaf conditions and display the confidence level of each prediction as either early blight, late blight or healthy.

---

###   Requirements / Dependencies
Before running, ensure the following are installed:

- **Hardware:** Raspberry Pi (with Camera Module 2 - IMX219)
- **Operating System:** Raspberry Pi OS or Ubuntu 22.04 (with ROS 2 Humble)
- **Dependencies:**
  - Python 3.10+
  - ROS 2 (installed and sourced)
  - OpenCV (`opencv-python`)
  - PyTorch (`torch`, `torchvision`)
  - FFmpeg (for video streaming)
  - rpicam-vid (camera streaming utility)

Install missing dependencies via:

```bash
sudo apt install ffmpeg
pip install torch torchvision opencv-python
```

---

###   Start Raspberry Pi Camera Stream
Before running the required nodes, ensure the **Raspberry Pi Camera Module 2** is streaming using:

```bash
rpicam-vid -t 0 --nopreview --inline --codec mjpeg --width 1920 --height 1080 \
--framerate 10 --hflip --vflip --awb daylight --autofocus-mode manual \
-o - | ffmpeg -f mjpeg -i - -c:v copy -f rtsp -rtsp_flags listen rtsp://0.0.0.0:8554/cam
```

You may modify parameters such as autofocus or white balance, but these defaults provided stable results.

  **Reference:** [Raspberry Pi Camera Software Documentation (IMX219 V2)](https://www.raspberrypi.com/documentation/computers/camera_software.html)

---

###  Running the ROS 2 Nodes
To run the two nodes, open separate terminals and execute:

1. **Potato Disease Detection Node:**
   ```bash
   ros2 run rdj2025_potato_disease_detection potato_disease_detection_node
   ```

2. **Image Publisher Node:**
   ```bash
   ros2 run rdj2025_potato_disease_detection publish_test_image
   ```

The second node can be configured in `setup.py` to switch between **RTSP streaming** or **local images**.

---

###  Node Locations
- Detection Node:  
  `pi-code/src/rdj2025_potato_disease_detection/rdj2025_potato_disease_detection/potato_disease_detection_node.py`

- RTSP Image Publisher:  
  `pi-code/src/rdj2025_potato_disease_detection/rdj2025_potato_disease_detection/rtsp_image_publisher.py`

- Test Image Publisher (Local):  
  `pi-code/src/rdj2025_potato_disease_detection/rdj2025_potato_disease_detection/test_image_publisher.py`

---

###  Model Details
- **Commented code** in the following files uses the model trained by **RDJ 2025 organizers**:
  - `inference_engine.py`
  - `potato_disease_detection_node.py`

    Repository: [https://github.com/roboticsdojo/rdj2025_potato_disease_detection](https://github.com/roboticsdojo/rdj2025_potato_disease_detection)

- **Uncommented sections** use the custom model trained by this team:
  - `pi-code/src/rdj2025_potato_disease_detection/models/potato_disease_model.pth`

This custom model achieved higher accuracy and provided classification confidence levels for each prediction.

## Colour Detection

## [colour_door_controller](pi-code/src/colour_door_controller/)

This package is part of the **Computer Vision** system and controls a **servo-driven door mechanism** based on detected colours from a live RTSP video stream.

---

###  Overview
The package contains two main nodes:

1. **Colour Detector Node** – Publishes the detected colour from the RTSP stream.  
   Location: `pi-code/src/colour_door_controller/colour_door_controller/colour_detector.py`

2. **Colour to Door Service Node** – Controls the servo motor based on the detected colour (supports **blue** and **white**).  
   Location: `pi-code/src/colour_door_controller/colour_door_controller/colour_to_door_service.py`

These nodes together enable a door automation mechanism that responds to specific colour detections in real-time.

---

###  Prerequisites
Before running these nodes, ensure that the **Raspberry Pi Camera Module 2** is streaming using the following command:

```bash
rpicam-vid -t 0 --nopreview --inline --codec mjpeg --width 1920 --height 1080 \
--framerate 10 --hflip --vflip --awb daylight --autofocus-mode manual \
-o - | ffmpeg -f mjpeg -i - -c:v copy -f rtsp -rtsp_flags listen rtsp://0.0.0.0:8554/cam
```

You may modify the camera parameters (e.g., white balance, autofocus), but the above configuration is recommended for best performance.

---

###  IP Configuration
To ensure proper RTSP streaming, verify that **all instances of the IP address** in the scripts match your Raspberry Pi's IP address.

In VS Code, you can search for all IP instances using:
```
Ctrl + Shift + F
```

Alternatively, you can pass the IP as a runtime parameter when launching the detector node:

```bash
ros2 run colour_door_controller colour_detector --ros-args -p IP:=<your_IP>
```

---

###  Running the Nodes
1. **Run the Colour Detector Node:**
   ```bash
   ros2 run colour_door_controller colour_detector
   ```

2. **Run the Colour to Door Service Node:**
   ```bash
   ros2 run colour_door_controller colour_to_door_service
   ```

These two nodes will work together — the detector identifies the colour (blue or white), and the service node controls the servo motor accordingly.

---

### Notes
The current implementation is a little biased toward white detection. A gentle blur before processing might help. 

# Demos
This section describes how to run the robot. Just to make sure you have all the required packages, run this:
```bash
sudo apt-get update && sudo apt-get install \
    ros-humble-demo-nodes-cpp \
    ros-humble-teleop-twist-keyboard \
    ros-humble-xacro \
    ros-humble-twist-mux \
    ros-humble-robot-localization \
    ros-humble-ros2-control \
    ros-humble-ros2-controllers \
    ros-humble-rplidar-ros \
    ros-humble-ros-gz \
    ros-humble-slam-toolbox \
    ros-humble-navigation2 \
    ros-humble-nav2-bringup
```
## Simulation Demo
This is for autonomous navigation demo only. Two launch files are provided for running the simulation, one for the mapping phase, another for the navigation phase. Run SLAM follows:
```bash
cd pi-code
colcon build --symlink-install && source install/setup.bash
ros2 launch ./launch_sim_slam.yaml
```

in another terminal
```bash
ros2 run teleop_twist_keyboard teleop_twist_keyboard
```

Drive around until your map looks good. Then save the map using the slam_toolbox panel in RViz. The map will be saved in the directory where you ran the launch file.

Then run navigation:
```bash
cd pi-code
colcon build --symlink-install && source install/setup.bash
ros2 launch ./launch_sim_nav.yaml map:=./yourMapName.yaml
```

# Misc.
- the lidar sometimes acts up when attempting to launch. If it fails to launch even after mutliple retries, (1) try **unplug and replug** connections, i.e., usb connection on pi, usb on lidar, the lidar daughter board cables, other daughter board cable motor side and laser side, you get the idea (2) try using a different type-c cable, or different power supply altogether for the raspberry pi. This was an issue in 2024 but not 2025.
- Update on above issue: Josh posted [a video](https://youtu.be/PddIeZP-wgw?si=uCTXcbRP3fAoC-ss) about power supplies for Pi. He suggested this change as one fix at timestamp 5:45.
```bash
sudo rpi-eeprom-config --edit
```
A file will open with nano-like editor. Add this line to it
```
PSU_MAX_CURRENT=5000
```
Tried 3000 (coz its Pi 4), Josh tried 4000 on his Pi 5 but it would not work in both cases, even though it seemingly should. But 5000 seems a little reliable for both. 
- you can save changes to comp.rviz (or the default rviz config) when you hit Ctrl+S so that you don't have to make the same changes when relaunching rviz
- for simulations, slow computers may produce unrealistic navigation behaviours, like aborting too soon. Gazebo is a resource hog. So try to launch launch_sim in one pc then everything else in another pc configured in the same way, connected in the same network 
- use float where float is used in yaml files, otherwise stuff may fail to launch
- sometimes things don't work very well, relaunch and reboot are a must for both Pi and PC
- rplidar.launch.py sometimes takes a few tries to get working, also try disconnecting arduino then launch lidar first
- use `export ROS_DOMAIN_ID=some-number-that-is-not-the-default-0` per terminal or in bashrc to avoid communicating with other people's ROS environments. Do this for both your PC and PI
- cyclonedds is recommended over fastrtps because ["it doesn't work well with nav2"](https://roboticsbackend.com/ros2-nav2-tutorial/). Install using `sudo apt install ros-humble-rmw-cyclonedds-cpp` and then per terminal or in bashrc `export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp`
- Raspberry running ubuntu 22.04 sometimes takes a long time to boot. Using `systemd-analyze blame`, it seems to be caused by `systemd-networkd-wait-online.service`. A workaround is turning `optional` to `false` in `/etc/netplan/50-cloud-init.yaml`. A better fix should be identified. 

# Future
- Fix weaving. When the path is relatively straight the robot follows a winding path, it weaves left and right, which might cause it to hit an obstacle. In the competition, we reduced the maximum angular velocity to work around this.
- Make the robot smaller (particularly wheel_separation), like [Atom x RoboQueens robot](atom-roboqueens-robot.png)  ---> ✔️
- Isolate the arduino and motor driver, tristate buffer and/or relay? 
- Find a way to power lidar motor properly. Why was the USB hub buzzing when switched on? Modify a USB cable, so that an additional power cable is soldered to the 5V line, for simplicity and compactness
- Get a really good USB PD power bank for the Raspberry Pi
- Get USB-C trigger boards, or build battery pack with BMS to avoid overdischarging motor power supply like last time
- Wifi sucks. Check if you can get a better AP or ditch wifi altogether and use thin ethernet cable
- Try increasing rates (like decreasing map update interval, increase controller frequency, map publish rates, robot velocities etc) in mapper params and nav2 params
- Try [RPP](https://docs.nav2.org/configuration/packages/configuring-regulated-pp.html#), TEB, [Graceful](https://docs.nav2.org/configuration/packages/configuring-graceful-motion-controller.html) and [DWB](https://docs.nav2.org/configuration/packages/configuring-dwb-controller.html) controller servers (with and without [rotation shim](https://docs.nav2.org/configuration/packages/configuring-rotation-shim-controller.html))
- Restore the original ISR in ROSArduinoBridge using the newfound knowledge in register operations. Should have a small speed boost.
- Try using a differert fork of supporting packages (like serial and serial motor demo) because the ones used have easy_install deprecation warnings during colcon build --symlink-install
- Try using very high resolution laser scan and local and global costmaps to see if the bot ever gets stuck (without modifying robot radius that is)
- Try a different path planner, like [smac](https://docs.nav2.org/configuration/packages/configuring-smac-planner.html) or [thetastar](https://docs.nav2.org/configuration/packages/configuring-thetastar.html), maybe they will perform better with higher resolution costmaps. When thete-star was tested with resolution 0.05, generated paths were straighter and often passing through the robot radius regions hence the robot would get stuck in recovery easily. It also fixed the weaving issue mentioned earlier in this section. Also explore effect of deadband velocity, as well as other params n this file.
- Try and use `ros2 launch sllidar_ros2 sllidar_a1_launch.py` or `ros2 launch rplidar_ros rplidar_a1_launch.py` instead of jkl rplidar.launch.py to see if the 8kHz and 'Sensitivity' vs 2kHz and 'Standard' makes it better ---> ✔️ Used `ros2 launch rplidar_ros rplidar_a1_launch.py frame_id:=laser_frame`
- Try using a different bt_navigator xml such as the ones in jkl/config/behavior_trees. You specify the one to use in nav2_params.yaml
- Explore effect of other params in my_controllers.yaml. There's also closed-loop there, might be valuable
- Make the mapping phase also autonomous
- Incorporate IMU, maybe this has benefits ---> ✔️
- Try depth camera instead of lidar
- Make a simple BMS, can add a simple MOSFET switch to prevent overdischarge from battery. Can also configure battery pack to be able to charge using LiPo charger, something like 3s 2p config?
- a guest on Tech Expo said we should check AWS Deepracer 
- Maybe explore [Ackermann](https://en.wikipedia.org/wiki/Ackermann_steering_geometry) steering? :)
- Have an mini PC as part of the robot to run all needed software to do away with WIFI & Hotspot problems during the competition. Maybe this might do [HP EliteDesk 705 G4 Microtower PC - Ryzen 5 Pro 2400GE 3.2GHz, 8 CPUs, 8GB RAM, 256GB NVMe M.2 SSD, Radeon RX Vega 11 Graphics, DP, Gigabit LAN](https://bestsella.co.ke/product/hp-elitedesk-705-g4-microtower-pc-ryzen-5-pro-2400ge-3-2ghz-8-cpus-8gb-256gb-nvme-m-2-ssd-radeon-rx-vega-11-graphics-dp-gigabit-lan/)























