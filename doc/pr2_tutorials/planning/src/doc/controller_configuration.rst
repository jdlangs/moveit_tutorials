Controllers Configuration Tutorial
===================================

In this section, we will walk through configuring MoveIt! with the controllers on your robot. We will assume that your robot offers a ``FollowJointTrajectory`` action service for the arms on your robot and (optionally) a ``GripperCommand`` service for your gripper.

YAML Configuration
------------------

The first file to create is a YAML configuration file (call it *controllers.yaml* and place it in the *config* directory of your MoveIt! config directory). This will specify the controller configuration for your robot. Here's an example file for configuring a ``FollowJointTrajectory`` action controller for two different arms (left and right) and a ``GripperCommand`` gripper controller for two grippers ::

 controller_list:
  - name: r_arm_controller
    action_ns: follow_joint_trajectory
    type: FollowJointTrajectory
    default: true
    joints:
      - r_shoulder_pan_joint
      - r_shoulder_lift_joint
      - r_upper_arm_roll_joint
      - r_elbow_flex_joint
      - r_forearm_roll_joint
      - r_wrist_flex_joint
      - r_wrist_roll_joint
  - name: l_arm_controller
    action_ns: follow_joint_trajectory
    type: FollowJointTrajectory
    default: true
    joints:
      - l_shoulder_pan_joint
      - l_shoulder_lift_joint
      - l_upper_arm_roll_joint
      - l_elbow_flex_joint
      - l_forearm_roll_joint
      - l_wrist_flex_joint
      - l_wrist_roll_joint
  - name: gripper_controller
    action_ns: gripper_action
    type: GripperCommand
    default: true
    joints:
      - l_gripper_joint
      - r_gripper_joint

We will walk through the parameters for both types of controllers.

FollowJointTrajectory Controller Interface
++++++++++++++++++++++++++++++++++++++++++

The parameters are:
 * *name*: The name of the controller.  (See debugging information below for important notes).
 * *action_ns*: The action namespace for the controller. (See debugging information below for important notes).
 * *type*: The type of action being used (here FollowJointTrajectory).
 * *default*: The default controller is the primary controller chosen by MoveIt! for communicating with a particular set of joints.
 * *joints*: Names of all the joints that are being addressed by this interface.

GripperCommand Controller Interface
+++++++++++++++++++++++++++++++++++

The parameters are:
 * *name*: The name of the controller.  (See debugging information below for important notes).
 * *action_ns*: The action namespace for the controller. (See debugging information below for important notes).
 * *type*: The type of action being used (here GripperCommand).
 * *default*: The default controller is the primary controller chosen by MoveIt! for communicating with a particular set of joints.
 * *joints*: Names of all the joints that are being addressed by this interface.


Create the Controller launch file
---------------------------------

Now, create the controller launch file (call it *robot_moveit_controller_manager.launch* where *robot* is the name of your robot - the robot name needs to match the name specified when you created your MoveIt! config directory).

Add the following lines to this file ::

 <launch>
  <!-- Set the param that trajectory_execution_manager needs to find the controller plugin -->
  <arg name="moveit_controller_manager" default="moveit_simple_controller_manager/MoveItSimpleControllerManager" />
  <param name="moveit_controller_manager" value="$(arg moveit_controller_manager)"/>
  <!-- load controller_list -->
  <rosparam file="$(find my_robot_name_moveit_config)/config/controllers.yaml"/>
 </launch>

MAKE SURE to replace *my_robot_name_moveit_config* with the correct path for your MoveIt! config directory.

Now, you should be ready to have MoveIt! talk to your robot.

Debugging Information
---------------------

The ``FollowJointTrajectory`` or ``GripperCommand`` interfaces on your robot must be communicating in the namespace: ``\name\action_ns``. In the above example, you should be able to see the following topics (using *rostopic list*) on your robot:

 * /r_arm_controller/follow_joint_trajectory/goal
 * /r_arm_controller/follow_joint_trajectory/feedback
 * /r_arm_controller/follow_joint_trajectory/result
 * /l_arm_controller/follow_joint_trajectory/goal
 * /l_arm_controller/follow_joint_trajectory/feedback
 * /l_arm_controller/follow_joint_trajectory/result
 * /gripper_controller/gripper_action/goal
 * /gripper_controller/gripper_action/feedback
 * /gripper_controller/gripper_action/result

You should also be able to see (using *rostopic info topic_name*) that the topics are published/subscribed to by the controllers on your robot and also by the *move_group* node.

Remapping /joint_states topic
-----------------------------

When you run a `move group node <http://docs.ros.org/kinetic/api/moveit_tutorials/html/doc/pr2_tutorials/planning/src/doc/move_group_interface_tutorial.html>`_, you may need to remap the topic /joint_states to /robot/joint_states, otherwise MoveIt! won't have feedback from the joints. To do this remapping you could make a simple launch file for your node as follows ::

  <node pkg="moveit_ros_move_group" type="move_group" name="any_name" output="screen">
    <remap from="joint_states" to="robot/joint_states"/>
  </node>

Or you can make a subscriber with the correct topic name and then ensure that the starting robot state for your move group corresponds to a correct joints angle by using the call back of this subscriber.   

MoveIt! ROS Control Plugin
--------------------------

This package provides plugins of base class `moveit_controller_manager::MoveItControllerManager` and a new plugin base class for `moveit_controller_manager::MoveItControllerHandle` allocators.
The allocator class is necessary because `moveit_controller_manager::MoveItControllerHandle` needs a name passed to the constructor.
Two variantes are provided, `moveit_ros_control_interface::MoveItControllerManager` for interfacing a singe ros_control node and `moveit_ros_control_interface::MoveItMultiControllerManager` for seamless integration with any number of ros_control nodes.


moveit_ros_control_interface::MoveItControllerManager
+++++++++++++++++++++++++++++++++++++++++++++++++++++

This plugin interfaces a single ros_control-driven node in the namespace given in the `~ros_control_namespace` ROS parameter.
It polls all controllers via the `list_controllers` service and passes their properties to MoveIt!.
The polling is throttled to 1 Hertz.

Handle plugins
""""""""""""""

The actual handle creation is delegated to allocator plugins of base class `moveit_ros_control_interface::ControllerHandleAllocator`.
These plugins should be registered with lookup names that match the corresponding controller types.

Currently plugins for `position_controllers/JointTrajectoryController`, `velocity_controllers/JointTrajectoryController` and `effort_controllers/JointTrajectoryController` are available, which simply wrap `moveit_simple_controller_manager::FollowJointTrajectoryControllerHandle` instances.

Setup
"""""

In your MoveIt! launch file (e.g. `ROBOT_moveit_config/launch/ROBOT_moveit_controller_manager.launch.xml`) set the `moveit_controller_manager` parameter ::

  <arg name="moveit_controller_manager" default="moveit_ros_control_interface::MoveItControllerManager" />

And make sure to set the `ros_control_namespace` parameter to the namespace (without the /controller_manager/ part) of the ros_control-based node you like to interface.
If you are using the `moveit_setup_assistent` you can add it to `ROBOT_moveit_config/config/controllers.yaml`, e.g. ::

  ros_control_namespace: /ROS_CONTROL_NODE
  controller_list:
    - name: /ROS_CONTROL_NODE/position_trajectory_controller
      action_ns: follow_joint_trajectory
      type: FollowJointTrajectory
      default: true
      joints:
        - joint_a1
        - joint_a2
        - joint_a3
        - joint_a4
        - joint_a5
        - joint_a6
        - joint_a7

Controller switching
""""""""""""""""""""

MoveIt! can decide which controllers have to be started and stopped.
Since only controller names with registered allocator plugins are handed over to MoveIt!, this implementation takes care of stopping other conflicting controllers based on their claimed resources and the resources for the to-be-started controllers.

Namespaces
""""""""""

All controller names get prefixed by the namespace of the ros_control node.
For this to work the controller names should not contain slashes. This is a strict requirement if the ros_control  namespace is `/`.

moveit_ros_control_interface::MoveItMultiControllerManager
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

This plugin does not need further configuration. It polls the ROS master for services and identifies ros_control nodes automatically.
It spawns `moveit_ros_control_interface::MoveItControllerManager` instances with their namespace and takes cares of proper delegation.

Setup
"""""

Just set the `moveit_controller_manager` parameter in your MoveIt! launch file (e.g. `ROBOT_moveit_config/launch/ROBOT_moveit_controller_manager.launch.xml`) ::

  <arg name="moveit_controller_manager" default="moveit_ros_control_interface::MoveItMultiControllerManager" />
