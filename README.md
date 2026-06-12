# capture-card-viewer

A dead-simple, low-latency viewer for a USB HDMI capture card on Linux. It uses
[`ffplay`](https://ffmpeg.org/ffplay.html) to display the video and
[`pactl`](https://www.freedesktop.org/wiki/Software/PulseAudio/) to loop the
capture card's audio straight to your speakers — so you can watch and hear a
console (or any HDMI source) on your desktop with minimal delay.

It was written for an **Elgato Game Capture HD60 S+**, but it will work with any
capture device that exposes a V4L2 video node and a PulseAudio/PipeWire audio
source. You just need to point the two variables at the right devices (see
[Configuration](#configuration)).

## Requirements

- `ffplay` — ships with [FFmpeg](https://ffmpeg.org/)
- `pactl` — ships with PulseAudio (also provided by PipeWire's `pipewire-pulse`)
- `v4l2-ctl` — from `v4l-utils`, only needed to discover your video device

On Debian/Ubuntu:

```sh
sudo apt install ffmpeg v4l-utils pulseaudio-utils
```

On Arch:

```sh
sudo pacman -S ffmpeg v4l-utils libpulse
```

## Usage

```sh
./capture-card-viewer
```

A window titled `capture-card-viewer` opens with your capture feed. Press `q` or close the
window to quit. On exit the script unloads the audio loopback module it created,
so it cleans up after itself.

## Configuration

Open the `capture-card-viewer` script and set these two variables near the top:

```sh
VIDEO=/dev/video2
AUDIO=alsa_input.usb-Elgato_Game_Capture_HD60_S__0004F0D20C000-03.analog-stereo
```

### Finding your video device (`VIDEO`)

List the V4L2 video devices on your system:

```sh
v4l2-ctl --list-devices
```

You'll get something like:

```
Elgato Game Capture HD60 S+ (usb-0000:00:14.0-5):
        /dev/video2
        /dev/video3
```

Use the **first** `/dev/videoN` listed under your capture card. A single device
often exposes several nodes; the lowest-numbered one is usually the capture
stream. If video fails to open, try the next node.

To check what formats and framerates a node actually supports:

```sh
v4l2-ctl -d /dev/video2 --list-formats-ext
```

This tells you the valid values for the `-input_format`, `-framerate`, and
resolution flags in the `ffplay` command (the script uses `yuyv422` at `240`
fps — adjust to whatever your card and source report).

### Finding your audio source (`AUDIO`)

List the audio sources PulseAudio/PipeWire knows about:

```sh
pactl list sources short
```

You'll get something like:

```
44  alsa_input.usb-Elgato_Game_Capture_HD60_S__0004F0D20C000-03.analog-stereo  ...
45  alsa_output.pci-0000_00_1f.3.analog-stereo.monitor                         ...
```

Copy the **name** (the second column) of the source that belongs to your capture
card — typically the one containing `alsa_input` and your device's name.

## How it works

1. `pactl load-module module-loopback source=$AUDIO` routes the capture card's
   audio to your default output and returns the new module's ID.
2. A `trap` on `INT` unloads that exact module on exit, so you don't leave
   orphaned loopbacks piling up (run `pactl list modules short` to check).
3. `ffplay` plays the video with low-latency flags (`-fflags nobuffer`,
   `-flags low_delay`, `-framedrop`) to keep the delay as small as possible.

## License

MIT
