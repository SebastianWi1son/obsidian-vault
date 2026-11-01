
# 1. TF - TransForm
## 1.1 Purpose
> Keep a structured tree for all the joints/frames over time

> We want to know:
> - How frames are placed and moved relative to one another
> - We should describe the pose and location
## 1.2 TF
An TF description
```
translation:
	x: 0.2
	y: 0.0
	z: 0.3
pose:
	x: 0.0
	y: 0.0
	z: 0.0
	w: 1.0
```
why using quaternion to describe pose?
- Euler Angle
- Rotation Matrix
- Quaternion

# 2. URDF - Unified Robot Description Format
## 2.1 URDF file
>  Description of all the elements in a robot
>  **Used to generate TFs**
>  XML format
```xml

```
## 2.2 RViz
> Visualize the Robot on RViz
## 2.3 Link
> 一个刚性部件，内部相对位置固定，不会彼此运动
```xml
<link name="base_link">
    <visual>
        <geometry>
            <box size="0.6 0.4 0.2" />
        </geometry>
        <origin xyz="0 0 0.1" rpy="0 0 0" />
        <material name="green" />
    </visual>
</link>
```
- 定义一个名为 `base_link` 的刚性部件
- 用绿色长方体显示它
![425](assets/TF/file-20261101195438990.png)

## 2.4 Policy
- link的origin在底部
- link的visual在中心