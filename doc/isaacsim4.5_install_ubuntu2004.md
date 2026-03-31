# Isaac Sim 4.5.0 on Ubuntu 20.04 - Additional Notes

Supplementary notes for issues not covered in the official install doc (`isaacsim4.5_install.md` Section 2.3).

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
