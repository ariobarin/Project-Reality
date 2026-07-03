# Project Reality

Real-time 3D ball tracking that feeds live coordinates into a Unity game. Two Raspberry Pi cameras watch the play area, locate a colored ball through computer vision, triangulate its position in 3D space, and stream the result over TCP to a Unity server that uses it in-game.

## What it does and the problem it solves

The goal is to bring a physical ball into a virtual world without depth sensors or motion-capture rigs. A player moves a real ball around a tracked surface, and the game reacts to that ball's 3D position in real time.

Concretely, the pipeline does the following:

- Captures synchronized frames from two cameras (a stereo pair).
- Detects the ball in each frame using HSV color thresholding and contour analysis.
- Converts the ball's pixel position in each camera into an angular direction using each camera's field of view.
- Triangulates the two angular rays into a single x, y, z coordinate in real-world units (centimeters).
- Sends that coordinate over a TCP socket to a Unity host, which consumes it each frame.

This gives a low-cost, low-latency positional input for interactive games and demos, built entirely from commodity parts (a pair of cameras, a small computer, and a Unity client).

## Key features

- Stereo triangulation: two-camera geometry turns 2D detections into a 3D world position, with explicit guards against degenerate (near-parallel) ray configurations.
- HSV-based detection with a live tuner: an OpenCV trackbar UI lets you adjust hue, saturation, and value bounds per camera and persist them to `hsv_config.json`. The mask logic combines the H, S, and V channels with a majority-vote rule so detection stays robust to lighting.
- Threaded networking: the Python client runs frame capture, target detection, and TCP sending on separate threads so a slow network never stalls the vision loop. The sender is non-blocking with a queue and a clean QUIT/shutdown handshake.
- Unity TCP server: a `MonoBehaviour` TCP listener accepts the coordinate stream on the main thread-safe Unity Update loop, plus a calibration protocol that maps tracked coordinates to the four corners of a 1920x1080 play surface.
- Per-camera configuration: each camera keeps its own HSV bounds and mounting angles, since the two views differ in lighting and geometry.

## Repository layout

- `Vision/` - the Python computer-vision and networking side.
  - `camera.py`: Raspberry Pi camera wrapper (Picamera2) that produces HSV frames at a fixed resolution and FPS.
  - `analyze_frame.py`: HSV thresholding, mask combination, contour finding, and the interactive trackbar tuner.
  - `hello.py`: the `BallDetector` class, pixel-to-angle conversion and the two-camera triangulation math.
  - `tcp_client.py`: the threaded TCP client that reads detections and streams `x,y,z` to the game server.
  - `hsv_config.json`: persisted per-camera HSV bounds.
- `ELVINBOSS/` - the Unity/C# side.
  - `TCPServerUnity.cs`: Unity TCP server that accepts connections and queues incoming coordinate lines.
  - `ServerInfoParse.cs`: parses incoming messages, with a calibration mode (four-corner registration) followed by an update mode.
  - `generate_data_points.py` and `TCP_Client`: standalone test harnesses for the message protocol, useful for exercising the Unity server without cameras attached.
- `Game_Demo.mp4`, `Mask_Demo.mp4`: short videos showing the running game and the vision masks.

## Tech stack

- Python 3, OpenCV (`opencv-python`), NumPy, and `picamera2` for camera capture on Raspberry Pi.
- C# / Unity for the game server and message parsing.
- Raw TCP sockets (`socket` on the Python side, `TcpListener` on the Unity side) for inter-process communication, with `TCP_NODELAY` enabled for low latency.
- Standard-library `threading`, `queue`, and `concurrent.futures` for parallel camera reads.

## How to run

The vision side runs on a Raspberry Pi (or any Linux box with two cameras and `picamera2`):

1. Install dependencies: `pip install -r requirements.txt`.
2. Connect and align the two cameras so they share an overlapping view of the play area, separated by the baseline distance defined as `SEPERATION_DISTANCE` in `hello.py`.
3. Tune detection by running `python analyze_frame.py`, switching between cameras with `1`/`2`, dragging the HSV trackbars until only the ball is masked, then pressing Enter to save the bounds to `hsv_config.json`.
4. Start the Unity scene that hosts `TCPServerUnity` and `ServerInfoParse`, and note the host machine's IP.
5. Launch the client: `python tcp_client.py` (optionally updating the `host`/`port` to point at the Unity machine). It connects, then continuously sends `x,y,z` coordinates as the ball moves.
6. Walk through the four-corner calibration so tracked positions map onto the real play surface.

For development without cameras, `ELVINBOSS/TCP_Client` and `ELVINBOSS/generate_data_points.py` together act as a mock producer that sends randomized, protocol-shaped messages to the Unity server.

## Notes and outcomes

- Detection accuracy depends heavily on consistent lighting and a distinctly colored ball; the per-camera HSV bounds exist precisely because each camera sees color and brightness differently.
- The triangulation math assumes the two cameras are roughly coplanar with a known baseline, and it explicitly returns no position when the rays are too close to parallel to resolve cleanly.
- Frame capture from both cameras runs in parallel via a thread pool to keep the loop near the cameras' target FPS.
- The included demo videos (`Game_Demo.mp4`, `Mask_Demo.mp4`) show the end-to-end result: the live game responding to a physical ball, and the raw HSV masks used for detection.
