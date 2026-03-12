# URDF construction and visualization

```bash
mkdir ros2_ws

cd ros2_ws/

mkdir src

gedit ~/.bashrc
```
type in:
source <Path to>/ros2_ws/install/setup.bash

```bash
cd src/

ros2 pkg create kxsmoveit_robot_description
```

```bash
cd kxsmoveit_robot_description/

rm -r include/ src/

mkdir urdf launch rviz

cd ..
```
in CMakeLists.txt
```
install(
  DIRECTORY launch rviz urdf
  DESTINATION share/${PROJECT_NAME}
)
```

under src
```bash
code .

colcon build --packages-select kxsmoveit_robot_description
```
```bash
cd ros2_ws/src/
cd kxsmoveit_robot_description/urdf

touch arm.urdf
```
edit arm info accordingly in ros2_ws/src/kxsmoveit_robot_description/urdf/arm.urdf

**Meaning of link `origin`:The coordinates of the geometric center of this shape (relative to the previous joint, the first one is relative to the origin)**

**Meaning of joint `origin`: The position relationship between this joint and the previous joint in the initial state (the first one is related to Fixed coordinate system origin)**

---
### setting in urdf

every origin is suggesed to set to <origin xyz="0 0 0" rpy="0 0 0"/>
and adjust later to avoid mess

Why you should still use unit vectors (sum=1)for <axis xyz="1 0 0"/>:

  Clarity: It's immediately obvious to human readers what the direction is.
  Compatibility: While standard ROS tools handle it, some custom parsers or 3rd party visualizers might behave unexpectedly if the vector has a length other than 1.

---

for quick visualization of urdf 3D model:
```bash
sudo apt install ros-humble-urdf-tutorial 

```
For preview visualize in rviz
```bash

ros2 launch urdf_tutorial display.launch.py model:= <Absolute_Path_to_>/ros2_ws/src/kxsmoveit_robot_description/urdf/arm.urdf 

```
https://wiki.ros.org/urdf/XML/link
Reference of link element

After finish constructing urdf of robotic arm
```bash
ros2 run tf2_tools view_frames
```
to print TF tree 

Result: [text](../frames_2026-01-21_22.06.37.pdf)

---

### Improve urdf with Xacro

in ros2_ws/src/kxsmoveit_robot_description/urdf,  
create ```common_properties.xacro``` and ```kxsmoveit_robot.urdf.xacro``` as shown

change file name ```arm.urdf``` to `arm.xacro` 

this makes the project more scalable.

(I kept my older version of `arm.urdf`)


To test whether xacro has built up successfuly:
```bash
ros2 launch urdf_tutorial display.launch.py model:= <Absolute_Path_to_>/ros2_ws/src/kxsmoveit_robot_description/urdf/kxsmoveit_robot.urdf.xacro

```
If rviz shows up and correctly shows the model then its correct.

---
### Create a Launch file

As shown in here
[text](src/kxsmoveit_robot_description/launch/display.launch.xml)

then you launch 

```bash
ros2 launch kxsmoveit_robot_description display.launch.xml 
```
![alt text](<Screenshot from 2026-01-21 23-51-12.png>)
at the first time, rviz is not configured

so we can add-> RobotModel->OK
and Change "description topic" from empty to "/robot_description"

also add TF 

after you're satisfied with your panel, file->save configuation as->

and save in your `/ros2_ws/src/kxsmoveit_robot_description/rviz`
configure launch file `ros2_ws/src/kxsmoveit_robot_description/launch/display.launch.xml`again, adding
```xml
<!-- ... -->
    <let name="rviz_config_path" 
        value="$(find-pkg-share kxsmoveit_robot_description)/rviz/urdf_config.rviz"/>
<!-- ... -->
    <node pkg="rviz2" exec="rviz2" output="screen" args="-d $(var rviz_config_path)"/>   
<!--  -->
```
then launch again
```bash
colcon build
source install/setup.bash
ros2 launch kxsmoveit_robot_description display.launch.xml
```
No need to change configuration manually anymore

# Path & Motion Planning

### Add collision to every link of robotarm

```xml
      <collision>
            <geometry>
                <box size="0.4 0.4 0.1"/>
            </geometry>
            <origin xyz="0 0 0.05" rpy="0 0 0"/>
      </collision>
```
then 
```bash
cd ros2_ws
colcon build
source install/setup.bash 
ros2 launch kxsmoveit_robot_description display.launch.xml 
```
### Run Moveit setup Assistant
```bash
ros2 launch moveit_setup_assistant setup_assistant.launch.py 
```
in the window, Create New... -> Browse

-> Choose your file -> Load files

Left side: self collision -> Generate collision matrix ok

visual joints -> add -> ![alt text](<Screenshot from 2026-02-02 23-52-51.png>)

Planning groups - add - Kinematics solver:"kdl..." -Add joints(select all and click `>` -> `save`)

Define robot poses -> (diy)

End effectors -> 

Passive joints(no in this one)

ros2 control urdf Mod: already given(only choose `position`in command and state inerfaces)

Ros2 Controllers -> Auto add...

Moveit controllers -> 

Perception ->

Launch files : keep select all

Configuration files -> browse -> Save path: `src/kxsmoveit_robot_moveit_config` (a new folder should be created to avoid any errors) **then a new folder `ros2_ws/src/kxsmoveit_robot_moveit_config`with, most importantly, a srdf file is generated**

then you can see a folder is created


```bash
cd ros2_ws
colcon build
source install/setup.bash 
ros2 launch kxsmoveit_robot_moveit_config demo.launch.py
```

if you see warnings like `[move_group-3] what(): parameter 'robot_description_planning.joint_limits.joint1.max_velocity' has invalid type: expected [double] got [integer] `

change every integer parameter in `ros2_ws/src/kxsmoveit_robot_moveit_config/config/joint_limits.yaml`
to double type and build again.

In rviz, you can plan and execute a path from ant to some state

A linear path can be generated by select `Use Cartesian Path` only when it doesn't encounter singularity. More links means higher chance of success in generating linear path.

### Create a Gripper

Add a new file in `ros2_ws/src/kxsmoveit_robot_description` called gripper.xacro

edit accordingly, similar process to arm urdf.

to preview whether the configuration is working well:
```bash
cd 
ros2 launch urdf_tutorial display.launch.py model:=/home/< Path to >/gripper.xacro

```

The fixed joint belongs in kxsmoveit_robot.urdf.xacro
Add this joint in`ros2_ws/src/kxsmoveit_robot_description/urdf/kxsmoveit_robot.urdf.xacro`:
```xml
<joint name="gripper_base_joint" type="fixed">
    <parent link="tool_link"/>
    <child link="gripper_base_link"/>
    <origin xyz="0 0 0.0" rpy="0 0 0"/>
</joint>
```
connects the arm to the gripper.

The structure is correct because `gripper.xacro` is used as a subcomponent of the full robot, not as a standalone robot.

**Intended role of files:**
```
common_properties.xacro → shared materials/properties
arm.xacro → arm links/joints
gripper.xacro → gripper links/joints
kxsmoveit_robot.urdf.xacro → assembles everything into one robot
```

use 
```bash
ros2 run tf2_tools view_frames 
```
to see the structure clearly: [Stucture of files](../frames_2026-03-12_12.44.14.pdf)

