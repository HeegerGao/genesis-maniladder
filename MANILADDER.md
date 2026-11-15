# genesis-maniladder: what differs from Genesis

This repository is [Genesis](https://github.com/Genesis-Embodied-AI/Genesis) 0.3.6 at upstream commit
`f44047923efad37c78a9a2200878cb977580780c`, plus five changes that the
[ManiLadder benchmark](https://github.com/HeegerGao/ManiLadder-Benchmark) needs. The upstream history is kept; the
changes are the five commits whose subject starts with `[maniladder-rl]`. Everything else, including the Python
package name (`genesis-world`) and version (`0.3.6`), is upstream Genesis, distributed under its Apache-2.0 license
([LICENSE](LICENSE)).

## Install

```bash
pip uninstall -y genesis-world          # if another Genesis is installed
pip install git+https://github.com/HeegerGao/genesis-maniladder.git
```

Check that Python imports this build:

```bash
python -c "import genesis, pathlib; print('RASTERIZER_BATCHED_CAMERA' in (pathlib.Path(genesis.__file__).parent / 'vis' / 'camera.py').read_text())"   # True
```

## Changes

| Commit subject | Files | What it does |
|---|---|---|
| Rasterizer: one image and one camera pose per environment | `genesis/vis/camera.py`, `genesis/ext/pyrender/renderer.py`, `genesis/ext/pyrender/scene.py` | Cameras of a batched scene (`scene.build(n_envs=N)` with `VisOptions.env_separate_rigid=True`) hold one pose per rendered environment and may be attached to a link, so every environment gets its own image and a wrist camera follows the hand of its own environment. Backport of the relevant parts of upstream PRs #2212, #2560 and #2564 (Genesis 0.4.4). `genesis.vis.camera.RASTERIZER_BATCHED_CAMERA = True` marks this build. |
| Rasterizer: draw meshes and list lights in creation order | `genesis/ext/pyrender/scene.py` | pyrender kept mesh and light nodes in Python sets, so the draw order followed object addresses: two processes rendering the same state could produce different pictures of coplanar or blended surfaces. Nodes are now visited in the order they were added. |
| Rasterizer: deformable bodies of one environment only in its image | `genesis/vis/rasterizer_context.py`, `genesis/ext/pyrender/jit_render.py` | Particle and mesh nodes of the non-rigid solvers (MPM, SPH, PBD, FEM) are tagged with their environment and only drawn into that environment's image; upstream drew the cloth of all environments into every image. |
| Rigid solver: environment-specific poses of fixed links | `genesis/engine/solvers/rigid/rigid_solver_decomp.py` | `set_pos` / `set_quat` of a fixed body accept a different pose per environment (upstream raises). The solver sets `_env_specific_fixed_poses`; the cached world-frame vertices of fixed geoms (`get_verts`, SAP coupler, raycaster) still exist once for all environments. |
| Inverse kinematics: Jacobian over the requested dofs only | `genesis/engine/entities/rigid_entity/rigid_entity.py` | Inverse kinematics called with a subset of the dofs (`dofs_idx_local`, e.g. the arm joints of a mobile manipulator) builds its Jacobian from those dofs only, so the solve no longer moves the other joints on the chain (the mobile base). `entity.get_jacobian()` shares the kernel and returns the columns of the dofs of the last inverse-kinematics call. |

Scenes with one environment behave as in upstream Genesis, with two exceptions: the draw order of coplanar /
blended surfaces is now fixed (the picture is one of those upstream produces), and inverse kinematics restricted to
some dofs no longer moves the others.

The same changes are kept as patches against the upstream commit in the ManiLadder repository
(`third_party/genesis_rl/`).
