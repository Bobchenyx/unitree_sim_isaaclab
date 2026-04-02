# Isaac Sim 4.5.0 on Ubuntu 20.04 - Additional Notes

Supplementary notes for issues not covered in the official install doc (`isaacsim4.5_install.md` Section 2.3).

## Isaac Lab Version Compatibility (Critical)

Each Isaac Sim version requires a specific Isaac Lab commit. Using the wrong version will cause runtime errors (e.g., `wp.transform_compose` not found in Warp).

| Isaac Sim | Isaac Lab commit |
|-----------|-----------------|
| 4.5.0 | `91ad4944f2b7fad29d52c04a5264a082bcaad71d` |
| 5.0.0 | `v2.2.0` |
| 5.1.0 | `80094be3245aa5c8376a7464d29cb4412ea518f5` |

After cloning Isaac Lab, **you must checkout the correct version before running `isaaclab.sh --install`**:
```bash
cd IsaacLab
git checkout 91ad4944f2b7fad29d52c04a5264a082bcaad71d
./isaaclab.sh --install
```

If you already installed the wrong version, fix it by:
```bash
cd IsaacLab
git checkout 91ad4944f2b7fad29d52c04a5264a082bcaad71d
./isaaclab.sh --install
# Then reinstall PyTorch to fix CUDA library version conflicts:
pip install torch==2.5.1 torchvision==0.20.1 --index-url https://download.pytorch.org/whl/cu121
pip install numpy==1.26.4
```

## Why `auto_setup_env.sh` Fails on Ubuntu 20.04

Ubuntu 20.04 has glibc 2.31, but all Isaac Sim pip wheels require `manylinux_2_34` (glibc >= 2.34). This cannot be fixed without upgrading the OS. Must use binary install instead.

## Post-Install Fixes

### numpy version conflict
`unitree_sdk2_python` pulls in numpy 2.x, but Isaac Lab requires numpy < 2. After installing unitree_sdk2_python, run:
```bash
pip install "numpy<2"
```

### Stale pycache after branch switch
When switching git branches, clear Python cache to avoid stale bytecode issues:
```bash
find . -type d -name __pycache__ -exec rm -rf {} + 2>/dev/null
```

## `isaaclab.sh --conda` Behavior

`./isaaclab.sh --conda <env_name>` permanently writes Isaac Sim's environment setup into the conda env's activate hooks. After this, `conda activate <env_name>` is sufficient — no need to manually `source setup_conda_env.sh` each time.

## `--device cpu` vs `--device cuda`

This flag controls the **physics/tensor compute** device, not rendering. Rendering always uses the GPU. With `--device cpu`, expect ~11Hz loop frequency; `--device cuda` is faster.

## Known Warnings (Safe to Ignore)

- `No carb::graphics::DescriptorSet for pool 1` — Vulkan init, harmless
- `rendering_modes extension.toml doesn't exist` — missing optional extension config
- `InstanceAdapter - cannot find cache item for proto ...` — instanced mesh cache miss during cleanup
- `pip check` dependency conflicts (usd-core, lxml, msal, etc.) — non-critical, does not affect simulation
- `Spatial tendons are not supported in Isaac Sim < 5.0` — expected on Isaac Sim 4.5
