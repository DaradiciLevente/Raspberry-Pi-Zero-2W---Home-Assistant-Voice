<h1 align="center">🎙️ DIY Wyoming Satellite - Raspberry Pi Zero 2W </h1>

<p align="center">
  <img src="assets/logo.png" alt="Logo" width="300"/>
</p>


# This repository provides a step-by-step guide and all necessary configuration files to build a high-performance, standalone voice satellite for Home Assistant using the Wyoming protocol.
---

### 🛠️ Hardware Stack

---

Computing: Raspberry Pi Zero 2W


Audio Output (DAC): MAX98357A (I2S Mono Amplifier)


Audio Input (Microphone): INMP441 (I2S Omnidirectional)


Interface: I2S Digital Audio


### 📐 Hardware Wiring (Pinout)

---

To ensure the I2S interface works correctly, connect your components to the following GPIO pins:

```Component	Pin Label	    
                                Raspberry Pi GPIO	     Physical Pin
MAX98357A (DAC)	  LRC	          GPIO 19	               Pin 35 (PCM_FS)
                  BCLK	          GPIO 18	               Pin 12 (PCM_CLK)
                  DIN	          GPIO 21	               Pin 40 (PCM_DOUT)
                  Vin	          5V	                   Pin 2 or 4
                  GND	          GND	                   Pin 6 or 9
                  
INMP441 (Mic)	  WS	          GPIO 19	               Pin 35 (PCM_FS)
                  SCK	          GPIO 18	               Pin 12 (PCM_CLK)
                  SD	          GPIO 20	               Pin 38 (PCM_DIN)
                  VCC	          3.3V	                   Pin 1 or 17
                  GND	          GND	                   Pin 6 or 9
                  L/R	GND	Set to GND for Left Channel
```


### 💾 Software Installation & Configuration

---

## 1. System Preparation

Start with Raspberry Pi OS Lite (64-bit). Install the required audio utilities and Python dependencies:

```
sudo apt update
sudo apt install -y git python3-pip python3-venv alsa-utils alsa-tools sox libsox-fmt-all python3-pyaudio python3-numpy swh-plugins raspi-gpio
```

## 2. Enable I2S Audio (/boot/firmware/config.txt)

Edit the config file to enable the I2S interface and the specific audio overlay:

```
sudo nano /boot/firmware/config.txt
```


```
dtparam=i2s=on
dtparam=audio=off
dtoverlay=max98357a
dtoverlay=googlevoicehat-soundcard
```

Reboot your Pi after saving this file.

## 3. Audio Architecture (/etc/asound.conf)

Since the DAC lacks hardware volume control, we use a softvol (software volume) device. This file also enables full-duplex audio (simultaneous mic and speaker).

Important: Use the asound.conf file provided in this repository to replace your /etc/asound.conf. 

```sudo nano /etc/asound.conf```


```
pcm.mic {
    type hw
    card 0
    device 0
    format "S32_LE"
    channels 1
    rate 16000
}

pcm.dac {
    type hw
    card 0
    device 0
}

pcm.softvol {
    type softvol
    slave.pcm "dac"
    control {
        name "Master"
        card 0
    }
}

pcm.!default {
    type asym
    playback.pcm "softvol"
    capture.pcm "mic"
}
```


## 🛰️ 4. Wyoming Satellite Setup

Installation via Python Virtual Environment

Install the latest version of the Wyoming Satellite using a virtual environment:


```
python3 -m venv ~/wyoming
~/wyoming/bin/pip3 install --upgrade pip
~/wyoming/bin/pip3 install wyoming-satellite
```

## 🔊 5. Feedback Sounds

The satellite requires local .wav files for "wake" and "done" sounds to ensure zero-latency audio feedback.


```
mkdir -p /home/levente/sounds
cd /home/levente/sounds

# Download official Rhasspy/Wyoming sounds
wget https://github.com/rhasspy/wyoming-satellite/raw/master/sounds/awake.wav -O wake.wav
wget https://github.com/rhasspy/wyoming-satellite/raw/master/sounds/done.wav -O done.wav
```

Note: If your username is not levente, update the paths in the service file accordingly.


## 🚀 6. Auto-Start Service

To make the satellite run automatically on boot, create a systemd service:

```
sudo nano /etc/systemd/system/wyoming-satellite.service
```

```
[Unit]
Description=Wyoming Satellite
After=network-online.target sound.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/home/levente/wyoming/bin/python3 -m wyoming_satellite \
  --name "PiZero2W" \
  --uri tcp://0.0.0.0:10700 \
  --mic-command "arecord -D plughw:CARD=sndrpigooglevoi,DEV=0 -r 16000 -c 1 -f S16_LE -t raw" \
  --snd-command "aplay -D plug:softvol -r 22050 -c 1 -f S16_LE -t raw" \
  --awake-wav "/home/levente/sounds/wake.wav" \
  --done-wav "/home/levente/sounds/done.wav" \
  --debug

[Install]
WantedBy=multi-user.target
```

Copy the service configuration from this repository. It is pre-configured to handle the specific arecord and aplay commands for your I2S hardware.


## 🔧 Calibration & Testing

---


After starting the service, you must calibrate the volume level:


Open the mixer: ```alsamixer -D softvol```

Set the "Master" level to approximately 60-70% (to avoid distortion).


Save the settings permanently: ```sudo alsactl store```


###🚀 7. Activating the Service (Systemd)

After creating the wyoming-satellite.service file, you need to tell Linux to recognize it, enable it for auto-start, and finally start it. 
Run these commands:


```
# 1. Reload the systemd manager configuration to recognize the new service
sudo systemctl daemon-reload
```

```
# 2. Enable the service to start automatically on every boot
sudo systemctl enable wyoming-satellite.service
```

```
# 3. Start the service now
sudo systemctl start wyoming-satellite.service
```

```
# 4. Check the status to see if it's running correctly (Active: active (running))
sudo systemctl status wyoming-satellite.service
```




