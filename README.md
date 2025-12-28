# Linux-Tinnitus-Filter

A simple filter for tinnitus therapy for Linux systems (tested on Ubuntu). You can implement both notch therapy or peaking therapy.

> [!WARNING]  
> You should definitely see an ear doctor and discuss whether you need filter therapy for tinnitus! Organic damage, stress, psychological and other health problems can also be the cause! In addition, a hearing test is necessary to determine the frequencies or frequency ranges that need to be either filtered out (notch) or amplified (peaking). Your doctor should also decide which therapy would be best for you individually!

The audio output of your Linux system (tested on Ubuntu) filters appropriately configured frequencies to support your tinnitus therapy. It doesn't matter whether you're playing Netflix, Prime Video, YouTube, web radio, Spotify, etc., local audio and video files, or even sounds from video games through your standard speakers. These filters work in real time, amplifying or attenuating the corresponding frequencies. A notch filter can even be used to completely eliminate and filter out a frequency from the sound output.

![Tinnitus Reliever Configurator](https://github.com/Michdo93/test2/blob/main/tinnitus_reliever.png?raw=true)

## Features

- Starts and loads the filter settings after each system startup.
- Filters can be adjusted and configured at runtime.
- Multiple peaking and notch filters can be combined.
- Multiple frequency ranges can be filtered simultaneously.
- A simple GUI makes configuration very easy and flexible.

## Peaking therapy vs. Notch therapy

### Method 1: Peaking (Bell) filter

The idea behind this is that tinnitus sounds are caused by hearing impairments, such as after sudden hearing loss, and the whistling sound occurs because the ear can no longer perceive these noises well enough. In this case, the measured frequencies would need to be amplified.

In audio technology, a so-called peaking filter (also known as a bell filter, because the amplification curve looks like a bell) is used for this purpose.

- **Function**: Boosts a selected frequency range in a bell-shaped curve.
- **Purpose**: To compensate for hearing loss (hearing aid function) or to mask tinnitus with a more pleasant background noise in the same pitch.
- **Sound**: Music or ambient noise sounds “brighter” or “more present” in this range.

### Method 2: Notch (Removing) filter

The underlying idea here is that the nerve pathways are essentially going haywire. This can be caused by stress, but it can also have many other physical and psychological causes. You can think of it as a kind of sensory overload. Ergo, the solution here is to reduce these measured frequencies as much as possible or eliminate them entirely in order to relieve the burden on the auditory system.

- **Function**: Creates an extremely narrow and deep “notch” in the frequency spectrum.
- **Purpose**: The tinnitus frequency is removed from the audio signal to “wean” the brain off it.
- **Sound**: When listening to music, you will hardly notice this filter, as it only removes a tiny fraction of the sound.

### Why you often need both

In cases of “quite severe” tinnitus, there is often a combination of factors:

1. You have a hearing dip (mild hearing loss) at a certain frequency. Here, the peaking filter is used to compensate for the deficit.
2. At the same time, you want to calm the overactivity of the nerve cells. Here, you use the notch filter directly on the tinnitus frequency.

## Setup and configuration

### Install dependencies

First, we need the audio plugins (LADSPA) and the Python environment for the graphical user interface, as well as other dependencies.

```
sudo apt update
sudo apt install swh-plugins python3 python3-dev python3-pip python3-pyqt5 pulseaudio-utils pipewire-audio-client-libraries -y
```

**Why we use pactl (even though you have PipeWire)**

You may be wondering, “If I have PipeWire, why am I using PulseAudio commands?”


- Stability: The native PipeWire commands (`pw-cli`, `pw-load-module`) are currently still undergoing frequent syntax changes (as we saw earlier).
- Compatibility: `pactl` has been the standard for over 15 years. PipeWire has a perfect translation layer built in for it. It is the safest way to avoid crashes in a VMware environment.

### The core: The startup script

This script dynamically builds the filter chain at system startup or when changes are made.

1. Create the file: nano `~/apply_tinnitus_filter.sh`
2. Insert this code:

```
#!/bin/bash
# Short pause to allow the audio system to be ready
sleep 5

SETTINGS="$HOME/.tinnitus_settings"
# If no settings exist, set default values
if [ ! -f "$SETTINGS" ]; then
    echo "notch;8000;50" > "$SETTINGS"
fi

# Discharge existing filter modules for safety reasons
pactl unload-module module-ladspa-sink 2>/dev/null

# The starting point is your physical sound card.
LAST_SINK="alsa_output.pci-0000_02_02.0.analog-stereo"
COUNT=0

# Build the filter chain line by line
while read -r line; do
    [ -z "$line" ] && continue
    TYPE=$(echo $line | cut -d';' -f1)
    FREQ=$(echo $line | cut -d';' -f2)
    VAL=$(echo $line | cut -d';' -f3)
    
    CURRENT_SINK="tinnitus_stage_$COUNT"
    
    if [ "$TYPE" == "notch" ]; then
        pactl load-module module-ladspa-sink sink_name=$CURRENT_SINK master=$LAST_SINK \
        plugin=notch_iir_1894 label=notch_iir control=$FREQ,$VAL
    elif [ "$TYPE" == "peak" ]; then
        pactl load-module module-ladspa-sink sink_name=$CURRENT_SINK master=$LAST_SINK \
        plugin=single_para_1203 label=singlePara control=$VAL,$FREQ,2.0
    fi
    
    LAST_SINK=$CURRENT_SINK
    COUNT=$((COUNT+1))
done < "$SETTINGS"

# Set the last filter in the chain as the system default
pactl set-default-sink $LAST_SINK
```

3. Make file executable: `make chmod +x ~/apply_tinnitus_filter.sh`

### Automatic system startup (service)

To ensure that the filter is loaded during bootup.

1. Create the service folder: `mkdir -p ~/.config/systemd/user/`
2. Create the file: `nano ~/.config/systemd/user/tinnitus-filter-init.service`
3. Insert content:

```
[Unit]
Description=Initilisation Tinnitus Filter
After=pipewire.service

[Service]
Type=oneshot
ExecStart=%h/apply_tinnitus_filter.sh
RemainAfterExit=yes

[Install]
WantedBy=default.target
```

4. Activation

```
systemctl --user daemon-reload
systemctl --user enable tinnitus-filter-init.service
```

### The configuration program (Python GUI)

Save this code as `tinnitus_reliver_config.py`. This program loads your old values when it starts and allows live changes.

```
import sys
import subprocess
import os
from PyQt5.QtWidgets import (QApplication, QWidget, QVBoxLayout, QHBoxLayout, 
                             QSlider, QLabel, QPushButton, QComboBox, QCheckBox, QScrollArea)
from PyQt5.QtCore import Qt

SETTINGS_FILE = os.path.user_home = os.path.expanduser("~/.tinnitus_settings")

class FilterWidget(QWidget):
    def __init__(self, f_type="notch", freq=8000, val=50, active=True, on_remove=None):
        super().__init__()
        self.on_remove = on_remove
        
        layout = QVBoxLayout()
        header_layout = QHBoxLayout()
        
        self.chk_active = QCheckBox("Active")
        self.chk_active.setChecked(active)
        header_layout.addWidget(self.chk_active)

        self.combo = QComboBox()
        self.combo.addItems(["notch", "peak"])
        self.combo.setCurrentText(f_type)
        header_layout.addWidget(self.combo)

        btn_del = QPushButton("Remove")
        btn_del.setStyleSheet("background-color: #ff4444; color: white;")
        btn_del.clicked.connect(lambda: self.on_remove(self))
        header_layout.addWidget(btn_del)
        
        layout.addLayout(header_layout)

        # Frequenz
        freq_layout = QHBoxLayout()
        self.lbl_freq = QLabel(f"Frequency: {freq} Hz")
        self.sld_freq = QSlider(Qt.Horizontal)
        self.sld_freq.setRange(100, 18000)
        self.sld_freq.setValue(int(freq))
        self.sld_freq.valueChanged.connect(lambda v: self.lbl_freq.setText(f"Frequency: {v} Hz"))
        freq_layout.addWidget(self.sld_freq)
        freq_layout.addWidget(self.lbl_freq)
        layout.addLayout(freq_layout)

        # Intensität
        val_layout = QHBoxLayout()
        self.lbl_val = QLabel(f"Gain factor: {val}")
        self.sld_val = QSlider(Qt.Horizontal)
        self.sld_val.setRange(1, 100)
        self.sld_val.setValue(int(val))
        self.sld_val.valueChanged.connect(lambda v: self.lbl_val.setText(f"Gain factor: {v}"))
        val_layout.addWidget(self.sld_val)
        val_layout.addWidget(self.lbl_val)
        layout.addLayout(val_layout)

        self.setLayout(layout)
        self.setStyleSheet("border-bottom: 1px solid #ccc; padding: 10px; margin-bottom: 5px;")

class TinnitusApp(QWidget):
    def __init__(self):
        super().__init__()
        self.filters = []
        self.initUI()
        self.load_settings() # Load existing settings at startup

    def initUI(self):
        self.main_layout = QVBoxLayout()
        
        scroll = QScrollArea()
        self.container = QWidget()
        self.scroll_layout = QVBoxLayout(self.container)
        self.scroll_layout.setAlignment(Qt.AlignTop)
        scroll.setWidget(self.container)
        scroll.setWidgetResizable(True)
        self.main_layout.addWidget(scroll)

        btn_add = QPushButton("+ Add new filter")
        btn_add.clicked.connect(lambda: self.add_filter_ui())
        self.main_layout.addWidget(btn_add)

        btn_apply = QPushButton("SAVE & APPLY")
        btn_apply.setFixedHeight(50)
        btn_apply.setStyleSheet("background-color: #4CAF50; color: white; font-weight: bold;")
        btn_apply.clicked.connect(self.save_and_apply)
        self.main_layout.addWidget(btn_apply)

        self.setLayout(self.main_layout)
        self.setWindowTitle("Tinnitus Reliever Configurator v3")
        self.resize(500, 600)
        self.show()

    def load_settings(self):
        """Reads the .tinnitus_settings file and builds the UI"""
        if os.path.exists(SETTINGS_FILE):
            try:
                with open(SETTINGS_FILE, "r") as f:
                    for line in f:
                        if ';' in line:
                            f_type, freq, val = line.strip().split(';')
                            self.add_filter_ui(f_type, int(freq), int(val))
            except Exception as e:
                print(f"Error loading settings: {e}")

    def add_filter_ui(self, f_type="notch", freq=8000, val=50):
        fw = FilterWidget(f_type, freq, val, on_remove=self.remove_filter_ui)
        self.filters.append(fw)
        self.scroll_layout.addWidget(fw)

    def remove_filter_ui(self, widget):
        self.filters.remove(widget)
        widget.deleteLater()

    def save_and_apply(self):
        # 1. Save (overwrites the old file)
        with open(SETTINGS_FILE, "w") as f:
            for fw in self.filters:
                if fw.chk_active.isChecked():
                    f.write(f"{fw.combo.currentText()};{fw.sld_freq.value()};{fw.sld_val.value()}\n")
        
        # 2. Delete existing filters in the system
        subprocess.run(["pactl", "unload-module", "module-ladspa-sink"], capture_output=True)
        
        # 3. Rebuild chain via start script
        subprocess.run([os.path.expanduser("~/apply_tinnitus_filter.sh")])
        print("Settings saved and audio engine restarted.")

if __name__ == '__main__':
    app = QApplication(sys.argv)
    ex = TinnitusApp()
    sys.exit(app.exec_())
```

## Usage

1. Start the Python script: python3 tinnitus_app.py.
2. Add as many filters as you like (e.g., a notch at your tinnitus frequency and a peak at your hearing threshold).
3. Click `Save & Apply`.
4. The last “Tinnitus Stage” device will now be automatically selected in the Ubuntu sound settings.
