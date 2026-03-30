# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

unitree_sim_isaaclab is a simulation platform for **Unitree robots** (G1/H1-2) built on **NVIDIA Isaac Sim + Isaac Lab**. It supports teleoperation, data collection/replay/generation, and model validation. Communication with the simulated robot uses the same **DDS protocol** (CycloneDDS via unitree_sdk2_python) as physical Unitree robots.

## Environment Setup

### Automated setup (recommended)
```bash
bash auto_setup_env.sh <4.5|5.0|5.1> <conda_env_name> [cuda_version]
# Example: bash auto_setup_env.sh 5.1 unitree_sim_env cu126
```
This creates a conda environment, installs Isaac Sim, Isaac Lab, CycloneDDS, unitree_sdk2_python, teleimager (git submodule), and downloads assets.

### Asset download (required separately if not using auto_setup_env.sh)
```bash
sudo apt install git-lfs
. fetch_assets.sh
```
Assets are fetched from HuggingFace (`unitree_sim_isaaclab_usds`) and extracted to `./assets/`.

### Docker (Ubuntu 22.04 / Isaac Sim 5.1)
```bash
sudo docker build -t unitree-sim:latest -f Dockerfile .
```

## Running the Simulator

### Teleoperation
```bash
python sim_main.py --device cpu --enable_cameras --task <TASK_NAME> --enable_dex1_dds --robot_type g129
```

### Data Replay
```bash
python sim_main.py --device cpu --enable_cameras --task <TASK_NAME> --enable_dex1_dds --robot_type g129 --replay_data --file_path <DATASET_DIR>
```

### Data Generation (augmentation via modified lighting/cameras)
```bash
python sim_main.py --device cpu --enable_cameras --task <TASK_NAME> --enable_dex1_dds --robot_type g129 --replay_data --file_path <DATASET_DIR> --generate_data --generate_data_dir ./data2
```

### Key CLI flags
- `--task`: Task name (e.g., `Isaac-PickPlace-Cylinder-G129-Dex1-Joint`)
- `--robot_type`: `g129` (G1 29-DOF) or `h1_2` (H1-2 27-DOF)
- `--enable_dex1_dds` / `--enable_dex3_dds` / `--enable_inspire_dds`: Enable DDS for 2-finger gripper / 3-finger dex hand / Inspire hand (mutually exclusive)
- `--no_render`: Headless mode with WebRTC streaming
- `--step_hz`: Control loop frequency (default 100)
- Tasks with `Wholebody` in the name enable mobile locomotion

### Movement control
Use `send_commands_8bit.py` or `send_commands_keyboard.py` to publish movement commands (only for `Wholebody` tasks).

## Architecture

### Entry point
`sim_main.py` orchestrates the full pipeline: parses args, launches Isaac Sim via `AppLauncher`, creates the gym environment, sets up DDS communication, creates an action provider, and runs the control loop.

### Core modules

- **`action_provider/`** - Action source abstraction layer. `ActionProvider` base class with three implementations:
  - `DDSActionProvider` - receives joint commands via DDS (teleoperation)
  - `DDSRLActionProvider` - receives commands via DDS for wholebody/RL tasks
  - `FileActionProviderReplay` - replays recorded datasets
  - Factory: `create_action_provider()` selects provider based on `--action_source`

- **`dds/`** - DDS communication layer (CycloneDDS). Each DDS entity (G1 robot, gripper, dex3 hand, inspire hand, run commands, reset pose, sim state, rewards) has its own class. `dds_master.py` manages registration, publishing, and subscribing. DDS operates on **channel 1**.

- **`layeredcontrol/`** - `RobotController` runs the main control loop at a configurable frequency. Gets actions from the action provider, steps the environment, and handles timing.

- **`tasks/`** - Isaac Lab task definitions using gymnasium registration pattern:
  - `common_config/` - shared camera and robot configurations
  - `common_observations/` - observation functions (robot state, gripper state, camera images)
  - `common_scene/` - base scene configs for different object types (cylinder, red block)
  - `common_termination/` - termination/reset conditions per task type
  - `common_rewards/` - reward computation
  - `common_event/` - event manager registration
  - `g1_tasks/` - G1 robot task variants (each has `__init__.py` with `gym.register()`, env config, MDP observations, terminations)
  - `h1-2_tasks/` - H1-2 robot task variants

- **`robots/`** - Robot USD/URDF configuration (`unitree.py` contains articulation configs)

- **`teleimager/`** - Git submodule (`teleimager` package). Multi-camera image streaming service using ZMQ. Installed as editable package (`pip install -e .`).

- **`tools/`** - Utilities: USD conversion, data augmentation (lighting/camera), data loading, episode writing, reward computation, shared memory, rerun visualization.

### Adding a new task
1. Create a common scene in `tasks/common_scene/` (if needed)
2. Add termination conditions in `tasks/common_termination/` (if needed)
3. Create a new task directory under `tasks/g1_tasks/` (or `tasks/h1-2_tasks/`) with:
   - `mdp/observations.py` - import observation functions from `common_observations`
   - `mdp/terminations.py` - import termination functions from `common_termination`
   - `__init__.py` - register task with `gym.register(id="Isaac-...", ...)`
   - `*_env_cfg.py` - environment config importing common scene + robot/camera configs
4. Add the import to the parent `__init__.py` (e.g., `tasks/g1_tasks/__init__.py`)

### Task naming convention
`Isaac-{Action}-{Object}-{Robot}-{Hand}-{Mode}`
- Action: `PickPlace`, `Stack`, `Move`
- Object: `Cylinder`, `RedBlock`, `RgyBlock`
- Robot: `G129` (G1 29-DOF), `H12-27dof`
- Hand: `Dex1` (gripper), `Dex3` (3-finger), `Inspire`
- Mode: `Joint` (fixed base) or `Wholebody` (mobile)

## Dependencies
- Python 3.10 (Isaac Sim 4.5) or 3.11 (Isaac Sim 5.x)
- Isaac Sim 4.5.0 / 5.0.0 / 5.1.0
- Isaac Lab (installed via `isaaclab.sh --install`)
- CycloneDDS 0.10.x (built from source)
- unitree_sdk2_python
- PyTorch with CUDA
- See `requirements.txt` for additional Python deps (rerun-sdk, pyzmq, pynput, onnxruntime, aiortc, aiohttp)
