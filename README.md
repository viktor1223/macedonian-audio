# Macedonian Audio Dictionary

5,736 word pronunciation clips for Macedonian language learners, extracted from open-licensed speech corpora and hosted via GitHub Pages for integration with a [Notion vocabulary database](https://github.com/viktor1223/Macedonian-Notion-Updates).

## Audio Sources

| Source | Type | Clip Count | License | Description |
|--------|------|-----------|---------|-------------|
| [Mozilla Common Voice 17.0 MK](https://commonvoice.mozilla.org/) | Sentence → word extraction | 5,580 | CC-0 (Public Domain) | Words clipped from sentence recordings by Macedonian volunteer speakers using forced alignment |
| [Lingua Libre](https://lingualibre.org/) (Wikimedia Commons) | Direct word recordings | 156 | CC-BY-SA 4.0 | Single-word pronunciations recorded by native speakers Bjankuloski06 and Jovan.kostov |

## Repository Structure

```
clips/          # 5,580 word clips extracted from Common Voice sentences
                # Filename: {word}.wav (e.g. вода.wav, куче.wav)
                # Format: 16kHz mono WAV
README.md
```

## How These Files Were Created

The entire audio pipeline is automated and lives in the companion repository:
**[Macedonian-Notion-Updates](https://github.com/viktor1223/Macedonian-Notion-Updates)**

### Pipeline Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│  1. INDEX          Build word→sentence mapping from Common Voice    │
│                    (download_corpora.py → common_voice_word_index)  │
├─────────────────────────────────────────────────────────────────────┤
│  2. FETCH          Download Lingua Libre recordings via             │
│                    Wikimedia Commons API (fetch_audio.py)           │
├─────────────────────────────────────────────────────────────────────┤
│  3. CLIP           Extract words from CV sentences using            │
│                    MMS forced alignment (clip_audio.py)             │
├─────────────────────────────────────────────────────────────────────┤
│  4. VALIDATE       Verify each clip contains the correct word       │
│                    using alignment confidence (validate_audio.py)   │
├─────────────────────────────────────────────────────────────────────┤
│  5. VERIFY         Cross-source MFCC comparison where both          │
│                    sources exist (verify_audio.py)                  │
├─────────────────────────────────────────────────────────────────────┤
│  6. PUBLISH        Push validated clips to this repo + link         │
│                    audio URLs in Notion (push_to_notion.py)         │
└─────────────────────────────────────────────────────────────────────┘
```

### Step-by-Step Detail

#### 1. Word Indexing

The pipeline first builds a word-to-sentence index from the Common Voice Macedonian dataset. Each word in the vocabulary is mapped to a sentence recording that contains it.

- **Script:** [`download_corpora.py`](https://github.com/viktor1223/Macedonian-Notion-Updates/blob/main/download_corpora.py)
- **Output:** `audio/common_voice_word_index.csv` — maps each word to a sentence, audio file path, source, and license
- **Dataset:** [`fsicoli/common_voice_17_0`](https://huggingface.co/datasets/fsicoli/common_voice_17_0) (Macedonian split)

#### 2. Audio Fetching (Lingua Libre)

For words with direct single-word recordings on Wikimedia Commons, the pipeline downloads them via the MediaWiki API.

- **Script:** [`fetch_audio.py`](https://github.com/viktor1223/Macedonian-Notion-Updates/blob/main/fetch_audio.py)
- **Source config:** [`audio_sources.yaml`](https://github.com/viktor1223/Macedonian-Notion-Updates/blob/main/audio_sources.yaml)
- **Speakers:** Bjankuloski06, Jovan.kostov
- **Output:** `audio/words/{word}.wav`
- **Resolution strategy:** Exact word match → lemma fallback → flag as missing

#### 3. Forced Alignment Clipping (Common Voice)

This is where the majority of clips (5,580) come from. The pipeline:

1. Streams sentence audio from the Common Voice dataset
2. Romanizes Cyrillic text (Macedonian → Latin transliteration for the alignment model)
3. Runs **Meta's MMS CTC forced alignment model** (`torchaudio.pipelines.MMS_FA`) to locate exact word boundaries within each sentence
4. Extracts the target word with 40ms padding on each side
5. Saves as 16kHz mono WAV

- **Script:** [`clip_audio.py`](https://github.com/viktor1223/Macedonian-Notion-Updates/blob/main/clip_audio.py)
- **Model:** Meta MMS (Massively Multilingual Speech) — supports 1,100+ languages
- **Transliteration:** [`core/romanize.py`](https://github.com/viktor1223/Macedonian-Notion-Updates/blob/main/core/romanize.py)
- **Minimum confidence:** 0.4 (clips below this threshold are rejected)
- **Output:** `audio/clips/{word}.wav` + `audio/clip_dictionary_{timestamp}.csv` with per-clip metadata

Each clip in the dictionary records: word, confidence score, duration, source sentence, original audio file, source, and license.

#### 4. Validation

After clipping, every audio file is validated to confirm it actually contains the expected word.

- **Script:** [`validate_audio.py`](https://github.com/viktor1223/Macedonian-Notion-Updates/blob/main/validate_audio.py)
- **Method:** Re-runs MMS forced alignment on the isolated clip and checks:
  - Alignment succeeds (word is detectable)
  - Confidence score ≥ 0.75 (PASS) or ≥ 0.50 (WARN)
  - Word occupies a reasonable portion of the clip duration
- **Grades:** PASS / WARN / FAIL — only PASS clips are published

#### 5. Cross-Source Verification

When a word has audio from both Lingua Libre and Common Voice, the pipeline runs a spectral comparison to confirm agreement.

- **Script:** [`verify_audio.py`](https://github.com/viktor1223/Macedonian-Notion-Updates/blob/main/verify_audio.py)
- **Method:** Computes MFCC (Mel-Frequency Cepstral Coefficients) fingerprints for both clips and measures cosine similarity
- **Threshold:** ≥ 0.70 cosine similarity → CONFIRMED
- **Additional heuristics:** Duration-per-character check flags outliers
- **No AI/API calls** — pure numpy + scipy computation
- **Verdicts:** CONFIRMED (multi-source agree) / PASS (single source, passes checks) / SUSPECT / FAIL

#### 6. Publishing

Validated clips are committed to this repository and served via GitHub Pages. The audio URLs are then linked into the Notion vocabulary database.

- **Push script:** [`push_to_notion.py`](https://github.com/viktor1223/Macedonian-Notion-Updates/blob/main/push_to_notion.py)
- **URL pattern:** `https://viktor1223.github.io/macedonian-audio/clips/{word}.wav`

## Per-File Attribution

Every clip can be traced back to its source:

| Directory | Source | License | Attribution |
|-----------|--------|---------|-------------|
| `clips/*.wav` | Mozilla Common Voice 17.0 (Macedonian) | CC-0 (Public Domain) | Extracted from sentence recordings by anonymous Macedonian volunteer speakers via forced alignment |
| `words/*.wav` (if present) | Lingua Libre / Wikimedia Commons | CC-BY-SA 4.0 | Recorded by speakers **Bjankuloski06** and **Jovan.kostov** |

A full per-word attribution log with source sentence, speaker, confidence score, and original filename is maintained in the pipeline repo at [`audio/clip_dictionary_*.csv`](https://github.com/viktor1223/Macedonian-Notion-Updates/tree/main/audio) and [`audio/attribution.csv`](https://github.com/viktor1223/Macedonian-Notion-Updates/blob/main/audio/attribution.csv).

## Audio Quality Guarantees

- **Minimum alignment confidence:** 0.4 (extraction) → 0.75 (validation)
- **Cross-source verification** where multiple sources available
- **Duration sanity check:** 40–250ms per character expected
- **Format:** 16kHz mono WAV, consistent across all clips
- **No AI-generated speech** — all audio is from real human speakers

## Technology Stack

| Component | Technology |
|-----------|-----------|
| Forced alignment | [Meta MMS](https://github.com/facebookresearch/fairseq/tree/main/examples/mms) via torchaudio |
| Audio processing | torchaudio, soundfile, numpy |
| Spectral comparison | MFCC computation (scipy DCT + numpy) |
| Transliteration | Custom Cyrillic→Latin mapping ([`core/romanize.py`](https://github.com/viktor1223/Macedonian-Notion-Updates/blob/main/core/romanize.py)) |
| Source datasets | HuggingFace datasets library (streaming mode) |
| API downloads | Wikimedia Commons MediaWiki API |

## License

- **Common Voice clips** (`clips/`): **CC-0** (Public Domain) — no attribution required
- **Lingua Libre clips** (`words/`): **CC-BY-SA 4.0** — attribution: Bjankuloski06, Jovan.kostov via Wikimedia Commons

## Related Projects

- **[Macedonian-Notion-Updates](https://github.com/viktor1223/Macedonian-Notion-Updates)** — Full pipeline code: frequency analysis, dictionary enrichment, audio extraction, and Notion database sync
- **[Mozilla Common Voice](https://commonvoice.mozilla.org/mk)** — Contribute Macedonian voice recordings
- **[Lingua Libre](https://lingualibre.org/)** — Record word pronunciations for Wikimedia projects
