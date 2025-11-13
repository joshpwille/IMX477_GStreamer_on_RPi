
## Overview and Goals

This document describes a minimal but reasonably high-quality video pipeline using:

- A Raspberry Pi 5 as the video sender.
- A Raspberry Pi HQ Camera (Sony IMX477) as the image sensor.
- libcamera as the camera stack on the Pi.
- GStreamer 1.0 as the multimedia framework.
- A laptop (Linux) as the optional network receiver.

The main goals are:

1. Verify that the IMX477 can be driven through `libcamerasrc` inside GStreamer.
2. Achieve a stable local preview at a useful resolution (e.g. 1280×720 @ 30 fps).
3. Provide a clear path to RTP/UDP network streaming from the Pi to a laptop, without any CCSDS/DTN complexity.

The idea is intentionally simple: use GStreamer as the “plumbing” between the camera (IMX477),
the encoder (H.264), and the sinks (local display, file, or network).

---

## Conceptual System Diagrams

### Local Preview (On the Pi)

IMX477 camera → libcamera / `libcamerasrc` → GStreamer pipeline → display (preview window)

**Interpretation**

- The IMX477 raw sensor output is exposed via the libcamera stack.
- `libcamerasrc` is the GStreamer element that talks to libcamera and yields frames into GStreamer.
- The pipeline converts those frames to a display-friendly format and pushes them to a video sink
  (`autovideosink`).

### Record to File (Future Extension)

IMX477 → `libcamerasrc` → GStreamer (convert + H.264 encoder + muxer) → file (e.g. `.mp4`)

**Interpretation**

- Same camera source as local preview.
- Insert an H.264 encoder and a container muxer (e.g. MP4).
- Write the resulting compressed video to disk via a `filesink`.

### Network Stream

IMX477 → `libcamerasrc` → GStreamer (H.264 encoder + RTP payloader + `udpsink`) → LAN →  
GStreamer receiver (`udpsrc` + depay + decode + display)

**Interpretation**

- The Pi 5 acts as a live camera source and encoder.
- The encoded H.264 video is wrapped into RTP packets and transmitted via UDP on the LAN.
- The laptop receives RTP packets, depayloads, decodes, and displays them in real time using another
  GStreamer pipeline.

---

## 3.3 Networking

- Ensure the Pi 5 and the laptop are on the same LAN (Ethernet or Wi-Fi).
- Confirm IP addresses on each device, for example:

On the Pi:

```bash
ip addr
```

On the laptop:

```bash
ip addr
```

---

## 4. Software Dependencies

### 4.1 Laptop (Receiver)

Install GStreamer and its core plugins on the laptop:

```bash
sudo apt update
sudo apt install gstreamer1.0-tools gstreamer1.0-plugins-base   gstreamer1.0-plugins-good gstreamer1.0-plugins-bad   gstreamer1.0-plugins-ugly gstreamer1.0-libav -y
```

### 4.2 Raspberry Pi 5 (Sender)

On the Pi 5 (running Raspberry Pi OS with the libcamera stack):

```bash
sudo apt update
sudo apt install gstreamer1.0-tools gstreamer1.0-plugins-base   gstreamer1.0-plugins-good gstreamer1.0-plugins-bad   gstreamer1.0-plugins-ugly gstreamer1.0-libav -y
```

Verify that `libcamerasrc` is present:

```bash
gst-inspect-1.0 libcamerasrc
```

Test a simple local preview:

```bash
gst-launch-1.0 -v libcamerasrc !   "video/x-raw,format=NV12,width=1280,height=720,framerate=30/1" !   videoconvert ! autovideosink
```

---

## X11 Forwarding (Optional)

If you are using SSH from a laptop to the Pi, you can forward the preview window over X11:

- Connect with:

  ```bash
  ssh -X pi@<pi-ip-address>
  ```

- Confirm that `$DISPLAY` is set:

  ```bash
  echo $DISPLAY
  ```

---

## Network Streaming

Once the local preview is working, it is straightforward to extend the concept to a network streaming
test. The idea is:

- **Sender (Pi 5):** encode IMX477 frames as H.264, wrap them in RTP, and send over UDP to the laptop.
- **Receiver (Laptop):** listen on the UDP port, depayload the RTP stream, decode H.264, and display.

---

## Receiver Pipeline (Laptop)

On the laptop, run:

```bash
gst-launch-1.0 -v   udpsrc port=5000 caps="application/x-rtp,media=video,encoding-name=H264,payload=96" !   rtph264depay !   avdec_h264 !   videoconvert !   autovideosink
```

**Explanation**

- `udpsrc port=5000`: listen for UDP packets on port 5000.
- `caps="application/x-rtp,media=video,encoding-name=H264,payload=96"`:
  - Tells GStreamer that the incoming UDP packets are RTP with H.264 payload.
  - `payload=96` is an arbitrary dynamic payload type ID; it just needs to match the sender.
- `rtph264depay`: strips RTP headers, reconstructs a clean H.264 byte stream.
- `avdec_h264`: software H.264 decoder (from libav).
- `videoconvert ! autovideosink`: convert to a sink-friendly format and display the video.  
  Leave this running; it will block and wait for incoming RTP packets.

---

## Sender Pipeline (Pi 5)

On the Pi 5, run:

```bash
gst-launch-1.0 -v libcamerasrc !   "video/x-raw,format=NV12,width=1280,height=720,framerate=30/1" !   videoconvert !   x264enc tune=zerolatency !   rtph264pay config-interval=1 pt=96 !   udpsink host=192.168.1.100 port=5000
```

---

## 7. Tuning Knobs and Considerations

### 7.1 Resolution and Frame Rate

- Start with 1280×720 at 30 fps:
  - Low enough bandwidth and CPU load to be stable.
  - High enough resolution to be visually useful.
- Once stable, consider:
  - 1920×1080 at 30 fps for higher quality.
  - Lower frame rates (e.g. 15 fps) if bandwidth or CPU becomes an issue.

### 7.2 Encoder Settings

If using `x264enc`:

- `tune=zerolatency` for low-latency streaming.
- `bitrate` property to control bandwidth/quality.
- `speed-preset` to choose between speed and compression efficiency.

If using a hardware encoder (`v4l2h264enc`):

- You may need to adjust the caps to match hardware-supported formats.
- Hardware encoder typically reduces CPU load on the Pi.

### 7.3 Latency and Buffering

- Extra `queue` elements increase buffering and stability but add latency.
- For low latency:
  - Avoid unnecessary queues.
  - Use encoders/decoders tuned for low-latency modes.
- Network quality (Wi-Fi vs Ethernet) also affects jitter and latency.
