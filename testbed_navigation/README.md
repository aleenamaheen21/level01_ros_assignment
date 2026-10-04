# testbed_navigation

## Overview

`testbed_navigation` brings up the Nav2 components one by one, instead of calling `nav2_bringup`, so that the Testbed-T1.0.0 robot can drive to a goal on the provided map. The work is split into three launch files:

- `map_loader.launch.py` starts `map_server`, which loads the saved map (an image of the walls plus a YAML file with its size and resolution) and publishes it for the other nodes, and `lifecycle_manager_map`.
- `localization.launch.py` starts `amcl`, which estimates where the robot is on that map by comparing laser scans to the walls, and `lifecycle_manager_localization`.
- `navigation.launch.py` starts `planner_server` (plans a path to the goal), `controller_server` (drives along that path), `behavior_server` (recovery behaviors such as spin, backup and wait), `bt_navigator` (coordinates them with a behavior tree), and `lifecycle_manager_navigation`.

Each launch file has its own lifecycle manager, which activates that group's nodes in order. Because of this, each part can be started, tested and restarted separately from the others. Navigation still needs the map and localization running in order to work.

## Package layout

- `config/amcl_params.yaml`: parameters for AMCL
- `config/map_server_params.yaml`: parameters for `map_server` and its lifecycle manager (the map image path is added by the launch file, since it depends on where the package is installed)
- `config/nav2_params.yaml`: parameters for the planner, controller, behavior server, `bt_navigator` and the costmaps
- `launch/`: the three launch files
- `rviz/navigation.rviz`: RViz configuration with the map, costmap, scan, plan and footprint displays
- `CMakeLists.txt` and `package.xml`: build and dependency description; `CMakeLists.txt` installs `config`, `launch` and `rviz`

## Build and run

1. Build the workspace and source it:

```bash
cd ~/assignment_ws
colcon build
source install/setup.bash
```

2. In every terminal, use Cyclone DDS:

```bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

3. Start the parts in this order, one launch file per terminal (each terminal needs steps 1 and 2 first):

```bash
ros2 launch testbed_bringup testbed_full_bringup.launch.py
ros2 launch testbed_navigation map_loader.launch.py
ros2 launch testbed_navigation localization.launch.py
ros2 launch testbed_navigation navigation.launch.py
```

4. The simulation launch opens Gazebo and an RViz window with its own configuration. In RViz, use File > Open Config and load `rviz/navigation.rviz` (the installed copy is under `install/testbed_navigation/share/testbed_navigation/rviz/`). Then send a goal with **2D Goal Pose**.

Start localization soon after the simulator. In one run the laser scan was misaligned with the walls after the simulator had been running about 9 minutes before AMCL started; after restarting everything within about a minute, the scan lined up. A likely cause is that AMCL starts from the fixed initial pose in `amcl_params.yaml` while the robot had drifted, but this was not verified.

## Design decisions

1. **Footprint:** a four-point polygon in `base_link`, measured from the STL meshes of the chassis and wheels, with about 1 cm margin. The origin is not at the body centre and the wheels stick out past the chassis, so a circular radius would not describe the robot.
2. **Local costmap:** `odom` frame, 3 x 3 m rolling window, obstacle and inflation layers. It only needs the robot's surroundings. The plain obstacle layer is used instead of the voxel layer because the lidar is 2D.
3. **Global costmap:** `map` frame, with static, obstacle and inflation layers.
4. **Global planner:** `NavfnPlanner` (Dijkstra), goal tolerance 0.5 m, `allow_unknown: true`.
5. **Local controller:** Regulated Pure Pursuit at 0.3 m/s with a 0.5 m lookahead. Its label is `FollowPath`, because the behavior tree refers to it by that name. The speeds are chosen values, since the robot's drive plugin does not impose limits.
6. **Goal and progress checking:** the goal is reached within 0.25 m and 0.25 rad. The robot counts as stuck if it moves less than 0.5 m in 10 s.
7. **Behavior server and tree:** spin, backup and wait behaviors, with the default behavior tree shipped with Nav2 (no custom tree file is configured).
8. **AMCL:** `scan_topic: scan` matches the robot's lidar topic, 500 to 2000 particles, and a fixed initial pose set in `amcl_params.yaml`.
9. **Optional plugins:** `velocity_smoother` and `collision_monitor` are not used, since only basic navigation is required.

## Verifying each part

Each part can be checked on its own before starting the next:

1. **Map loading:** start the simulator and `map_loader.launch.py`. `ros2 lifecycle get /map_server` should print `active [3]`, and the map appears in RViz (Durability set to Transient Local on the map display).
2. **Localization:** start `localization.launch.py`. `lifecycle_manager_localization` reports that the managed nodes are active, and the laser scan lines up with the walls in RViz. AMCL publishes the `map` to `odom` transform.
3. **Navigation:** start `navigation.launch.py`, then send a goal with **2D Goal Pose**. The log shows that the goal was reached, and the Navigation 2 panel shows navigation and localization active.

In the final test two goals were sent this way. Both were reached without any recovery behavior.

## Challenges and observations

- **Startup timing:** see the note under Build and run. Scan misalignment appeared when localization was started long after the simulator, and disappeared after a quick full restart. The cause is not verified.
- **`Frame [map] does not exist` in RViz:** with only the simulator and the map server running, RViz reports this and the robot model turns red. The map server publishes no transforms. The `map` to `odom` transform comes from AMCL, so the message disappears once localization runs.
- **Costmap display showing "No map received":** this appeared once, although the topic and durability settings were correct. A full restart fixed it. The cause is unknown.
- **Misspelled parameter names are ignored silently:** a wrong key in a YAML file produces no error. Parameter names were checked against the Nav2 libraries with `strings`.
- **Plugin names use two styles:** some use `/` (for example `nav2_behaviors/Spin`) and others use `::` (for example `nav2_controller::SimpleProgressChecker`).
- **Installed files:** `colcon` only copies the folders listed in `install(DIRECTORY ...)`, so the new `rviz` folder had to be added there.
- **Costmap sensor ranges** are left at the reference defaults (2.5 m obstacle, 3.0 m raytrace), although the lidar reaches 10 m.

## Bugs in the starter code

The bugs found and fixed in the provided packages are listed in `BUGS.txt` in the repository root.

## Evidence

- `docs/localization.png`: the robot localized on the map
- `docs/costmaps_overview.png` and `docs/costmaps_closeup.png`: global and local costmaps
- `docs/navigation_goal_reached.png`: a goal reached
- [`docs/navigation_demo.webm`](../docs/navigation_demo.webm): a video of navigation (GitHub does not play it inline, so open or download the file)
