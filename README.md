# Linux-Tinnitus-Filter

A simple filter for tinnitus therapy for Linux systems (tested on Ubuntu). You can implement both notch therapy or peaking therapy.

> [!WARNING]  
> You should definitely see an ear doctor and discuss whether you need filter therapy for tinnitus! Organic damage, stress, psychological and other health problems can also be the cause! In addition, a hearing test is necessary to determine the frequencies or frequency ranges that need to be either filtered out (notch) or amplified (peaking). Your doctor should also decide which therapy would be best for you individually!

The audio output of your Linux system (tested on Ubuntu) filters appropriately configured frequencies to support your tinnitus therapy. It doesn't matter whether you're playing Netflix, Prime Video, YouTube, web radio, Spotify, etc., local audio and video files, or even sounds from video games through your standard speakers. These filters work in real time, amplifying or attenuating the corresponding frequencies. A notch filter can even be used to completely eliminate and filter out a frequency from the sound output.

![Tinnitus Reliever Configurator](https://github.com/Michdo93/test2/blob/main/tinnitus_reliever.png?raw=true)

## Features

- Starts and loads the filter settings after each system startup.
- Multiple peaking and notch filters can be combined.
- Multiple frequency ranges can be filtered simultaneously.
- A simple GUI makes configuration very easy and flexible.
- **Automatic hardware detection**: Select your output device (Bluetooth, USB headset, internal speakers) directly in the GUI.
- **Precision filters (Q factor)**: Determine the quality (width) of your filters. Use high Q values for surgically sharp notches or low values for natural gain curves.
- **Individual gain management**: Control the gain for each peaking filter individually.
- **Built-in clipping protection**: Use the master gain to lower the overall volume before amplifying frequencies – for distortion-free sound.
- **Persistent configuration**: Automatically loads your settings at system startup via Systemd service.
- **Real-time adjustment**: Changes take effect immediately without restarting the system.

## Functionality

### Peaking therapy vs. Notch therapy

#### Method 1: Peaking (Bell) filter

The idea behind this is that tinnitus sounds are caused by hearing impairments, such as after sudden hearing loss, and the whistling sound occurs because the ear can no longer perceive these noises well enough. In this case, the measured frequencies would need to be amplified.

In audio technology, a so-called peaking filter (also known as a bell filter, because the amplification curve looks like a bell) is used for this purpose.

- **Function**: Boosts a selected frequency range in a bell-shaped curve.
- **Purpose**: To compensate for hearing loss (hearing aid function) or to mask tinnitus with a more pleasant background noise in the same pitch.
- **Sound**: Music or ambient noise sounds “brighter” or “more present” in this range.

#### Method 2: Notch (Removing) filter

The underlying idea here is that the nerve pathways are essentially going haywire. This can be caused by stress, but it can also have many other physical and psychological causes. You can think of it as a kind of sensory overload. Ergo, the solution here is to reduce these measured frequencies as much as possible or eliminate them entirely in order to relieve the burden on the auditory system.

- **Function**: Creates an extremely narrow and deep “notch” in the frequency spectrum.
- **Purpose**: The tinnitus frequency is removed from the audio signal to “wean” the brain off it.
- **Sound**: When listening to music, you will hardly notice this filter, as it only removes a tiny fraction of the sound.

#### Why you often need both

In cases of “quite severe” tinnitus, there is often a combination of factors:

1. You have a hearing dip (mild hearing loss) at a certain frequency. Here, the peaking filter is used to compensate for the deficit.
2. At the same time, you want to calm the overactivity of the nerve cells. Here, you use the notch filter directly on the tinnitus frequency.

### Q factor (quality)

The Q factor determines how narrow the filter is.

- **High Q** (e.g., 10.0): Creates an extremely narrow cut. Ideal for notch filters to precisely target the tinnitus frequency without distorting the music.
- **Low Q** (e.g., 1.0): Creates a wide curve. Ideal for peaking filters to gently compensate for hearing loss.

### Master Gain

When you amplify frequencies (peak), the digital signal can “overdrive” (clipping).

- **Solution**: If you set a peak of +6 dB, you should set the master gain to -6 dB. This preserves the dynamics and keeps the sound clean.

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
SETTINGS="$HOME/.tinnitus_settings"

# The first argument of the script is now our master device
REAL_MASTER=$1

if [ -z "$REAL_MASTER" ]; then
    # Fallback if no argument was passed
    REAL_MASTER=$(pactl get-default-sink)
fi

pactl unload-module module-ladspa-sink 2>/dev/null

LAST_SINK=$REAL_MASTER
COUNT=0

while read -r line; do
    [ -z "$line" ] || [[ "$line" == master_gain* ]] && continue
    IFS=';' read -r TYPE FREQ Q GAIN <<< "$line"
    
    CURRENT_SINK="tinnitus_stage_$COUNT"
    BW=$(echo "scale=2; $FREQ / $Q" | bc)
    
    if [ "$TYPE" == "notch" ]; then
        pactl load-module module-ladspa-sink sink_name=$CURRENT_SINK master=$LAST_SINK \
        plugin=notch_iir_1894 label=notch_iir control=$FREQ,$BW
    elif [ "$TYPE" == "peak" ]; then
        pactl load-module module-ladspa-sink sink_name=$CURRENT_SINK master=$LAST_SINK \
        plugin=single_para_1203 label=singlePara control=$GAIN,$FREQ,$BW
    fi
    LAST_SINK=$CURRENT_SINK
    COUNT=$((COUNT+1))
done < "$SETTINGS"

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

# Path to the settings file
SETTINGS_FILE = os.path.expanduser("~/.tinnitus_settings")
# Path to the Bash script
SCRIPT_PATH = os.path.expanduser("~/apply_tinnitus_filter.sh")

class FilterWidget(QWidget):
    """Single filter module in the UI"""
    def __init__(self, f_type="notch", freq=8000, q=4.0, gain=0, active=True, on_remove=None):
        super().__init__()
        self.on_remove = on_remove

        layout = QVBoxLayout()

        # Top row: Active check mark, type selection, delete button
        header_layout = QHBoxLayout()
        self.chk_active = QCheckBox("Active")
        self.chk_active.setChecked(active)
        header_layout.addWidget(self.chk_active)

        self.combo = QComboBox()
        self.combo.addItems(["notch", "peak"])
        self.combo.setCurrentText(f_type)
        header_layout.addWidget(self.combo)

        header_layout.addStretch()

        btn_del = QPushButton("Remove")
        btn_del.setStyleSheet("background-color: #ff4444; color: white; padding: 5px;")
        btn_del.clicked.connect(lambda: self.on_remove(self))
        header_layout.addWidget(btn_del)
        layout.addLayout(header_layout)

        # Frequency slider (100 Hz - 18000 Hz)
        self.lbl_freq = QLabel(f"Frequency: {freq} Hz")
        self.sld_freq = self.create_slider(100, 18000, int(freq))
        self.sld_freq.valueChanged.connect(lambda v: self.lbl_freq.setText(f"Frequency: {v} Hz"))
        layout.addWidget(self.lbl_freq)
        layout.addWidget(self.sld_freq)

        # Q-Factor slider (0.1 to 10.0 -> intern 1 to 100)
        q_val = int(q * 10)
        self.lbl_q = QLabel(f"Q-Factor (Quality): {q}")
        self.sld_q = self.create_slider(1, 100, q_val)
        self.sld_q.valueChanged.connect(lambda v: self.lbl_q.setText(f"Q-Factor (Quality): {v/10.0}"))
        layout.addWidget(self.lbl_q)
        layout.addWidget(self.sld_q)

        # Gain slider (-20 dB bis +20 dB) - Mainly relevant for Peak
        self.lbl_gain = QLabel(f"Gain: {gain} dB")
        self.sld_gain = self.create_slider(-20, 20, int(gain))
        self.sld_gain.valueChanged.connect(lambda v: self.lbl_gain.setText(f"Gain: {v} dB"))
        layout.addWidget(self.lbl_gain)
        layout.addWidget(self.sld_gain)

        self.setLayout(layout)
        self.setStyleSheet("border-bottom: 2px solid #555; padding: 10px; margin-bottom: 5px; background-color: #f9f9f9;")

    def create_slider(self, min_v, max_v, curr_v):
        s = QSlider(Qt.Horizontal)
        s.setRange(min_v, max_v)
        s.setValue(curr_v)
        return s

class TinnitusApp(QWidget):
    def __init__(self):
        super().__init__()
        self.filters = []
        self.initUI()
        self.refresh_devices()
        self.load_settings()

    def initUI(self):
        self.main_layout = QVBoxLayout()

        # Section 1: Geräteauswahl
        dev_group = QVBoxLayout()
        dev_label = QLabel("<b>1. Select output device:</b>")
        dev_group.addWidget(dev_label)

        dev_sub_layout = QHBoxLayout()
        self.combo_devices = QComboBox()
        btn_refresh = QPushButton("🔄")
        btn_refresh.setFixedWidth(40)
        btn_refresh.clicked.connect(self.refresh_devices)
        dev_sub_layout.addWidget(self.combo_devices)
        dev_sub_layout.addWidget(btn_refresh)
        dev_group.addLayout(dev_sub_layout)
        self.main_layout.addLayout(dev_group)

        # Section 2: Master Gain
        self.main_layout.addSpacing(10)
        self.lbl_mg = QLabel("<b>2. Master Gain (Security): 0 dB</b>")
        self.sld_mg = QSlider(Qt.Horizontal)
        self.sld_mg.setRange(-30, 0) # Nur absenken um Clipping zu vermeiden
        self.sld_mg.setValue(0)
        self.sld_mg.valueChanged.connect(lambda v: self.lbl_mg.setText(f"<b>2. Master Gain (Security): {v} dB</b>"))
        self.main_layout.addWidget(self.lbl_mg)
        self.main_layout.addWidget(self.sld_mg)

        # Section 3: Filter List (Scrollbar)
        self.main_layout.addSpacing(10)
        self.main_layout.addWidget(QLabel("<b>3. Filter Chain:</b>"))
        scroll = QScrollArea()
        self.container = QWidget()
        self.scroll_layout = QVBoxLayout(self.container)
        self.scroll_layout.setAlignment(Qt.AlignTop)
        scroll.setWidget(self.container)
        scroll.setWidgetResizable(True)
        self.main_layout.addWidget(scroll)

        # Section 4: Buttons
        btn_add = QPushButton("+ Add new filter")
        btn_add.setFixedHeight(40)
        btn_add.clicked.connect(lambda: self.add_filter_ui())
        self.main_layout.addWidget(btn_add)

        btn_apply = QPushButton("SAVE && APPLY")
        btn_apply.setFixedHeight(60)
        btn_apply.setStyleSheet("background-color: #4CAF50; color: white; font-weight: bold; font-size: 14px;")
        btn_apply.clicked.connect(self.save_and_apply)
        self.main_layout.addWidget(btn_apply)

        self.setLayout(self.main_layout)
        self.setWindowTitle("Tinnitus Reliever Configurator v5")
        self.resize(600, 800)
        self.show()

    def refresh_devices(self):
        """Reads physical sinks from PipeWire/PulseAudio"""
        self.combo_devices.clear()
        try:
            output = subprocess.check_output(["pactl", "list", "sinks", "short"]).decode()
            for line in output.splitlines():
                parts = line.split()
                if len(parts) > 1:
                    name = parts[1]
                    # Ignore filter stages, show only real hardware
                    if "tinnitus_stage" not in name:
                        self.combo_devices.addItem(name)
        except Exception as e:
            print(f"Error loading devices: {e}")

    def load_settings(self):
        """Loading the .tinnitus_settings file"""
        if os.path.exists(SETTINGS_FILE):
            try:
                with open(SETTINGS_FILE, "r") as f:
                    for line in f:
                        parts = line.strip().split(';')
                        if parts[0] == "master_gain":
                            self.sld_mg.setValue(int(parts[1]))
                        elif len(parts) >= 3:
                            f_type = parts[0]
                            freq = int(parts[1])
                            q = float(parts[2])
                            gain = int(parts[3]) if len(parts) > 3 else 0
                            self.add_filter_ui(f_type, freq, q, gain)
            except Exception as e:
                print(f"Error loading settings: {e}")

    def add_filter_ui(self, f_type="notch", freq=8000, q=4.0, gain=0):
        fw = FilterWidget(f_type, freq, q, gain, on_remove=self.remove_filter_ui)
        self.filters.append(fw)
        self.scroll_layout.addWidget(fw)

    def remove_filter_ui(self, widget):
        self.filters.remove(widget)
        widget.deleteLater()

    def save_and_apply(self):
        """Saves settings and executes the Bash script"""
        selected_device = self.combo_devices.currentText()
        if not selected_device:
            print("No output device selected!")
            return

        # 1. Save
        with open(SETTINGS_FILE, "w") as f:
            f.write(f"master_gain;{self.sld_mg.value()}\n")
            for fw in self.filters:
                if fw.chk_active.isChecked():
                    f.write(f"{fw.combo.currentText()};{fw.sld_freq.value()};{fw.sld_q.value()/10.0};{fw.sld_gain.value()}\n")

        # 2. Reset filter modules in the system
        subprocess.run(["pactl", "unload-module", "module-ladspa-sink"], capture_output=True)

        # 3. Adjust master volume for clipping protection (optional via pactl)
        # Here we call the script with the master device
        if os.path.exists(SCRIPT_PATH):
            subprocess.run([SCRIPT_PATH, selected_device])
            print(f"Filter applied to {selected_device}.")
        else:
            print(f"Error: {SCRIPT_PATH} not found!")

if __name__ == '__main__':
    app = QApplication(sys.argv)
    ex = TinnitusApp()
    sys.exit(app.exec_())
```

### Desktop file

At least we can run a `create_launcher.py` file to create our `.desktop` file. The desktop file will add the application to your Desktop and to your Dashboard. If you want also to your Favorites.

#### The creation script (create_launcher.py)

Copy this code and run it once:

```
import os

# paths definition
home = os.path.expanduser("~")
launcher_path = os.path.join(home, ".local/share/applications/tinnitus_reliever.desktop")
script_path = os.path.join(home, "tinnitus_reliver.py")
python_path = "/usr/bin/python3"

# desktop file content
desktop_file_content = f"""[Desktop Entry]
Version=1.0
Type=Application
Name=Tinnitus Reliever
Comment=Configuring the tinnitus filter chain
Exec={python_path} {script_path}
Icon=audio-card
Terminal=false
Categories=AudioVideo;Audio;Settings;
Keywords=Tinnitus;Filter;Audio;Notch;Peak;
"""

# write file
try:
    with open(launcher_path, "w") as f:
        f.write(desktop_file_content)
    
    # make executable
    os.chmod(launcher_path, 0o755)
    print(f"Success! The launcher was created at: {launcher_path}")
    print("You can now find the app in your Start menu under 'Tinnitus Reliever'.")
except Exception as e:
    print(f"Error: {e}")
```

#### What the script does

- **Path**: It saves the file in ~/.local/share/applications/. This is the default location for user-defined launchers in Ubuntu.
- **Icon**: I chose audio-card as the default icon. This is a system icon that looks like a sound card.
- **Integration**: The app will immediately appear in your search when you press the Super key (Windows key) and search for “Tinnitus.”

## Usage

1. Start the Python script: python3 tinnitus_app.py.
2. Add as many filters as you like (e.g., a notch at your tinnitus frequency and a peak at your hearing threshold).
3. Click `Save & Apply`.
4. The last “Tinnitus Stage” device will now be automatically selected in the Ubuntu sound settings.
