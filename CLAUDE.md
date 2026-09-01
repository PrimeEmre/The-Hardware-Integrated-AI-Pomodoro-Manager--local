# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A two-piece hardware/software Pomodoro timer: an Arduino sketch ([hardware.ino](hardware.ino)) drives a 4-digit
7-segment countdown display and physical switches/LEDs, and a Python script ([main.py](main.py)) sits on the host
machine listening over serial for signals from the Arduino, kicks off a cloud-hosted CrewAI crew, and speaks the
result back with local TTS.

## Running

- Flash [hardware.ino](hardware.ino) to the Arduino with the Arduino IDE / `arduino-cli` (no build step in this repo).
- Run the host script: `python main.py`
- Config lives in `.env` (not committed) with these keys: `ARDUINO_PORT`, `CREWAI_CREW_URL`, `CREWAI_CREW_TOKEN`,
  `OMNIVOICE_VOICE`. `ARDUINO_PORT` must match the Arduino's actual serial port (e.g. `COM3` on Windows,
  `/dev/ttyACM0` on Linux).
- There is no requirements.txt; dependencies observed in [main.py](main.py) are `pyserial`, `python-dotenv`,
  `requests`, `kittentts`, `soundfile` (plus stdlib `winsound`, so the TTS playback path is Windows-only).
- KittenTTS requires eSpeak NG installed; [main.py](main.py) hardcodes the default winget install path
  (`C:\Program Files\eSpeak NG`) and sets `PHONEMIZER_ESPEAK_LIBRARY`/`PATH` for it at import time.

## Architecture / protocol

The two halves communicate over a tiny newline-delimited serial protocol at 9600 baud:

- **Arduino → Python**: `TRIGGER_AI` — sent either when the countdown reaches zero or when the manual AI switch
  (`manualAISwitch`, pin A3) is pressed. This starts the red `aiStatusLED` blinking on the Arduino side
  (non-blocking, driven from `loop()`'s millis() timer) as a "still working" indicator.
- **Python → Arduino**: `AI_DONE` — sent from a `finally` block in `run_crew()` once the crew call finishes
  (success or failure), so the LED always stops blinking even on error/timeout.

Python's `run_crew()` flow: POST to `{CREW_URL}/kickoff`, then poll `{CREW_URL}/status/{kickoff_id}` every
`POLL_SECONDS` (5s) up to `POLL_TIMEOUT_SECONDS` (300s) until state is one of
`SUCCESS`/`COMPLETED`/`FAILED`/`ERROR`. The crew's `topic` input is currently hardcoded in `run_crew()`. If the
result is a string, it's spoken sentence-by-sentence via KittenTTS (long multi-sentence text fails in the nano
model, hence the per-sentence splitting).

On the Arduino side, `timerSwitch` (A0) toggles the countdown running/paused and `pomodoroLED` (A1) tracks that
state; the countdown resets to 5 minutes after firing. Display multiplexing (`displayNumber`) and the AI-status
blink are both non-blocking so the countdown timing stays accurate.
