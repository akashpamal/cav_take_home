# Cavalier Autonomous Racing

Hello! The goal of this take-home is to give you an introduction to what it is like to work with C++ / ROS2 and data we record during every run. 

We want to see you become comfortable with writing code with the tools that run directly on our car and testing this code in open-loop (i.e. by replaying old data). A strong understanding of how to visualize your results will be critical to succeed here. We will evaluate every submission, along with the attached videos (further explained below), on quality and completion.

This task is difficult, so we highly recommend joining our Slack (see the bottom of this document), ask strong questions in Slack, and start this work well before the deadline. The primary focus should be putting in effort for quality work while the secondary focus is finishing all the metrics. It is ok to use generative AI (this is not a homework assignment!), but in our experience ChatGPT / Claude can be misleading for working in ROS2 so don't put to much trust in it. 

## Task 0: Install ROS2

Robot Operating System (ROS) is a framework that we use to develop our racing stack and interface with the car. The first step of this take home assignment is to install ROS2. **Make sure to install version humble.** These instructions are easiest to follow using Linux (Ubuntu 22.04). We recommend using Linux natively with dual boot, but you we also have instructions to set up Ubuntu 22.04 on MacOS or Windows.

[MacOS Instructions here](./MacOS.md)

[Windows (WSL) Instructions here](./WSL.md)

Regardless of how you get to Ubuntu 22.04, you should install ROS2 using the following instructions:

Linux: https://docs.ros.org/en/humble/Installation/Alternatives/Ubuntu-Development-Setup.html



## Task 1: RACECAR Dataset

Provided here (https://github.com/linklab-uva/RACECAR_DATA) is a link to a public dataset our team has published. In it is an explanation of the data and tutorials that explain how to use it. After installing ROS2, please follow Tutorial 1 & 2. Although we aren't checking for the completion of this task explictly, it will make the following task significantly easier. One recommendation is instead of using RViz as seen in the tutorial is to use foxglove since it is more modern and is what we use heavily in our testing.

1. Install foxglove https://foxglove.dev/ and `sudo apt install ros-humble-foxglove-bridge`

2. Launch foxglove and launch the foxglove bridge `ros2 launch foxglove_bridge foxglove_bridge_launch.xml`. Make sure you source your install before launching the bridge (see below about sourcing).


## Task 2: Write code to compute metrics

There is a take_home package we have incluced with some stub code. Add to this ROS node some metrics, and create a separate topic for each metric. Download the example bag provided in the Slack.

To get started do the following:

1. Git clone this package.

2. Source ROS2 and build the template package and message dependencies. If you encounter any build issues after running the following, let us know.

```
cd ~/
git clone https://github.com/linklab-uva/cav_take_home.git
cd cav_take_home
source /opt/ros/humble/setup.bash
colcon build
```

3. To run the template node, source the install folder and run the executable.

```
source install/setup.bash
ros2 run take_home take_home_node
```

4. In a separate terminal, source your installation folder, and run the bag file.

```
source install/setup.bash
ros2 bag play -s mcap cavalier_take_home.mcap
```

5. In a separate terminal, run the following.

```
ros2 topic list
ros2 topic echo /metrics_output
```

The template node is subscribing to a message /vehicle/uva_odometry which contains the pose (position and orientation) and twist (linear and angular velocity) of our vehicle. There is a sample calculation that processes the pose and twist of the vehicle and outputs a value to the /metrics_output topic. It is your job to compute the meaningful metrics listed below, and publish them to new topics as we have shown in the example.

### A. Wheel Slip Ratio

Wheel slip ratio is the ratio in the rotational speed of the wheel and the actual linear speed of the car. 

Relevant topics:
- Vehicle Odometry (i.e. Pose and Twist) `/vehicle/uva_odometry`
- Wheel Speed: `/raptor_dbw_interface/wheel_speed_report` (kmph)


The formula for calculating wheel slip ratio varies slightly between each of the four wheels. For the rear right wheel, we have the following formula:

$v_{x,rr} = v_x - 0.5 * \omega * w_r$

$\kappa_{rr} = (v_{rr}^w - v_{x,rr})/v_{x,rr}$

A similar calculation can be used for the rear left wheel:

$v_{x,rl} = v_x + 0.5 * \omega * w_r$

$\kappa_{rl} = (v_{rl}^w - v_{x,rl})/v_{x,rl}$

Here, $v_x$ refers to the car's longitudinal (forwards) linear speed in $m/s$. $\omega$ is the angular velocity (yaw rate) of the vehicle in $rad / s$. $w_r$ refers to the rear track width (the distance between left and right tire) in meters. $v^w$ refers to the wheel speed. $\kappa$ refers to the slip ratio of that wheel. $rr$ refers to the rear right wheel and $rl$ refers to the rear left wheel.

The front two wheels have similar formulas, but they have an extra transformation calculation since their orientation varies with the steering angle. For the front right wheel the full formula is:

$v_{x,fr} = v_x - 0.5 * \omega * w_f$

$v_{x,fr}^\delta = cos(\delta) * v_{x,fr} - sin(\delta) * v_{y,fr}$

$\alpha_{fr} = (v_{fr}^w - v_{x,fr}^\delta)/v_{x,fr}^\delta$

For the front left wheel the formula is:

$v_{x,fl} = v_x + 0.5 * \omega * w_f$

$v_{x,fl}^\delta = cos(\delta) * v_{x,fl} - sin(\delta) * v_{y,fl}$

$\alpha_{fl} = (v_{fl}^w - v_{x,fl}^\delta)/v_{x,fl}^\delta$

Here, the same variables from before are used, with the addition of $\delta$, which refers to steering angle (in radians), and $w_f$, which refers to the front track width (in meters). Here, $fr$ refers to the front right wheel and $fl$ refers to the front left wheel.

Use the following values for the constants:
- **$w_f$** = 1.638
- **$w_r$** = 1.523

### B. Slip Angle

Slip angle is the difference in the direction the wheels are pointing (i.e. steering angle) and the direction the vehicle is actually moving (i.e. linear velocity).

Relevant topics:
- Vehicle Odometry (i.e. Pose and Twist) `/vehicle/uva_odometry`
- Wheel Speed: `/raptor_dbw_interface/wheel_speed_report` (kmph)
- Steering angle: `raptor_dbw_interface/steering_extended_report` (primary_steering_angle_fbk - deg)

Computing the slip angle re-utilizes some of the formulas from the slip ratio calculation. The additional formulas needed for slip angle ($\alpha$) for the rear wheels are below:

$v_{y,r} = v_y - \omega * L_r$

$\alpha_{rl} = tan^{-1}(\frac{v_{y,r}}{v_{x,rl}})$

$\alpha_{rr} = tan^{-1}(\frac{v_{y,r}}{v_{x,rr}})$

Here, $v_y$ refers to the car's lateral (tangential) linear speed, in $m/s$. $l_r$ is to the longitudinal distance from the COG of the car to the rear wheels (in meters). $\alpha$ refers to the slip angle for that corresponding wheel.

The report includes the steering wheel angle and needs to be converted to wheel angle. This wheel angle will be in degrees. Use a steering ratio of $15.0$ to do so:

$ \text{wheel angle} = \text{steering wheel angle} / \text{steering ratio} $

For the front wheels, it is a similar formula, but we have to do the same steering transformations from before:

$v_{y,fr} = v_y + \omega * l_f$

$v_{y,fr}^\delta = sin(\delta) * v_{x,fr} + cos(\delta) * v_{y,fr}$

$v_{y,fl} = v_y + \omega * l_f$

$v_{y,fl}^\delta = sin(\delta) * v_{x,fl} + cos(\delta) * v_{y,fl}$

$\alpha_{fr} = tan^{-1}(v_{y,fr}^\delta / v_{x,fr}^\delta)$

$\alpha_{fl} = tan^{-1}(v_{y,fl}^\delta / v_{x,fl}^\delta)$

Here, $l_f$ is to the longitudinal distance from the COG of the car to the front wheels (in meters). Make sure to mind the units for the input to trigonometric methods. 

Use the following values for the constants:
- **$l_f$** = 1.7238
- **$l_r$** = 1.248

### C. Moving average of lateral error

Part of our racing stack is the planner which creates a reference line for the vehicle to follow (the optimal raceline precomputed if there are no other cars). Our lateral controller does some math to figure out the optimal steering angle / steering rate so that the vehicle stays on this raceline. Of course, no controller is perfect, so we report the lateral error we observe at every timestep (i.e. the distance off the raceline). 

To smooth out this signal, we want to compute a moving average of the reported lateral error. For this metric, compute the moving average over the published lateral errors in a `200ms` window. 

Relevant topics:
- Lateral Error: `lateral_control/lateral_error`

### D. Jitter in IMU data

Our car runs with two GNSS sensors from novatel which both report IMU measurements. Ideally, we would like these sensors to publish at a consistent ∆t (ie a fixed rate). Jitter is defiend as the variance of the ∆t between consecutive measurements. Compute the jitter for the top IMU and bottom IMU independently and report them to two different topics. Compute this metric using a sliding window and consider the last `1s` of data in this sliding window. 

Relevant toipcs: 
- Top IMU: `/novatel_top/rawimu`
- Bottom IMU: `/novatel_bottom/rawimu`

### E. Lap time

How long does it take in seconds for the vehicle to complete 1 lap.

Relevant topics:
- Vehicle Odometry (i.e. Pose and Twist) `/vehicle/uva_odometry`
- Curvilinear Distance (Forward progress along the track in meters) `curvilinear_distance`

## Task 3: Run your code and visualize the results

1. Please refer to the pinned message in the shared slack to get access to a ROS2 bag with data from our time trial run. 

2. Foxglove supports making some really useful panels with plots, 3D visualization position / sensor data, and more. We have attached a panel to start with (`panel.json`) which just creates an empty plot and a 3D visualization that shows the LiDAR scans. An important part of working on the stack invovles adding to foxglove panels to visualize and plot your data, so adjust the given panel to include all of the metrics A-E above. 

3. Screen record the foxglove app and play the entire bag, so we can see the metrics you computed. 

## Task 4: Make a PR for your fork back into the main repository

Please upload your screen recording and share the link to it as a part of the PR. 

**To get credit for completing any of the metrics in task 3, this step is mandatory**

## Getting Started + Tips Tricks

### Clone this repo into your home directory

```{bash}
git clone git@github.com:linklab-uva/cav_take_home.git
```

### Building

In ROS 2 we use `colcon` for building all of our packages. Always build your packages from `~/cav_take_home`

If there is **any** questions or ambiguities please send a message in `#all-cavalier-take-home-slack`. We encourage you to ask questions in this Slack channel so everyone can benefit from the answers. Furthermore, join `#office-hours` to stop by and get help in person. 

### Building All Relevant Messages

```{bash}
colcon build --packages-up-to deep_orange_msgs uva_iac_msgs localization_msgs raptor_dbw_msgs
```

### Sourcing
Whenever you run anything, it is critical that you source your workspace. This allows you to work with our custom ROS2 message types and run the packages you've built. 

```{bash}
source ~/cav_take_home/install/setup.sh
```
