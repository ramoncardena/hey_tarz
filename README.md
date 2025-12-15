# Hey Tarz Wake Word
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![ESPHome Ready](https://img.shields.io/badge/ESPHome-ready-blue.svg)](#m5stack-echo-example)

Custom wake-word assets for triggering assistants with the phrase **“hey tarz”** on tiny embedded hardware (ESPHome voice assistant / `micro_wake_word`). The repository ships the quantized TensorFlow Lite Micro model together with its metadata so it can be dropped directly into an ESPHome deployment or re-used as a starting point for further tuning.

## Contents
- `hey_tarz.tflite` – 8‑bit quantized model compiled for TensorFlow Lite Micro runtimes (ESP32, RP2040, etc.).
- `hey_tarz.json` – metadata consumed by ESPHome describing thresholds, tensor arena requirements, and minimum firmware version (`2024.7.0`).
- `hey_tarz.onnx` – full‑precision reference model that can be re-exported or fine-tuned if you need to retrain.

## Using with ESPHome
1. Copy `hey_tarz.tflite` and `hey_tarz.json` to your ESPHome `config/wake_words/hey_tarz/` (or any directory that is synced to the device).
2. Reference the model in your ESPHome YAML:

```yaml
voice_assistant:
  id: va
  microphone: i2s_mic
  speaker: i2s_spk
  wake_word:
    engine: micro_wake_word
    model:
      file: /config/wake_words/hey_tarz/hey_tarz.tflite
    metadata:
      file: /config/wake_words/hey_tarz/hey_tarz.json
    detection_threshold: 0.5  # matches probability_cutoff in metadata
```

Adjust the `detection_threshold` if you need a more/less sensitive detector, but keep it aligned with `probability_cutoff` from the metadata for best results.

## M5Stack Echo Example
The M5Stack Echo ships with an ESP32, on-board I2S MEMS mic, class-D amp, and speaker, so you can run the wake word end-to-end on a single device. Drop the assets into `/config/wake_words/hey_tarz/` and use a configuration similar to this:

```yaml
esphome:
  name: m5stack-echo-tarz
  min_version: 2024.7.0

esp32:
  board: m5stack-core-esp32

i2s_audio:
  - id: bus0
    i2s_lrclk_pin: GPIO25
    i2s_bclk_pin: GPIO26
    i2s_mclk_pin: GPIO0

microphone:
  - platform: i2s_audio
    id: i2s_mic
    i2s_audio_id: bus0
    adc_type: external
    channel: left
    bits_per_sample: 32
    i2s_din_pin: GPIO32

speaker:
  - platform: i2s_audio
    id: i2s_spk
    i2s_audio_id: bus0
    dac_type: external
    mode: mono
    i2s_dout_pin: GPIO22

voice_assistant:
  id: va
  microphone: i2s_mic
  speaker: i2s_spk
  auto_gain: 16dB
  use_wake_word: true
  wake_word:
    engine: micro_wake_word
    model:
      file: /config/wake_words/hey_tarz/hey_tarz.tflite
    metadata:
      file: /config/wake_words/hey_tarz/hey_tarz.json
    detection_threshold: 0.5
```

Tweak the `auto_gain`, `detection_threshold`, or pin assignments if you are running a board revision that wires the codec differently.

## Metadata Highlights
- **Sliding window:** 10 frames with a step size of 10, tuned for low-latency streaming inference.
- **Tensor arena:** Requires ~22 KB (`tensor_arena_size: 22860`), so plan memory accordingly on ESP32-class devices.
- **Language:** Trained on English data for a neutral accent; retrain if you need other languages.

## Regenerating / Retraining
If you retrain the ONNX model:
1. Export to TFLite (`float32`) with standard audio pre-processing.
2. Run post-training quantization for int8/int8 to stay compatible with TensorFlow Lite Micro.
3. Update the metadata JSON (especially arena size and cutoff) before deploying.

Feel free to adapt the model or thresholds to match your acoustic environment and assistant stack.

## Training Notebook
The model was produced in this Colab notebook: [Hey Tarz wake-word training](https://colab.research.google.com/drive/1q1oe2zOyZp7UsB3jJiQ1IFn8z5YfjwEb?usp=sharing#scrollTo=qgaKWIY6WlJ1). It walks through:
- collecting/curating positive and negative clips,
- feature extraction with log-mel MFCCs,
- training an ONNX model with a small CNN,
- exporting to TFLite and quantizing for TensorFlow Lite Micro,
- generating the `hey_tarz.json` metadata (arena sizing, cutoff, sliding window).

Use the notebook as a template if you want to adapt the wake word to another phrase or language—swap the dataset, retrain, and drop the new artifacts back into this repo structure.
