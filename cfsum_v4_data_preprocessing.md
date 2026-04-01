# Data Preprocessing Pipeline — QVHighlights Dataset

Complete preprocessing pipeline for the Audio-Aware Coarse-to-Fine Summarization model.

---

## 1. Dataset Overview

**QVHighlights** — a query-conditioned video highlight detection benchmark.

- Source: YouTube videos
- Each video is split into **150-second clips**
- Human annotators write natural language queries with per-clip saliency scores
- Standard splits: **Train / Val / Test**

---

## 2. Annotation Format

Annotations stored in JSONL files (one JSON object per line):

| File | Purpose |
|---|---|
| `highlight_train_release.jsonl` | Training set with full annotations |
| `highlight_val_release.jsonl` | Validation set with full annotations |
| `highlight_test_release.jsonl` | Test set (no ground truth) |
| `highlight_test_with_gt.jsonl` | Test set with ground truth |

### Schema

```json
{
  "qid": 9769,
  "query": "some military patriots takes us through their safety procedures.",
  "duration": 150,
  "vid": "j7rJstUseKg_360.0_510.0",
  "relevant_clip_ids": [36, 37, 38],
  "saliency_scores": [[4, 3, 2], [4, 1, 3], [3, 2, 4]],
  "relevant_windows": [[72, 82], [84, 94]]
}
```

| Field | Description |
|---|---|
| `qid` | Unique query ID |
| `query` | Natural language query string |
| `duration` | Clip duration in seconds (typically 150) |
| `vid` | Clip identifier: `{youtube_id}_{start_sec}_{end_sec}` |
| `relevant_clip_ids` | Indices of relevant 2-second clips within the 150s window |
| `saliency_scores` | 3 annotators' scores per relevant clip, each in range [1, 5] |
| `relevant_windows` | Temporal windows `[start, end]` in seconds |

### Saliency Target Construction

```python
saliency[clip_idx] = (mean(annotator_scores) - 1.0) / 4.0
# Maps annotator scores [1, 5] → [0, 1]
# Non-relevant clips default to 0.0
```

---

## 3. Feature Extraction Pipeline

### 3.1 Audio Download (`download_audios.py`)

| Parameter | Value |
|---|---|
| Source | YouTube via `yt-dlp` |
| Input | `yt_ids.txt` — one YouTube ID per line |
| Output | `audio_downloads/{yt_id}.{ext}` |
| Preferred formats | m4a, webm, opus, mp3, wav, flac, aac |
| Target count | 3000 valid downloads |

**Error handling**:
- `audio_downloads/unavailable.txt` — permanently unavailable (private/deleted/copyright)
- `audio_downloads/failed.txt` — transient failures (network errors)

Already-downloaded files are skipped on re-run.

### 3.2 Audio Features — PANN CNN14 (`extract_audio_features.py`)

| Parameter | Value |
|---|---|
| Model | PANN CNN14 (`panns_inference.AudioTagging`) |
| Sample rate | 32,000 Hz |
| Embedding dim | **2048** |
| Window size | 2 seconds (non-overlapping) |
| Windows per clip | 75 (150s ÷ 2s) |
| Batch size | 32 windows per forward pass |

**Process**:
1. Load audio with `librosa.load(sr=32000, mono=True)`
2. Slice the relevant 150-second segment using clip start/end times
3. Pad or trim to exactly **4,800,000 samples** (150s × 32kHz)
4. Reshape into **75 windows × 64,000 samples** (75 two-second windows)
5. Batch-feed through CNN14 → extract 2048-dim embedding per window
6. Save as `.npz` with key `features`, shape **(75, 2048)**

**Output**: `features/audio_features/{vid_stem}.npz`

### 3.3 Video Features — SlowFast R50 (Pre-extracted)

| Parameter | Value |
|---|---|
| Model | SlowFast R50 (`pytorchvideo.models.hub.slowfast_r50`) |
| Feature dim | **2304** |
| Slow pathway | 8 frames |
| Fast pathway | 32 frames |
| Head | Replaced with `nn.Identity()` for feature extraction |

**Output**: `features/slowfast_features/{vid_stem}.npz` — key `features`, shape **(T, 2304)**

### 3.4 CLIP Visual Features (Pre-extracted)

| Parameter | Value |
|---|---|
| Model | `openai/clip-vit-base-patch32` (HuggingFace) |
| Feature dim | **512** |
| Sampling | 1 frame per 2-second clip interval |
| Normalization | L2-normalized |

**Output**: `features/clip_features/{vid_stem}.npz` — key `features`, shape **(T, 512)**

### 3.5 CLIP Text Features (Pre-extracted)

| Parameter | Value |
|---|---|
| Model | `openai/clip-vit-base-patch32` text encoder |
| Feature dim | **512** |
| Max token length | 32 |
| Output | Word-level hidden states (not pooled) |

**Output**: `features/clip_text_features/qid{N}.npz` — key `last_hidden_state`, shape **(S, 512)**

One file per query (~10,000 total).

---

## 4. Feature File Summary

| Directory | Filename Pattern | Key | Shape | Modality |
|---|---|---|---|---|
| `clip_features/` | `{vid}.npz` | `features` | (T, 512) | CLIP visual |
| `slowfast_features/` | `{vid}.npz` | `features` | (T, 2304) | SlowFast video |
| `audio_features/` | `{vid}.npz` | `features` | (75, 2048) | PANN CNN14 audio |
| `clip_text_features/` | `qid{N}.npz` | `last_hidden_state` | (S, 512) | CLIP text tokens |

**Naming convention**: `{youtube_id}_{start_sec}_{end_sec}.npz`
- Example: `_-vRAyh0p8w_210.0_360.0.npz`

---

## 5. Dataset Loading Pipeline

### 5.1 Stem Cache (`stem_cache.json`)

Pre-built index of all available feature file stems to avoid expensive filesystem scans:

```json
{
  "clip": ["vid1_0.0_150.0", "vid2_150.0_300.0", ...],
  "slowfast": [...],
  "audio": [...],
  "text": ["qid0", "qid1", ...]
}
```

Built once on first run, reused on subsequent runs.

### 5.2 Sample Matching

A sample requires **both** CLIP and SlowFast features AND an annotation entry:

```python
valid_ids = clip_stems ∩ slowfast_stems ∩ annotated_vids
```

Audio is optional — missing audio is zero-filled.

### 5.3 RAM Caching (`RAMCachedDataset`)

All `.npz` features are loaded into RAM dictionaries at startup for zero-disk-IO training:

```python
ram_clip[vid]     = np.load(...)['features']     # (T, 512)
ram_slowfast[vid] = np.load(...)['features']     # (T, 2304)
ram_audio[vid]    = np.load(...)['features']      # (75, 2048)
ram_text[qid]     = np.load(...)['last_hidden_state']  # (S, 512)
```

### 5.4 Temporal Alignment

Features from different modalities may have different temporal lengths. All are aligned to the CLIP feature length `T` using adaptive average pooling:

```python
aligned = F.adaptive_avg_pool1d(features.T.unsqueeze(0), target_T).squeeze().T
```

### 5.5 Collation (Batching)

Samples within a batch are padded to the maximum sequence length:

| Tensor | Padding | Mask |
|---|---|---|
| Video (B, T, 2304) | Zero-pad to max T | `mask` = 1 for real, 0 for pad |
| Audio (B, T, 2048) | Zero-pad to max T | `audio_mask` = 1 if has audio, 0 otherwise |
| CLIP (B, T, 512) | Zero-pad to max T | Shared `mask` |
| Text (B, S, 512) | Zero-pad to max S | `text_mask` = 1 for real tokens |
| Saliency (B, T) | Zero-pad to max T | — |

---

## 6. Dataset Statistics

| Split | Videos | Samples (queries) |
|---|---|---|
| Train | ~1,000+ | ~4,000+ |
| Val | ~400+ | ~1,500+ |
| Test | ~400+ | ~1,500+ |

Audio coverage: ~60–70% of videos have downloadable audio.

---

## 7. Additional Files

| File | Purpose |
|---|---|
| `yt_ids.txt` | Master list of YouTube video IDs for audio downloading |
| `stem_cache.json` | Pre-computed feature file index |
| `audio_downloads/failed.txt` | YouTube IDs with transient download failures |
| `audio_downloads/unavailable.txt` | YouTube IDs permanently unavailable |
| `qv_dataset.py` | Legacy dataset class (SlowFast + subtitle features only, no audio/CLIP) |
