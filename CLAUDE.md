# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Chatterbox Pro is a Python GUI application for generating audiobooks using the Chatterbox TTS (text-to-speech) model. It provides voice cloning, multi-GPU processing, ASR validation, and professional audio post-processing.

## Commands

```bash
# Run the application
python chatter_pro.py

# Install dependencies (requires CUDA 12.1)
pip install -r requirements_pro.txt
```

**System dependencies:** FFmpeg (normalization), auto-editor (silence removal), Pandoc (optional, for docx/mobi)

## Architecture

### Entry Point
- `chatter_pro.py` - Application launcher, sets up multiprocessing spawn method and initializes GUI

### Core Modules (`core/`)
- `orchestrator.py` - `GenerationOrchestrator` manages multi-GPU TTS generation workflow using `ProcessPoolExecutor`
- `audio_manager.py` - `AudioManager` handles audiobook assembly, normalization (EBU R128), and silence removal

### UI Layer (`ui/`)
- `main_window.py` - `ChatterboxProGUI` (CustomTkinter CTk) is the main application class containing all state variables
- `playlist.py` - `PlaylistFrame` manages the text chunk playlist with pagination
- `controls_frame.py` - Playback and playlist manipulation controls
- `tabs/` - Tab-specific UI: `setup_tab.py`, `generation_tab.py`, `finalize_tab.py`, `advanced_tab.py`

### Workers (`workers/`)
- `tts_worker.py` - `worker_process_chunk()` runs in separate processes, handles TTS generation with Whisper ASR validation

### Utilities (`utils/`)
- `text_processor.py` - `TextPreprocessor` handles text extraction, sentence splitting, and chunking
- `dependency_checker.py` - `DependencyManager` checks for FFmpeg, auto-editor, Pandoc

### Chatterbox TTS (`chatterbox/`)
- Embedded Chatterbox model code (not modified by this project)
- `tts.py` - Main TTS interface via `ChatterboxTTS.from_pretrained()`

## Key Data Flow

1. Text files → `TextPreprocessor` splits into sentences/chunks → stored in `app.sentences` list
2. `GenerationOrchestrator.run()` distributes chunks to `ProcessPoolExecutor` workers
3. Each worker loads models once (`get_or_init_worker_models`), generates audio, validates with Whisper
4. Results saved to `Outputs_Pro/<session>/Sentence_wavs/audio_<uuid>.wav`
5. `AudioManager.assemble_audiobook()` combines chunks with pydub, applies FFmpeg normalization

## State Management

All application state lives in `ChatterboxProGUI` as `ctk.StringVar`/`BooleanVar`/`DoubleVar` instances. Session data (sentences, settings) is persisted as JSON in `Outputs_Pro/<session>/`.

## Multiprocessing Notes

- Uses `spawn` method on Windows/macOS for CUDA compatibility
- Workers initialize TTS and Whisper models once per process via global singletons
- Main thread communicates via `app.after()` for thread-safe UI updates
