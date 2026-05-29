# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Layout

This is a three-component monorepo for an automated cup-stacking robot system:

```
ros2-cup-stack/
├── cup_stack/      # ROS 2 Humble workspace — robot motion control
├── server/         # FastAPI backend — bridges ROS and the web frontend
├── frontend/       # React/TypeScript/Vite — web dashboard UI
└── dsr_practice/   # Earlier Doosan MoveIt2 practice workspace (reference only)
```

Each sub-directory has its own git history. `cup_stack/` and `dsr_practice/` contain git submodules pointing to the upstream Doosan driver (`doosan-robot2`).

## System Architecture

```
Browser
  └─ React frontend (Vite, port 80 via nginx)
       ├─ HTTP REST  →  FastAPI server (port 8000 or per-domain ports)
       └─ WebSocket  →  FastAPI server  →  RosBridge (port 9090, WebSocket)
                                                └─ ROS 2 Humble topics/services
                                                        └─ Doosan M0609 robot
```

The server connects to ROS through `roslibpy` (rosbridge protocol). It never runs ROS nodes directly — all robot communication goes through `server/server/ros/bridge.py` (`RosBridge` singleton).

## cup_stack (ROS 2 Layer)

See `cup_stack/CLAUDE.md` for full details. Quick reference:

```bash
# Build (from cup_stack/)
git submodule update --init --recursive
cd ros2
colcon build --symlink-install
source install/setup.bash

# Syntax check without ROS deps
python3 -m compileall ros2/src/cup_stack

# Run tests
colcon test --packages-select cup_stack && colcon test-result --verbose

# Bringup (sim)
ros2/src/cup_stack/bringup_sim.sh

# Bringup (real robot at 192.168.1.100)
ros2/src/cup_stack/bringup_real.sh 192.168.1.100

# Launch tasks (second terminal, after bringup)
ros2 launch cup_stack cup_pyramid.launch.py nest_inc:=0.0127
ros2 launch cup_stack cup_pyramid_select.launch.py nest_inc:=0.0127
ros2 launch cup_stack cup_unstack.launch.py nest_inc:=0.0127
ros2 launch cup_stack cup_unstack_select.launch.py nest_inc:=0.0127
```

Key files in `cup_stack/ros2/src/cup_stack/cup_stack/`:
- `config.py` — robot frames, gripper settings, cup geometry constants (change here first when hardware changes)
- `runtime.py` — `CupStackRuntime`: MoveItPy init, pose goals, HOME moves, gripper commands
- `tasks/cup_pyramid.py` — 6-cup 3-2-1 pyramid build sequence
- `tasks/cup_unstack.py` — reverse pyramid → nested stack sequence
- `onrobot.py` — Modbus TCP driver for OnRobot RG2/RG6 gripper (IP `192.168.1.1:502`)
- `vision.py` — OpenCV click-to-coordinate selector for `_select` node variants

Planning: Pilz `PTP` for XY moves, Pilz `LIN` for vertical approach/retract (falls back to `PTP` when `strict=False`). Config in `cup_stack/ros2/src/cup_stack/config/moveit_py.yaml`.

## server (FastAPI Backend)

```bash
cd server
pip install -e ".[dev]"
cup-server          # start full server (all domains)
cup-robot           # robot-only entrypoint
cup-handineye       # hand-in-eye calibration entrypoint
cup-handtoeye       # hand-to-eye calibration entrypoint

# Tests
pytest
```

Docker Compose (from `server/`):
```bash
docker-compose up --build
```

Runs nginx + three domain services (robot:8001, handineye:8002, handtoeye:8003) + rosbridge container.

**Server internals** (`server/server/`):

| Layer | Path | Role |
|-------|------|------|
| Config | `config.py` | All settings as frozen dataclasses; override via env vars `ROSBRIDGE_HOST`, `ROSBRIDGE_PORT` |
| ROS bridge | `ros/bridge.py` | `RosBridge` singleton — subscribe/publish/call_service over rosbridge WebSocket |
| Launch | `ros/launch.py` | `LaunchManager` — spawns/stops ROS 2 launch processes from the server |
| Domains | `domains/` | Business logic for robot, hand-in-eye, hand-to-eye calibration |
| Routers | `routers/` | FastAPI routes; `dashboard.py` has WebSocket endpoints for live state/camera/logs |
| Services | `services/camera.py`, `services/calibration.py` | Camera frame streaming, calibration file I/O |
| Entrypoints | `entrypoints/` | `main()` wrappers that start uvicorn for each domain variant |

WebSocket endpoints (from `routers/dashboard.py`):
- `GET /ws/robot/state` — 10 Hz robot joint state JSON
- `GET /ws/camera/{handineye|handtoeye}` — compressed JPEG frames
- `GET /ws/task/log` — active task name, status, last 5 log lines

## frontend (React Dashboard)

```bash
cd frontend
npm install
npm run dev     # dev server
npm run build   # production build
npm run lint    # eslint
```

React 19 + Vite 8 + TypeScript. Deployed via the nginx container in `server/docker-compose.yml`. The nginx config in `server/nginx/nginx.conf` proxies `/api/` to the backend services.

Components: `Header`, `RobotStatus`, `CameraPanel`, `CommandInput`, `LogFeed`.

## Commit Style

Use Conventional Commits: `feat:`, `fix:`, `docs:`, `refactor:`, etc. PRs should state affected task steps, frames, and whether real hardware was used for motion changes.
