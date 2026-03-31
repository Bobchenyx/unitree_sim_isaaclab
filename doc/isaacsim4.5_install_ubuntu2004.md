# Isaac Sim 4.5.0 Installation on Ubuntu 20.04

Ubuntu 20.04 ships with glibc 2.31, but all Isaac Sim pip wheels require `manylinux_2_34` (glibc >= 2.34).
The pip install method (`auto_setup_env.sh`) does **not** work on Ubuntu 20.04. Binary install is required.

## Environment

- OS: Ubuntu 20.04.6 LTS (glibc 2.31)
- GPU: NVIDIA TITAN RTX 24GB
- Driver: 545.23.08 / CUDA 12.3
- Python: 3.10

---

## Step 1: Download Isaac Sim Binary

Download `isaac-sim-standalone-4.5.0-linux-x86_64.zip` (6.7 GB) from:
https://docs.isaacsim.omniverse.nvidia.com/4.5.0/installation/download.html

Extract to a permanent location (do **not** move it later):
```bash
unzip isaac-sim-standalone-4.5.0-linux-x86_64.zip -d /path/to/isaac-sim
```

In this setup, extracted to:
```
/home/yixiao/Downloads/isaac-sim-standalone-4.5.0-linux-x86_64
```

---

## Step 2: Set Environment Variables

Add to `~/.bashrc`:
```bash
export ISAACSIM_PATH="/home/yixiao/Downloads/isaac-sim-standalone-4.5.0-linux-x86_64"
export ISAACSIM_PYTHON_EXE="${ISAACSIM_PATH}/python.sh"
```

Apply:
```bash
source ~/.bashrc
```

Verify:
```bash
${ISAACSIM_PYTHON_EXE} -c "print('Isaac Sim configuration is now complete.')"
```

---

## Step 3: Clone Repositories

```bash
cd /home/yixiao/Workspace/humanoid

git clone https://github.com/isaac-sim/IsaacLab.git
git clone https://github.com/eclipse-cyclonedds/cyclonedds -b releases/0.10.x
git clone https://github.com/unitreerobotics/unitree_sdk2_python
```

Init submodules for unitree_sim_isaaclab:
```bash
cd unitree_sim_isaaclab
git submodule update --init --depth 1
```

---

## Step 4: Build CycloneDDS

```bash
cd /home/yixiao/Workspace/humanoid/cyclonedds
mkdir -p build install
cd build
cmake .. -DCMAKE_INSTALL_PREFIX=../install
cmake --build . --target install
```

---

## Step 5: Set Up Conda Environment

Create and configure the conda environment with Isaac Sim integration:
```bash
conda create -y -n unitree_sim_env python=3.10
conda activate unitree_sim_env
```

Install PyTorch (CUDA 12.1):
```bash
pip install torch==2.5.1 torchvision==0.20.1 --index-url https://download.pytorch.org/whl/cu121
```

---

## Step 6: Install Isaac Lab

```bash
cd /home/yixiao/Workspace/humanoid/IsaacLab
ln -s /home/yixiao/Downloads/isaac-sim-standalone-4.5.0-linux-x86_64 _isaac_sim
./isaaclab.sh --conda unitree_sim_env
conda activate unitree_sim_env
./isaaclab.sh --install
```

> `ln -s` only needs to be run once. `./isaaclab.sh --conda` permanently integrates Isaac Sim into the conda environment's activate hooks.

---

## Step 7: Install Remaining Dependencies

```bash
conda activate unitree_sim_env

# unitree_sdk2_python
cd /home/yixiao/Workspace/humanoid/unitree_sdk2_python
pip install -e .

# Fix numpy version conflict (Isaac Lab requires numpy < 2)
pip install "numpy<2"

# unitree_sim_isaaclab dependencies
cd /home/yixiao/Workspace/humanoid/unitree_sim_isaaclab
pip install -r requirements.txt

# teleimager
cd teleimager
pip install -e .
```

---

## Step 8: Download Assets

```bash
cd /home/yixiao/Workspace/humanoid/unitree_sim_isaaclab
. fetch_assets.sh
```

---

## Running the Simulation

```bash
conda activate unitree_sim_env
cd /home/yixiao/Workspace/humanoid/unitree_sim_isaaclab
python sim_main.py --device cpu --enable_cameras --task Isaac-PickPlace-Cylinder-G129-Dex1-Joint --enable_dex1_dds --robot_type g129
```

- First launch is slow (shader compilation). Subsequent launches are faster.
- Close with `Ctrl+C`.
- `--device cpu` controls the physics/tensor compute device, not rendering (rendering always uses GPU).

### Optional: Shell Alias

```bash
echo 'alias unitreesim="conda activate unitree_sim_env && cd /home/yixiao/Workspace/humanoid/unitree_sim_isaaclab"' >> ~/.bashrc
source ~/.bashrc
```

---

## Known Issues

| Warning/Error | Impact |
|---|---|
| `No carb::graphics::DescriptorSet for pool 1` | None, safe to ignore |
| `rendering_modes extension.toml doesn't exist` | None, safe to ignore |
| `pip check` dependency conflicts (usd-core, lxml, etc.) | None for normal usage |
| numpy conflicts after unitree_sdk2 install | Fix with `pip install "numpy<2"` |
