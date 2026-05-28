# Multilingual Support

> UDARA AI supports 12+ African languages across NER, ASR, embeddings, and UI localization. This document covers the language architecture, model choices, and implementation patterns.

---

## Table of Contents

1. [Language Priority Matrix](#1-language-priority-matrix)
2. [Named Entity Recognition (NER)](#2-named-entity-recognition-ner)
3. [Automatic Speech Recognition (ASR)](#3-automatic-speech-recognition-asr)
4. [Multilingual Embeddings](#4-multilingual-embeddings)
5. [Large Language Model (LLM) Layer](#5-large-language-model-llm-layer)
6. [Language Detection](#6-language-detection)
7. [UI Localization](#7-ui-localization)
8. [Language Configuration](#8-language-configuration)
9. [Testing Multilingual Pipelines](#9-testing-multilingual-pipelines)

---

## 1. Language Priority Matrix

### 1.1 Supported Languages by Priority

| Priority | Language | ISO 639-1/3 | Script | Speakers (M) | Countries | Status |
|----------|----------|-------------|--------|---------------|-----------|--------|
| **P0** | English | `en` | Latin | 700+ | Widespread (lingua franca) | ✅ Full support |
| **P0** | Swahili | `sw` | Latin | 100+ | Tanzania, Kenya, Uganda, DRC | ✅ Full support |
| **P0** | Yoruba | `yo` | Latin | 45+ | Nigeria, Benin, Togo | ✅ Full support |
| **P0** | Hausa | `ha` | Latin (Ajami) | 150+ | Nigeria, Niger, Ghana, Cameroon | ✅ Full support |
| **P0** | Igbo | `ig` | Latin | 45+ | Nigeria | ✅ Full support |
| **P1** | Amharic | `am` | Ge'ez (Ethiopic) | 60+ | Ethiopia | 🔄 In progress |
| **P1** | Nigerian Pidgin | `pcm` | Latin | 75+ | Nigeria | 🔄 In progress |
| **P2** | Luganda | `lg` | Latin | 20+ | Uganda | 📋 Planned |
| **P2** | Twi | `tw` | Latin | 11+ | Ghana | 📋 Planned |
| **P2** | Somali | `so` | Latin (Osmanya) | 21+ | Somalia, Ethiopia, Kenya | 📋 Planned |
| **P3** | Zulu | `zu` | Latin | 28+ | South Africa, Eswatini | 📋 Planned |
| **P3** | Amharic (advanced) | `am` | Ge'ez | 60+ | Ethiopia | 📋 Planned |

### 1.2 Priority Definitions

| Priority | Definition | NER | ASR | Embeddings | UI |
|----------|-----------|-----|-----|------------|-----|
| **P0** | Launch-critical. Must work on Day 1. | ✅ XLM-RoBERTa fine-tuned | ✅ MMS-ASR | ✅ multilingual-e5 | ✅ Full strings |
| **P1** | High-value. Ship within 3 months of launch. | ✅ XLM-RoBERTa (zero-shot) | ✅ MMS-ASR | ✅ multilingual-e5 | ✅ Core strings |
| **P2** | Medium-value. Ship within 6 months. | ⚠️ Cross-lingual transfer | ⚠️ MMS-ASR (lower accuracy) | ✅ multilingual-e5 | ⚠️ Basic strings |
| **P3** | Nice-to-have. Ship within 12 months. | ⚠️ Cross-lingual transfer | ⚠️ MMS-ASR (experimental) | ⚠️ English fallback | ❌ English only |

### 1.3 Language Coverage by ML Component

```
Language    │ NER      │ ASR      │ Embeddings │ LLM      │ UI
────────────┼──────────┼──────────┼────────────┼──────────┼────────
English     │ ★★★★★   │ ★★★★★   │ ★★★★★     │ ★★★★★   │ ★★★★★
Swahili     │ ★★★★★   │ ★★★★★   │ ★★★★★     │ ★★★★☆   │ ★★★★★
Yoruba      │ ★★★★☆   │ ★★★★☆   │ ★★★★★     │ ★★★☆☆   │ ★★★★★
Hausa       │ ★★★★☆   │ ★★★★☆   │ ★★★★★     │ ★★★☆☆   │ ★★★★★
Igbo        │ ★★★★☆   │ ★★★★☆   │ ★★★★★     │ ★★★☆☆   │ ★★★★★
Amharic     │ ★★★★☆   │ ★★★☆☆   │ ★★★★☆     │ ★★☆☆☆   │ ★★★★☆
Nig. Pidgin │ ★★★☆☆   │ ★★☆☆☆   │ ★★★★☆     │ ★★☆☆☆   │ ★★★☆☆
Luganda     │ ★★★☆☆   │ ★★★☆☆   │ ★★★★☆     │ ★★☆☆☆   │ ★★★☆☆
Twi         │ ★★★☆☆   │ ★★★☆☆   │ ★★★★☆     │ ★★☆☆☆   │ ★★☆☆☆
Somali      │ ★★★☆☆   │ ★★★☆☆   │ ★★★★☆     │ ★★☆☆☆   │ ★★☆☆☆
Zulu        │ ★★☆☆☆   │ ★★★☆☆   │ ★★★★☆     │ ★☆☆☆☆   │ ★★☆☆☆

★ = Support level (5 = production, 1 = experimental)
```

---

## 2. Named Entity Recognition (NER)

### 2.1 Model Selection: XLM-RoBERTa

We use **XLM-RoBERTa-large** (`xlm-roberta-large`) as the base NER model, fine-tuned for African language medical entity extraction.

```
┌──────────────────────────────────────────────────┐
│                NER PIPELINE                       │
│                                                  │
│  Input Text ──▶ Language Detection ──▶ NER Model │
│  (any lang)     (fastText lid)      (XLM-R)      │
│                      │                  │        │
│                      ▼                  ▼        │
│               ┌────────────┐   ┌──────────────┐  │
│               │ lang_code  │   │ Entities:    │  │
│               │ confidence │   │ - SYMPTOM    │  │
│               └────────────┘   │ - MEDICINE   │  │
│                                 │ - DURATION   │  │
│                                 │ - BODY_PART  │  │
│                                 │ - SEVERITY   │  │
│                                 │ - PATIENT    │  │
│                                 └──────────────┘  │
└──────────────────────────────────────────────────┘
```

### 2.2 Why XLM-RoBERTa

| Feature | XLM-RoBERTa | AfriBERTa | mBERT |
|---------|-------------|-----------|-------|
| Languages covered | 100 | 12 African | 104 |
| African lang quality | ★★★★☆ | ★★★★★ | ★★☆☆☆ |
| Medical domain | Fine-tuned | Pre-trained only | Fine-tuned |
| Model size | 560M params | 125M params | 110M params |
| Inference (INT4) | ~1.5s | ~0.4s | ~0.3s |
| Availability | Hugging Face | Hugging Face | Hugging Face |

### 2.3 Entity Types

```python
# services/agent-a/src/ner/entities.py

from dataclasses import dataclass
from enum import Enum
from typing import Optional


class EntityType(Enum):
    """UDARA-specific medical entity types for African AMR surveillance."""

    SYMPTOM = "SYMPTOM"           # Fever, cough, diarrhea, rash
    MEDICINE = "MEDICINE"         # Amoxicillin, paracetamol, ACT
    DURATION = "DURATION"          # 3 days, 2 weeks, since Monday
    BODY_PART = "BODY_PART"        # Chest, abdomen, throat, head
    SEVERITY = "SEVERITY"          # Mild, severe, very bad, slight
    PATIENT_AGE = "PATIENT_AGE"    # 2 years, 6 months, adult
    PATIENT_SEX = "PATIENT_SEX"    # Male, female, boy, girl
    RESISTANCE_INDICATOR = "RESISTENCE_IND"  # "not working", "no improvement"
    DIAGNOSIS = "DIAGNOSIS"        # Malaria, UTI, typhoid
    DOSAGE = "DOSAGE"              # 500mg, twice daily, 3 tablets
    SPECIMEN_TYPE = "SPECIMEN_TYPE" # Blood, urine, stool
    FACILITY_TYPE = "FACILITY_TYPE" # PHC, hospital, chemist
    ANTIMICROBIAL_CLASS = "ANTIMICROBIAL_CLASS"  # Antibiotic, antimalarial


@dataclass
class ExtractedEntity:
    """A single named entity extracted from case text."""

    entity_type: EntityType
    text: str                    # Original text span
    text_en: str                 # English translation (if applicable)
    start_char: int
    end_char: int
    confidence: float            # 0.0 - 1.0
    language: str                # Source language code
    normalized_value: Optional[str] = None

    def to_dict(self) -> dict:
        return {
            "entity_type": self.entity_type.value,
            "text": self.text,
            "text_en": self.text_en,
            "start_char": self.start_char,
            "end_char": self.end_char,
            "confidence": round(self.confidence, 3),
            "language": self.language,
            "normalized_value": self.normalized_value,
        }
```

### 2.4 Fine-Tuning for African Languages

```python
# training/ner/finetune_xlmr.py
"""
Fine-tuning XLM-RoBERTa for medical NER on African languages.
Dataset: Custom annotated corpus in en, sw, yo, ha, ig
"""

from transformers import (
    AutoTokenizer,
    AutoModelForTokenClassification,
    TrainingArguments,
    Trainer,
    DataCollatorForTokenClassification,
)
import evaluate
import numpy as np

MODEL_NAME = "xlm-roberta-large"
LANGUAGES = ["en", "sw", "yo", "ha", "ig"]
LABEL_LIST = [
    "O",  # Outside
    "B-SYMPTOM", "I-SYMPTOM",
    "B-MEDICINE", "I-MEDICINE",
    "B-DURATION", "I-DURATION",
    "B-BODY_PART", "I-BODY_PART",
    "B-SEVERITY", "I-SEVERITY",
    "B-PATIENT_AGE", "I-PATIENT_AGE",
    "B-PATIENT_SEX", "I-PATIENT_SEX",
    "B-DIAGNOSIS", "I-DIAGNOSIS",
    "B-DOSAGE", "I-DOSAGE",
    "B-ANTIMICROBIAL_CLASS", "I-ANTIMICROBIAL_CLASS",
    "B-RESISTANCE_INDICATOR", "I-RESISTANCE_INDICATOR",
]
LABEL2ID = {label: i for i, label in enumerate(LABEL_LIST)}
ID2LABEL = {i: label for i, label in enumerate(LABEL_LIST)}


def load_multilingual_dataset(data_dir: str, languages: list[str]):
    """Load annotated NER datasets for all target languages."""
    from datasets import load_dataset

    datasets = []
    for lang in languages:
        ds = load_dataset("json", data_files={
            "train": f"{data_dir}/{lang}/train.jsonl",
            "validation": f"{data_dir}/{lang}/val.jsonl",
            "test": f"{data_dir}/{lang}/test.jsonl",
        })
        ds = ds.map(lambda x: {"language": lang})
        datasets.append(ds)

    # Combine all languages into one training set
    from datasets import concatenate_datasets
    combined = concatenate_datasets([
        concatenate_datasets([ds["train"] for ds in datasets]),
    ])
    val_combined = concatenate_datasets([
        concatenate_datasets([ds["validation"] for ds in datasets]),
    ])
    return combined, val_combined


def compute_metrics(p):
    """Compute token-level F1 score for NER."""
    metric = evaluate.load("seqeval")

    predictions, labels = p
    predictions = np.argmax(predictions, axis=2)

    # Convert to label strings, ignoring -100 (special tokens)
    true_labels = [
        [LABEL_LIST[l] for l in label if l != -100]
        for label in labels
    ]
    true_predictions = [
        [LABEL_LIST[p] for (p, l) in zip(prediction, label) if l != -100]
        for prediction, label in zip(predictions, labels)
    ]

    results = metric.compute(predictions=true_predictions, references=true_labels)
    return {
        "precision": results["overall_precision"],
        "recall": results["overall_recall"],
        "f1": results["overall_f1"],
        "accuracy": results["overall_accuracy"],
    }


def train():
    """Main training function."""
    tokenizer = AutoTokenizer.from_pretrained(MODEL_NAME)

    # Load dataset
    train_dataset, val_dataset = load_multilingual_dataset(
        "data/ner/african_medical", LANGUAGES
    )

    # Tokenize
    def tokenize_and_align_labels(examples):
        tokenized_inputs = tokenizer(
            examples["tokens"],
            truncation=True,
            is_split_into_words=True,
            max_length=256,
        )
        labels = []
        for i, label in enumerate(examples["ner_tags"]):
            word_ids = tokenized_inputs.word_ids(batch_index=i)
            previous_word_idx = None
            label_ids = []
            for word_idx in word_ids:
                if word_idx is None:
                    label_ids.append(-100)
                elif word_idx != previous_word_idx:
                    label_ids.append(label[word_idx])
                else:
                    label_ids.append(label[word_idx] if label[word_idx] % 2 == 1 else -100)
                previous_word_idx = word_idx
            labels.append(label_ids)
        tokenized_inputs["labels"] = labels
        return tokenized_inputs

    train_dataset = train_dataset.map(
        tokenize_and_align_labels,
        batched=True,
        remove_columns=train_dataset.column_names,
    )
    val_dataset = val_dataset.map(
        tokenize_and_align_labels,
        batched=True,
        remove_columns=val_dataset.column_names,
    )

    # Model
    model = AutoModelForTokenClassification.from_pretrained(
        MODEL_NAME,
        num_labels=len(LABEL_LIST),
        id2label=ID2LABEL,
        label2id=LABEL2ID,
    )

    # Training
    training_args = TrainingArguments(
        output_dir="./models/xlm-roberta-ner-african-medical",
        evaluation_strategy="epoch",
        learning_rate=2e-5,
        per_device_train_batch_size=16,
        per_device_eval_batch_size=16,
        num_train_epochs=10,
        weight_decay=0.01,
        warmup_ratio=0.1,
        logging_dir="./logs/ner",
        logging_steps=100,
        save_strategy="epoch",
        load_best_model_at_end=True,
        metric_for_best_model="f1",
        greater_is_better=True,
        fp16=True,  # Use mixed precision
        gradient_accumulation_steps=4,
    )

    trainer = Trainer(
        model=model,
        args=training_args,
        train_dataset=train_dataset,
        eval_dataset=val_dataset,
        tokenizer=tokenizer,
        data_collator=DataCollatorForTokenClassification(tokenizer),
        compute_metrics=compute_metrics,
    )

    trainer.train()
    trainer.save_model("./models/xlm-roberta-ner-african-medical")


if __name__ == "__main__":
    train()
```

### 2.5 NER Inference Service

```python
# services/agent-a/src/ner/inference.py
"""
NER inference service for UDARA Agent-A.
Supports 100+ languages via XLM-RoBERTa with medical fine-tuning.
"""

import torch
from transformers import AutoTokenizer, AutoModelForTokenClassification
from typing import Optional
import logging

logger = logging.getLogger(__name__)


class NERService:
    """Multilingual medical NER service."""

    def __init__(
        self,
        model_path: str = "./models/xlm-roberta-ner-african-medical",
        device: str = "auto",
    ):
        self.device = self._get_device(device)
        self.tokenizer = AutoTokenizer.from_pretrained(model_path)
        self.model = AutoModelForTokenClassification.from_pretrained(model_path)
        self.model.to(self.device)
        self.model.eval()
        logger.info(f"NER model loaded on {self.device}")

    @staticmethod
    def _get_device(device: str) -> torch.device:
        if device == "auto":
            return torch.device("cuda" if torch.cuda.is_available() else "cpu")
        return torch.device(device)

    def extract_entities(
        self,
        text: str,
        language: Optional[str] = None,
        confidence_threshold: float = 0.7,
    ) -> list[dict]:
        """Extract medical entities from text in any supported language.

        Args:
            text: Input text (can be in any of 100+ languages)
            language: ISO 639-1 code for hinting (optional, improves accuracy)
            confidence_threshold: Minimum confidence score (0.0-1.0)

        Returns:
            List of extracted entities with type, text, confidence
        """
        inputs = self.tokenizer(
            text,
            return_tensors="pt",
            truncation=True,
            max_length=256,
        ).to(self.device)

        with torch.no_grad():
            logits = self.model(**inputs).logits

        predictions = torch.argmax(logits, dim=2)
        tokens = self.tokenizer.convert_ids_to_tokens(inputs["input_ids"][0])

        # Aggregate BIO-tagged entities
        entities = []
        current_entity = None

        for token, pred_idx in zip(tokens, predictions[0].tolist()):
            label = self.model.config.id2label[pred_idx]

            if label.startswith("B-"):
                if current_entity:
                    entities.append(current_entity)
                current_entity = {
                    "entity_type": label[2:],
                    "text": token.replace("▁", " "),
                    "confidence": torch.softmax(logits[0, len(entities) + 1], dim=0)[pred_idx].item(),
                }
            elif label.startswith("I-") and current_entity:
                current_entity["text"] += token.replace("▁", " ")
            elif label == "O" and current_entity:
                entities.append(current_entity)
                current_entity = None

        if current_entity:
            entities.append(current_entity)

        # Filter by confidence and clean up
        cleaned = []
        for ent in entities:
            ent["text"] = ent["text"].strip()
            if ent["text"] and ent["confidence"] >= confidence_threshold:
                cleaned.append({
                    **ent,
                    "confidence": round(ent["confidence"], 3),
                })

        logger.info(
            f"Extracted {len(cleaned)} entities from {len(text)} chars "
            f"(lang={language})"
        )
        return cleaned
```

### 2.6 Cross-Lingual Entity Mapping

When extracting entities in non-English languages, we map common medical terms to English equivalents:

```python
# services/agent-a/src/ner/entity_mapping.py

# Cross-lingual symptom vocabulary
SYNONYM_MAP = {
    # Swahili
    "homa": ("SYMPTOM", "fever"),
    "mashiko": ("SYMPTOM", "chills"),
    "kikohozi": ("SYMPTOM", "cough"),
    "kutapika": ("SYMPTOM", "vomiting"),
    "kuhara": ("SYMPTOM", "diarrhea"),
    "maumivu ya kichwa": ("SYMPTOM", "headache"),
    "maumivu ya tumbo": ("SYMPTOM", "abdominal pain"),
    "kupumua kwa shida": ("SYMPTOM", "difficulty breathing"),

    # Yoruba
    "ìbà": ("SYMPTOM", "fever"),
    "ìyọ̀nù": ("SYMPTOM", "pain"),
    "aṣáyán": ("SYMPTOM", "cough"),
    "ìtésígun": ("SYMPTOM", "diarrhea"),
    "ìrora": ("SYMPTOM", "headache"),

    # Hausa
    "zazzabi": ("SYMPTOM", "fever"),
    "ciwon ciki": ("SYMPTOM", "abdominal pain"),
    "yaji": ("SYMPTOM", "cough"),
    "hashiƙe": ("SYMPTOM", "diarrhea"),
    "ciwon kai": ("SYMPTOM", "headache"),

    # Igbo
    "ịba": ("SYMPTOM", "fever"),
    "ọrịa": ("SYMPTOM", "sickness/disease"),
    "ịba ahụ": ("SYMPTOM", "body pain"),
    "ịkpatakpa": ("SYMPTOM", "cough"),
    "ịnye ahụ": ("SYMPTOM", "diarrhea"),

    # Amharic
    "ትኩሳት": ("SYMPTOM", "fever"),
    "ራስ ምታት": ("SYMPTOM", "headache"),
    "ሆድ ምታት": ("SYMPTOM", "abdominal pain"),
    "ጭርቀት": ("SYMPTOM", "cough"),
}

# Medicine name mapping (common antimicrobials)
MEDICINE_MAP = {
    # Brand names in African markets
    "amoxil": "amoxicillin",
    "augmentin": "amoxicillin-clavulanate",
    "septrin": "co-trimoxazole",
    "flagyl": "metronidazole",
    "ciprotab": "ciprofloxacin",
    "azithrom": "azithromycin",
    "arthemeter": "artemether",
    "coartem": "artemether-lumefantrine",
    "lonart": "artemether-lumefantrine",
    "fansidar": "sulfadoxine-pyrimethamine",
    "chloroquin": "chloroquine",
    "ampiclox": "ampicillin-cloxacillin",
    "ceftriaxone": "ceftriaxone",
    "gentamicin": "gentamicin",
    "metronidazole": "metronidazole",
}
```

---

## 3. Automatic Speech Recognition (ASR)

### 3.1 Model Selection: MMS-ASR

We use **Meta MMS-ASR** (Massively Multilingual Speech), which supports 1,100+ languages.

```
┌────────────────────────────────────────────────────────────┐
│                    ASR PIPELINE                             │
│                                                            │
│  Audio Input                                               │
│      │                                                     │
│      ▼                                                     │
│  ┌─────────────┐    ┌─────────────┐    ┌──────────────┐   │
│  │  Pre-process│───▶│  MMS-ASR    │───▶│  Post-process│   │
│  │  (normalize,│    │  (lang-det  │    │  (normalize, │   │
│  │   VAD, 16kHz│    │   + transcr)│    │   spell-chk) │   │
│  └─────────────┘    └─────────────┘    └──────┬───────┘   │
│                                               │           │
│                                               ▼           │
│                                    ┌──────────────────┐   │
│                                    │ NER Processing   │   │
│                                    │ (English text is  │   │
│                                    │  best for models) │   │
│                                    └──────────────────┘   │
└────────────────────────────────────────────────────────────┘
```

### 3.2 MMS-ASR Configuration

```python
# services/whisper-asr/src/asr_service.py
"""
ASR service using Meta MMS-ASR for multilingual speech recognition.
"""

import torch
from transformers import Wav2Vec2ForCTC, AutoProcessor
import torchaudio
import logging
from typing import Optional

logger = logging.getLogger(__name__)

# Language to MMS ASR model ID mapping
MMS_MODELS = {
    "en": "facebook/mms-1b-all",         # English (general)
    "sw": "facebook/mms-1b-all",         # Swahili
    "yo": "facebook/mms-1b-fl102",       # Yoruba
    "ha": "facebook/mms-1b-fl102",       # Hausa
    "ig": "facebook/mms-1b-fl102",       # Igbo
    "am": "facebook/mms-1b-fl102",       # Amharic
    "lg": "facebook/mms-1b-fl102",       # Luganda
    "tw": "facebook/mms-1b-fl102",       # Twi
    "so": "facebook/mms-1b-fl102",       # Somali
    "zu": "facebook/mms-1b-fl102",       # Zulu
    "pcm": "en",  # Nigerian Pidgin → use English model
}

# ISO 639-3 to MMS adapter language codes
MMS_LANG_ADAPTERS = {
    "swa": "swa",  # Swahili
    "yor": "yor",  # Yoruba
    "hau": "hau",  # Hausa
    "ibo": "ibo",  # Igbo
    "amh": "amh",  # Amharic
    "lug": "lug",  # Luganda
    "twi": "twi",  # Twi
    "som": "som",  # Somali
    "zul": "zul",  # Zulu
}


class ASRService:
    """Multilingual ASR service using MMS-ASR."""

    def __init__(
        self,
        model_id: str = "facebook/mms-1b-all",
        device: str = "auto",
        languages: Optional[list[str]] = None,
    ):
        self.device = self._get_device(device)
        self.processor = AutoProcessor.from_pretrained(model_id)
        self.model = Wav2Vec2ForCTC.from_pretrained(model_id)
        self.model.to(self.device)
        self.model.eval()

        self.supported_languages = languages or list(MMS_MODELS.keys())
        logger.info(
            f"ASR model loaded: {model_id} on {self.device}, "
            f"languages: {self.supported_languages}"
        )

    @staticmethod
    def _get_device(device: str) -> torch.device:
        if device == "auto":
            return torch.device("cuda" if torch.cuda.is_available() else "cpu")
        return torch.device(device)

    def transcribe(
        self,
        audio_path: str,
        language: Optional[str] = None,
        return_timestamps: bool = False,
    ) -> dict:
        """Transcribe audio file to text.

        Args:
            audio_path: Path to audio file (WAV, MP3, OGG)
            language: ISO 639-1/3 language code (auto-detect if None)
            return_timestamps: Include word-level timestamps

        Returns:
            {
                "text": "transcribed text",
                "language": "detected/used language",
                "language_confidence": 0.95,
                "duration_seconds": 12.5,
                "words": [...]  # if return_timestamps=True
            }
        """
        # Load and preprocess audio
        waveform, sample_rate = torchaudio.load(audio_path)

        # Resample to 16kHz (MMS requirement)
        if sample_rate != 16000:
            resampler = torchaudio.transforms.Resample(sample_rate, 16000)
            waveform = resampler(waveform)

        # Convert to mono if stereo
        if waveform.shape[0] > 1:
            waveform = waveform.mean(dim=0, keepdim=True)

        # Set language adapter
        lang_code = MMS_LANG_ADAPTERS.get(language, language)
        if lang_code and lang_code in self.processor.tokenizer.lang_codes:
            self.processor.tokenizer.set_target_lang(lang_code)
        else:
            logger.warning(f"Language {language} not in MMS lang codes, using default")

        # Process
        inputs = self.processor(
            waveform.squeeze().numpy(),
            sampling_rate=16000,
            return_tensors="pt",
            padding=True,
        ).to(self.device)

        with torch.no_grad():
            logits = self.model(**inputs).logits

        # Decode
        predicted_ids = torch.argmax(logits, dim=-1)
        transcription = self.processor.batch_decode(predicted_ids)[0]

        return {
            "text": transcription.strip(),
            "language": language or "auto",
            "duration_seconds": waveform.shape[1] / 16000,
        }

    async def transcribe_streaming(
        self,
        audio_chunk: bytes,
        language: Optional[str] = None,
    ) -> str:
        """Transcribe a streaming audio chunk.

        Used for real-time USSD voice prompts and Telegram voice messages.
        """
        # Process chunk using same pipeline
        import io
        import soundfile as sf

        waveform, sr = sf.read(io.BytesIO(audio_chunk))
        # ... (similar processing as transcribe)
        return "transcription_placeholder"
```

### 3.3 ASR Accuracy by Language (Expected WER)

| Language | Training Data (hrs) | Expected WER | Notes |
|----------|--------------------:|-------------:|-------|
| English | 12,000+ | < 10% | Excellent on medical vocabulary |
| Swahili | 200+ | < 20% | Good coverage, moderate medical terms |
| Yoruba | 100+ | < 25% | Decent, improving with custom data |
| Hausa | 150+ | < 25% | Good, widely spoken in Nigeria |
| Igbo | 80+ | < 30% | Moderate, less training data |
| Amharic | 100+ | < 25% | Good, unique script handled well |
| Nigerian Pidgin | 0 (uses EN) | < 35% | Using English model as proxy |
| Luganda | 30+ | < 40% | Limited data, needs augmentation |
| Twi | 20+ | < 40% | Limited data |
| Somali | 50+ | < 35% | Moderate |

> **Note:** WER (Word Error Rate) — lower is better. <25% is acceptable for our use case since NER can handle imperfect transcription.

---

## 4. Multilingual Embeddings

### 4.1 Model: multilingual-e5-large

We use **`intfloat/multilingual-e5-large`** for semantic search and case similarity across languages.

```python
# services/agent-b/src/embeddings.py
"""
Multilingual embeddings using intfloat/multilingual-e5-large.
Supports 100+ languages for semantic similarity search.
"""

from sentence_transformers import SentenceTransformer
from typing import Optional
import numpy as np
import logging

logger = logging.getLogger(__name__)


class EmbeddingService:
    """Multilingual embedding service for case similarity."""

    def __init__(
        self,
        model_name: str = "intfloat/multilingual-e5-large",
        device: str = "auto",
        dimension: int = 1024,
    ):
        self.model = SentenceTransformer(model_name)
        self.dimension = dimension
        self.device = device

        logger.info(f"Embedding model loaded: {model_name}")

    def embed_text(
        self,
        text: str,
        instruction: str = "query: ",
    ) -> np.ndarray:
        """Generate embedding for text in any supported language.

        Args:
            text: Input text (any of 100+ languages)
            instruction: E5 prefix ("query: " or "passage: ")

        Returns:
            Normalized embedding vector of shape (1024,)
        """
        embedded = self.model.encode(
            f"{instruction}{text}",
            normalize_embeddings=True,
            show_progress_bar=False,
        )
        return embedded

    def embed_case_for_search(
        self,
        case_text: str,
        symptoms: list[str],
        diagnosis: Optional[str] = None,
    ) -> np.ndarray:
        """Generate embedding for a case record.

        Combines symptoms, diagnosis, and free text for rich representation.
        """
        parts = [f"Symptoms: {', '.join(symptoms)}"]
        if diagnosis:
            parts.append(f"Diagnosis: {diagnosis}")
        if case_text:
            parts.append(f"Details: {case_text}")

        combined = " | ".join(parts)
        return self.embed_text(combined, instruction="passage: ")

    def compute_similarity(
        self,
        embedding_a: np.ndarray,
        embedding_b: np.ndarray,
    ) -> float:
        """Compute cosine similarity between two embeddings."""
        return float(np.dot(embedding_a, embedding_b))

    def find_similar_cases(
        self,
        query_embedding: np.ndarray,
        candidate_embeddings: np.ndarray,
        top_k: int = 10,
    ) -> list[dict]:
        """Find most similar cases from candidates.

        Args:
            query_embedding: Query case embedding (1, 1024)
            candidate_embeddings: Candidate embeddings (N, 1024)
            top_k: Number of results to return

        Returns:
            List of {index, similarity_score} dicts
        """
        similarities = np.dot(candidate_embeddings, query_embedding)
        top_indices = np.argsort(similarities)[::-1][:top_k]

        return [
            {"index": int(idx), "similarity": float(similarities[idx])}
            for idx in top_indices
        ]
```

### 4.2 Cross-Lingual Search Example

```python
# A query in Swahili can find similar English cases
service = EmbeddingService()

# Swahili query
swahili_query = "Mtoto ana homa kali na kikohozi kwa siku tatu"
sw_embedding = service.embed_text(swahili_query)

# English case in database
english_case = "Child has high fever and cough for 3 days, not responding to amoxicillin"
en_embedding = service.embed_text(english_case, instruction="passage: ")

# Similarity score (0.0 to 1.0)
score = service.compute_similarity(sw_embedding, en_embedding)
# Expected: 0.82-0.90 (high similarity across languages!)
```

---

## 5. Large Language Model (LLM) Layer

### 5.1 Model: Llama 3.2 3B

The LLM is English-primary. Best strategy: use NER to extract structured entities, then pass English text to LLM.

```python
# services/agent-c/src/llm/llm_service.py
"""
LLM service using Llama 3.2 3B for therapy recommendation generation.
Key insight: Always translate NER-extracted entities to English before LLM.
"""

from llama_cpp import Llama
from typing import Optional
import logging

logger = logging.getLogger(__name__)

# System prompt for AMR therapy recommendation
SYSTEM_PROMPT = """You are UDARA Therapy Assistant, an AI that helps Community
Health Workers in sub-Saharan Africa with antimicrobial therapy recommendations.

Your role:
1. Assess AMR risk based on patient symptoms, history, and local resistance data
2. Recommend first-line and alternative antibiotic treatments
3. Provide clear dosage instructions suitable for community health settings
4. Identify when to refer to a higher-level facility

Important rules:
- Always consider local AMR resistance patterns
- Prefer narrow-spectrum antibiotics when appropriate
- Include red flags that require immediate referral
- Use simple language suitable for CHWs with basic training
- Never recommend antibiotics for viral infections
- Always ask about allergies and current medications"""

THERAPY_PROMPT_TEMPLATE = """
Based on the following patient assessment, provide therapy recommendations.

PATIENT INFORMATION:
- Age: {age}
- Sex: {sex}
- Weight (if known): {weight}

SYMPTOMS:
{symptoms}

DIAGNOSIS: {diagnosis}

CURRENT MEDICATIONS: {current_medications}

LOCAL RESISTANCE DATA:
- Region resistance profile: {resistance_profile}

PREVIOUS TREATMENTS:
{previous_treatments}

Please provide:
1. AMR Risk Level (Low / Moderate / High / Critical)
2. Recommended Treatment (first-line)
3. Dosage and Duration
4. Alternative Treatment (if first-line not suitable)
5. Red Flags for Referral
6. Patient Education Points

Format your response in clear, numbered sections."""


class LLMService:
    """LLM service for therapy recommendation."""

    def __init__(
        self,
        model_path: str = "./models/llama-3.2-3b-instruct-Q4_K_M.gguf",
        n_ctx: int = 4096,
        n_gpu_layers: int = -1,  # -1 = all layers on GPU
    ):
        self.model = Llama(
            model_path=model_path,
            n_ctx=n_ctx,
            n_gpu_layers=n_gpu_layers,
            verbose=False,
        )
        logger.info(f"LLM loaded: {model_path}")

    def generate_therapy_recommendation(
        self,
        entities: dict,  # NER-extracted entities (always in English)
        resistance_data: dict,
        language: str = "en",
    ) -> str:
        """Generate therapy recommendation from structured entities.

        CRITICAL: entities should always be in English (translated by NER service).
        The LLM is English-primary; translation is done upstream.
        """
        prompt = THERAPY_PROMPT_TEMPLATE.format(
            age=entities.get("patient_age", "Unknown"),
            sex=entities.get("patient_sex", "Unknown"),
            weight=entities.get("weight", "Unknown"),
            symptoms="\n".join(f"- {s}" for s in entities.get("symptoms", [])),
            diagnosis=entities.get("diagnosis", "Not confirmed"),
            current_medications=", ".join(
                entities.get("current_medications", ["None reported"])
            ),
            resistance_data=resistance_data.get("profile", "Standard"),
            previous_treatments="\n".join(
                entities.get("previous_treatments", ["None"])
            ),
        )

        response = self.model.create_chat_completion(
            messages=[
                {"role": "system", "content": SYSTEM_PROMPT},
                {"role": "user", "content": prompt},
            ],
            max_tokens=1024,
            temperature=0.3,  # Low temperature for consistent medical advice
            top_p=0.9,
        )

        return response["choices"][0]["message"]["content"]

    def generate_response_in_language(
        self,
        english_response: str,
        target_language: str = "en",
    ) -> str:
        """Translate LLM response to target language.

        Uses a secondary translation call or template-based approach.
        """
        if target_language == "en":
            return english_response

        # For P0 languages, use template-based response (see Section 7)
        # For P1-P3, attempt LLM translation
        translation_prompt = f"""Translate the following medical advice to {target_language}.
Keep medical terms in their original English form when there is no common translation.
Keep the format and structure identical.

Text to translate:
{english_response}"""

        response = self.model.create_chat_completion(
            messages=[
                {
                    "role": "system",
                    "content": "You are a medical translator. Translate accurately while keeping medical terminology clear.",
                },
                {"role": "user", "content": translation_prompt},
            ],
            max_tokens=1024,
            temperature=0.1,
        )

        return response["choices"][0]["message"]["content"]
```

### 5.2 Why English-First for LLM

| Aspect | English Input | Non-English Input |
|--------|--------------|-------------------|
| NER accuracy | ★★★★★ (F1 > 90%) | ★★★★☆ (F1 > 80%) |
| LLM reasoning quality | ★★★★★ | ★★☆☆☆ (hallucination risk) |
| Therapy recommendation | ★★★★★ | ★★☆☆☆ (unsafe!) |
| Token efficiency | ~200 tokens | ~300+ tokens |
| Consistency | High | Low |

**Conclusion:** Always translate NER-extracted entities to English before passing to LLM. The UI response is then translated back to the user's language using templates or a secondary translation step.

---

## 6. Language Detection

### 6.1 fastText Language Identification

```python
# services/agent-a/src/language_detection.py
"""
Language detection using fastText lid.176.bin.
Supports 176 languages including all target African languages.
"""

import fasttext
import re
from typing import Optional
import logging

logger = logging.getLogger(__name__)

# Map fastText ISO codes to UDARA language codes
FASTTEXT_TO_UDARA = {
    "en": "en",
    "sw": "sw",
    "yo": "yo",
    "ha": "ha",
    "ig": "ig",
    "am": "am",
    "lg": "lg",
    "tw": "tw",
    "so": "so",
    "zu": "zu",
    # Nigerian Pidgin is not in fastText; detect via heuristics
}

# Pidgin detection heuristics
PIDGIN_PATTERNS = [
    r"\bdey\b",           # is/are
    r"\bna\b",            # is/it is
    r"\bwetin\b",         # what
    r"\bhow\b.*\bbe\b",   # how are you
    r"\babi\b",           # right?/isn't it?
    r"\boya\b",           # okay/let's go
    r"\bwahala\b",        # problem
    r"\bsabi\b",          # know
    r"\bnna\b",           # doesn't
    r"\bgo\b.*\bcome\b",  # complex pidgin pattern
    r"\bdem\b",           # them
    r"\bim\b",            # he/she/it
]


class LanguageDetector:
    """Fast and accurate language detection for African languages."""

    def __init__(self, model_path: str = "./models/lid.176.bin"):
        self.model = fasttext.load_model(model_path)
        logger.info("Language detection model loaded")

    def detect(
        self,
        text: str,
        threshold: float = 0.5,
    ) -> dict:
        """Detect language of input text.

        Args:
            text: Input text in any language
            threshold: Minimum confidence score

        Returns:
            {
                "language": "en",
                "confidence": 0.98,
                "udara_code": "en",
                "is_pidgin": False
            }
        """
        # Check for Nigerian Pidgin first (not in fastText)
        is_pidgin, pidgin_confidence = self._detect_pidgin(text)

        if is_pidgin:
            return {
                "language": "pcm",
                "confidence": pidgin_confidence,
                "udara_code": "pcm",
                "is_pidgin": True,
            }

        # fastText prediction
        predictions = self.model.predict(text.replace("\n", " "), k=3)

        language = predictions[0][0].replace("__label__", "")
        confidence = float(predictions[1][0])

        # Map to UDARA code
        udara_code = FASTTEXT_TO_UDARA.get(language, language)

        # If confidence is below threshold, fall back to English
        if confidence < threshold:
            logger.warning(
                f"Low confidence language detection: {language} "
                f"({confidence:.2f}), falling back to English"
            )
            return {
                "language": "en",
                "confidence": confidence,
                "udara_code": "en",
                "is_pidgin": False,
                "fallback": True,
                "original_detection": language,
            }

        return {
            "language": language,
            "confidence": round(confidence, 3),
            "udara_code": udara_code,
            "is_pidgin": False,
        }

    def _detect_pidgin(self, text: str) -> tuple[bool, float]:
        """Heuristic detection for Nigerian Pidgin."""
        text_lower = text.lower()
        matches = sum(
            1 for pattern in PIDGIN_PATTERNS
            if re.search(pattern, text_lower)
        )
        confidence = min(matches / 3.0, 1.0)  # 3+ matches = high confidence
        return matches >= 2, round(confidence, 3)
```

### 6.2 Usage Example

```python
# Example usage across different languages
detector = LanguageDetector()

examples = [
    "The child has had fever for three days and is not responding to amoxicillin",
    "Mtoto amekuwa na homa kwa siku tatu na hana maendeleo tiba ya amoxicillin",
    "Ọmọde náà ti ní ìbà fún ọjọ́ mẹ́ta kí amoxicillin tó fi yọ̀",
    "Yaro yana da zazzabi kwana uku kuma amoxicillin bai aiki ba",
    "Nwa a nwere ịba ruo ụbọchị atọ ma ọ naghị azụ mma na amoxicillin",
    "ልጁ ሶስት ቀናት ትኩሳት ያለው ነው እና በአሞክሲሲሊን አይረዳም",
    "E don get fever for three days and the amoxicillin no dey work",
]

for text in examples:
    result = detector.detect(text)
    print(f"{text[:50]:50s} → {result['udara_code']:5s} ({result['confidence']:.2f})")
```

**Expected output:**
```
The child has had fever for three days and is   → en    (0.99)
Mtoto amekuwa na homa kwa siku tatu na hana    → sw    (0.97)
Ọmọde náà ti ní ìbà fún ọjọ́ mẹ́ta kí amoxici → yo    (0.95)
Yaro yana da zazzabi kwana uku kuma amoxicil   → ha    (0.96)
Nwa a nwere ịba ruo ụbọchị atọ ma ọ naghị azụ  → ig    (0.93)
ልጁ ሶስት ቀናት ትኩሳት ያለው ነው እና በአሞክሲሲሊን  → am    (0.94)
E don get fever for three days and the amoxic  → pcm   (0.83)
```

---

## 7. UI Localization

### 7.1 Template-Based Response Strings

For critical UI responses (therapy recommendations, dosage instructions), we use template-based localization for safety and consistency.

```python
# services/agent-c/src/localization/strings.py
"""
Template-based UI response strings per language.
These are used for critical medical information where accuracy is paramount.
LLM-generated translations are used only for non-critical supplementary text.
"""

RESPONSE_STRINGS = {
    "en": {
        "risk_low": "Risk Level: LOW — Standard antibiotic therapy is likely effective.",
        "risk_moderate": "Risk Level: MODERATE — Consider local resistance patterns. First-line may have reduced efficacy.",
        "risk_high": "Risk Level: HIGH — High local resistance detected. Alternative therapy recommended.",
        "risk_critical": "Risk Level: CRITICAL — Widespread resistance. Immediate specialist referral advised.",
        "dosage_header": "Recommended Dosage:",
        "alternative_header": "Alternative if first-line not available:",
        "red_flags_header": "⚠️ REFER IMMEDIATELY if:",
        "red_flags_list": [
            "Fever above 39°C for more than 3 days",
            "Difficulty breathing",
            "Unable to keep fluids down",
            "No improvement after 48 hours of treatment",
            "Signs of dehydration (no urine, sunken eyes)",
        ],
        "allergy_warning": "⚠️ IMPORTANT: Ask about allergies before prescribing.",
        "duration_format": "Take for {days} days. Do not stop even if feeling better.",
        "complete_course": "Complete the full course. Stopping early causes resistance.",
        "follow_up": "Follow up in {days} days if no improvement.",
        "referral_needed": "This case requires referral to {facility_type}.",
    },

    "sw": {
        "risk_low": "Kiwango cha Hatari: CHINI — Matibabu ya kawaida ya antibiotiki yanaweza kufanya kazi.",
        "risk_moderate": "Kiwango cha Hatari: WASTANI — Zingatia mfumo wa upinzani wa eneo.",
        "risk_high": "Kiwango cha Hatari: JUU — Upinzani wa juu umegunduliwa. Matibabu mbadala yanashauriwa.",
        "risk_critical": "Kiwango cha Hatari: CHOKONOZI — Upinzani upana. Rufaa ya haraka ya daktari bingwa.",
        "dosage_header": "Dozi Iliyoshauriwa:",
        "alternative_header": "Mbadala kama matibabu ya kwanza hayapatikani:",
        "red_flags_header": "⚠️ TUMA KWA HOSPITALI HARAKA ikiwa:",
        "red_flags_list": [
            "Homa juu ya 39°C kwa zaidi ya siku 3",
            "Ushindi wa kupumua",
            "Haiwezi kubakiza maji mwilini",
            "Hakuna maendeleo baada ya masaa 48 ya matibabu",
        ],
        "allergy_warning": "⚠️ MUHIMU: Uliza kuhusu mzio kabla ya kutoa dawa.",
        "duration_format": "Chukua kwa siku {days}. Usisitize hata kama umepata nafuu.",
        "complete_course": "Maliza dozi kamili. Kuacha mapema husababisha upinzani.",
        "follow_up": "Fuata upya baada ya siku {days} kama hakuna maendeleo.",
        "referral_needed": "Kesi hii inahitaji rufaa kwenda {facility_type}.",
    },

    "yo": {
        "risk_low": "Ipele EÀbájà: KÈÈRÍ — Ìtọjú àjẹsára àdánùkọ yóò dára.",
        "risk_moderate": "Ipele EÀbájà: ÀÀRIN — Gbà ọ̀nà ìbámu ipó àyíkà pa.",
        "risk_high": "Ipele EÀbájà: GÁÀ — Ìbámu ipó gágá àwárí. Ìtọjú àyọ̀kúrò ní ìṣedédé.",
        "risk_critical": "Ipele EÀbájà: PÀTÀKÌ — Ìbámu ipó tóbi. Pàdé ọ̀dọ́ òṣìṣẹ́ ààyanfẹ́ lórí tán.",
        "dosage_header": "Dózì Títọ́:",
        "alternative_header": "Àyọ̀kúró tí ìtọjú àkọ́kọ́ kò ní:",
        "red_flags_header": "⚠️ FÍ GBA ÌTÌJÚ TÁN LỌ́ ÀYÍKÀ ÒYÌNBÓ TI:",
        "allergy_warning": "⚠️ PÀTÀKÌ: Bí èèyàn ní àlèrìgí kí o sọ̀.
        "duration_format": "Lo fún {days} ọjọ́. Má ṣe dúró nítorí o wárí ara.",
    },

    "ha": {
        "risk_low": "Matakin Haɗari: ƘANƏ — Maganin rigakafi na al'ada zai yi aiki.",
        "risk_moderate": "Matakin Haɗari: TSAYA — Ka lura da yanayin juriyar gida.",
        "risk_high": "Matakin Haɗari: GABA — An gano ƙarancin juriya mai yawa. Ana ba da shawarar magani.",
        "risk_critical": "Matakin Haɗari: MUКAMA — Juriya da ta yawa. Fitar da maras lafiya ga likita.",
        "dosage_header": "Matsayin Magani:",
        "red_flags_header": "⚠️ AURA TẢKARA GIMDI AGA DA:",
        "allergy_warning": "⚠️ MUHIMMI: Ka tambayi game da lalacewar fata kafin ba da magani.",
    },

    "ig": {
        "risk_low": "Ọkwa Ihe Iche Iche: NKEƠ — Ọgwụ mgbochi ọkụkọ ga-arụ ọrụ nke ọma.",
        "risk_moderate": "Ọkwa Ihe Iche Iche: N'ETITI — Lelee ụdị mmegide mpaghara.",
        "risk_high": "Ọkwa Ihe Iche Iche: NKEỌ - A chọpụtara mmegide dị elu. A na-ahụ maka ọgwụ.",
        "risk_critical": "Ọkwa Ihe Iche Iche: SỌSỌ - Mmegide sara mbara. Zigara onye ọrịa gaa n'ụlọ ọgwụ.",
        "dosage_header": "Ọnụ ọgụgụ Ọgwụ A Na-atụ Aro:",
        "red_flags_header": "⚠ BỊA TỤZỤ OKE RỤMỤNỌ AGA NA:",
    },

    "am": {
        "risk_low": "የሚገኝ ደረጃ: ዝቅተኛ — መደበኛ አንቲባዮቲክ ትኩሳት ይሰራል.",
        "risk_high": "የሚገኝ ደረጃ: ከፍ ተርፎ — ከፍተኛ ስምምነት ተገኝቷል.",
        "dosage_header": "የሚመከር መጠን:",
        "red_flags_header": "⚠️ ወደ ሆስፒታል ያስተላልፉ ፈጣን ከሆነ:",
    },
}


def get_localized_string(
    key: str,
    language: str,
    **kwargs,
) -> str:
    """Get a localized response string with template substitution.

    Args:
        key: String key (e.g., "risk_high", "duration_format")
        language: ISO 639-1 language code
        **kwargs: Template variables (e.g., days=7, facility_type="hospital")

    Returns:
        Localized string with variables substituted

    Raises:
        KeyError: If key not found for language (falls back to English)
    """
    lang_strings = RESPONSE_STRINGS.get(language, RESPONSE_STRINGS["en"])

    template = lang_strings.get(key, RESPONSE_STRINGS["en"].get(key, key))

    if kwargs:
        try:
            return template.format(**kwargs)
        except KeyError:
            logger.warning(f"Missing template variable in key '{key}' for lang '{language}'")

    return template
```

### 7.2 Localization Best Practices

| Practice | Do | Don't |
|----------|-----|-------|
| Critical medical text | Use pre-translated templates | Use LLM translation |
| User-facing prompts | Use pre-translated strings | Concatenate dynamically |
| Supplementary info | LLM translation with review | Use raw LLM output |
| Numbers/dosages | Use `format()` with `**kwargs` | Hardcode in translation |
| Medical terminology | Keep English terms with local explanation | Translate everything |
| Pluralization | Use separate keys for singular/plural | Assume English plural rules |

---

## 8. Language Configuration

### 8.1 Language Map Configuration JSON

```json
{
  "version": "1.0.0",
  "languages": {
    "en": {
      "name": "English",
      "native_name": "English",
      "iso_639_1": "en",
      "iso_639_3": "eng",
      "priority": "P0",
      "script": "Latin",
      "rtl": false,
      "ner": {
        "supported": true,
        "model": "xlm-roberta-large",
        "fine_tuned": true,
        "expected_f1": 0.92
      },
      "asr": {
        "supported": true,
        "model": "mms-1b-all",
        "expected_wer": 0.10,
        "training_hours": 12000
      },
      "embeddings": {
        "supported": true,
        "model": "multilingual-e5-large"
      },
      "llm": {
        "supported": true,
        "primary": true,
        "direct_input": true
      },
      "ui": {
        "supported": true,
        "coverage": "full",
        "strings_file": "locales/en.json"
      }
    },
    "sw": {
      "name": "Swahili",
      "native_name": "Kiswahili",
      "iso_639_1": "sw",
      "iso_639_3": "swa",
      "priority": "P0",
      "script": "Latin",
      "rtl": false,
      "ner": {
        "supported": true,
        "model": "xlm-roberta-large",
        "fine_tuned": true,
        "expected_f1": 0.85
      },
      "asr": {
        "supported": true,
        "model": "mms-1b-all",
        "expected_wer": 0.20,
        "training_hours": 200
      },
      "embeddings": {
        "supported": true,
        "model": "multilingual-e5-large"
      },
      "llm": {
        "supported": true,
        "primary": false,
        "direct_input": false,
        "translate_to_english": true
      },
      "ui": {
        "supported": true,
        "coverage": "full",
        "strings_file": "locales/sw.json"
      }
    },
    "yo": {
      "name": "Yoruba",
      "native_name": "Yorùbá",
      "iso_639_1": "yo",
      "iso_639_3": "yor",
      "priority": "P0",
      "script": "Latin",
      "rtl": false,
      "ner": {
        "supported": true,
        "model": "xlm-roberta-large",
        "fine_tuned": true,
        "expected_f1": 0.83
      },
      "asr": {
        "supported": true,
        "model": "mms-1b-all",
        "expected_wer": 0.25,
        "training_hours": 100
      },
      "embeddings": {
        "supported": true,
        "model": "multilingual-e5-large"
      },
      "llm": {
        "supported": true,
        "primary": false,
        "direct_input": false,
        "translate_to_english": true
      },
      "ui": {
        "supported": true,
        "coverage": "full",
        "strings_file": "locales/yo.json"
      }
    },
    "ha": {
      "name": "Hausa",
      "native_name": "Hausa",
      "iso_639_1": "ha",
      "iso_639_3": "hau",
      "priority": "P0",
      "script": "Latin",
      "rtl": false,
      "ner": {
        "supported": true,
        "model": "xlm-roberta-large",
        "fine_tuned": true,
        "expected_f1": 0.84
      },
      "asr": {
        "supported": true,
        "model": "mms-1b-all",
        "expected_wer": 0.25,
        "training_hours": 150
      },
      "embeddings": {
        "supported": true,
        "model": "multilingual-e5-large"
      },
      "llm": {
        "supported": true,
        "primary": false,
        "direct_input": false,
        "translate_to_english": true
      },
      "ui": {
        "supported": true,
        "coverage": "full",
        "strings_file": "locales/ha.json"
      }
    },
    "ig": {
      "name": "Igbo",
      "native_name": "Igbo",
      "iso_639_1": "ig",
      "iso_639_3": "ibo",
      "priority": "P0",
      "script": "Latin",
      "rtl": false,
      "ner": {
        "supported": true,
        "model": "xlm-roberta-large",
        "fine_tuned": true,
        "expected_f1": 0.82
      },
      "asr": {
        "supported": true,
        "model": "mms-1b-all",
        "expected_wer": 0.30,
        "training_hours": 80
      },
      "embeddings": {
        "supported": true,
        "model": "multilingual-e5-large"
      },
      "llm": {
        "supported": true,
        "primary": false,
        "direct_input": false,
        "translate_to_english": true
      },
      "ui": {
        "supported": true,
        "coverage": "full",
        "strings_file": "locales/ig.json"
      }
    },
    "am": {
      "name": "Amharic",
      "native_name": "አማርኛ",
      "iso_639_1": "am",
      "iso_639_3": "amh",
      "priority": "P1",
      "script": "Ethiopic",
      "rtl": false,
      "ner": {
        "supported": true,
        "model": "xlm-roberta-large",
        "fine_tuned": false,
        "expected_f1": 0.78
      },
      "asr": {
        "supported": true,
        "model": "mms-1b-all",
        "expected_wer": 0.25,
        "training_hours": 100
      },
      "embeddings": {
        "supported": true,
        "model": "multilingual-e5-large"
      },
      "llm": {
        "supported": false,
        "primary": false,
        "direct_input": false,
        "translate_to_english": true
      },
      "ui": {
        "supported": true,
        "coverage": "core",
        "strings_file": "locales/am.json"
      }
    },
    "pcm": {
      "name": "Nigerian Pidgin",
      "native_name": "Naijá",
      "iso_639_1": null,
      "iso_639_3": "pcm",
      "priority": "P1",
      "script": "Latin",
      "rtl": false,
      "ner": {
        "supported": true,
        "model": "xlm-roberta-large",
        "fine_tuned": false,
        "expected_f1": 0.75
      },
      "asr": {
        "supported": false,
        "model": null,
        "expected_wer": null,
        "training_hours": 0,
        "notes": "Uses English ASR as proxy"
      },
      "embeddings": {
        "supported": true,
        "model": "multilingual-e5-large"
      },
      "llm": {
        "supported": false,
        "primary": false,
        "direct_input": false,
        "translate_to_english": true,
        "notes": "Requires custom pidgin vocabulary mapping"
      },
      "ui": {
        "supported": true,
        "coverage": "core",
        "strings_file": "locales/pcm.json"
      }
    }
  }
}
```

---

## 9. Testing Multilingual Pipelines

### 9.1 Test Cases Per Language

```python
# tests/test_multilingual_pipeline.py
"""
End-to-end multilingual pipeline tests.
Run: pytest tests/test_multilingual_pipeline.py -v
"""

import pytest
from services.agent_a.src.language_detection import LanguageDetector
from services.agent_a.src.ner.inference import NERService
from services.agent_b.src.embeddings import EmbeddingService


class TestLanguageDetection:
    """Language detection accuracy tests."""

    @pytest.fixture
    def detector(self):
        return LanguageDetector()

    @pytest.mark.parametrize("text, expected_lang, min_confidence", [
        # English
        ("Child has fever for 3 days", "en", 0.90),
        ("Patient not responding to antibiotics", "en", 0.90),

        # Swahili
        ("Mtoto ana homa kwa siku tatu", "sw", 0.85),
        ("Mgonjwa hapo amekuwa akitumia dawa", "sw", 0.85),

        # Yoruba
        ("Ọmọdé ní ìbà fún ọjọ́ mẹ́ta", "yo", 0.80),
        ("Aláìsàn kò ní ìmúṣẹṣẹ́ lórí òògùn", "yo", 0.80),

        # Hausa
        ("Yaro yana da zazzabi kwana uku", "ha", 0.80),
        ("Maganin bai yi aiki ba", "ha", 0.80),

        # Igbo
        ("Nwa a nwere ịba ruo ụbọchị atọ", "ig", 0.80),
        ("Ọgwụ ahụ anaghị arụ ọrụ", "ig", 0.80),

        # Nigerian Pidgin
        ("E don get fever for three days now", "pcm", 0.70),
        ("The medicine no dey work at all", "pcm", 0.70),
    ])
    def test_detection_accuracy(self, detector, text, expected_lang, min_confidence):
        result = detector.detect(text)
        assert result["udara_code"] == expected_lang
        assert result["confidence"] >= min_confidence


class TestNERMultilingual:
    """NER accuracy tests across languages."""

    @pytest.fixture
    def ner_service(self):
        return NERService(model_path="./models/xlm-roberta-ner-african-medical")

    @pytest.mark.parametrize("text, language, expected_entity_types", [
        ("Mtoto ana homa na kikohozi kwa siku tatu", "sw", ["SYMPTOM", "SYMPTOM", "DURATION"]),
        ("Patient was given amoxicillin 500mg twice daily", "en", ["MEDICINE", "DOSAGE"]),
        ("Yaro yana da zazzabi kuma ya sha maganin amoxicillin", "ha", ["SYMPTOM", "MEDICINE"]),
        ("Ọmọdé náà ní ìbà àti kíkohozì", "yo", ["SYMPTOM", "SYMPTOM"]),
    ])
    def test_entity_extraction(self, ner_service, text, language, expected_entity_types):
        entities = ner_service.extract_entities(text, language=language)
        entity_types = {e["entity_type"] for e in entities}

        for expected_type in expected_entity_types:
            assert expected_type in entity_types, \
                f"Expected {expected_type} in {entity_types} for text: {text}"


class TestCrossLingualEmbeddings:
    """Test that cross-lingual embeddings find similar cases."""

    @pytest.fixture
    def embedding_service(self):
        return EmbeddingService()

    def test_swahili_english_similarity(self, embedding_service):
        """Swahili symptom description should match English equivalent."""
        sw_text = "Mtoto ana homa kali, kikohozi, na kutapika"
        en_text = "Child has severe fever, cough, and vomiting"

        sw_embed = embedding_service.embed_text(sw_text)
        en_embed = embedding_service.embed_text(en_text)

        similarity = embedding_service.compute_similarity(sw_embed, en_embed)
        assert similarity > 0.75, f"Cross-lingual similarity too low: {similarity}"

    def test_dissimilar_cases(self, embedding_service):
        """Unrelated cases should have low similarity."""
        text_a = "Child with fever and cough"
        text_b = "Adult with skin rash and joint pain"

        embed_a = embedding_service.embed_text(text_a)
        embed_b = embedding_service.embed_text(text_b)

        similarity = embedding_service.compute_similarity(embed_a, embed_b)
        assert similarity < 0.70, f"Unrelated cases too similar: {similarity}"
```

---

*Last updated: 2025-01-15 | Maintainer: ML Team*
