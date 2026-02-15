<h1 align="center">🎙️ DIY Wyoming Satellite - Raspberry Pi Zero 2W </h1>

<p align="center">
  <img src="assets/logo2.png" alt="Logo" width="300"/>
</p>


### This repository provides a step-by-step guide and all necessary configuration files to build a high-performance, standalone voice satellite for Home Assistant using the Wyoming protocol.
---

# 🛠️ Hardware Stack

---

- ** Computing: Raspberry Pi Zero 2W


- ** Audio Output (DAC): MAX98357A (I2S Mono Amplifier)


Audio Input (Microphone): INMP441 (I2S Omnidirectional)


Interface: I2S Digital Audio


# 📐 Hardware Wiring (Pinout)

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


# 💾 Software Installation & Configuration

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


Save the settings permanently: ```sudo alsactl store``` or ```alsamixer```


## 🚀 7. Activating the Service (Systemd)

---

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

# 🔄 Managing the Service

---

If you make changes to asound.conf or the service arguments (like changing the volume or the name of the satellite), use these commands:


To apply changes: ```sudo systemctl restart wyoming-satellite.service```


To stop it: ```sudo systemctl stop wyoming-satellite.service```


To see live logs (Troubleshooting): ```journalctl -u wyoming-satellite.service -f```


# 🏠 8. Integrating with Home Assistant

---

Once the service is running, it's time to connect it to your Home Assistant instance:


Discovery: Home Assistant should automatically discover the satellite via the Wyoming integration. 
Go to Settings > Devices & Services.


Manual Add: If it doesn't appear, click Add Integration, search for Wyoming, and enter the IP address of your Raspberry Pi and port 10700.


Voice Assistant Setup: * Go to Settings > Voice Assistants.


Create a new pipeline or edit the existing one.


Select your preferred STT (Speech-to-Text), TTS (Text-to-Speech), and Conversation Agent (e.g., Assist or Gemini).


# 🛠️ Troubleshooting (Common Issues)

---

No Sound or Microphone Not Working?

Check if the cards are detected by the system:


```
aplay -l   # List playback devices
arecord -l # List recording devices
```

If you don't see sndrpigooglevoi, check your wiring and ensure config.txt is correct.


"Device or resource busy"

This usually means another process is using the audio card. Ensure you don't have PulseAudio or another media player running in the background.


Audio Clipping or Distortion

If the feedback sounds are too loud or distorted, lower the "Master" volume in alsamixer -D softvol below 70%.


## 📺 Project Showcase (Video Demos)

---

See the Wyoming Satellite in action! These short clips demonstrate the response time and integration with Home Assistant:


<p align="center">
<a href="https://youtube.com/shorts/ag_uql1x-wI">
<img src="https://img.youtube.com/vi/ag_uql1x-wI/0.jpg" alt="Demo 1" width="250" style="margin: 10px;"/>
</a>

  
<a href="https://youtube.com/shorts/bD9Pid_7iNs">
<img src="https://img.youtube.com/vi/bD9Pid_7iNs/0.jpg" alt="Demo 2" width="250" style="margin: 10px;"/>
</a>


<a href="https://youtube.com/shorts/CC92k0u0qHU">
<img src="https://img.youtube.com/vi/CC92k0u0qHU/0.jpg" alt="Demo 3" width="250" style="margin: 10px;"/>
</a>
</p>


# 📄 License

---

This project is open-source. Feel free to use and modify it for your smart home setup!

