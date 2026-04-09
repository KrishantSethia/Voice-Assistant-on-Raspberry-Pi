# Raspberry Pi Voice Assistant - Project Context

## Overview
Setting up a Google Gemini-powered voice assistant on Raspberry Pi 5, based on the [TechMakerAI tutorial](https://github.com/techmakerai/Hands-on-Tutorial-Voice-Assistant-on-Raspberry-Pi).

---

## Hardware Setup

| Component | Details |
|-----------|---------|
| Board | Raspberry Pi 5 |
| OS | Raspberry Pi OS (Bookworm) |
| Network | Ethernet (LAN) connected to router |
| Audio Input | Galaxy Buds2 Pro (Bluetooth) |
| Audio Output | Galaxy Buds2 Pro (Bluetooth) |
| LEDs | Optional - GPIO 24 (green), GPIO 25 (red) |

---

## Network Configuration

- **SSH enabled** via `sudo raspi-config` → Interface Options → SSH
- **Static IP**: Check with `hostname -I` (IP may change on reboot)
- **DNS fix applied**: `echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf`
- **SSH config on Mac** (`~/.ssh/config`):
  ```
  Host pi
      HostName <raspberry-pi-ip>
      User krishant
  ```

---

## Software Installation

### System packages installed:
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install ufw -y
sudo ufw allow ssh
sudo ufw enable

sudo apt install python3 python3-pip python3-venv -y
sudo apt install portaudio19-dev python3-pyaudio flac espeak -y
sudo apt install python3-lgpio -y
sudo apt install pulseaudio pulseaudio-module-bluetooth pavucontrol -y
sudo apt install neovim -y
```

### Python virtual environment:
```bash
cd $HOME
python3 -m venv .venv --system-site-packages
source $HOME/.venv/bin/activate
```

**Note**: `--system-site-packages` is required so the venv can access `lgpio` installed via apt.

### Python packages installed:
```bash
pip install speechrecognition sounddevice pyaudio
pip install google-generativeai
pip install gtts pygame gpiozero
```

---

## Project Location

```bash
$HOME/code/Hands-on-Tutorial-Voice-Assistant-on-Raspberry-Pi/
```

Cloned from:
```bash
git clone https://github.com/techmakerai/Hands-on-Tutorial-Voice-Assistant-on-Raspberry-Pi.git
```

---

## Bluetooth Audio Setup

### Galaxy Buds2 Pro connected via:
```bash
bluetoothctl
# Already paired and connected
```

### Switched to headset profile (enables microphone):
```bash
pactl set-card-profile 100 headset-head-unit
```

### Available audio profiles for Buds:
- `off` — disabled
- `a2dp-sink` — high quality playback only (no mic)
- `a2dp-sink-sbc_xq` — high quality playback only (no mic)
- `headset-head-unit-cvsd` — headset with mic (lower quality codec)
- `headset-head-unit` — headset with mic (mSBC codec) ← **USE THIS**

### Check audio sources:
```bash
pactl list sources short
```

### List available microphones in Python:
```bash
python3 -c "
import speech_recognition as sr
for i, name in enumerate(sr.Microphone.list_microphone_names()):
    print(f'{i}: {name}')
"
```

Current output shows:
- `0: vc4-hdmi-1: MAI PCM i2s-hifi-0 (hw:1,0)` — HDMI audio
- `1: pulse` — PulseAudio (use this for Bluetooth)
- `2: default`

---

## Code Modifications Needed

### 1. Add Google Gemini API Key
In `gva7_led.py`, find:
```python
my_api_key = " "
```
Replace with your actual API key from https://aistudio.google.com/app/apikey

### 2. Set Microphone Device Index
In `gva7_led.py`, find:
```python
mic = sr.Microphone()
```
Change to:
```python
mic = sr.Microphone(device_index=1)  # Use PulseAudio for Bluetooth
```

---

## How the Voice Assistant Works

1. **Wake word**: Say "Jack" to activate
2. **Listening**: Green LED on (GPIO 24)
3. **Speech-to-text**: Uses Google Speech Recognition
4. **AI Processing**: Sends text to Google Gemini API
5. **Text-to-speech**: Uses gTTS (Google Text-to-Speech)
6. **Speaking**: Red LED on (GPIO 25), plays audio via pygame
7. **Sleep command**: Say "That's all" to deactivate

### Threading Architecture:
- **Thread 1**: LLM response streaming (chatfun)
- **Thread 2**: Text-to-speech conversion (text2speech)
- **Thread 3**: Audio playback (play_audio)

Uses queues to pass data between threads for smooth streaming.

---

## Known Issues & Fixes

### 1. Deprecated google.generativeai package
**Warning**: Package is deprecated, should migrate to `google.genai`
**Status**: Still works for now, migration recommended for future

### 2. GPIO access in virtual environment
**Problem**: `lgpio` not found in venv
**Fix**: Create venv with `--system-site-packages` flag

### 3. DNS resolution failure
**Problem**: `ping google.com` fails with "temporary failure in name resolution"
**Fix**: `echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf`

### 4. Bluetooth mic not detected
**Problem**: Buds connected but no microphone available
**Fix**: Switch from A2DP to headset profile:
```bash
pactl set-card-profile 100 headset-head-unit
```

### 5. Keyboard layout showing Hindi
**Problem**: Physical keyboard outputs Hindi characters
**Fix**: `sudo raspi-config` → Localisation Options → Keyboard → English (US)

---

## Running the Voice Assistant

```bash
cd $HOME/code/Hands-on-Tutorial-Voice-Assistant-on-Raspberry-Pi
source $HOME/.venv/bin/activate
python3 gva7_led.py
```

Press `Ctrl+C` to exit.

---

## Next Steps / TODO

- [ ] Test microphone with Bluetooth Buds
- [ ] Verify audio output works
- [ ] Test full conversation flow
- [ ] Consider migrating to `google.genai` package
- [ ] Consider using USB microphone for more reliable input
- [ ] Make DNS fix permanent (edit `/etc/resolv.conf` or configure DHCP)
- [ ] Set up script to auto-run on boot (optional)

---

## Useful Commands

```bash
# Check IP address
hostname -I

# SSH from Mac
ssh pi  # or ssh krishant@<ip>

# Check Bluetooth status
bluetoothctl
# Inside: devices, info <MAC>, connect <MAC>

# Check audio devices
pactl list cards short
pactl list sources short
pactl list sinks short

# Restart PulseAudio
pulseaudio --kill
pulseaudio --start

# Activate Python venv
source $HOME/.venv/bin/activate

# Check if services are running
sudo systemctl status ssh
sudo systemctl status bluetooth
```

---

## File Structure

```
$HOME/
├── .venv/                    # Python virtual environment
├── code/
│   └── Hands-on-Tutorial-Voice-Assistant-on-Raspberry-Pi/
│       ├── gva7_led.py       # Main voice assistant script
│       ├── gva7_led_old.py   # Older version
│       ├── schematic1.png    # LED wiring diagram
│       ├── README.md
│       └── LICENSE
```

---

## References

- [GitHub Repo](https://github.com/techmakerai/Hands-on-Tutorial-Voice-Assistant-on-Raspberry-Pi)
- [YouTube Tutorial](https://youtu.be/uV6hJQcuW4w)
- [Google Gemini API](https://aistudio.google.com/app/apikey)