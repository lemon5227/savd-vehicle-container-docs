# SAVD Vehicle Container Documentation

This repository documents the Docker/ROS2 container composition of the SAVD small vehicle test platform.

The documentation is based on a read-only live inspection of the vehicle at `172.21.16.162` on 2026-06-02. No remote files were modified, no containers were restarted, and no recovery scripts were executed during the inspection.

## Documents

- [中文：SAVD 小车容器组成与源码职责说明](docs/SAVD_container_composition_zh.md)
- [English: SAVD Vehicle Container Composition and Source Responsibilities](docs/SAVD_container_composition_en.md)
- [English onboarding and test guide](SAVD_onboarding_and_test_guide.md)
- [Original Chinese analysis baseline](SAVD_container_analysis.md)

## Scope

The main documents explain:

- Current Docker Compose stack and container inventory.
- Runtime image, command, mount, and status of each container.
- Source-code paths inside each container.
- ROS2 nodes, topics, services, and actions owned by each container.
- GUI-to-API-to-ROS2 control flow.
- VESC, ESP32/micro-ROS, camera, GPS, diagnostics, and Foxglove chains.
- Current live issues observed during inspection.

## Safety and Credentials

This repository must not contain SSH passwords, tokens, private keys, or other credentials.

Some API endpoints and GUI controls can command the physical vehicle. Read the warning sections in the documentation before using control APIs or running recovery scripts.

## Current Key Findings

- The active stack is `savd_docker` with 17 running containers.
- Core control containers are `savd_sysmode`, `savd_syscontrol`, `savd_manop`, `savd_wpo`, and `savd_vehicle`.
- Hardware interface containers are `vesc_driver`, `vesc_ackermann`, and `savd_micro_ros_agent`.
- The front camera stream is available through MediaMTX; the rear camera stream is currently unavailable because the rear RTSP source returns `503 Service Unavailable`.
- The U-Blox container currently runs static transforms only; the real GPS node is commented out in the launch file.
- `savd_jetson_stats` is exited, so JetsonStats diagnostics are stale.
- The physical joystick device `/dev/input/js*` was not present during inspection.

