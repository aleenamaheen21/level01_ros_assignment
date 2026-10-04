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
