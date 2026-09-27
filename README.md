# headtrack-lan
Free, open-source iPhone head tracking for DCS World and other flight simulators. Uses Safari, your local Wi-Fi, and opentrack—no paid apps or extra hardware.
## What it does

```text
iPhone Safari camera → MediaPipe Face Landmarker → HTTPS over your Wi-Fi → Windows bridge → UDP → opentrack → simulator
```

Face detection and pose estimation run in Safari. Camera frames stay on the iPhone; a six-axis pose packet is sent to the PC (X/Y are currently zero; yaw/pitch/roll and Z are active). The page includes center calibration, yaw/pitch/roll sensitivity, axis inversion, Z movement gain, and an adaptive One Euro filter. Settings are saved in that browser. The interface offers English and Spanish and chooses an initial language based on the device locale.

The Z position uses MediaPipe's facial pose matrix, whose canonical model uses centimeters. The filter smooths small motion and responds more quickly to faster motion. The filter cutoff slider trades stability for responsiveness.

## Requirements

- Windows 10/11 PC with Python 3.10 or newer and the `py` launcher.
- iPhone with Safari; phone and PC on the same private Wi-Fi/LAN.
- Free [opentrack](https://github.com/opentrack/opentrack).
- Internet access when starting the page to download MediaPipe's browser bundle, WebAssembly files, and face model. Camera frames are not uploaded.

## Quick setup

1. Download this repository using **Code → Download ZIP** and extract it.
2. Double-click `setup.bat`. It installs Python dependencies, detects the PC's LAN IPv4 address, and creates a local HTTPS certificate and `config.json`. Confirm or type the correct Wi-Fi IPv4 address when asked.
3. Transfer `HeadTrack-CA.cer` to the iPhone using AirDrop or iCloud Drive. Install the profile, then enable full trust at **Settings → General → About → Certificate Trust Settings**. The CA is generated locally by setup. Never share its private key.
4. Double-click `start.bat`. If Windows Firewall asks, allow Python on **Private networks**. Leave this window open while tracking.
5. In opentrack, choose **Input: UDP over network**, set port **4242** in its input settings, choose **Output: freetrack 2.0 Enhanced**, and press **Start**. Keep the output configuration appropriate for the simulator.
6. On the iPhone, open the address printed by setup, such as `https://192.168.1.20:8765/`. Allow camera access, press **Start camera and tracking**, look forward, then press **Calibrate center**.
7. Launch or return to the simulator. For DCS, start opentrack before launching DCS if the game does not detect tracking in an already-open session.

`0.0.0.0` in the bridge configuration means “listen on the PC's network interfaces.” Open the iPhone page using the specific LAN address printed by setup.

## Controls

- **Calibrate center**: sets the current head pose and distance as neutral.
- **Turn sensitivity**: raises or lowers view rotation for a given head turn.
- **Invert yaw/pitch/roll**: corrects an axis whose direction is reversed.
- **Z position multiplier**: changes how strongly leaning toward/away from the phone moves the simulator view.
- **Invert Z**: reverses forward/back movement if needed.
- **Adaptive filter cutoff**: lower values are steadier but slower; higher values respond faster but allow more small movement.

For Z movement, make sure the Z axis is enabled in opentrack's **Mapping** window. Tune the curve to taste. opentrack's own filter can also be adjusted; avoid stacking strong smoothing in both places.

## Configuration

`setup.bat` creates a per-PC `config.json` from the detected address. The checked-in `config.example.json` documents the available settings:

- `lan_ip`: address used in the iPhone URL and HTTPS certificate.
- `https_bind` / `https_port`: local web server bind and port.
- `udp_host` / `udp_port`: opentrack receiver; default `127.0.0.1:4242`.
- `cert_file` / `key_file`: local HTTPS certificate files.

You can edit the sliders in the iPhone page at any time; their values persist in Safari. For another PC/IP, run `setup.bat` there to generate its own certificate. If you change the IP after setup, regenerate the certificate and install/trust the new CA on the phone.

## Protocol

The bridge sends the opentrack **UDP over network** input format: six little-endian IEEE-754 64-bit doubles, in order `x, y, z, yaw, pitch, roll`, 48 bytes total, no header or checksum. This matches [opentrack's UDP tracker source](https://github.com/opentrack/opentrack/blob/master/tracker-udp/ftnoir_tracker_udp.cpp), which reads a `double[6]` directly into its six pose axes. Position values are centimeters and angles are degrees. The bridge forwards to loopback by default, so the iPhone never sends UDP directly.

## Troubleshooting

- **Safari says it cannot open the page**: confirm `start.bat` is still open, use the exact HTTPS address printed by setup, and allow Python through Windows Firewall on Private networks.
- **Camera does not start**: use Safari and allow camera access. Camera access requires HTTPS and the trusted local CA.
- **Page loads but opentrack does not move**: check `UDP over network`, port `4242`, and press Start. The bridge console should print `RX` and `48 bytes` once per second.
- **DCS does not move**: keep opentrack tracking before starting DCS; verify the output protocol is `freetrack 2.0 Enhanced` and that the Z mapping is enabled for zoom.
- **Zoom moves the wrong way**: toggle **Invert Z**. Adjust its multiplier and opentrack Z curve.
- **Tracking jitters or feels delayed**: move the filter cutoff slightly up for faster response or down for more stability. Adjust opentrack's filter conservatively.
- **The PC's LAN IP changed**: rerun setup and recreate the certificate, then reinstall/trust the new CA on the iPhone.

## Privacy and security

The camera stream is processed on the iPhone and is not sent to the PC. Pose values travel only over the local network to the bridge. The local root CA private key can issue certificates trusted by the iPhone: do not publish `ca.key.pem`, `server.key`, generated certificates, or `config.json`. Do not expose port 8765 to the public Internet.

## License and acknowledgements

This project is MIT licensed. It uses MediaPipe Tasks Vision (Apache-2.0) and the MediaPipe Face Landmarker model, loaded from Google's hosting at runtime. opentrack is a separate GPL-licensed project; install it from its official project page.
