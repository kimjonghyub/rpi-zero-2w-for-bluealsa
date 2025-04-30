# Raspberry Pi: Bluetooth A2DP Audio Input using BlueALSA (Successfully Verified)

This guide explains how to install and configure `bluez-alsa` to receive audio via Bluetooth (e.g., from an iPhone) and use it as an input stream for audio processing or playback on Raspberry Pi. This setup is confirmed working with real-time DSP audio using `alsaaudio`.

---

## ✅ Verified System Environment

- **Raspberry Pi OS**: Debian Bullseye (ARMv7)
- **BlueZ Version**: Default from Bullseye
- **Python 3.9**
- **`alsaaudio` (pyalsaaudio)**: Python interface for ALSA
- **Bluetooth Source**: iPhone (A2DP)

---

## 📦 Required Dependencies

Install required packages before building BlueALSA:

```bash
sudo apt update
sudo apt install -y \
  build-essential \
  git \
  libbluetooth-dev \
  libasound2-dev \
  libdbus-1-dev \
  libglib2.0-dev \
  libortp-dev \
  check \
  automake \
  autoconf \
  libtool \
  libsbc-dev \
  cmake \
  pkg-config
```

---

## 🔧 Building and Installing `bluez-alsa` (from source)

```bash
cd ~
git clone https://github.com/Arkq/bluez-alsa.git
cd bluez-alsa

./configure
make -j4
sudo make install
```

> 💡 Ignore `--enable-a2dp-sink` option if it shows a warning — modern versions auto-enable A2DP profiles.

---

## 🔄 BlueALSA Daemon Setup

Start the BlueALSA daemon with A2DP Sink enabled:

```bash
sudo /usr/bin/bluealsad --profile=a2dp-sink
```

Optionally make this a systemd service (not covered here).

---

## 🔊 Connect iPhone (or other source device)

```bash
bluetoothctl

# In bluetoothctl shell:
power on
agent on
default-agent
scan on           # Find MAC address (e.g., 28:8E:EC:0F:9B:20)
pair MAC_ADDRESS
connect MAC_ADDRESS
trust MAC_ADDRESS
```

---

## 🦢 Test Audio Playback from Bluetooth

```bash
bluealsa-aplay 28:8E:EC:0F:9B:20
```

This confirms real-time audio streaming via ALSA. You should hear iPhone audio on the speaker connected to the Pi.

---

## 🎛 ALSA Configuration (Optional for `/etc/asound.conf`)

Basic device listing:

```bash
aplay -L | grep bluealsa
```

Example output:

```
bluealsa:HCI=hci0,DEV=28:8E:EC:0F:9B:20,PROFILE=a2dp-source
```

To simplify usage in code, you can create `.asoundrc` (optional):

```bash
pcm.!default {
    type plug
    slave.pcm {
        type bluealsa
        device "28:8E:EC:0F:9B:20"
        profile "a2dp-source"
    }
}
```

---

## 🐍 Using `alsaaudio` in Python

Python code sample (for capture):

```python
import alsaaudio

inp = alsaaudio.PCM(type=alsaaudio.PCM_CAPTURE, device='bluealsa')
inp.setchannels(2)
inp.setrate(48000)
inp.setformat(alsaaudio.PCM_FORMAT_S16_LE)
inp.setperiodsize(480)

while True:
    length, data = inp.read()
    if length:
        # process audio
        pass
```

Playback device should be `hw:0,0` or use `aplay -L` to find your active sink.

---

## 🚀 DSP Audio Processing (Summary)

- Sample rate: `48000`
- Chunk size: `480`
- DSP threading: capture → queue → filter → output
- Filtering: `highpass`, `notch`, `bandpass` via `scipy.signal.sosfilt`
- Output via `alsaaudio.PCM_PLAYBACK`

---

## ✅ Final Status

- ✅ Real-time audio capture from iPhone via Bluetooth A2DP
- ✅ Clean playback through speaker (`hw:0,0`)
- ✅ DSP processing integrated and optimized (no jitter)
- ✅ `bluealsa` built and configured manually for full control

---

## 📜 Notes

- If audio is jittery, ensure you use 48000Hz sample rate with 480 chunk size
- Ensure **only one application accesses the playback device (`hw:0,0`)** at a time
- Use `bluealsa-aplay` only for testing — your DSP app should directly connect to `bluealsa` as input

---

## 📚 References

- https://github.com/Arkq/bluez-alsa
- https://www.freedesktop.org/wiki/Software/PulseAudio/Documentation/User/Bluetooth/
- https://linux.die.net/man/7/bluealsa

---

