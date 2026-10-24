# Multi-parameter Vital Signs Monitor

A low-cost prototype patient monitor: an ESP32 acquires ECG, photoplethysmography (red/IR), cuff pressure and body temperature, and a Python desktop application displays the signals in real time, computes heart rate and SpO₂, and raises clinical alarms.

> **Context:** Final project for *Measurement Systems and Medical Transducers* (Sistemas de Medición y Transductores Médicos), Biomedical Systems Engineering, Universidad Nacional Autónoma de México (UNAM), May 2024.
>
> **Not a medical device.** This is an educational prototype. None of the measurements have been calibrated or validated against reference equipment.

<img width="674" height="396" alt="image" src="https://github.com/user-attachments/assets/317bfc15-3f71-4cea-8a2b-26197a88e449" />

*Monitor view: real-time ECG, cuff pressure and oximetry plots (left) and numeric displays with status labels (right). Screenshot taken during interface testing.*

---

## System overview

```
 Sensors                        ESP32 firmware                Python desktop app (PyQt5)
 ─────────────────────          ───────────────────           ─────────────────────────────────
 AD8232 ECG front end ─┐                                      • COM port / baud rate selection
                       ├─ ADS1115 16-bit ADC ─┐               • Real-time plots (pyqtgraph, threaded reader)
 Cuff pressure sensor ─┘   (250 samples/s)    │  USB serial   • Real-time filters (Butterworth, 60 Hz notch)
 MAX30102 (red / IR) ──────── I²C ────────────┼─────────────▶ • ECG R-peak detection → heart rate
 MLX90614 (IR temp) ───────── I²C ────────────┘  Bluetooth    • SpO₂ (ratio of ratios)
                                                               • Clinical alarms (visual + audio)
                                                               • CSV recording + offline analysis
```

## Hardware

| Component | Role |
|---|---|
| ESP32 development board | Acquisition and streaming over USB serial and Bluetooth |
| AD8232 | ECG analog front end (amplification and filtering) |
| ADS1115 | 16-bit ADC for the ECG and cuff-pressure channels, 250 samples/s |
| Cuff pressure sensor | Analog pressure transducer, converted with a linear calibration (64.64 mmHg/V) |
| MAX30102 | Red and infrared PPG for pulse oximetry |
| MLX90614 | Non-contact infrared temperature |

## Software

### Firmware (`firmware/vitalSignMonitor`)
Reads the five channels in each loop and streams them as comma-separated lines over USB serial and Bluetooth:

```
ECG (V), cuff pressure (mmHg), IR, red, temperature (°C)
```

### Desktop application (`app/`)
- **Settings tab:** serial port and baud rate, file name, variables to display, split plots, real-time filter options
- **Monitor tab:** live ECG, pressure and oximetry plots; numeric displays for elapsed time, ECG, heart rate, SpO₂, pressure and temperature; Start, Stop, Save, Analysis, Close port and Restart controls
- **Signal processing**
  - Real-time zero-phase Butterworth filters (high-pass 0.3 Hz, low-pass 5 Hz) on selected channels
  - ECG: 60 Hz notch filter → R-peak detection → RR intervals → heart rate
  - SpO₂: normalized AC components of red and IR → ratio R → `SpO₂ = 110 − 25·R`
- **Alarms:** each variable is classified into clinical ranges and shown with a status label; an optional sound plays when a value is abnormal

| Variable | Ranges used |
|---|---|
| SpO₂ | Normal ≥ 95 %; mild hypoxemia 90–95 %; moderate 75–90 %; severe < 75 % |
| Heart rate | Bradycardia < 60 BPM; normal 60–100; tachycardia > 100 |
| Temperature | Nine categories, from very deep hypothermia (< 17 °C) to hyperpyrexia (> 41.5 °C) |
| Blood pressure | Normal < 120/80 mmHg through grade 3 hypertension (≥ 180/110) |

- **Recording:** each session is saved as a time-stamped CSV (`time, value1…value5`) plus a PNG of the plots

![Settings tab](docs/images/gui-settings.png)
*Settings tab: serial configuration, display options and filters.*

## Getting started

**Firmware:** open `firmware/vitalSignMonitor/vitalSignMonitor.ino` in the Arduino IDE with the ESP32 board package and these libraries: Adafruit ADS1X15, Adafruit MLX90614, SparkFun MAX3010x. Upload, then connect the board over USB.

**Desktop application** (Python 3.12 or newer):
```bash
pip install -r requirements.txt
cd app
python main.py
```
Select the COM port and a baud rate of **115200**, then press Start. For an audible alert, place any short audio file at `resources/alert.mp3` and tick *SoundON*.

To edit the interface, modify `app/ui/vitalSignInterface.ui` in Qt Designer and regenerate it with `pyuic5 -x ui/vitalSignInterface.ui -o mainwindow2.py`.

## Known limitations

- **Blood pressure is experimental.** The cuff pressure is acquired and plotted, but the offline oscillometric estimation of systolic and diastolic pressure is not validated, and it did not converge on the recordings from the test sessions. When it fails, the application now reports *"measurement failed"* instead of a value.
- **No calibration.** SpO₂ uses a generic empirical equation, and none of the measurements were compared with reference devices.
- **Real-time filtering** re-filters a sliding buffer for every new sample, which is computationally heavy; a streaming (stateful) filter would scale better.
- **Interface language** is mixed (Spanish labels, English code).

## Notes on this version

Cleaned for publication in 2026 from the original course repository (2024):
- **Removed a placeholder in the blood-pressure analysis** that returned random values when no estimate could be computed; failures are now reported as such
- Fixed an undefined variable that crashed the analysis, the mean-arterial-pressure formula, and the alarm-sound call
- Replaced hard-coded file paths with paths relative to the repository; the alert sound is now optional
- Removed earlier interface versions and unused files
- Personal recordings from the test sessions are not included

## Team

Developed by **Ignacio Romero Fragoso** and **Gerardo Rodríguez Cárdenas** (firmware, desktop application and analysis scripts), with **Ana Isabel Fernández Díaz**, **Gustavo Maximilian Jaime Rodríguez** and **Daniela Ameyalli Valadez José** (documentation and research).

Biomedical Instrumentation Academy, Faculty of Engineering, UNAM.
