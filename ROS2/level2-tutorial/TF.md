
# 1. TF (TransForm)
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
## 2.2 RViz
> Visualize the Robot on RViz

