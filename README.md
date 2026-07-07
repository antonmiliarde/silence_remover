# BezPauz (BezPauses)

**Desktop video editor that automatically removes silence — manual timeline edits, and multi-camera audio sync.**

BezPauz targets creators who record talking-head or tutorial videos and lose time trimming dead air in NLEs. The app analyzes audio, builds a cut plan, lets you review and adjust removals on a dual-track waveform timeline, then exports via FFmpeg — locally, without cloud upload.

> **Note for reviewers:** Silence detection is **classical signal processing** (RMS windows, FFmpeg `silencedetect`, cross-correlation sync) — not deep learning. The project demonstrates production-grade **audio/video pipelines**, desktop UX, and a freemium SaaS integration.

---

## ✨ Key Features

### Free (guest / Basic tier)
- **7 silence-detection modes** — peak-percent (`volumedetect` + `silencedetect`), fixed dBFS, percentile RMS, noise-floor RMS, hysteresis RMS, spectral high-pass, mean RMS
- **Interactive timeline** — dual waveform tracks (main + optional B-roll), zoom/pan, compressed “output duration” view
- **Manual cut editing** — mark/unmark regions, undo/redo, configurable hotkeys
- **In-app preview** — OpenCV frame decode + FFmpeg PCM playback (`sounddevice`), scrubbing, playhead sync
- **Export** with quality presets (100% / 75% / 50% / 25% scale + CRF) and cancellable progress UI
- **CLI** — `remove_silence.py` for scripted batch cuts
- **6 UI languages** — RU, EN, IT, DE, ES, FR
- **3 layout modes** — classic, split, wide timeline

### Pro (subscription)
- **Unlimited export length** — Basic / guest exports are capped at **120 seconds** after cuts (`FEATURE_EXPORT_UNLIMITED`)
- Account login, license verification, HWID binding via backend API

### Multi-camera / B-roll sync
- **Audio-based alignment** — normalized mono PCM + **numpy cross-correlation** (`estimate_main_to_sync_time_offset_sec`)
- Handles **different clip lengths** (sliding-window search when B-roll is longer than A-roll)
- Synced export applies the same keep-intervals as the main video, shifted by the estimated offset
- Overlap visualization on the secondary timeline (`sync_overlap_on_main_timeline`)

---

## 🛠 Tech Stack & Architecture

| Layer | Technologies |
|--------|----------------|
| **Core engine** | Python 3.12+, `remove_silence.py` — interval algebra, cut plans, FFmpeg orchestration |
| **Media** | [static-ffmpeg](https://pypi.org/project/static-ffmpeg/) (bundled FFmpeg/ffprobe), `libx264` + AAC |
| **Signal / “ML-adjacent” DSP** | NumPy (cross-correlation, RMS stats), FFmpeg `silencedetect` / `volumedetect`, windowed RMS (8 kHz, 400-sample windows) |
| **Preview** | OpenCV (`opencv-python-headless`), Pillow, FFmpeg → PCM → `sounddevice` |
| **Desktop UI** | Tkinter + `ttk`, custom theme (`bezpauz_ui_theme.py`), canvas timelines (`bezpauz_time_ruler.py`) |
| **Concurrency** | `threading` — background analysis/export, preview pump, sync alignment; `tkinter.after` for UI thread safety |
| **Caching** | Disk waveform cache (`.waveform_cache/`, 0.1 s buckets, content-addressed keys) |
| **Accounts / billing** | `vorozheikin_account.py` — REST login, HWID verify, HMAC-signed API, tariff gating |
| **i18n** | `bezpauz_ui_i18n.py` + account strings |
| **Packaging** | PyInstaller (onedir), Inno Setup → `Bezpauz_Setup.exe` |
| **Optional** | SwiftUI iOS prototype under `ios/Bezpauz/` (separate from the Windows desktop build) |

### High-level modules

```
remove_silence_gui.py   # App shell, timeline, preview, export dialogs
remove_silence.py       # Silence detection, CutPlan, FFmpeg filter graphs
vorozheikin_account.py  # Auth, tariffs, settings persistence
bezpauz_ui_*.py         # Theme, i18n, time ruler
экзе/                   # PyInstaller + Inno Setup build pipeline
```

---

## ⚙️ How It Works (Under the Hood)

### 1. Silence analysis → `CutPlan`

1. **Probe** duration with `ffprobe`.
2. **Detect silence** using the selected mode (`SilenceDetectMode`):
   - *Peak percent* — `volumedetect` peak → threshold in dBFS → `silencedetect`
   - *RMS modes* — `collect_window_rms()` decodes short PCM windows; percentile / mean / hysteresis / noise-floor logic builds intervals without neural models
   - *Spectral* — high-pass @ 180 Hz before detection
3. **Post-process** — merge intervals, optional before/after pause buffers, invert to **keep** segments.
4. Output: immutable `CutPlan { duration_sec, noise_db, remove[], keep[], detect_mode }`.

### 2. Timeline & manual overrides

- Waveforms: `compute_waveform_envelope()` → normalized RMS per 0.1 s bucket (cached on disk).
- User edits mutate `_effective_remove`; undo/redo stacks store interval lists.
- Export uses `_timeline_remove_intervals_for_export()` so manual marks match the final file.

### 3. Export pipeline

- `build_filter_complex()` — per-segment `trim` + `atrim`, `concat` for video/audio.
- Large graphs written to a **temp filter script** (avoids Windows ~8 KB command-line limit).
- `run_remove_silence()` / `transcode_with_keep_intervals()` — progress callbacks, cooperative cancel via `ExportCancelToken`.
- Basic tier: `max_output_duration_sec=120` clips the concatenated output.

### 4. B-roll sync

1. Extract mono f32le PCM (800 Hz) from both files.
2. **Cross-correlate** main template against the longer sync track (or vice versa) with adaptive `max_lag_sec` (`compute_sync_search_lag_sec`).
3. Peak-to-median ratio gate (`min_peak_ratio`) rejects weak matches.
4. On export: shift keep-intervals by offset, clamp to sync file duration, transcode with the same cut structure.

### 5. Account gating

- Guest / **Basic**: full analysis & preview; export ≤ 120 s.
- **Pro**: `export_unlimited` feature flag from `/me` + license HWID verify.

---

## 🚀 Installation & Local Setup (Developers)

**Requirements:** Windows 10/11 (primary target), Python 3.11+ (3.12+ recommended), internet on first run (FFmpeg bootstrap via `static-ffmpeg`).

### 1. Clone

```bash
git clone https://github.com/<your-username>/bezpauz.git
cd bezpauz
```

### 2. Create a virtual environment (recommended)

```bash
python -m venv .venv
.venv\Scripts\activate        # Windows
# source .venv/bin/activate   # macOS/Linux (core engine only; GUI is Windows-focused)
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

| Package | Role |
|---------|------|
| `static-ffmpeg` | Bundled FFmpeg/ffprobe |
| `opencv-python-headless` | Preview frames |
| `Pillow` | Icons / header branding |
| `sounddevice` + `numpy` | Playback & sync correlation |
| `numpy` | RMS stats, cross-correlation |

### 4. Run from source

**Debug (console visible — recommended for development):**

```bat
ОТЛАДКА.bat
```

Or manually:

```bash
python remove_silence_gui.py
```

**Normal launch (minimized console on Windows):**

```bat
Start from here.bat
```

> On Windows, if `python` points to the Microsoft Store stub, use `py -3` or run `ОТЛАДКА.bat` — it resolves a working interpreter via `_find_python.bat`.

**CLI example:**

```bash
python remove_silence.py -i "talk.mp4" -o "talk_cut.mp4" --percent 8 --min-silence 0.4 --buffer 0.15
```

### 5. Build the Windows installer (optional)

```bat
cd экзе
BUILD_CLIENT.bat
```

Output: `экзе/ГОТОВО/Bezpauz_Setup.exe` (PyInstaller onedir + Inno Setup).

---

## 📦 How to Run the Pre-built Binary

1. Open **[Releases](https://github.com/<your-username>/bezpauz/releases)** on GitHub.
2. Download **`Bezpauz_Setup.exe`** (or the latest installer asset).
3. Run the installer and launch **BezPauz** from the Start menu / desktop shortcut.
4. On first launch, allow network access once — FFmpeg binaries are fetched by `static-ffmpeg` if not bundled.

No Python installation required for end users.

---

## 📄 License & Commercial Model

This repository is a **portfolio / product codebase** for **BezPauz** by Vorozheikin & Company.

| Tier | Export | Account |
|------|--------|---------|
| **Guest / Basic** | Free; max **120 s** output after cuts | Optional login |
| **Pro** | Unlimited duration | Subscription via [itsoulutions.ru](https://itsoulutions.ru/vorozheikinithub/) |

- There is **no root `LICENSE` file** in this tree — the distributed app is **freemium**, not open-source MIT.
- Third-party deps retain their own licenses (FFmpeg, NumPy, OpenCV, etc.).
- For **portfolio review**: you may reference algorithms and architecture in this README; **commercial use, redistribution, or Pro features** require permission from the author.

If you plan to open-source a fork for GitHub, add an explicit `LICENSE` (e.g. MIT) and strip or mock `vorozheikin_account.py` API credentials.

---

## 👤 Author

**Vorozheikin & Company** — desktop automation & video tooling.

- Product site: [itsoulutions.ru/bezpauz](https://itsoulutions.ru/vorozheikinithub/)
- Portfolio focus: signal-processing pipelines, desktop UX, FFmpeg automation, production Python
