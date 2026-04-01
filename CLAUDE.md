# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

unitree_sim_isaaclab is a simulation platform for **Unitree robots** (G1/H1-2) built on **NVIDIA Isaac Sim + Isaac Lab**. It supports teleoperation, data collection/replay/generation, and model validation. Communication with the simulated robot uses the same **DDS protocol** (CycloneDDS via unitree_sdk2_python) as physical Unitree robots — all DDS communication operates on **channel 1** (`ChannelFactoryInitialize(1)`), and external DDS clients must use the same channel.

## Environment Setup

```bash
# Automated setup (recommended) — creates conda env, installs all deps, downloads assets
bash auto_setup_env.sh <4.5|5.0|5.1> <conda_env_name> [cuda_version]
# Example: bash auto_setup_env.sh 5.1 unitree_sim_env cu126

# Asset download only (if not using auto_setup_env.sh)
sudo apt install git-lfs && . fetch_assets.sh
# Assets fetched from HuggingFace (unitree_sim_isaaclab_usds) → ./assets/

# Docker (Ubuntu 22.04 / Isaac Sim 5.1)
sudo docker build -t unitree-sim:latest -f Dockerfile .
```

## Running the Simulator

```bash
# Teleoperation
python sim_main.py --device cpu --enable_cameras --task <TASK_NAME> --enable_dex1_dds --robot_type g129

# Data replay
python sim_main.py --device cpu --enable_cameras --task <TASK_NAME> --enable_dex1_dds --robot_type g129 --replay_data --file_path <DATASET_DIR>

# Data generation (augmented lighting/cameras during replay)
python sim_main.py --device cpu --enable_cameras --task <TASK_NAME> --enable_dex1_dds --robot_type g129 --replay_data --file_path <DATASET_DIR> --generate_data --generate_data_dir ./data2
```

### Key CLI flags
- `--task`: Task name (e.g., `Isaac-PickPlace-Cylinder-G129-Dex1-Joint`)
- `--robot_type`: `g129` (G1 29-DOF) or `h1_2` (H1-2 27-DOF)
- `--enable_dex1_dds` / `--enable_dex3_dds` / `--enable_inspire_dds`: Enable DDS for 2-finger gripper / 3-finger dex hand / Inspire hand (mutually exclusive)
- `--no_render`: Headless mode with WebRTC streaming
- `--step_hz`: Control loop frequency (default 100)
- `--action_source`: `dds` (default), `replay`, or `dds_wholebody` (auto-set for Wholebody tasks)
- `--modify_light` / `--modify_camera`: Enable light/camera augmentation during data generation
- `--rerun_log`: Enable rerun visualization logging during data generation
- Tasks with `Wholebody` in the name automatically enable mobile locomotion and switch to `dds_wholebody` action source

### Movement control
Use `send_commands_8bit.py` or `send_commands_keyboard.py` to publish movement commands (only for `Wholebody` tasks).

## Architecture

### Startup sequence (`sim_main.py`)
1. Sets `PROJECT_ROOT` env var (used by robot USD configs to locate asset paths)
2. Parses args → imports `pinocchio` (must happen before AppLauncher) → launches Isaac Sim via `AppLauncher`
3. Creates gym environment from task config via `gym.make()`
4. Sets up DDS objects via `dds/dds_create.py` (different sets for live vs replay mode)
5. Creates action provider via factory (`action_provider/create_action_provider.py`)
6. Creates `RobotController` with the action provider and runs the main control loop

### Control loop data flow
```
DDS Subscribe → ActionProvider.get_action() → RobotController.step() → env.step(action)
                                                                      ↓
                                                          env state → DDS Publish (sim_state, rewards)
                                                                      ↓
                                                          camera frames → teleimager ZMQ publish
```

### Core modules

- **`action_provider/`** — Action source abstraction. `ActionProvider` (ABC in `action_base.py`) runs a background thread, subclasses implement `get_action(env) → Tensor`. Three implementations:
  - `DDSActionProvider` — receives joint commands via DDS (teleoperation)
  - `DDSRLActionProvider` — receives commands via DDS for wholebody/RL tasks
  - `FileActionProviderReplay` — replays recorded datasets from JSON files
  - Factory: `create_action_provider()` selects based on `--action_source`

- **`dds/`** — DDS communication layer (CycloneDDS). `DDSObject` (ABC in `dds_base.py`) defines the publish/subscribe interface with optional shared memory. `DDSManager` (`dds_master.py`) is a **thread-safe singleton** that manages registration, publish loop (background thread at 100Hz default), and subscribing. Each entity (g1_robot, gripper, dex3, inspire, run_command, reset_pose, sim_state, rewards) has its own `DDSObject` subclass.

- **`layeredcontrol/`** — `RobotController` runs the main control loop at configurable frequency. Each `step()`: gets action from provider → `env.step(action)` → frequency-limited sleep. In replay/wholebody mode, `env.step()` is skipped (action provider handles env stepping directly).

- **`tasks/`** — Isaac Lab task definitions using gymnasium registration pattern. `tasks/__init__.py` auto-discovers and imports all task packages via `import_packages()` (blacklist: `utils`, `.mdp`, `pick_place`). Task hierarchy:
  - `common_config/` — shared camera (`camera_configs.py`) and robot (`robot_configs.py`) configurations
  - `common_observations/` — observation functions: `g1_29dof_state.py`, `gripper_state.py`, `dex3_state.py`, `camera_state.py`
  - `common_scene/` — base scene configs per object type (cylinder, red block, etc.)
  - `common_termination/` — reset conditions (object out of workspace bounds)
  - `common_rewards/` — reward computation functions
  - `common_event/` — event manager registration (reset_object_self, reset_all_self)
  - `g1_tasks/` — G1 robot task variants (15 tasks)
  - `h1-2_tasks/` — H1-2 robot task variants (3 tasks)

- **`robots/`** — `unitree.py` contains `ArticulationCfg` definitions for each robot+hand combination. USD paths reference `PROJECT_ROOT` env var.

- **`teleimager/`** — Git submodule (branch: `sim`) from `unitreerobotics/teleimager`. Multi-camera image streaming via ZMQ. Installed as editable package.

- **`tools/`** — Utilities: `augmentation_utils.py` (lighting/camera augmentation), `data_json_load.py` (dataset loading/state serialization), `episode_writer.py`, `get_reward.py`, `get_stiffness.py`, `rerun_visualizer.py`, `shared_memory_utils.py`, USD conversion tools.

### Task naming convention
`Isaac-{Action}-{Object}-{Robot}-{Hand}-{Mode}`
- Action: `PickPlace`, `Stack`, `Move`, `Pick`
- Object: `Cylinder`, `RedBlock`, `RgyBlock`
- Robot: `G129` (G1 29-DOF), `H12-27dof`
- Hand: `Dex1` (gripper), `Dex3` (3-finger), `Inspire`
- Mode: `Joint` (fixed base) or `Wholebody` (mobile)

### Adding a new task
1. Create a common scene in `tasks/common_scene/` (if needed)
2. Add termination conditions in `tasks/common_termination/` (if needed)
3. Create a new task directory under `tasks/g1_tasks/` (or `tasks/h1-2_tasks/`) with:
   - `mdp/observations.py` — import observation functions from `common_observations`
   - `mdp/terminations.py` — import termination functions from `common_termination`
   - `__init__.py` — register task with `gym.register(id="Isaac-...", entry_point="isaaclab.envs:ManagerBasedRLEnv", ...)`
   - `*_env_cfg.py` — environment config importing common scene + robot/camera configs
4. Add the import to the parent `__init__.py` (e.g., `tasks/g1_tasks/__init__.py`) and update `__all__`

## Dependencies
- Python 3.10 (Isaac Sim 4.5) or 3.11 (Isaac Sim 5.x)
- Isaac Sim 4.5.0 / 5.0.0 / 5.1.0 + Isaac Lab
- CycloneDDS 0.10.x (built from source) + unitree_sdk2_python
- pinocchio (imported before AppLauncher in sim_main.py — order matters)
- PyTorch with CUDA
- See `requirements.txt` for additional deps (rerun-sdk, pyzmq, pynput, onnxruntime, aiortc, aiohttp)
