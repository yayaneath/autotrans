# AutOTranS: an Autonomous Open World Transportation System
## Brayan S. Zapata-Impata<sup>1</sup>, Vikrant Shah<sup>2</sup>, Hanumant Singh<sup>2</sup> and Robert Platt<sup>3</sup>

<sup>1</sup> Dept. of Physics, System Engineering and Signal Theory, University of Alicante, Alicante, Spain. Email: brayan (dot) impata (at) ua (dot) es

<sup>2</sup> Dept. of Electrical and Computer Engineering, Northeastern University, Boston, Massachusetts, USA. Email: shah (dot) vi (at) husky (dot) neu (dot) edu, ha (dot) singh (at) northeastern (dot) edu

<sup>3</sup> College of Computer and Information Science, Northeastern University, Boston, Massachusetts, USA. Email: rplatt (at) ccs (dot) neu (dot) edu

###### Reference
Zapata-Impata, B.S., Shah, V., Singh, H., Platt, R. *AutOTranS: an Autonomous Open World Transportation System*. arXiv https://arxiv.org/abs/1810.03400

###### AutOTranS in the media
- IEEE Spectrum Video Friday (October 12th, 2018): https://spectrum.ieee.org/automaton/robotics/robotics-hardware/video-friday-boston-dynamics-spot-goes-to-work-and-more
- ClearPath Robotics Blog (November 28th, 2018): https://www.clearpathrobotics.com/blog/2018/11/warthog-ugv-mobile-manipulation-research-autotrans/


### System Video
[![Video of the system working outdoors](https://github.com/yayaneath/autotrans/blob/master/robot.png)](https://youtu.be/93nWXhaGEWA)

### Motivation

In many cities, trash bags often accumulate through the week, waiting to be picked up. Farm workers must often pick up and carry heavy tools daily. Construction workers spend a lot of time transporting materials through the construction site. These are all labor intensive tasks that could benefit from mobile robotic manipulation. **We focus on the problem of picking and dropping novel objects in an open world environment**. The only input to our system are the the pick and drop points selected by an operator using a map of a previously explored area. Once these points are identified, the robot navigates to the pick location, picks up whatever is found there, transports it and drops it to a bin at the drop location. Our main contributions are as follows:

1. We describe a method for navigating to the pick and drop points autonomously. We propose two transport strategies to solve this task: *collect all* and *collect one by one*.
2. We describe a method for selecting grasps in order to carry out the picking that does not make any assumptions about the objects.
3. We describe the outdoor mobile manipulator used in detail.
4. We experimentally characterize the navigation and grasping systems and report success rates and times over four transport tasks on different objects (trash bags, general garbage, gardening tools and fruits).

### Robot Hardware

The mobile base is the **Clearpath Warthog**, which offers a payload of 276kg and measures 1.52 x 1.38 x 0.83 (m). Since the default setup gave us steering issues, we disengaged the rear wheels and put them on casters, converting it to a differential drive system. The manipulator is the **Universal Robots UR10**, which has 6 Degrees of Freedom (DoF) and a payload of 10kg. The end effector is a **Robotiq 2-Finger 85** gripper, with a payload of 5kg and a maximum aperture of 85mm.
	
For perception, we use three **Intel RealSense D415** depth cameras. Two of them are fixed to the two sides in front of the robot pointing downwards to cover the target picking area. The third one is mounted on the gripper configured as a hand-eye camera. The primary sensor used for vehicle localization is a front mounted single line **SICK LMS511** lidar which has a field of view of 190 degrees.

### System Architecture

![alt text](https://github.com/yayaneath/autotrans/blob/master/proj-nodes.png "System architecture")

Our system is developed using the Robot Operating System (ROS). The three main parts are: the **navigation stack**, the **grasping stack** and the **task planner**. Given their computational requirements, we split the system onto two computers: 1) the on-board PC in the Warthog, which runs all of the navigation stack and 2) the PC mounted on top, which runs the grasping nodes and the task planner.

###### Navigation Stack
The main goal of the navigation is to deliver the robot base to a position such that the target objects are within the manipulator workspace. We use the existing ROS navigation stack, which uses GMapping SLAM for creating maps, AMCL for localization in existing maps and the move\_base stack for route planning and control of the robot.

###### Grasping Stack
We calculate grasps using the Grasp Pose Detection package ([GPD](https://github.com/atenpas/gpd)). This method calculates grasps poses given a 3D point cloud without having to segment the objects: the method samples grasps hypotheses over the point cloud (500 seeds in our case) and ranks their potential success using a custom grasp descriptor. Then, it returns the top K grasps found (50 in our setup). Before selecting the best grasp, we prune kinematically infeasible grasps by checking inverse kinematics (IK) solutions against the environment constraints. Thus, using the IK solver, we discard grasp poses that are in collision with obstacles or are otherwise unreachable.

###### Task Planner
The task planner node is in charge of sending goals to the Warthog and requests to the grasping service in order to provide the mobile manipulation functionality. It requires three inputs: the type of task, the pick position, and the drop position. Two tasks are considered:

- **Collect all:** the robot must collect everything from the pick point before moving to the drop point.
- **Collect one by one:** the robot moves between the pick and drop points transporting only one object at a time.

The type of task is passed as an argument to the task node on launch. For the pick and drop points, the RViz window from the navigation side is used as the human interface. By clicking on a position in the map using the *2D Nav Goal* functionality the user sets goals for the task. The first set goal is the pick point and the second one is the drop point. Afterwards, the autonomous task can start:

1. **Moving to pick point:** the task planner sends the pick position to the Warthog, waiting for this goal to be accomplished. After reaching the pick point, the Warthog adjust its final position.
2. **Collecting:** a request is sent to the grasping service specifying that it has to perform grasps on the floor and drops in the basket. If this is a *collect all* task, this request is sent until no more grasps are found in the floor, meaning that there are no more objects left.
3. **Moving to drop point:** the task planner sends the drop position as the new goal to the Warthog, waiting for this goal to be accomplished. Then, the Warthog adjust its final position but this time with respect to the bin.
4. **Dropping:** a request is sent to the grasping service indicating that this time it has to perform grasps in the basket and drops in the bin. Again, if its a *collect all* task this is done until no more grasps are found.
	
For *collect all* tasks, only a single pass through these steps is needed. For a *collect one by one* task, these steps are repeated until no more objects are detected at the pick point.

### Experiments

We performed experiments to evaluate the variety of objects the system can handle and the success rates and times of various parts of the process. We performed the experiments on city streets in the vicinity of a loading dock. On each trial, we dropped a set of objects at a random location, placed the bin at a different random location and started the robot from a third random location. The set of objects used in these experiments were selected to be graspable by our gripper:

- **Trash bags:** 3 black trash bags made of plastic, which are deformable so their shape changed from test to test.	
- **General garbage:** 15 objects that could be found lying in the street like plastic bottles, cans and paper cups.
- **Gardening tools:** 4 gardening tools made of steel with wood handles, except for one with rubber handle.
- **Fruits:** 3 green apples and 3 oranges.

![alt text](https://github.com/yayaneath/autotrans/blob/master/test-objs.png "Objects used in experimentation")


