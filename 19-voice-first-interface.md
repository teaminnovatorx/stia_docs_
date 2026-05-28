# Voice-First Interface — MMS-ASR Pipeline for African Languages

> **Document ID**: UDARA-ARCH-019  
> **Version**: 2.5.0  
> **Last Updated**: 2026-05-27  
> **Author**: UDARA AI Voice & NLP Engineering Team  
> **Classification**: Technical Deep Dive — Voice Pipeline  
> **Audience**: NLP Engineers, ML Engineers, Backend Engineers, Product Designers

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Why Voice-First?](#2-why-voice-first)
3. [MMS-ASR Technical Foundation](#3-mms-asr-technical-foundation)
   - 3.1 [Model Architecture](#31-model-architecture)
   - 3.2 [INT8 Quantization for Edge](#32-int8-quantization-for-edge)
   - 3.3 [Streaming Inference with Chunked Audio](#33-streaming-inference-with-chunked-audio)
   - 3.4 [Target Languages](#34-target-languages)
4. [Language Detection Pipeline](#4-language-detection-pipeline)
   - 4.1 [fastText LID Model](#41-fasttext-lid-model)
   - 4.2 [Confidence Thresholds & Fallback](#42-confidence-thresholds--fallback)
   - 4.3 [Language Confirmation Flow](#43-language-confirmation-flow)
5. [ASR Pipeline Architecture](#5-asr-pipeline-architecture)
   - 5.1 [Audio Preprocessing](#51-audio-preprocessing)
   - 5.2 [Feature Extraction](#52-feature-extraction)
   - 5.3 [Model Inference](#53-model-inference)
   - 5.4 [Post-Processing](#54-post-processing)
   - 5.5 [Confidence Scoring](#55-confidence-scoring)
   - 5.6 [Decision Logic](#56-decision-logic)
6. [Audio Input Channels](#6-audio-input-channels)
   - 6.1 [WhatsApp Voice Notes](#61-whatsapp-voice-notes)
   - 6.2 [Telegram Voice Messages](#62-telegram-voice-messages)
   - 6.3 [USSD / DTMF Fallback](#63-ussd--dtmf-fallback)
   - 6.4 [Web Microphone](#64-web-microphone)
   - 6.5 [Phone Call (IVR)](#65-phone-call-ivr)
7. [Complete Code Implementation](#7-complete-code-implementation)
8. [Medical Vocabulary & Post-Processing](#8-medical-vocabulary--post-processing)
9. [Performance on Raspberry Pi 5](#9-performance-on-raspberry-pi-5)
10. [Error Handling & Graceful Degradation](#10-error-handling--graceful-degradation)
11. [Testing & Evaluation](#11-testing--evaluation)
12. [Configuration Reference](#12-configuration-reference)
13. [Monitoring & Analytics](#13-monitoring--analytics)
14. [Appendix](#14-appendix)

---

## 1. Executive Summary

The UDARA AI Voice-First Interface is a multilingual Automatic Speech Recognition (ASR) pipeline designed specifically for Community Health Workers (CHWs) in sub-Saharan Africa. It enables CHWs to report AMR cases using natural speech in their native language, dramatically reducing reporting friction and improving data quality.

The pipeline is built on Meta's **MMS (Massively Multilingual Speech)** model, quantized to INT8 for edge deployment on Raspberry Pi 5, and supports **12 African languages** at launch.

### Key Performance Targets

| Metric | Target | Current |
|--------|--------|---------|
| Word Error Rate (WER) — Swahili | < 15% | 12.3% |
| Word Error Rate (WER) — Yoruba | < 18% | 16.1% |
| WER — average across 12 languages | < 20% | 17.8% |
| Medical term accuracy | > 90% | 91.2% |
| Transcription latency (10s audio, RPi 5) | < 5s | 3.2s |
| Language detection accuracy | > 95% | 96.4% |
| Memory usage (RPi 5, single language loaded) | < 1GB | 820MB |
| Model size per language (INT8) | < 350MB | 305MB |
| Simultaneous language models in memory | 3 | 3 |
| Audio channels supported | 5 | 5 |
| Daily voice reports processed (cloud) | 50,000+ | 42,000 |

---

## 2. Why Voice-First?

### 2.1 The Literacy Challenge

```
┌──────────────────────────────────────────────────────────────────┐
│               LITERACY RATES IN SUB-SAHARAN AFRICA               │
│                                                                  │
│  Adult Literacy Rate (%)                                         │
│                                                                  │
│  100 ┤                                                          │
│   80 ┤    ██                                                    │
│   60 ┤    ██ ██                                                 │
│   40 ┤    ██ ██ ██              ██                              │
│   20 ┤    ██ ██ ██    ██       ██ ██                            │
│    0 ┼────┬─┬─┬─┬────┬───┬──────┬──┬─────┬───────┬──             │
│       Nig Eth Mali Burk Chad SLe Lib  Nig  Malawi  Sen           │
│       eri  pia     Fas o   ier    eria       Mali               │
│                                                                  │
│  ██ = Adult literacy (age 15+)  — Many below 50%                │
│                                                                  │
│  Rural female literacy can be as low as 15-25%                  │
│  CHW programs often recruit from local communities               │
│  where literacy rates reflect these statistics                   │
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 The Reporting Friction Problem

| Challenge | Text-Based Reporting | Voice Reporting | Improvement |
|-----------|---------------------|-----------------|-------------|
| Literacy requirement | High — must read/write | None — speak naturally | 3× more CHWs can report |
| Time per report | 5–8 minutes (type carefully) | 1.5–2 minutes (speak) | 3–4× faster |
| Feature phone compatibility | Difficult (T9 typing) | Natural (voice note) | Native UX |
| Medical terminology | Often misspelled | Corrected by post-processor | Higher accuracy |
| Data completeness | Often skip fields | Naturally conveyed in speech | More complete |
| User satisfaction | Low (frustrating) | High (natural) | Better retention |

### 2.3 Voice Reporting Adoption Evidence

```
┌──────────────────────────────────────────────────────────────────┐
│           REPORTING METHOD COMPARISON — PILOT DATA               │
│                                                                  │
│  Reports per CHW per Week                                        │
│   12 ┤                                                          │
│   10 ┤                              ╭──╮                        │
│    8 ┤                              │  │                        │
│    6 ┤         ╭──╮                  │  │   ╭──╮                │
│    4 ┤    ╭────╯  ╰────╮      ╭─────╯  ╰───╯  ╰──╮             │
│    2 ┤───╯            ╰─────╯                  ╰───╮             │
│    0 ┼────────────────────────────────────────────┼──►           │
│      Month 1   Month 2   Month 3   Month 4   Month 5           │
│                                                                  │
│  ━━━ Text-only (baseline)     Average: 2.8 reports/week        │
│  ─── Text + optional voice    Average: 5.1 reports/week        │
│  ┄┄┄ Voice-first (UDARA AI)   Average: 9.4 reports/week  ★     │
│                                                                  │
│  Voice-first CHWs are 3.4× more productive than text-only       │
└──────────────────────────────────────────────────────────────────┘
```

### 2.4 Design Principles

1. **Speak Your Language**: No need to switch to English or a colonial language
2. **Zero Training Required**: Works out of the box — no speech training needed
3. **Forgiving**: Handles accents, background noise, incomplete sentences
4. **Fast**: Transcription completes while you prepare your next report
5. **Reliable**: Fallback to text if voice fails; never lose a report
6. **Private**: Audio is processed locally when possible; minimal data sent to cloud

---

## 3. MMS-ASR Technical Foundation

### 3.1 Model Architecture

Meta's MMS (Massively Multilingual Speech) model is built on the **wav2vec 2.0** architecture with a CTC (Connectionist Temporal Classification) decoder:

```
┌──────────────────────────────────────────────────────────────────┐
│                 MMS-ASR MODEL ARCHITECTURE                        │
│                                                                  │
│  Audio (16kHz PCM)                                               │
│       │                                                          │
│       ▼                                                          │
│  ┌─────────────┐    ┌──────────────────────┐                    │
│  │  Feature    │───►│  CNN Feature Encoder  │                    │
│  │  Extractor  │    │  (7 layers, 512ch)    │                    │
│  └─────────────┘    └──────────┬───────────┘                    │
│                                │                                 │
│                                ▼                                 │
│  ┌──────────────────────────────────────────────┐               │
│  │        Transformer Encoder (24 layers)        │               │
│  │                                                │               │
│  │  ┌──────┐ ┌──────┐ ┌──────┐     ┌──────┐    │               │
│  │  │ MHA  │ │ FFN  │ │ MHA  │ ... │ FFN  │    │               │
│  │  │ + LN │ │ + LN │ │ + LN │     │ + LN │    │               │
│  │  └──┬───┘ └──┬───┘ └──┬───┘     └──┬───┘    │               │
│  │     └───────┴────────┴─────────────┘         │               │
│  │                                                │               │
│  │  Hidden dim: 1020    Heads: 16                │               │
│  │  Parameters: ~1B    (full precision)          │               │
│  └──────────────────┬───────────────────────────┘               │
│                      │                                           │
│                      ▼                                           │
│  ┌──────────────────────────────────────────────┐               │
│  │           Language Adapter Layer              │               │
│  │  Per-language adapter (shared encoder)        │               │
│  │  ~3M parameters per language                  │               │
│  └──────────────────┬───────────────────────────┘               │
│                      │                                           │
│                      ▼                                           │
│  ┌──────────────────────────────────────────────┐               │
│  │           CTC Decoder                         │               │
│  │  Character-level output → text                │               │
│  │  Vocabulary: 6,305 characters (all scripts)   │               │
│  └──────────────────────────────────────────────┘               │
│                                                                  │
│  Total parameters: ~1B (shared) + ~3M × N_languages (adapters) │
│  UDARA AI uses: 12 language-specific checkpoints               │
└──────────────────────────────────────────────────────────────────┘
```

### 3.2 INT8 Quantization for Edge

The full MMS-1B model requires ~4GB in FP32, which exceeds the Raspberry Pi 5's 8GB RAM when combined with the OS, database, and other services. INT8 quantization reduces the model to ~300MB per language checkpoint:

```python
#!/usr/bin/env python3
"""
MMS-ASR INT8 Quantization Script
==================================

Quantizes the MMS-1B model to INT8 for edge deployment.
Uses PyTorch's dynamic quantization for linear layers.

Target: Raspberry Pi 5 (ARM Cortex-A76, 8GB RAM)
"""

import torch
import torch.quantization
from transformers import Wav2Vec2ForCTC, Wav2Vec2Processor
import os
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger('udara.voice.quantize')


def quantize_model(model_name: str, 
                    output_dir: str,
                    language_code: str) -> str:
    """
    Quantize a MMS-ASR model to INT8.
    
    Process:
    1. Load FP32 model
    2. Apply dynamic INT8 quantization to Linear layers
    3. Save quantized model and processor
    4. Report size reduction
    
    Args:
        model_name: HuggingFace model identifier
        output_dir: Directory to save quantized model
        language_code: Language code (e.g., 'swa' for Swahili)
    
    Returns:
        Path to quantized model directory
    """
    logger.info(f"Loading FP32 model: {model_name}")
    
    # Load model and processor
    processor = Wav2Vec2Processor.from_pretrained(model_name)
    model = Wav2Vec2ForCTC.from_pretrained(model_name)
    
    # Set to eval mode for quantization
    model.eval()
    
    # Report original size
    original_size = _get_model_size(model)
    logger.info(f"Original model size: {original_size / 1e6:.1f} MB")
    
    # Dynamic INT8 quantization
    # This quantizes all Linear layers (which are the bulk of parameters)
    logger.info("Applying INT8 dynamic quantization...")
    
    quantized_model = torch.quantization.quantize_dynamic(
        model,
        {torch.nn.Linear},  # Quantize all Linear layers
        dtype=torch.qint8
    )
    
    # Report quantized size
    quantized_size = _get_model_size(quantized_model)
    reduction = (1 - quantized_size / original_size) * 100
    logger.info(f"Quantized model size: {quantized_size / 1e6:.1f} MB")
    logger.info(f"Size reduction: {reduction:.1f}%")
    
    # Save quantized model
    model_path = os.path.join(output_dir, f"mms-1b-{language_code}-int8")
    os.makedirs(model_path, exist_ok=True)
    
    # Save the quantized state dict
    torch.save(quantized_model.state_dict(), 
               os.path.join(model_path, "pytorch_model.bin"))
    
    # Save processor and config
    processor.save_pretrained(model_path)
    model.config.save_pretrained(model_path)
    
    # Save a flag indicating this is quantized
    with open(os.path.join(model_path, "quantized.txt"), 'w') as f:
        f.write(f"INT8 dynamic quantization\n")
        f.write(f"Original size: {original_size / 1e6:.1f} MB\n")
        f.write(f"Quantized size: {quantized_size / 1e6:.1f} MB\n")
        f.write(f"Reduction: {reduction:.1f}%\n")
    
    logger.info(f"Quantized model saved to: {model_path}")
    
    return model_path


def _get_model_size(model) -> int:
    """Calculate model size in bytes."""
    param_size = sum(p.nelement() * p.element_size() for p in model.parameters())
    buffer_size = sum(b.nelement() * b.element_size() for b in model.buffers())
    return param_size + buffer_size


def quantize_all_target_languages(output_base_dir: str = "./models"):
    """Quantize all 12 target languages."""
    
    TARGET_LANGUAGES = {
        'swa': 'Swahili',
        'yor': 'Yoruba',
        'hau': 'Hausa',
        'amh': 'Amharic',
        'lug': 'Luganda',
        'kin': 'Kinyarwanda',
        'zul': 'Zulu',
        'xho': 'Xhosa',
        'sna': 'Shona',
        'mlg': 'Malagasy',
        'sot': 'Sesotho',
        'lin': 'Lingala',
    }
    
    results = {}
    
    for code, name in TARGET_LANGUAGES.items():
        logger.info(f"\n{'='*60}")
        logger.info(f"Quantizing: {name} ({code})")
        logger.info(f"{'='*60}")
        
        model_name = f"facebook/mms-1b-{code}"
        model_path = quantize_model(
            model_name=model_name,
            output_dir=output_base_dir,
            language_code=code,
        )
        
        results[code] = {
            'name': name,
            'model_path': model_path,
            'huggingface_id': model_name,
        }
    
    # Print summary table
    logger.info("\n" + "="*60)
    logger.info("QUANTIZATION SUMMARY")
    logger.info("="*60)
    for code, info in results.items():
        model_file = os.path.join(info['model_path'], 'pytorch_model.bin')
        size_mb = os.path.getsize(model_file) / 1e6
        logger.info(f"  {info['name']:15s} ({code}): {size_mb:.0f} MB")
    
    return results


if __name__ == '__main__':
    quantize_all_target_languages()
```

#### Quantization Results

| Language | FP32 Size | INT8 Size | Reduction | WER Impact |
|----------|-----------|----------|-----------|------------|
| Swahili | 1,240 MB | 305 MB | 75.4% | +0.3% absolute |
| Yoruba | 1,238 MB | 303 MB | 75.5% | +0.5% absolute |
| Hausa | 1,241 MB | 306 MB | 75.3% | +0.4% absolute |
| Amharic | 1,245 MB | 308 MB | 75.3% | +0.8% absolute (script complexity) |
| Luganda | 1,239 MB | 304 MB | 75.5% | +0.3% absolute |
| Kinyarwanda | 1,237 MB | 302 MB | 75.6% | +0.2% absolute |
| Zulu | 1,243 MB | 307 MB | 75.3% | +0.5% absolute |
| Xhosa | 1,240 MB | 305 MB | 75.4% | +0.6% absolute |
| Shona | 1,238 MB | 303 MB | 75.5% | +0.4% absolute |
| Malagasy | 1,236 MB | 301 MB | 75.7% | +0.3% absolute |
| Sesotho | 1,239 MB | 304 MB | 75.5% | +0.5% absolute |
| Lingala | 1,242 MB | 306 MB | 75.4% | +0.4% absolute |

### 3.3 Streaming Inference with Chunked Audio

Long audio recordings are processed in chunks to avoid memory overflow and provide incremental results:

```python
import numpy as np
from typing import List, Tuple
import torch

class StreamingTranscriber:
    """
    Processes long audio in chunks for memory-efficient streaming inference.
    
    Strategy:
    1. Split audio into 10-second overlapping chunks
    2. Process each chunk independently
    3. Merge results with overlap compensation
    4. Provide incremental results as chunks complete
    """
    
    CHUNK_LENGTH_S = 10     # 10 seconds per chunk
    OVERLAP_S = 1.5          # 1.5 second overlap for continuity
    SAMPLE_RATE = 16000
    
    def __init__(self, processor, model, device='cpu'):
        self.processor = processor
        self.model = model
        self.device = device
        self.model.to(device)
        self.model.eval()
    
    def transcribe_streaming(self, 
                               audio: np.ndarray,
                               callback=None) -> str:
        """
        Transcribe audio in chunks with optional callback.
        
        Args:
            audio: Full audio waveform (1D numpy array, 16kHz)
            callback: Optional callback(chunk_idx, chunk_text, is_final)
                      called after each chunk is processed
        
        Returns:
            Full transcribed text
        """
        chunk_size = int(self.CHUNK_LENGTH_S * self.SAMPLE_RATE)
        overlap_size = int(self.OVERLAP_S * self.SAMPLE_RATE)
        step_size = chunk_size - overlap_size
        
        all_text = []
        n_chunks = max(1, (len(audio) - chunk_size) // step_size + 1)
        
        for i in range(n_chunks):
            start = i * step_size
            end = min(start + chunk_size, len(audio))
            
            if end - start < self.SAMPLE_RATE:  # Skip chunks < 1 second
                continue
            
            chunk = audio[start:end]
            chunk_text = self._transcribe_chunk(chunk)
            
            # Clean up overlap artifacts
            if i > 0 and len(chunk_text) > 5:
                # Remove first ~3 words (likely from overlap)
                words = chunk_text.split()
                chunk_text = ' '.join(words[3:])
            
            all_text.append(chunk_text)
            
            if callback:
                is_final = (i == n_chunks - 1)
                callback(i, chunk_text, is_final)
        
        return ' '.join(all_text).strip()
    
    def _transcribe_chunk(self, chunk: np.ndarray) -> str:
        """Transcribe a single audio chunk."""
        inputs = self.processor(
            chunk, 
            sampling_rate=self.SAMPLE_RATE, 
            return_tensors="pt",
            padding=True
        ).to(self.device)
        
        with torch.no_grad():
            logits = self.model(**inputs).logits
        
        predicted_ids = torch.argmax(logits, dim=-1)
        text = self.processor.decode(predicted_ids[0])
        
        return text


class StreamingASRServer:
    """
    WebSocket-based streaming ASR server for real-time transcription.
    
    Clients send audio chunks as they're recorded, and receive
    incremental transcription results.
    """
    
    def __init__(self, transcriber: StreamingTranscriber):
        self.transcriber = transcriber
        self.active_sessions = {}
    
    async def handle_websocket(self, websocket):
        """Handle a WebSocket connection for streaming ASR."""
        session_id = str(id(websocket))
        self.active_sessions[session_id] = {
            'audio_buffer': np.array([], dtype=np.float32),
            'full_text': '',
        }
        
        try:
            async for message in websocket:
                if isinstance(message, bytes):
                    # Binary message = audio chunk
                    audio_chunk = np.frombuffer(message, dtype=np.float32)
                    session = self.active_sessions[session_id]
                    session['audio_buffer'] = np.concatenate([
                        session['audio_buffer'], audio_chunk
                    ])
                    
                    # If we have enough for a chunk, process it
                    if len(session['audio_buffer']) >= self.transcriber.CHUNK_LENGTH_S * 16000:
                        text = self.transcriber._transcribe_chunk(
                            session['audio_buffer'][:self.transcriber.CHUNK_LENGTH_S * 16000]
                        )
                        session['full_text'] += ' ' + text
                        await websocket.send(json.dumps({
                            'type': 'partial',
                            'text': session['full_text'].strip(),
                            'chunk_text': text,
                        }))
                
                elif isinstance(message, str):
                    data = json.loads(message)
                    if data.get('type') == 'end':
                        # Final processing of remaining audio
                        session = self.active_sessions[session_id]
                        if len(session['audio_buffer']) > 16000:
                            text = self.transcriber._transcribe_chunk(
                                session['audio_buffer']
                            )
                            session['full_text'] += ' ' + text
                        
                        await websocket.send(json.dumps({
                            'type': 'final',
                            'text': session['full_text'].strip(),
                        }))
        
        finally:
            del self.active_sessions[session_id]
```

### 3.4 Target Languages

| # | Language | ISO 639-3 | Speakers (M) | Countries | Script | WER Target |
|---|----------|-----------|-------------|-----------|--------|------------|
| 1 | Swahili | swa | 100+ | Tanzania, Kenya, Uganda, DRC | Latin | < 15% |
| 2 | Yoruba | yor | 45+ | Nigeria, Benin, Togo | Latin | < 18% |
| 3 | Hausa | hau | 75+ | Nigeria, Niger, Ghana, Cameroon | Latin (Ajami) | < 18% |
| 4 | Amharic | amh | 57+ | Ethiopia | Ge'ez (Ethiopic) | < 22% |
| 5 | Luganda | lug | 20+ | Uganda | Latin | < 18% |
| 6 | Kinyarwanda | kin | 12+ | Rwanda | Latin | < 18% |
| 7 | Zulu | zul | 27+ | South Africa | Latin | < 20% |
| 8 | Xhosa | xho | 19+ | South Africa | Latin | < 20% |
| 9 | Shona | sna | 11+ | Zimbabwe, Mozambique | Latin | < 18% |
| 10 | Malagasy | mlg | 25+ | Madagascar | Latin | < 18% |
| 11 | Sesotho | sot | 8+ | Lesotho, South Africa | Latin | < 18% |
| 12 | Lingala | lin | 25+ | DRC, Congo, CAR | Latin | < 18% |

**Total primary coverage**: ~425 million speakers across 25+ countries.

---

## 4. Language Detection Pipeline

### 4.1 fastText LID Model

```python
#!/usr/bin/env python3
"""
Language Identification (LID) Module
=====================================

Uses fastText's language identification model to detect the language
of incoming audio before routing to the appropriate ASR model.

The fastText LID model (lid.176.bin) supports 176 languages and
can identify language from as little as 1-3 seconds of transcribed audio.
"""

import fasttext
import numpy as np
import tempfile
import os
import logging
from typing import Tuple, Optional
from dataclasses import dataclass

logger = logging.getLogger('udara.voice.lid')


@dataclass
class LIDResult:
    """Language identification result."""
    language_code: str       # ISO 639-3 code
    language_name: str       # Full name
    confidence: float        # 0.0 to 1.0
    is_african_target: bool  # Whether it's one of our 12 target languages
    alternatives: list       # Top-5 alternative predictions


SUPPORTED_LANGUAGES = {
    'swa': 'Swahili',
    'yor': 'Yoruba',
    'hau': 'Hausa',
    'amh': 'Amharic',
    'lug': 'Luganda',
    'kin': 'Kinyarwanda',
    'zul': 'Zulu',
    'xho': 'Xhosa',
    'sna': 'Shona',
    'mlg': 'Malagasy',
    'sot': 'Sesotho',
    'lin': 'Lingala',
    # Fallback languages (non-target but handled)
    'eng': 'English',
    'fra': 'French',
    'por': 'Portuguese',
    'ara': 'Arabic',
}

# fastText ISO code mapping (fastText uses ISO 639-1 where possible)
FASTTEXT_TO_ISO3 = {
    'sw': 'swa',  # Swahili
    'yo': 'yor',  # Yoruba
    'ha': 'hau',  # Hausa
    'am': 'amh',  # Amharic
    'lg': 'lug',  # Luganda
    'rw': 'kin',  # Kinyarwanda
    'zu': 'zul',  # Zulu
    'xh': 'xho',  # Xhosa
    'sn': 'sna',  # Shona
    'mg': 'mlg',  # Malagasy
    'st': 'sot',  # Sesotho
    'ln': 'lin',  # Lingala
}


class LanguageIdentifier:
    """
    Two-stage language identification:
    
    Stage 1: fastText LID on initial transcription (any ASR model)
    Stage 2: Confirm with user if confidence < 0.7
    """
    
    CONFIDENCE_THRESHOLD_AUTO = 0.7     # Auto-accept above this
    CONFIDENCE_THRESHOLD_REJECT = 0.4   # Reject below this
    CONFIDENCE_THRESHOLD_CONFIRM = 0.4  # Ask confirmation between 0.4-0.7
    
    def __init__(self, lid_model_path: str = "models/lid.176.bin"):
        self.model = fasttext.load_model(lid_model_path)
        logger.info(f"Loaded fastText LID model from {lid_model_path}")
    
    def identify(self, text: str, 
                  top_k: int = 5) -> LIDResult:
        """
        Identify language from transcribed text.
        
        Args:
            text: Transcribed text (can be partial, 1-3 sentences)
            top_k: Number of top predictions to return
        
        Returns:
            LIDResult with language identification and confidence
        """
        if not text or len(text.strip()) < 3:
            return LIDResult(
                language_code='und',
                language_name='Unknown',
                confidence=0.0,
                is_african_target=False,
                alternatives=[]
            )
        
        # fastText prediction
        predictions = self.model.predict(text.replace('\n', ' '), k=top_k)
        
        # Parse predictions
        results = []
        for label, score in zip(predictions[0], predictions[1]):
            # fastText labels are like '__label__sw'
            ft_code = label.replace('__label__', '')
            iso3 = FASTTEXT_TO_ISO3.get(ft_code, ft_code)
            name = SUPPORTED_LANGUAGES.get(iso3, ft_code)
            
            results.append({
                'ft_code': ft_code,
                'iso3': iso3,
                'name': name,
                'confidence': float(score),
            })
        
        top = results[0]
        
        return LIDResult(
            language_code=top['iso3'],
            language_name=top['name'],
            confidence=top['confidence'],
            is_african_target=top['iso3'] in SUPPORTED_LANGUAGES,
            alternatives=results[:top_k],
        )
    
    def identify_from_audio(self, audio: np.ndarray,
                             default_asr_processor=None,
                             default_asr_model=None) -> Tuple[LIDResult, str]:
        """
        Full pipeline: audio → quick ASR → LID.
        
        Uses a default (Swahili) ASR model for initial transcription,
        then identifies the actual language.
        """
        # Quick transcription with default model
        if default_asr_processor and default_asr_model:
            inputs = default_asr_processor(
                audio, sampling_rate=16000, return_tensors="pt"
            )
            with torch.no_grad():
                logits = default_asr_model(**inputs).logits
            predicted_ids = torch.argmax(logits, dim=-1)
            preliminary_text = default_asr_processor.decode(predicted_ids[0])
        else:
            preliminary_text = ""
        
        # Language identification
        lid_result = self.identify(preliminary_text)
        
        return lid_result, preliminary_text
```

### 4.2 Confidence Thresholds & Fallback

```
┌──────────────────────────────────────────────────────────────────┐
│              LANGUAGE DETECTION DECISION TREE                    │
│                                                                  │
│                    Audio Received                                 │
│                         │                                         │
│                    Quick ASR (default)                            │
│                         │                                         │
│                    fastText LID                                   │
│                         │                                         │
│              ┌──────────┴──────────┐                              │
│              │                     │                              │
│         confidence ≥ 0.7     confidence < 0.7                    │
│              │                     │                              │
│      ┌───────┴──────┐      ┌──────┴──────┐                      │
│      │              │      │              │                      │
│  Is target      Not target  0.4-0.7    < 0.4                    │
│  language?      language?    │           │                       │
│      │              │      Ask user    Reject,                  │
│      │              │      to confirm   ask to                  │
│      ▼              ▼      language     retry                    │
│  Route to      Try English           in English                  │
│  correct ASR   ASR fallback                                      │
│  model         then confirm                                       │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ CONFIDENCE BANDS                                        │    │
│  │                                                         │    │
│  │ ≥ 0.70: AUTO-ACCEPT — proceed with detected language   │    │
│  │ 0.40-0.70: CONFIRM — ask user "Is this [language]?"    │    │
│  │ < 0.40: REJECT — ask user to retry or type instead     │    │
│  │                                                         │    │
│  │ Note: If detected language is NOT one of our 12        │    │
│  │ target languages, we try English ASR as fallback.      │    │
│  │ If that fails, we route to cloud for expanded support.  │    │
│  └─────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

### 4.3 Language Confirmation Flow

```python
class LanguageConfirmationHandler:
    """
    Handle language confirmation via the user's preferred channel.
    """
    
    CONFIRMATION_PROMPTS = {
        'swa': {
            'whatsapp': "Nimegundua unazungumza Kiswahili. Je, ni sahihi? "
                       "(Tuma 'Ndiyo' au '1' kudhibitisha)",
            'telegram': "I detected Swahili. Is this correct? "
                       "(Reply 'Yes' or '1' to confirm)",
            'ussd': "1.Kiswahili\n2.English\n3.Luganda",
        },
        'yor': {
            'whatsapp': "Mo ṣe dédé Yorùbá ni? Jẹ́ o yẹ́? "
                       "(Fi 'Bẹ́ẹni' tàbí '1' bọwọ fún mi)",
        },
        'eng': {
            'whatsapp': "I detected English. Is this correct? "
                       "(Reply 'Yes' or '1')",
        },
        'default': {
            'whatsapp': "What language are you speaking?\n"
                       "1. Swahili\n2. Yoruba\n3. Hausa\n4. Amharic\n"
                       "5. Luganda\n6. English",
        }
    }
    
    async def send_confirmation(self, channel: str, language_code: str,
                                  user_id: str) -> str:
        """
        Send language confirmation prompt to user.
        
        Returns:
            The prompt message sent
        """
        prompts = self.CONFIRMATION_PROMPTS.get(
            language_code, 
            self.CONFIRMATION_PROMPTS['default']
        )
        
        prompt = prompts.get(channel, prompts.get('whatsapp', ''))
        
        # Send via appropriate channel
        await self._send_message(channel, user_id, prompt)
        
        return prompt
    
    async def handle_confirmation_response(self, response: str) -> Tuple[bool, Optional[str]]:
        """
        Process user's confirmation response.
        
        Returns:
            (confirmed: bool, corrected_language: Optional[str])
        """
        positive_responses = {
            'swa': ['ndiyo', '1', 'yes', 'sawa', 'kweli'],
            'eng': ['yes', '1', 'correct', 'right'],
        }
        
        response_lower = response.strip().lower()
        
        if response_lower in ['1', 'yes', 'ndiyo', 'bẹ́ẹni', 'eeyah', '正确']:
            return True, None
        elif response_lower.isdigit():
            # User selected a language number
            lang_map = {
                '1': 'swa', '2': 'eng', '3': 'yor', '4': 'hau',
                '5': 'amh', '6': 'lug', '7': 'kin', '8': 'zul',
            }
            return False, lang_map.get(response_lower)
        
        return False, None
```

---

## 5. ASR Pipeline Architecture

### 5.1 Complete Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│                    ASR PIPELINE ARCHITECTURE                          │
│                                                                       │
│  ┌────────┐   ┌──────────────┐   ┌───────────┐   ┌───────────┐    │
│  │ Audio  │──►│ Preprocess   │──►│ Language  │──►│ MMS-ASR   │    │
│  │ Input  │   │              │   │ Detection │   │ Inference │    │
│  └────────┘   └──────────────┘   └───────────┘   └─────┬─────┘    │
│                                                           │          │
│  Channels:                                                ▼          │
│  • WhatsApp voice (ogg/opus)                     ┌───────────────┐   │
│  • Telegram voice (ogg/opus)                     │ Post-process  │   │
│  • Web mic (webm/opus)                           │               │   │
│  • Phone IVR (pcm/wav)                           │ • Number norm │   │
│  • USB mic on RPi (wav)                          │ • Med vocab   │   │
│                                                  │ • Drug names  │   │
│                                                  └──────┬────────┘   │
│                                                         │            │
│                                                  ┌──────▼────────┐   │
│                                                  │  Confidence   │   │
│                                                  │  Scoring      │   │
│                                                  └──────┬────────┘   │
│                                                         │            │
│                    ┌────────────────┬────────────────────┤            │
│                    │                │                    │            │
│              confidence ≥ 0.85  0.5-0.85          < 0.5  │            │
│                    │                │                    │            │
│                    ▼                ▼                    ▼            │
│             ┌──────────┐   ┌────────────┐      ┌──────────┐       │
│             │  AUTO    │   │  FLAG FOR  │      │  REJECT  │       │
│             │  ACCEPT  │   │  REVIEW    │      │  RETRY   │       │
│             │          │   │            │      │          │       │
│             │ → Store  │   │ → Show     │      │ → Ask to │       │
│             │ → Struct │   │   to super- │      │   re-    │       │
│             │ → Map    │   │   visor for │      │   record │       │
│             │ → Alert  │   │   confirm   │      │   or type│       │
│             └──────────┘   └────────────┘      └──────────┘       │
└──────────────────────────────────────────────────────────────────────┘
```

### 5.2 Audio Preprocessing

```python
import librosa
import soundfile as sf
import numpy as np
from typing import Tuple

class AudioPreprocessor:
    """
    Preprocess audio from various input formats to the format
    expected by MMS-ASR (16kHz, mono, float32, PCM).
    
    Handles:
    - Format conversion (ogg/opus, webm, mp3, wav, pcm)
    - Sample rate normalization
    - Channel mixing (stereo → mono)
    - Noise reduction (spectral gating)
    - Volume normalization
    - Silence trimming
    """
    
    TARGET_SR = 16000       # MMS-ASR expects 16kHz
    TARGET_DTYPE = np.float32
    MAX_DURATION_S = 120    # Max 2 minutes per voice note
    MIN_DURATION_S = 1      # Min 1 second (too short = noise)
    
    def preprocess(self, audio_bytes: bytes, 
                    input_format: str = 'ogg') -> Tuple[np.ndarray, dict]:
        """
        Preprocess audio bytes to model-ready format.
        
        Args:
            audio_bytes: Raw audio bytes
            input_format: 'ogg', 'webm', 'wav', 'mp3', 'pcm'
        
        Returns:
            (waveform, metadata) where waveform is float32 numpy array
        """
        metadata = {'original_format': input_format}
        
        # Step 1: Decode audio
        waveform, sr = self._decode(audio_bytes, input_format)
        metadata['original_sr'] = sr
        metadata['original_duration'] = len(waveform) / sr
        
        # Step 2: Validate duration
        duration = metadata['original_duration']
        if duration > self.MAX_DURATION_S:
            waveform = waveform[:int(self.MAX_DURATION_S * sr)]
            metadata['truncated'] = True
        elif duration < self.MIN_DURATION_S:
            raise ValueError(f"Audio too short: {duration:.1f}s < {self.MIN_DURATION_S}s")
        
        # Step 3: Convert to mono
        if len(waveform.shape) > 1:
            waveform = np.mean(waveform, axis=1)
            metadata['channels'] = waveform.shape[1] if len(waveform.shape) > 1 else 1
        
        # Step 4: Resample to 16kHz
        if sr != self.TARGET_SR:
            waveform = librosa.resample(
                waveform.astype(np.float32), 
                orig_sr=sr, 
                target_sr=self.TARGET_SR
            )
            metadata['resampled'] = True
        
        # Step 5: Normalize volume (peak normalization to -3dB)
        waveform = self._normalize_volume(waveform)
        metadata['normalized'] = True
        
        # Step 6: Noise reduction
        waveform = self._reduce_noise(waveform)
        metadata['noise_reduced'] = True
        
        # Step 7: Trim silence
        waveform = self._trim_silence(waveform)
        metadata['silence_trimmed'] = True
        
        # Ensure float32
        waveform = waveform.astype(self.TARGET_DTYPE)
        
        metadata['final_sr'] = self.TARGET_SR
        metadata['final_duration'] = len(waveform) / self.TARGET_SR
        metadata['final_samples'] = len(waveform)
        
        return waveform, metadata
    
    def _decode(self, audio_bytes: bytes, 
                 fmt: str) -> Tuple[np.ndarray, int]:
        """Decode audio bytes to numpy array."""
        import io
        
        if fmt in ('ogg', 'webm', 'mp3', 'wav', 'flac'):
            # Use soundfile for standard formats
            waveform, sr = sf.read(io.BytesIO(audio_bytes))
        elif fmt == 'pcm':
            # Raw PCM — need to know the format
            waveform = np.frombuffer(audio_bytes, dtype=np.int16).astype(np.float32) / 32768.0
            sr = 16000
        elif fmt == 'ogg_opus':
            # WhatsApp/Telegram voice messages
            # Requires ffmpeg for opus decoding
            import subprocess
            proc = subprocess.run(
                ['ffmpeg', '-i', 'pipe:0', '-f', 'wav', '-ar', '16000',
                 '-ac', '1', 'pipe:1'],
                input=audio_bytes,
                capture_output=True,
            )
            waveform, sr = sf.read(io.BytesIO(proc.stdout))
        else:
            raise ValueError(f"Unsupported format: {fmt}")
        
        return waveform, sr
    
    def _normalize_volume(self, waveform: np.ndarray,
                            target_db: float = -3.0) -> np.ndarray:
        """Peak normalize to target dB."""
        peak = np.max(np.abs(waveform))
        if peak > 0:
            target_amp = 10 ** (target_db / 20)
            waveform = waveform * (target_amp / peak)
        return waveform
    
    def _reduce_noise(self, waveform: np.ndarray,
                       noise_threshold_db: float = -40.0) -> np.ndarray:
        """
        Simple spectral gating noise reduction.
        
        More sophisticated methods (RNNoise, DeepFilterNet) available
        on cloud; edge uses this lightweight approach.
        """
        # Compute STFT
        stft = librosa.stft(waveform)
        magnitude = np.abs(stft)
        phase = np.angle(stft)
        
        # Estimate noise floor
        noise_floor = np.median(magnitude, axis=1, keepdims=True)
        threshold = noise_floor * (10 ** (noise_threshold_db / 20))
        
        # Gate: zero out bins below threshold
        mask = magnitude > threshold
        magnitude = magnitude * mask
        
        # Inverse STFT
        cleaned = librosa.istft(magnitude * np.exp(1j * phase))
        
        # Match length
        if len(cleaned) < len(waveform):
            cleaned = np.pad(cleaned, (0, len(waveform) - len(cleaned)))
        elif len(cleaned) > len(waveform):
            cleaned = cleaned[:len(waveform)]
        
        return cleaned
    
    def _trim_silence(self, waveform: np.ndarray,
                       top_db: float = 30) -> np.ndarray:
        """Trim leading and trailing silence."""
        trimmed, _ = librosa.effects.trim(waveform, top_db=top_db)
        
        # Don't over-trim (keep at least 0.5s of context)
        min_samples = int(0.5 * self.TARGET_SR)
        if len(trimmed) < min_samples:
            return waveform
        
        return trimmed
```

### 5.3 Feature Extraction

MMS-ASR uses the wav2vec 2.0 feature extractor, which operates on raw audio waveforms. No manual feature extraction (MFCC, etc.) is needed — the model learns features end-to-end. However, the Wav2Vec2Processor handles normalization internally.

### 5.4 Post-Processing

```python
class MedicalPostProcessor:
    """
    Post-process ASR output for AMR reporting context.
    
    Handles:
    - Number normalization ("elfu moja" → "1,000")
    - Medical term correction ("amoxicilin" → "Amoxicillin")
    - Drug name alias mapping
    - Unit normalization ("mg" → "mg", "gram" → "g")
    - Structured field extraction
    """
    
    # Common misspellings in African English/French accents
    DRUG_NAME_CORRECTIONS = {
        'amoxicilin': 'Amoxicillin',
        'amoxiciline': 'Amoxicillin',
        'amoxycillin': 'Amoxicillin',
        'amoxil': 'Amoxicillin',
        'cipro': 'Ciprofloxacin',
        'ciproflox': 'Ciprofloxacin',
        'ciproflaxin': 'Ciprofloxacin',
        'ciprofloxacine': 'Ciprofloxacin',
        'septrin': 'Co-trimoxazole',
        'bactrim': 'Co-trimoxazole',
        'cotrim': 'Co-trimoxazole',
        'septran': 'Co-trimoxazole',
        'ceftriaxon': 'Ceftriaxone',
        'ceftriaxone': 'Ceftriaxone',
        'rocephin': 'Ceftriaxone',
        'gentamycin': 'Gentamicin',
        'gentamicine': 'Gentamicin',
        'genta': 'Gentamicin',
        'azithro': 'Azithromycin',
        'zithromax': 'Azithromycin',
        'metronidazol': 'Metronidazole',
        'flagyl': 'Metronidazole',
        'chloramphenicol': 'Chloramphenicol',
        'augmentin': 'Amoxicillin-Clavulanate',
        'amox-clav': 'Amoxicillin-Clavulanate',
    }
    
    # Number words in target languages
    NUMBER_WORDS = {
        'swa': {
            'sifuri': 0, 'moja': 1, 'mbili': 2, 'tatu': 3, 'nne': 4,
            'tano': 5, 'sita': 6, 'saba': 7, 'nane': 8, 'tisa': 9,
            'kumi': 10, 'elfu': 1000, 'milioni': 1000000,
        },
        'yor': {
            'o零': 0, 'ọkan': 1, 'eji': 2, 'ẹta': 3, 'ẹrin': 4,
            'arun': 5, 'ẹfa': 6, 'ẹje': 7, 'ẹjọ': 8, 'ẹsàn': 9,
            'ẹwá': 10,
        },
        'hau': {
            'sifiri': 0, 'daya': 1, 'biyu': 2, 'uku': 3, 'hudu': 4,
            'biyar': 5, 'shida': 6, 'bakwai': 7, 'takwas': 8, 'tara': 9,
            'goma': 10, 'dari': 100, ' dubu': 1000,
        },
    }
    
    # Dosage patterns
    DOSAGE_PATTERN = re.compile(
        r'(\d+(?:[.,]\d+)?)\s*(mg|g|ml|mcg|microgram|milligram|gram|milliliter)',
        re.IGNORECASE
    )
    
    # Date patterns
    DATE_PATTERNS = [
        re.compile(r'(\d{1,2})[/\-](\d{1,2})[/\-](\d{2,4})'),  # DD/MM/YYYY
        re.compile(r'(\d{4})[-/](\d{1,2})[-/](\d{1,2})'),       # YYYY-MM-DD
        re.compile(r'(\d{1,2})[/\-](\d{1,2})'),                  # MM/YYYY
    ]
    
    def process(self, text: str, language_code: str = 'eng') -> dict:
        """
        Full post-processing pipeline.
        
        Args:
            text: Raw ASR output
            language_code: ISO 639-3 language code
        
        Returns:
            {
                'cleaned_text': str,
                'normalized_text': str,
                'drug_mentions': List[str],
                'dosage_mentions': List[str],
                'structured_fields': Dict,
            }
        """
        result = {
            'original_text': text,
            'cleaned_text': text,
            'normalized_text': text,
            'drug_mentions': [],
            'dosage_mentions': [],
            'structured_fields': {},
        }
        
        # Step 1: Lowercase for matching
        text_lower = text.lower()
        
        # Step 2: Fix common ASR errors
        text_lower = self._fix_asr_errors(text_lower)
        
        # Step 3: Normalize numbers
        text_normalized = self._normalize_numbers(text_lower, language_code)
        
        # Step 4: Correct drug names
        text_corrected, drug_mentions = self._correct_drug_names(text_normalized)
        result['drug_mentions'] = drug_mentions
        
        # Step 5: Extract dosage information
        dosages = self._extract_dosages(text_corrected)
        result['dosage_mentions'] = dosages
        
        # Step 6: Extract structured fields
        result['structured_fields'] = self._extract_fields(
            text_corrected, drug_mentions, dosages
        )
        
        result['cleaned_text'] = text_corrected
        result['normalized_text'] = text_normalized
        
        return result
    
    def _fix_asr_errors(self, text: str) -> str:
        """Fix common ASR transcription errors."""
        corrections = {
            'resistense': 'resistance',
            'resistances': 'resistance',
            'susceptible': 'susceptible',
            'susceptibel': 'susceptible',
            'pathogen': 'pathogen',
            'patogen': 'pathogen',
            'bacteria': 'bacteria',
            'bacterium': 'bacteria',
            'antibiotic': 'antibiotic',
            'antibiyotik': 'antibiotic',
            'culture': 'culture',
            'sensitivity': 'sensitivity',
            'sensitiviti': 'sensitivity',
            'patient': 'patient',
            'sampuli': 'sample',
        }
        
        for wrong, correct in corrections.items():
            text = text.replace(wrong, correct)
        
        return text
    
    def _normalize_numbers(self, text: str, 
                            language_code: str) -> str:
        """Convert number words to digits."""
        number_map = self.NUMBER_WORDS.get(language_code, self.NUMBER_WORDS.get('swa', {}))
        
        for word, digit in sorted(number_map.items(), key=lambda x: -len(x[0])):
            text = text.replace(word, str(digit))
        
        return text
    
    def _correct_drug_names(self, text: str) -> Tuple[str, List[str]]:
        """Find and correct drug name mentions."""
        found_drugs = []
        
        for misspelling, correct_name in self.DRUG_NAME_CORRECTIONS.items():
            # Use word boundary matching
            pattern = re.compile(r'\b' + re.escape(misspelling) + r'\b', re.IGNORECASE)
            if pattern.search(text):
                text = pattern.sub(correct_name, text)
                if correct_name not in found_drugs:
                    found_drugs.append(correct_name)
        
        return text, found_drugs
    
    def _extract_dosages(self, text: str) -> List[str]:
        """Extract dosage mentions from text."""
        dosages = []
        for match in self.DOSAGE_PATTERN.finditer(text):
            dosages.append(f"{match.group(1)}{match.group(2)}")
        return dosages
    
    def _extract_fields(self, text: str, drugs: List[str], 
                         dosages: List[str]) -> Dict:
        """Extract structured AMR reporting fields from text."""
        fields = {}
        
        # Drug name
        if drugs:
            fields['drug_name'] = drugs[0]
            fields['all_drugs_mentioned'] = drugs
        
        # Dosage
        if dosages:
            fields['dosage'] = dosages[0]
            fields['all_dosages'] = dosages
        
        # Resistance indicator
        resistance_keywords = ['resistant', 'resistance', 'rr', 'r rate', 
                              'not responding', 'failed', 'treatment failure']
        susceptible_keywords = ['sensitive', 'susceptible', 'ss', 's rate',
                                'responding', 'responded']
        
        text_lower = text.lower()
        
        if any(kw in text_lower for kw in resistance_keywords):
            fields['resistance_status'] = 'resistant'
        elif any(kw in text_lower for kw in susceptible_keywords):
            fields['resistance_status'] = 'susceptible'
        
        # Pathogen keywords
        pathogen_keywords = {
            'ecoli': 'E. coli', 'e coli': 'E. coli', 'e-coli': 'E. coli',
            'klebsiella': 'Klebsiella spp.', 'pneumoniae': 'K. pneumoniae',
            'salmonella': 'Salmonella spp.', 'typhi': 'S. Typhi',
            'staphylococcus': 'Staphylococcus spp.', 'staph': 'S. aureus',
            'streptococcus': 'Streptococcus spp.', 'gonorrhoeae': 'N. gonorrhoeae',
        }
        
        for keyword, pathogen in pathogen_keywords.items():
            if keyword in text_lower:
                fields['pathogen'] = pathogen
                break
        
        # Age (if mentioned)
        age_match = re.search(r'(?:age|years? old|miaka)\s*(\d+)', text_lower)
        if age_match:
            fields['patient_age'] = int(age_match.group(1))
        
        return fields
```

### 5.5 Confidence Scoring

```python
class ASRConfidenceScorer:
    """
    Score the confidence of ASR transcription.
    
    Uses multiple signals:
    1. CTC posterior probability (model certainty)
    2. Language model perplexity (text fluency)
    3. Medical vocabulary match rate
    4. Audio quality metrics (SNR, duration)
    """
    
    def score(self, logits: torch.Tensor, 
              predicted_ids: torch.Tensor,
              audio_metadata: dict,
              post_processed_text: str) -> dict:
        """
        Compute composite confidence score.
        
        Returns:
            {
                'overall_confidence': float (0-1),
                'ctc_confidence': float (0-1),
                'vocabulary_confidence': float (0-1),
                'audio_quality_confidence': float (0-1),
                'decision': 'accept' | 'review' | 'reject',
                'issues': List[str],
            }
        """
        # 1. CTC confidence (average max probability per frame)
        probs = torch.softmax(logits, dim=-1)
        max_probs, _ = probs.max(dim=-1)
        ctc_confidence = float(max_probs.mean())
        
        # 2. Vocabulary confidence (how many words are recognized)
        vocab_confidence = self._vocabulary_score(post_processed_text)
        
        # 3. Audio quality confidence
        audio_confidence = self._audio_quality_score(audio_metadata)
        
        # 4. Weighted composite
        overall = (
            0.5 * ctc_confidence +
            0.3 * vocab_confidence +
            0.2 * audio_confidence
        )
        
        # Decision
        if overall >= 0.85:
            decision = 'accept'
        elif overall >= 0.50:
            decision = 'review'
        else:
            decision = 'reject'
        
        # Issues
        issues = []
        if ctc_confidence < 0.6:
            issues.append('low_ctc_confidence')
        if vocab_confidence < 0.4:
            issues.append('many_unknown_words')
        if audio_confidence < 0.5:
            issues.append('poor_audio_quality')
        
        return {
            'overall_confidence': overall,
            'ctc_confidence': ctc_confidence,
            'vocabulary_confidence': vocab_confidence,
            'audio_quality_confidence': audio_confidence,
            'decision': decision,
            'issues': issues,
        }
    
    def _vocabulary_score(self, text: str) -> float:
        """Score based on how many words match known vocabulary."""
        if not text:
            return 0.0
        
        words = text.split()
        if not words:
            return 0.0
        
        # Known word lists
        medical_words = set(self.DRUG_NAME_CORRECTIONS.values())
        medical_words.update([
            'patient', 'resistant', 'susceptible', 'resistance', 
            'treatment', 'drug', 'sample', 'culture', 'test',
            'infection', 'fever', 'cough', 'diarrhea', 'hospital',
            'clinic', 'community', 'health', 'worker',
        ])
        
        common_words = {
            'the', 'a', 'an', 'is', 'was', 'are', 'were', 'been',
            'and', 'or', 'but', 'in', 'on', 'at', 'to', 'for',
            'with', 'from', 'by', 'of', 'this', 'that', 'it',
            'i', 'we', 'you', 'they', 'he', 'she', 'my', 'our',
            'patient', 'has', 'have', 'had', 'do', 'does', 'did',
            'not', 'no', 'yes', 'one', 'two', 'three',
        }
        
        known_words = medical_words | common_words
        
        matched = sum(1 for w in words if w.lower() in known_words)
        return matched / len(words)
    
    def _audio_quality_score(self, metadata: dict) -> float:
        """Score based on audio quality metrics."""
        score = 1.0
        
        # Short audio → lower confidence
        duration = metadata.get('final_duration', 0)
        if duration < 3:
            score -= 0.3
        elif duration < 5:
            score -= 0.1
        
        # Very long audio → might have quality issues
        if duration > 60:
            score -= 0.1
        
        # If truncated
        if metadata.get('truncated', False):
            score -= 0.15
        
        return max(0.0, min(1.0, score))
```

---

## 6. Audio Input Channels

### 6.1 WhatsApp Voice Notes

```python
class WhatsAppAudioHandler:
    """
    Handle incoming WhatsApp voice notes via webhook.
    
    WhatsApp sends voice notes as ogg/opus format.
    We receive them via the Meta Cloud API webhook.
    """
    
    async def handle_webhook(self, payload: dict) -> dict:
        """
        Process incoming WhatsApp voice message webhook.
        
        Payload structure (simplified):
        {
            "object": "whatsapp_business_account",
            "entry": [{
                "changes": [{
                    "value": {
                        "messages": [{
                            "from": "255712345678",
                            "id": "wamid_abc123",
                            "type": "audio",
                            "audio": {
                                "mime_type": "audio/ogg",
                                "sha256": "...",
                                "id": "media_id_xyz"
                            },
                            "timestamp": "1705312800"
                        }]
                    }
                }]
            }]
        }
        """
        message = payload['entry'][0]['changes'][0]['value']['messages'][0]
        
        sender_phone = message['from']
        message_id = message['id']
        media_id = message['audio']['id']
        mime_type = message['audio']['mime_type']
        
        # Step 1: Download media from WhatsApp servers
        audio_bytes = await self._download_media(media_id)
        
        # Step 2: Process through voice pipeline
        result = await self.voice_pipeline.process(
            audio_bytes=audio_bytes,
            input_format='ogg_opus',
            channel='whatsapp',
            user_id=sender_phone,
            source_message_id=message_id,
        )
        
        # Step 3: Send confirmation to user
        await self._send_confirmation(sender_phone, result)
        
        return {
            'status': 'processed',
            'message_id': message_id,
            'transcription': result.text,
            'confidence': result.confidence,
            'decision': result.decision,
        }
    
    async def _download_media(self, media_id: str) -> bytes:
        """Download audio media from WhatsApp Cloud API."""
        # Get media URL
        url_response = await self.http_client.get(
            f"{self.whatsapp_api_url}/{media_id}",
            headers={'Authorization': f'Bearer {self.access_token}'}
        )
        media_url = url_response.json()['url']
        
        # Download media
        download_response = await self.http_client.get(media_url)
        return download_response.content
```

### 6.2 Telegram Voice Messages

```python
class TelegramAudioHandler:
    """
    Handle incoming Telegram voice messages via webhook.
    
    Telegram sends voice messages as ogg/opus format,
    typically 20-30 seconds per message.
    """
    
    async def handle_update(self, update: dict) -> dict:
        """
        Process Telegram bot update containing a voice message.
        
        Update structure:
        {
            "update_id": 12345,
            "message": {
                "message_id": 67,
                "from": {"id": 987654, "language_code": "sw"},
                "chat": {"id": 987654},
                "voice": {
                    "file_id": "AwADBAADbXXXXXXXX",
                    "duration": 15,
                    "mime_type": "audio/ogg"
                },
                "date": 1705312800
            }
        }
        """
        message = update['message']
        voice = message['voice']
        chat_id = message['chat']['id']
        user_id = message['from']['id']
        file_id = voice['file_id']
        duration = voice['duration']
        
        # Download voice file
        file_info = await self.bot.get_file(file_id)
        audio_bytes = await self.bot.download_file(file_info['file_path'])
        
        # Process
        result = await self.voice_pipeline.process(
            audio_bytes=audio_bytes,
            input_format='ogg_opus',
            channel='telegram',
            user_id=str(user_id),
            metadata={'duration': duration, 'language_code': message['from'].get('language_code', '')},
        )
        
        # Respond via Telegram
        if result.decision == 'accept':
            await self.bot.send_message(
                chat_id=chat_id,
                text=f"✅ Report received:\n\n{result.text}\n\n"
                     f"Drug: {result.structured_fields.get('drug_name', 'N/A')}\n"
                     f"Status: {result.structured_fields.get('resistance_status', 'N/A')}\n"
                     f"Confidence: {result.confidence:.0%}"
            )
        elif result.decision == 'review':
            await self.bot.send_message(
                chat_id=chat_id,
                text=f"⚠️ We captured your report but need to verify:\n\n"
                     f'"{result.text}"\n\nIs this correct? Reply Y/N.'
            )
        else:
            await self.bot.send_message(
                chat_id=chat_id,
                text="❌ Sorry, we couldn't understand that. "
                     "Please try again with less background noise, "
                     "or type your report as text."
            )
        
        return {'status': 'processed', 'transcription': result.text}
```

### 6.3 USSD / DTMF Fallback

```
┌──────────────────────────────────────────────────────────────────┐
│                    USSD FALLBACK FLOW                             │
│                                                                  │
│  CHW dials: *123#                                                │
│                                                                  │
│  1. UDARA AI AMR Report                                          │
│     ─────────────────                                             │
│     1. New Report                                                │
│     2. Check Points                                              │
│     3. Alerts                                                    │
│                                                                  │
│  CHW enters: 1                                                   │
│                                                                  │
│  2. Select Drug:                                                 │
│     ──────────────                                               │
│     1. Amoxicillin                                              │
│     2. Ciprofloxacin                                            │
│     3. Co-trimoxazole                                           │
│     4. Ceftriaxone                                              │
│     5. Gentamicin                                               │
│     6. Other                                                    │
│                                                                  │
│  CHW enters: 2                                                   │
│                                                                  │
│  3. Select Pathogen:                                             │
│     ────────────────                                             │
│     1. E. coli                                                  │
│     2. K. pneumoniae                                            │
│     3. S. pneumoniae                                            │
│     4. S. aureus                                                │
│     5. Other                                                    │
│                                                                  │
│  CHW enters: 1                                                   │
│                                                                  │
│  4. Result:                                                      │
│     ────────                                                     │
│     1. Resistant                                                │
│     2. Susceptible                                              │
│                                                                  │
│  CHW enters: 1                                                   │
│                                                                  │
│  5. Report submitted! Ref: RPT-20250115-0042                    │
│                                                                  │
│  Note: USSD is limited to structured menus.                     │
│  For free-form reporting, CHWs are encouraged to use voice.      │
└──────────────────────────────────────────────────────────────────┘
```

### 6.4 Web Microphone

```javascript
// Browser-based voice recording using MediaRecorder API
// Sends audio chunks to server via WebSocket

class VoiceRecorder {
  constructor(wsUrl, language = 'auto') {
    this.wsUrl = wsUrl;
    this.language = language;
    this.mediaRecorder = null;
    this.audioChunks = [];
    this.ws = null;
  }

  async start() {
    // Request microphone permission
    const stream = await navigator.mediaDevices.getUserMedia({
      audio: {
        channelCount: 1,
        sampleRate: 16000,
        echoCancellation: true,
        noiseSuppression: true,
      }
    });

    // Create MediaRecorder (opus codec)
    this.mediaRecorder = new MediaRecorder(stream, {
      mimeType: 'audio/webm;codecs=opus',
      audioBitsPerSecond: 16000,
    });

    // WebSocket connection for streaming
    this.ws = new WebSocket(this.wsUrl);
    this.ws.onopen = () => {
      this.ws.send(JSON.stringify({
        type: 'config',
        language: this.language,
        format: 'webm_opus',
      }));
    };

    // Collect audio chunks
    this.mediaRecorder.ondataavailable = (event) => {
      if (event.data.size > 0) {
        this.audioChunks.push(event.data);
        // Stream chunk to server
        if (this.ws && this.ws.readyState === WebSocket.OPEN) {
          this.ws.send(event.data);
        }
      }
    };

    this.mediaRecorder.start(1000); // Send chunks every 1s
  }

  stop() {
    this.mediaRecorder.stop();
    if (this.ws) {
      this.ws.send(JSON.stringify({ type: 'end' }));
    }
    return new Promise((resolve) => {
      this.mediaRecorder.onstop = () => {
        const blob = new Blob(this.audioChunks, { type: 'audio/webm' });
        resolve(blob);
        this.audioChunks = [];
      };
    });
  }
}
```

### 6.5 Phone Call (IVR)

```python
class IVRVoiceHandler:
    """
    Interactive Voice Response system for phone-based reporting.
    
    CHWs can call a toll-free number and report cases by voice.
    The IVR system records the audio, processes it, and reads back
    the transcription for confirmation.
    """
    
    async def handle_incoming_call(self, call: dict):
        """Handle incoming call to UDARA AI reporting line."""
        
        # Answer and greet
        await self.play_tts(
            "Karibu UDARA AI. Tafadhali sema ripoti yako baada ya [beep].",
            language='swa'
        )
        
        # Record voice report (max 60 seconds)
        audio = await self.record_audio(max_duration=60)
        
        # Process
        result = await self.voice_pipeline.process(
            audio_bytes=audio,
            input_format='wav',
            channel='phone',
            user_id=call['from'],
        )
        
        # Read back transcription for confirmation
        await self.play_tts(
            f"Nimesikia: {result.text}. "
            f"Dawa ni {result.structured_fields.get('drug_name', 'haijulikani')}. "
            f"Hali ni {result.structured_fields.get('resistance_status', 'haijulikani')}. "
            f"Ili kudhibitisha, bonyeza 1. Kukataa, bonyeza 2.",
            language='swa'
        )
        
        # Wait for DTMF response
        digit = await self.wait_for_dtmf(timeout=10)
        
        if digit == '1':
            # Save report
            report_id = await self._save_report(call['from'], result)
            await self.play_tts(
                f"Asante! Ripoti yako imerekodiwa. Namba yake ni {report_id}.",
                language='swa'
            )
        else:
            await self.play_tts(
                "Samahani, tafadhali jaribu tena.",
                language='swa'
            )
```

---

## 7. Complete Code Implementation

```python
#!/usr/bin/env python3
"""
UDARA AI — Complete Voice-First Pipeline
==========================================

End-to-end voice reporting pipeline:
Audio Input → Preprocess → LID → ASR → Post-process → Confidence → Action

Supports: WhatsApp, Telegram, Web, IVR, USB mic
Languages: 12 African languages + English fallback
"""

import asyncio
import logging
import os
import time
import torch
import numpy as np
from dataclasses import dataclass, field
from typing import Optional, Dict, List, Tuple
from enum import Enum

import torchaudio
from transformers import Wav2Vec2ForCTC, Wav2Vec2Processor

# Import our custom modules (defined above)
# from .lid import LanguageIdentifier, LIDResult
# from .preprocessor import AudioPreprocessor
# from .postprocessor import MedicalPostProcessor
# from .confidence import ASRConfidenceScorer

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger('udara.voice.pipeline')

# ============================================================
# DATA CLASSES
# ============================================================

@dataclass
class TranscriptionResult:
    """Complete result of voice transcription pipeline."""
    text: str                             # Final transcribed text
    language: str                         # Detected language code
    language_name: str                    # Language name
    confidence: float                     # Overall confidence (0-1)
    decision: str                         # 'accept', 'review', 'reject'
    audio_metadata: Dict                  # Audio preprocessing metadata
    structured_fields: Dict               # Extracted AMR fields
    drug_mentions: List[str]              # Drug names found
    dosage_mentions: List[str]            # Dosages found
    processing_time_ms: float             # Total pipeline latency
    confidence_details: Dict              # Breakdown of confidence scores
    issues: List[str] = field(default_factory=list)
    source_channel: str = 'unknown'
    source_user_id: str = ''


class PipelineDecision(Enum):
    AUTO_ACCEPT = 'accept'
    FLAG_FOR_REVIEW = 'review'
    REJECT_RETRY = 'reject'


# ============================================================
# MAIN PIPELINE CLASS
# ============================================================

class VoicePipeline:
    """
    UDARA AI Voice-First Pipeline.
    
    Orchestrates the full voice reporting flow:
    1. Audio preprocessing
    2. Language detection
    3. ASR transcription
    4. Medical post-processing
    5. Confidence scoring
    6. Decision routing
    """
    
    TARGET_LANGUAGES = [
        'swa', 'yor', 'hau', 'amh', 'lug', 'kin',
        'zul', 'xho', 'sna', 'mlg', 'sot', 'lin'
    ]
    
    LANGUAGE_NAMES = {
        'swa': 'Swahili', 'yor': 'Yoruba', 'hau': 'Hausa',
        'amh': 'Amharic', 'lug': 'Luganda', 'kin': 'Kinyarwanda',
        'zul': 'Zulu', 'xho': 'Xhosa', 'sna': 'Shona',
        'mlg': 'Malagasy', 'sot': 'Sesotho', 'lin': 'Lingala',
        'eng': 'English', 'fra': 'French', 'por': 'Portuguese',
    }
    
    def __init__(self, 
                  models_dir: str = "./models",
                  device: str = None,
                  max_loaded_models: int = 3):
        """
        Initialize voice pipeline.
        
        Args:
            models_dir: Directory containing quantized models
            device: 'cpu', 'cuda', or None (auto-detect)
            max_loaded_models: Max ASR models to keep in memory
        """
        self.models_dir = models_dir
        self.device = device or self._detect_device()
        self.max_loaded_models = max_loaded_models
        
        # Initialize components
        self.preprocessor = AudioPreprocessor()
        self.postprocessor = MedicalPostProcessor()
        self.confidence_scorer = ASRConfidenceScorer()
        
        # Initialize language identifier
        self.lid_model = self._load_lid_model()
        
        # ASR model cache (LRU)
        self.asr_models = {}
        self.asr_processors = {}
        self.model_access_order = []
        
        # Load default model (Swahili — most common)
        self.default_language = 'swa'
        self._load_asr_model(self.default_language)
        
        logger.info(f"Voice pipeline initialized on {self.device}")
        logger.info(f"Default language: {self.default_language}")
    
    def _detect_device(self) -> str:
        """Auto-detect best available device."""
        if torch.cuda.is_available():
            return 'cuda'
        return 'cpu'
    
    def _load_lid_model(self):
        """Load fastText language identification model."""
        import fasttext
        lid_path = os.path.join(self.models_dir, 'lid.176.bin')
        if os.path.exists(lid_path):
            return fasttext.load_model(lid_path)
        else:
            logger.warning(f"LID model not found at {lid_path}. "
                          "Language detection will be limited.")
            return None
    
    def _load_asr_model(self, language_code: str):
        """Load ASR model for a specific language (with LRU eviction)."""
        if language_code in self.asr_models:
            # Move to front of access order
            self.model_access_order.remove(language_code)
            self.model_access_order.append(language_code)
            return
        
        # Evict least recently used if at capacity
        while len(self.model_access_order) >= self.max_loaded_models:
            lru_lang = self.model_access_order.pop(0)
            logger.info(f"Evicting ASR model: {lru_lang}")
            del self.asr_models[lru_lang]
            del self.asr_processors[lru_lang]
        
        # Load new model
        model_path = os.path.join(
            self.models_dir, f"mms-1b-{language_code}-int8"
        )
        
        logger.info(f"Loading ASR model for {language_code} from {model_path}")
        
        try:
            processor = Wav2Vec2Processor.from_pretrained(model_path)
            model = Wav2Vec2ForCTC.from_pretrained(model_path)
            model.to(self.device)
            model.eval()
            
            self.asr_models[language_code] = model
            self.asr_processors[language_code] = processor
            self.model_access_order.append(language_code)
            
            logger.info(f"ASR model loaded for {language_code}")
        except Exception as e:
            logger.error(f"Failed to load ASR model for {language_code}: {e}")
            raise
    
    async def process(self, 
                       audio_bytes: bytes,
                       input_format: str = 'ogg',
                       channel: str = 'unknown',
                       user_id: str = '',
                       expected_language: str = None) -> TranscriptionResult:
        """
        Process audio through the full voice pipeline.
        
        This is the main entry point for voice reporting.
        """
        start_time = time.time()
        
        try:
            # Step 1: Audio Preprocessing
            t1 = time.time()
            waveform, audio_metadata = self.preprocessor.preprocess(
                audio_bytes, input_format
            )
            t_preprocess = (time.time() - t1) * 1000
            
            # Step 2: Language Detection
            t2 = time.time()
            if expected_language and expected_language in self.TARGET_LANGUAGES:
                language_code = expected_language
                lid_confidence = 1.0
            else:
                language_code, lid_confidence = await self._detect_language(
                    waveform
                )
            t_lid = (time.time() - t2) * 1000
            
            # Step 3: ASR Transcription
            t3 = time.time()
            text, logits = await self._transcribe(waveform, language_code)
            t_asr = (time.time() - t3) * 1000
            
            # Step 4: Medical Post-Processing
            t4 = time.time()
            post_result = self.postprocessor.process(text, language_code)
            t_post = (time.time() - t4) * 1000
            
            # Step 5: Confidence Scoring
            t5 = time.time()
            confidence_result = self.confidence_scorer.score(
                logits=logits,
                predicted_ids=torch.argmax(logits, dim=-1),
                audio_metadata=audio_metadata,
                post_processed_text=post_result['cleaned_text'],
            )
            t_conf = (time.time() - t5) * 1000
            
            total_time = (time.time() - start_time) * 1000
            
            return TranscriptionResult(
                text=post_result['cleaned_text'],
                language=language_code,
                language_name=self.LANGUAGE_NAMES.get(language_code, language_code),
                confidence=confidence_result['overall_confidence'],
                decision=confidence_result['decision'],
                audio_metadata=audio_metadata,
                structured_fields=post_result['structured_fields'],
                drug_mentions=post_result['drug_mentions'],
                dosage_mentions=post_result['dosage_mentions'],
                processing_time_ms=total_time,
                confidence_details={
                    'ctc_confidence': confidence_result['ctc_confidence'],
                    'vocabulary_confidence': confidence_result['vocabulary_confidence'],
                    'audio_quality': confidence_result['audio_quality_confidence'],
                    'lid_confidence': lid_confidence,
                    'timing': {
                        'preprocess_ms': t_preprocess,
                        'lid_ms': t_lid,
                        'asr_ms': t_asr,
                        'postprocess_ms': t_post,
                        'confidence_ms': t_conf,
                    }
                },
                issues=confidence_result['issues'],
                source_channel=channel,
                source_user_id=user_id,
            )
        
        except Exception as e:
            logger.error(f"Pipeline error: {e}", exc_info=True)
            return TranscriptionResult(
                text="",
                language="und",
                language_name="Unknown",
                confidence=0.0,
                decision='reject',
                audio_metadata={},
                structured_fields={},
                drug_mentions=[],
                dosage_mentions=[],
                processing_time_ms=(time.time() - start_time) * 1000,
                confidence_details={'error': str(e)},
                issues=[f'pipeline_error: {str(e)}'],
                source_channel=channel,
                source_user_id=user_id,
            )
    
    async def _detect_language(self, waveform: np.ndarray) -> Tuple[str, float]:
        """Detect language from audio."""
        if self.lid_model is None:
            return self.default_language, 1.0
        
        # Quick transcription with default model for LID
        processor = self.asr_processors[self.default_language]
        model = self.asr_models[self.default_language]
        
        inputs = processor(
            waveform, sampling_rate=16000, return_tensors="pt"
        ).to(self.device)
        
        with torch.no_grad():
            logits = model(**inputs).logits
        
        predicted_ids = torch.argmax(logits, dim=-1)
        preliminary_text = processor.decode(predicted_ids[0])
        
        # Run fastText LID
        lid_result = self.lid_model.predict(preliminary_text.replace('\n', ' '), k=5)
        
        if lid_result[0] and lid_result[1][0] >= 0.7:
            ft_code = lid_result[0][0].replace('__label__', '')
            iso3 = FASTTEXT_TO_ISO3.get(ft_code, ft_code)
            if iso3 in self.TARGET_LANGUAGES:
                return iso3, float(lid_result[1][0])
        
        # Default
        return self.default_language, 0.5
    
    async def _transcribe(self, waveform: np.ndarray, 
                           language_code: str) -> Tuple[str, torch.Tensor]:
        """Transcribe audio with the specified language model."""
        self._load_asr_model(language_code)
        
        processor = self.asr_processors[language_code]
        model = self.asr_models[language_code]
        
        inputs = processor(
            waveform, sampling_rate=16000, return_tensors="pt",
            padding=True
        ).to(self.device)
        
        with torch.no_grad():
            logits = model(**inputs).logits
        
        predicted_ids = torch.argmax(logits, dim=-1)
        text = processor.decode(predicted_ids[0])
        
        return text, logits
```

---

## 8. Medical Vocabulary & Post-Processing

### 8.1 Drug Name Normalization

CHWs often use local brand names, abbreviations, or phonetic spellings. This module maps them to standardized INN (International Nonproprietary Names).

```python
DRUG_VOCABULARY = {
    # Brand → INN mapping (West Africa focus)
    'septrin': 'Co-trimoxazole',
    'bactrim': 'Co-trimoxazole',
    'augmentin': 'Amoxicillin-Clavulanate',
    'zithromax': 'Azithromycin',
    'flagyl': 'Metronidazole',
    'rocephin': 'Ceftriaxone',
    'cipro': 'Ciprofloxacin',
    'amoxil': 'Amoxicillin',
    'genta': 'Gentamicin',
    'erythro': 'Erythromycin',
    'tetracycline': 'Tetracycline',
    'ampicillin': 'Ampicillin',
    'cloxacillin': 'Cloxacillin',
    'metronidazole': 'Metronidazole',
    'norfloxacine': 'Norfloxacin',
    'ofloxacine': 'Ofloxacin',
    'levofloxacine': 'Levofloxacin',
    'meropenem': 'Meropenem',
    'cefuroxime': 'Cefuroxime',
    'doxycycline': 'Doxycycline',
}

# Swahili drug terms
DRUG_NAMES_SWAHILI = {
    'dawa ya tumbo': 'Metronidazole',
    'dawa ya homa': 'Artemether-Lumefantrine',
    'dawa ya kufulua': 'Amoxicillin',
    'damu': 'Co-trimoxazole',
}

# Yoruba drug terms
DRUG_NAMES_YORUBA = {
    'ogun arun': 'Metronidazole',
    'ogun adagba': 'Amoxicillin',
}
```

### 8.2 Symptom Terminology Mapping

```python
SYMPTOM_VOCABULARY = {
    'swa': {
        'maumau ya tumbo': {'name': 'abdominal_pain', 'category': 'gastrointestinal'},
        'kuhara': {'name': 'diarrhea', 'category': 'gastrointestinal'},
        'kipindupindu': {'name': 'cholera', 'category': 'gastrointestinal'},
        'homa': {'name': 'fever', 'category': 'systemic'},
        'kikohozi': {'name': 'cough', 'category': 'respiratory'},
        'maumau ya kifua': {'name': 'chest_pain', 'category': 'respiratory'},
        'uchovu': {'name': 'vomiting', 'category': 'gastrointestinal'},
        'kaswende': {'name': 'dysentery', 'category': 'gastrointestinal'},
    },
    'yor': {
        'inu obara': {'name': 'fever', 'category': 'systemic'},
        'eje': {'name': 'diarrhea', 'category': 'gastrointestinal'},
        'arun inu': {'name': 'stomach_pain', 'category': 'gastrointestinal'},
    },
    'hausa': {
        'zazzarin cizon sauro': {'name': 'malaria', 'category': 'infectious'},
        'rashin lafiya': {'name': 'illness', 'category': 'systemic'},
        'ciwon kai': {'name': 'headache', 'category': 'neurological'},
    },
}
```

### 8.3 Duration & Dosage Extraction

```python
import re

class DosageExtractor:
    """Extract structured dosage info from ASR text."""
    
    DOSE_PATTERN = re.compile(
        r'(\d+)\s*(mg|g|ml|tablets?|capsules?|teaspoons?|tableti?)',
        re.IGNORECASE
    )
    DURATION_PATTERN = re.compile(
        r'(\d+)\s*(days?|siku|days?|wiki|weeks?|mwezi|months?)',
        re.IGNORECASE
    )
    FREQUENCY_PATTERN = re.compile(
        r'(mara\s+\d+|times?\s+\d+|siku\s+\d+|mara\s+moja|mara\s+mbili|'
        r'kila\s+\d+\s+masaa|once|twice|thrice|three\s+times)',
        re.IGNORECASE
    )

    def extract(self, text: str) -> dict:
        dose = self.DOSE_PATTERN.search(text)
        duration = self.DURATION_PATTERN.search(text)
        frequency = self.FREQUENCY_PATTERN.search(text)
        return {
            'dose': dose.group(0) if dose else None,
            'duration': duration.group(0) if duration else None,
            'frequency': frequency.group(0) if frequency else None,
        }
```

### 8.4 Integration with ASR Pipeline

```python
class VoicePipeline:
    """Complete voice → structured case pipeline."""
    
    def __init__(self):
        self.asr_model = load_mms_asr()
        self.lang_detector = LanguageDetector()
        self.post_processor = MedicalPostProcessor()
        self.dosage_extractor = DosageExtractor()
    
    async def process(self, audio_path: str) -> dict:
        # 1. Detect language
        lang, confidence = self.lang_detector.detect(audio_path)
        
        # 2. Transcribe
        raw_text = self.asr_model.transcribe(audio_path, language=lang)
        
        # 3. Post-process: normalize drugs, numbers, terms
        cleaned = self.post_processor.process(raw_text, lang)
        
        # 4. Extract structured fields
        dosage = self.dosage_extractor.extract(cleaned)
        
        # 5. Build structured case
        return {
            'language': lang,
            'transcription': raw_text,
            'cleaned_text': cleaned,
            'dosage': dosage,
            'confidence': confidence,
        }
```

---

## 9. Performance on Raspberry Pi 5

### 9.1 Benchmark Results

| Metric | RPi 5 (8GB) | Cloud (4 vCPU) | Ratio |
|--------|-------------|---------------|-------|
| Cold model load | 8.2s | 1.4s | 5.9× |
| Warm model load | 3.5s | 0.3s | 11.7× |
| 5s audio transcription | 1.8s | 0.4s | 4.5× |
| 10s audio transcription | 3.2s | 0.7s | 4.6× |
| 30s audio transcription | 8.7s | 1.9s | 4.6× |
| 60s audio transcription | 16.4s | 3.5s | 4.7× |
| Full pipeline (10s audio) | 4.1s | 0.9s | 4.6× |
| Memory per model | 820MB | 1.2GB | 0.7× |
| Memory with 3 models | 1.8GB | 3.6GB | 0.5× |
| Total RPi memory used | 2.4GB | N/A | — |

### 9.2 Optimization Techniques Applied

```python
# Optimization configurations for RPi 5

RPI5_OPTIMIZATIONS = {
    # Thread configuration
    'intra_op_parallelism_threads': 4,   # 4 cores on RPi 5
    'inter_op_parallelism_threads': 1,
    
    # Memory optimization
    'torch_memory_fraction': 0.7,        # Leave 30% for OS + other services
    
    # Model optimization
    'use_int8': True,                    # INT8 quantization
    'use_onnx': False,                   # ONNX Runtime not yet optimized for ARM
    
    # Audio processing
    'noise_reduction': 'light',          # Light noise reduction (faster)
    'chunk_size_s': 10,                  # 10-second chunks
    'overlap_s': 1.5,                    # 1.5s overlap
    
    # LRU cache
    'max_models_in_memory': 3,           # Max 3 language models
    'model_eviction_policy': 'lru',      # Least recently used
    
    # Batch processing
    'max_concurrent_requests': 2,        # Process 2 requests at a time
}
```

---

## 10. Error Handling & Graceful Degradation

### 10.1 Error Hierarchy

| Error Type | Severity | Recovery Action | User Message |
|-----------|----------|----------------|-------------|
| Audio too short | Warning | Reject with guidance | "Recording too short. Please speak for at least 3 seconds." |
| Audio too long | Warning | Truncate to 2 min | "We captured the first 2 minutes of your report." |
| Unsupported format | Error | Try format conversion | "Please send voice notes in standard format." |
| Model not loaded | Error | Load model (8s delay) | "Setting up language support... one moment." |
| OOM on RPi | Critical | Evict models, retry | "Processing... please wait." |
| LID confidence low | Warning | Ask user to confirm | "Are you speaking [language]? Reply 1 for Yes." |
| ASR confidence low | Warning | Flag for review | "We captured your report but need to verify. A supervisor will review." |
| Network timeout | Error | Queue for retry | "Connection issue. Your report will be sent when connectivity returns." |
| Server overload | Warning | Queue request | "System busy. Your report is queued and will be processed shortly." |

### 10.2 Retry Logic

```python
class RetryHandler:
    """Exponential backoff retry for pipeline failures."""
    
    MAX_RETRIES = 3
    BASE_DELAY_S = 2.0
    MAX_DELAY_S = 30.0
    RETRYABLE_ERRORS = ('audio_format_error', 'model_load_error', 'timeout')
    
    async def execute_with_retry(self, func, *args, **kwargs):
        """Execute function with exponential backoff retry."""
        last_error = None
        
        for attempt in range(self.MAX_RETRIES):
            try:
                return await func(*args, **kwargs)
            except Exception as e:
                last_error = e
                
                if attempt < self.MAX_RETRIES - 1:
                    delay = min(
                        self.BASE_DELAY_S * (2 ** attempt),
                        self.MAX_DELAY_S
                    )
                    jitter = delay * 0.1 * (np.random.random() - 0.5)
                    await asyncio.sleep(delay + jitter)
                    logger.warning(f"Retry {attempt + 1} after {delay:.1f}s")
        
        raise last_error
```

---

## 11. Testing & Evaluation

### 11.1 WER Evaluation Framework

```python
import jiwer  # Jiwer: Word Error Rate library

class ASREvaluator:
    """Evaluate ASR model performance."""
    
    def evaluate(self, 
                  test_set: List[Dict],
                  language: str) -> Dict:
        """
        Evaluate ASR on a test set.
        
        Args:
            test_set: List of {'audio_path': str, 'reference': str}
            language: Language code
        
        Returns:
            Evaluation metrics
        """
        hypotheses = []
        references = []
        
        for sample in test_set:
            # Load and process audio
            audio_bytes = open(sample['audio_path'], 'rb').read()
            result = await self.pipeline.process(audio_bytes, 'wav')
            hypotheses.append(result.text)
            references.append(sample['reference'])
        
        # Compute WER
        wer = jiwer.wer(references, hypotheses)
        mer = jiwer.mer(references, hypotheses)  # Match Error Rate
        wil = jiwer.wil(references, hypotheses)  # Word Information Lost
        
        # Compute CER (Character Error Rate) — useful for agglutinative languages
        cer = jiwer.cer(references, hypotheses)
        
        # Medical term accuracy
        med_acc = self._medical_term_accuracy(references, hypotheses)
        
        return {
            'language': language,
            'n_samples': len(test_set),
            'wer': wer,
            'cer': cer,
            'mer': mer,
            'wil': wil,
            'medical_accuracy': med_acc,
            'target_wer': self._get_target_wer(language),
            'meets_target': wer <= self._get_target_wer(language),
        }
    
    def _medical_term_accuracy(self, references: List[str], 
                                hypotheses: List[str]) -> float:
        """How accurately are medical terms transcribed?"""
        correct = 0
        total = 0
        
        med_terms = [
            'amoxicillin', 'ciprofloxacin', 'co-trimoxazole',
            'ceftriaxone', 'gentamicin', 'resistant', 'susceptible',
            'culture', 'sensitivity', 'pathogen', 'patient',
        ]
        
        for ref, hyp in zip(references, hypotheses):
            ref_lower = ref.lower()
            hyp_lower = hyp.lower()
            
            for term in med_terms:
                ref_has = term in ref_lower
                hyp_has = term in hyp_lower
                
                if ref_has:
                    total += 1
                    if hyp_has:
                        correct += 1
        
        return correct / total if total > 0 else 0.0
```

### 11.2 Per-Language Benchmark Results

| Language | WER (%) | CER (%) | Medical Acc (%) | Target Met |
|----------|---------|---------|-----------------|------------|
| Swahili | 12.3 | 4.1 | 94.2 | ✅ |
| Yoruba | 16.1 | 6.8 | 89.5 | ✅ |
| Hausa | 15.8 | 5.9 | 90.1 | ✅ |
| Amharic | 21.4 | 7.2 | 85.3 | ❌ (script) |
| Luganda | 14.7 | 5.3 | 91.8 | ✅ |
| Kinyarwanda | 13.2 | 4.8 | 92.4 | ✅ |
| Zulu | 18.6 | 7.1 | 87.9 | ✅ |
| Xhosa | 19.2 | 7.5 | 87.1 | ✅ |
| Shona | 15.5 | 5.6 | 90.3 | ✅ |
| Malagasy | 14.1 | 5.1 | 91.2 | ✅ |
| Sesotho | 16.8 | 6.3 | 89.8 | ✅ |
| Lingala | 14.9 | 5.4 | 90.7 | ✅ |
| **Average** | **16.1** | **5.9** | **90.3** | **11/12** |

---

## 12. Configuration Reference

| Environment Variable | Default | Description |
|---------------------|---------|-------------|
| `UDARA_VOICE_MODELS_DIR` | `./models` | Directory for ASR models |
| `UDARA_VOICE_DEFAULT_LANG` | `swa` | Default ASR language |
| `UDARA_VOICE_MAX_MODELS` | `3` | Max models in memory |
| `UDARA_VOICE_DEVICE` | `auto` | 'cpu', 'cuda', or 'auto' |
| `UDARA_VOICE_CONFIDENCE_ACCEPT` | `0.85` | Auto-accept threshold |
| `UDARA_VOICE_CONFIDENCE_REVIEW` | `0.50` | Flag for review threshold |
| `UDARA_VOICE_MAX_AUDIO_S` | `120` | Max audio duration |
| `UDARA_VOICE_NOISE_REDUCTION` | `light` | Noise reduction level |
| `UDARA_VOICE_LID_MODEL` | `./models/lid.176.bin` | fastText LID model path |

---

## 13. Monitoring & Analytics

### 13.1 Key Metrics

| Metric | Collection | Dashboard |
|--------|-----------|-----------|
| Daily voice reports | Counter | Operations |
| Average processing time | Histogram | Performance |
| WER (sampled) | Gauge | Quality |
| Language distribution | Counter | Operations |
| Channel distribution | Counter | Operations |
| Confidence distribution | Histogram | Quality |
| Reject rate | Counter | Quality |
| Model eviction count | Counter | Performance |
| Error rate by type | Counter | Reliability |

### 13.2 Prometheus Metrics

```python
from prometheus_client import Counter, Histogram, Gauge

VOICE_REPORTS_TOTAL = Counter(
    'udara_voice_reports_total',
    'Total voice reports processed',
    ['channel', 'language', 'decision']
)

VOICE_PROCESSING_TIME = Histogram(
    'udara_voice_processing_seconds',
    'Voice processing time in seconds',
    ['channel', 'language'],
    buckets=[0.5, 1, 2, 5, 10, 15, 30, 60]
)

VOICE_CONFIDENCE = Histogram(
    'udara_voice_confidence',
    'Transcription confidence scores',
    ['language', 'decision'],
    buckets=[0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 0.95, 1.0]
)

VOICE_MODELS_LOADED = Gauge(
    'udara_voice_models_loaded',
    'Number of ASR models currently loaded in memory'
)

VOICE_ERRORS_TOTAL = Counter(
    'udara_voice_errors_total',
    'Voice pipeline errors',
    ['error_type', 'channel']
)
```

---

## 14. Appendix

### A. Language Code Mapping

| Language | ISO 639-1 | ISO 639-3 | fastText Label | MMS Model ID |
|----------|-----------|-----------|---------------|-------------|
| Swahili | sw | swa | sw | facebook/mms-1b-swa |
| Yoruba | yo | yor | yo | facebook/mms-1b-yor |
| Hausa | ha | hau | ha | facebook/mms-1b-hau |
| Amharic | am | amh | am | facebook/mms-1b-amh |
| Luganda | lg | lug | lg | facebook/mms-1b-lug |
| Kinyarwanda | rw | kin | rw | facebook/mms-1b-kin |
| Zulu | zu | zul | zu | facebook/mms-1b-zul |
| Xhosa | xh | xho | xh | facebook/mms-1b-xho |
| Shona | sn | sna | sn | facebook/mms-1b-sna |
| Malagasy | mg | mlg | mg | facebook/mms-1b-mlg |
| Sesotho | st | sot | st | facebook/mms-1b-sot |
| Lingala | ln | lin | ln | facebook/mms-1b-lin |

### B. Audio Format Compatibility

| Input Source | Format | Codec | Sample Rate | Channels | Conversion Needed |
|-------------|--------|-------|-------------|----------|-------------------|
| WhatsApp | ogg | opus | 48kHz | mono | Resample to 16kHz |
| Telegram | ogg | opus | 48kHz | mono | Resample to 16kHz |
| Web (Chrome) | webm | opus | 48kHz | mono | Resample to 16kHz |
| Web (Safari) | mp4 | AAC | 44.1kHz | mono | Resample to 16kHz |
| Phone IVR | wav | PCM | 8kHz | mono | Resample to 16kHz |
| RPi USB mic | wav | PCM | 16kHz | mono | None |
| File upload | various | various | various | various | Depends |

### C. References

1. Pratap, V. et al. (2023). *Scaling Speech Technology to 1,000+ Languages*. Meta AI.
2. Schneider, S. et al. (2019). *wav2vec 2.0: A Framework for Self-Supervised Learning of Speech Representations*. NeurIPS.
3. Joulin, A. et al. (2017). *Bag of Tricks for Efficient Text Classification*. fastText.
4. WHO (2023). *Digital Health for AMR Surveillance in Resource-Limited Settings*.

---

*Document generated as part of the UDARA AI Technical Documentation Series. For questions, contact the Voice/NLP team at voice@udara-ai.org.*
