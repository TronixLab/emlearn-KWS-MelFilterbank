# emlearn-KWS-MelFilterbank

TinyML keyword spotting for **Jarvis-style wake-word detection** on the **Arduino Nano 33 BLE Sense**. This repository documents a workflow for building a compact audio classifier that can react to phrases centered on **"Jarvis"**, including examples such as **"Jarvis"**, **"Hey Jarvis"**, and **"Hello Jarvis"**.

The project combines:

- **Mel filterbank feature extraction** for speech audio
- **Small neural network training** in Python/Jupyter
- **Embedded deployment** for real-time inference on an nRF52840-based Arduino board
- **Prebuilt firmware artifacts** that can be flashed with the Arduino CLI

## What is in this repository

```text
KWS_training.ipynb                                 End-to-end training and experimentation notebook
mel_filterbank.h                                   Exported sparse mel filterbank coefficients
arduino.mbed_nano.nano33ble/                       Prebuilt Arduino Nano 33 BLE Sense firmware
arduino-nano-33-ble-sense-ei-firmware.zip          Firmware bundle with helper flash scripts
ei-ched-husay-kws-demo-arduino-1.0.3-impulse-#1.zip Edge Impulse exported inference library
```

## Project goal

The repository demonstrates a **keyword spotting (KWS)** pipeline for always-on audio detection with TinyML constraints. Instead of full speech transcription, the model focuses on deciding whether a short audio window contains the target wake word or not.

In practice, the classification setup in this repository is organized around three classes:

- **Jarvis** - target wake-word audio
- **Negative** - non-target spoken words
- **Noise** - background or synthetic noise

This setup is useful for:

- wake-word front ends
- offline voice interfaces
- low-power embedded assistants
- speech-triggered automation prototypes

## Technical overview

### Signal pipeline

The notebook and exported artifacts are aligned around a compact speech feature pipeline:

- **Sample rate:** 16 kHz
- **FFT length:** 256
- **Mel bands:** 40
- **Frequency range:** 20 Hz to 8 kHz
- **Frame length:** 20 ms
- **Frame stride:** 10 ms
- **Model window:** 1 second
- **Window slicing:** 4 slices per model window in the exported Edge Impulse artifact

The file [`mel_filterbank.h`](./mel_filterbank.h) contains the exported sparse mel filterbank lookup tables used to compress FFT bins into mel features suitable for embedded inference.

### Training workflow

The notebook [`KWS_training.ipynb`](./KWS_training.ipynb) walks through the end-to-end method:

1. Load audio archives from a local `dataset/` directory
2. Build a labeled dataset for `jarvis`, `negative`, and `noise`
3. Visualize waveforms and mel spectrograms
4. Generate a sparse mel filterbank
5. Apply sliding 1-second audio windows
6. Extract log-mel features
7. Flatten features for classification
8. Train a small dense neural network with Keras
9. Evaluate the model on held-out data
10. Use the exported model for embedded deployment

The current notebook references:

- **5,000 audio files**
- **3 classes**
- **flattened feature size of 3960 per example**

### Model shape

The notebook trains a compact fully connected classifier with:

- input size matching the flattened mel features
- hidden layers of **64 -> 32 -> 16 -> 8**
- softmax output for 3-class classification

This design keeps the model small enough for microcontroller deployment while preserving enough capacity for wake-word discrimination.

## Hardware target

The repository targets the **Arduino Nano 33 BLE Sense**, which is a good fit for TinyML audio applications because it provides:

- **nRF52840 MCU**
- onboard **microphone**
- enough flash and RAM for compact inference workloads
- USB programming support

## Firmware artifacts

Three firmware delivery paths are included:

### 1. Direct binary in the repository

Prebuilt binary:

- [`arduino.mbed_nano.nano33ble/nano_ble33_sense_microphone_continuous.ino.bin`](./arduino.mbed_nano.nano33ble/nano_ble33_sense_microphone_continuous.ino.bin)

Supporting note:

- [`arduino.mbed_nano.nano33ble/README.txt`](./arduino.mbed_nano.nano33ble/README.txt)

### 2. Zipped firmware package

The archive [`arduino-nano-33-ble-sense-ei-firmware.zip`](./arduino-nano-33-ble-sense-ei-firmware.zip) contains:

- `arduino-nano-33-ble-sense.ino.bin`
- `flash_linux.sh`
- `flash_mac.command`
- `flash_windows.bat`

### 3. Full exported inference SDK

The archive [`ei-ched-husay-kws-demo-arduino-1.0.3-impulse-#1.zip`](./ei-ched-husay-kws-demo-arduino-1.0.3-impulse-#1.zip) contains the generated Edge Impulse inference library, including metadata showing:

- **16000 raw audio samples per model window**
- **3960 neural-network input features**
- output labels: **Jarvis**, **Negative**, **Noise**

## Tutorial: understand and use the project

### Prerequisites

You need:

- an **Arduino Nano 33 BLE Sense**
- a **USB cable** with data support
- a desktop system running **Linux**, **macOS**, or **Windows**
- optional: **Jupyter Notebook** and Python tools if you want to inspect or retrain the model

## Tutorial part 1: inspect the training workflow

If you want to understand how the model is built:

1. Open `KWS_training.ipynb`
2. Review the dataset loading section
3. Inspect waveform and mel spectrogram visualizations
4. Compare the dense mel filterbank with the sparse exported representation
5. Follow the sliding-window feature extraction process
6. Review the Keras classifier architecture and evaluation cells

This is the best starting point if you want to:

- adapt the project to a different wake word
- expand the negative-word set
- tune feature extraction parameters
- retrain the model before redeploying

## Tutorial part 2: install Arduino CLI

The easiest way to flash the included firmware is with the **Arduino CLI**.

### Verify installation

After installing, confirm the CLI is available:

```bash
arduino-cli version
```

### Linux

Install with the official script:

```bash
curl -fsSL https://raw.githubusercontent.com/arduino/arduino-cli/master/install.sh | sh
sudo mv bin/arduino-cli /usr/local/bin/
arduino-cli version
```

### macOS

If you use Homebrew:

```bash
brew update
brew install arduino-cli
arduino-cli version
```

### Windows

If you use Winget:

```powershell
winget install ArduinoSA.CLI
arduino-cli version
```

If `arduino-cli` is not found after installation, reopen the terminal so the updated `PATH` is loaded.

## Tutorial part 3: prepare the Arduino CLI for Nano 33 BLE Sense

Initialize the CLI and install the board core:

```bash
arduino-cli config init
arduino-cli core update-index
arduino-cli core install arduino:mbed_nano
```

Then confirm the board is detected:

```bash
arduino-cli board list
```

You should see a line similar to:

```text
Port   Protocol Type         Board Name           FQBN
COM6   serial   Serial Port  Arduino Nano 33 BLE arduino:mbed_nano:nano33ble
```

On Linux or macOS, the port will usually look like `/dev/ttyACM0`, `/dev/ttyUSB0`, or `/dev/cu.usbmodem*`.

## Tutorial part 4: flash the firmware

### Option A: flash the binary already stored in this repository

From the repository root:

```bash
arduino-cli upload \
  -p <PORT> \
  --fqbn arduino:mbed_nano:nano33ble \
  -i ./arduino.mbed_nano.nano33ble/nano_ble33_sense_microphone_continuous.ino.bin
```

Example on Windows from the included notes:

```bash
arduino-cli upload -p COM6 --fqbn arduino:mbed_nano:nano33ble -i ./arduino.mbed_nano.nano33ble/nano_ble33_sense_microphone_continuous.ino.bin
```

### Option B: use the zipped firmware package

Unzip `arduino-nano-33-ble-sense-ei-firmware.zip`, then upload the included binary or run the platform-specific helper script inside the archive.

## Troubleshooting flashing

- If the board does not appear in `arduino-cli board list`, reconnect the USB cable.
- If upload fails, press the **reset button twice** to enter bootloader mode, then retry.
- Make sure the installed board core is `arduino:mbed_nano`.
- Verify that the selected FQBN is exactly `arduino:mbed_nano:nano33ble`.
- Close any serial monitor application that may be locking the port.

## How to extend the project

Common next steps are:

- add more recordings for "Jarvis", "Hey Jarvis", and "Hello Jarvis"
- collect device-specific background noise
- balance the negative/noise classes
- retrain the notebook with updated data
- regenerate embedded artifacts for deployment

## Notes

- This repository currently focuses on **documentation, notebook-based experimentation, exported mel coefficients, and prebuilt firmware artifacts**.
- The training workflow runs on a host machine, not on the microcontroller.
- The included firmware is intended for the **Arduino Nano 33 BLE Sense** target.

## License and third-party artifacts

Please review the headers and bundled export artifacts for third-party licensing terms, especially the Edge Impulse generated package included in this repository.
