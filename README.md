# Soft Bubble Gripper

Soft Bubble Gripper is a Raspberry Pi based control and sensing console for a soft gripper with a bubble tactile surface. The project combines live camera detection, contact/deformation estimation, pressure sensing, load-cell validation, limit-switch monitoring, stepper control, and force calibration in one Python/Tk application.

## Key Features

- Live dot/blob detection from a Pi camera.
- 2D and 3D tactile contact-map visualization.
- Pressure-gated contact zeroing for repeatable deformation measurements.
- MPRLS pressure sensor support over I2C.
- HX711 load-cell support with saved calibration.
- Pressure-to-force calibration against the load cell.
- Stepper and limit-switch control through GPIO or native `libgpiod` helpers.
- Optional remote OpenCV/CUDA vision server for offloading image processing.

## Project Layout

```text
.
|-- main.py                                # stable entry point
|-- launch_soft_bubble_gripper.py          # Python launcher with GUI fallback
|-- launch_soft_bubble_gripper.sh          # Raspberry Pi/Linux launcher
|-- soft_bubble_gripper_gui.py             # modern Tk console shell
|-- soft_bubble_gripper_core.py            # core hardware, vision, force, and GUI runtime
|-- soft_bubble_remote_vision_server.py    # optional remote vision HTTP server
|-- vision/
|   |-- dot_pipeline.py                    # dot/blob detection pipeline
|   |-- native_detector.py                 # Python wrapper for native detector
|   `-- __init__.py
|-- native/                                # C helpers for detection, stepper, and limit reader
|-- scripts/setup_raspberry_pi_venv.sh     # Pi virtualenv setup helper
|-- requirements-rpi-bookworm.txt          # pip packages for Raspberry Pi OS Bookworm
|-- RASPBERRY_PI_SETUP.md                  # hardware setup and smoke-test checklist
|-- surface_measurement.py                 # surface/deformation geometry helpers
|-- tactile_contact_pipeline.py            # tactile contact-map pipeline
`-- hx711v0_5_1.py                         # HX711 load-cell driver
```

## Hardware Defaults

| Device | Default |
| --- | --- |
| Limit switch 1 | GPIO17 |
| Limit switch 2 | GPIO27 |
| Stepper PUL | GPIO24 |
| Stepper DIR | GPIO23 |
| Stepper ENA | GPIO26 |
| HX711 DT | GPIO5 |
| HX711 SCK | GPIO6 |
| MPRLS pressure sensor | I2C bus 1, address `0x18` |

## Raspberry Pi Setup

Target platform: Raspberry Pi OS Bookworm 64-bit with desktop.

Enable Camera and I2C:

```bash
sudo raspi-config
```

Install system packages:

```bash
sudo apt update
sudo apt install -y \
  python3-venv python3-tk python3-pil python3-pil.imagetk \
  python3-numpy python3-opencv python3-picamera2 \
  python3-rpi-lgpio python3-libgpiod libgpiod-dev \
  i2c-tools gcc pkg-config
```

Create the virtual environment:

```bash
cd ~/soft-bubble-gripper
chmod +x scripts/setup_raspberry_pi_venv.sh launch_soft_bubble_gripper.sh
./scripts/setup_raspberry_pi_venv.sh
```

The setup script creates `.venv` with `--system-site-packages` so apt-installed camera, OpenCV, Tk, GPIO, and Picamera2 packages remain available.

## Run

```bash
./launch_soft_bubble_gripper.sh
```

or:

```bash
.venv/bin/python main.py
```

`main.py` imports `launch_soft_bubble_gripper.py`, which starts `soft_bubble_gripper_gui.py`. If the modern GUI shell fails, the launcher falls back to the core legacy shell in `soft_bubble_gripper_core.py` and writes diagnostics to `launch_errors.log`.

## Optional Native Helpers

Build the native blob detector, stepper runner, and limit reader:

```bash
cd native
chmod +x build_pi.sh
./build_pi.sh
cd ..
```

When `native/build/libbubble_detect.so` exists, `vision/native_detector.py` and `vision/dot_pipeline.py` can use the C detector path. When `native/build/stepper_runner` and `native/build/limit_reader` exist, the GUI can use native `libgpiod` helpers for more reliable hardware timing.

## Remote Vision Server

Run this on the machine that should process camera frames:

```bash
python3 soft_bubble_remote_vision_server.py --host 0.0.0.0 --port 8765
```

Health check:

```bash
curl http://<host>:8765/health
```

Set the GUI remote URL with:

```bash
BUBBLE_REMOTE_VISION_URL=http://<host>:8765 ./launch_soft_bubble_gripper.sh
```

## Useful Environment Variables

```bash
# Pressure contact gate
BUBBLE_PRESSURE_CONTACT_GATE=0 ./launch_soft_bubble_gripper.sh
BUBBLE_PRESSURE_CONTACT_DELTA_HPA=1.5 ./launch_soft_bubble_gripper.sh

# Stepper depth tracking
BUBBLE_STEPPER_DEPTH_ENABLED=1 ./launch_soft_bubble_gripper.sh
BUBBLE_STEPPER_MM_PER_REV=8.0 ./launch_soft_bubble_gripper.sh

# Native GPIO helpers
PI_BUBBLE_GPIO_CHIP=gpiochip0 ./launch_soft_bubble_gripper.sh
PI_BUBBLE_LIMIT_BACKEND=native ./launch_soft_bubble_gripper.sh
PI_BUBBLE_NATIVE_LIB=/path/to/libbubble_detect.so ./launch_soft_bubble_gripper.sh
```

## Calibration Files

The GUI can create or update these machine-specific files:

- `loadcell_calibration.json`
- `force_calibration.json`
- `camera_deform_calibration.json`

Recalibrate when the gripper, tubing, load-cell mount, pressure sensor, camera position, or test geometry changes. These files are ignored by default in `.gitignore`.

## First Hardware Test Order

1. Launch the GUI only.
2. Start Live Detection with the `picamera2` backend.
3. Use the measurement preset, wait for the image to settle, then press Reset Ref before applying load.
4. Test limit switches on GPIO17 and GPIO27.
5. Test stepper movement with motor power disabled first, then enabled.
6. Confirm I2C with `i2cdetect -y 1`, then test the MPRLS pressure sensor.
7. Test HX711 load cell and save load-cell calibration.
8. Run Force Calibration with pressure and load-cell readings active.

See [RASPBERRY_PI_SETUP.md](RASPBERRY_PI_SETUP.md) for the detailed Raspberry Pi checklist.

## GitHub Notes

Generated runtime files are ignored by default: `.venv/`, `__pycache__/`, `logs/`, `native/build/`, `launch_errors.log`, and local calibration JSON files. Commit calibration files only when you intentionally want to preserve one exact hardware setup.
