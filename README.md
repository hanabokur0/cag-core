# CAG Core — Cognitive-Action Gap Engine

> **LoPAS-SEED v1.15 → v1.20**  
> カーネル化リリース：Core / Normalizer / Wave（外付け）の3層分離

---

## Overview

CAG（Cognitive-Action Gap）は、認知層と行動層の位相差を測定するメタ指標。

```
CAG = Cognition Score − Action Score
```

| CAG符号 | 状態 | 判定 |
|--------|------|------|
| CAG >> 0 | 認知先行（バズが実需を上回る） | STOP / WAIT |
| CAG ≈ 0 | 同期（競合が多い） | WAIT |
| CAG << 0 | 行動先行（静かな実需） | **GO** |

---

## Architecture

```
raw_signals
  ↓
signal_normalizer   ← 差し替え可能（I/Oのみ固定）
  ↓
cag_core            ← 不変のカーネル
  ├─ cag_value
  ├─ delta_cag
  ├─ delta2_cag
  ├─ phase_label
  └─ confidence
  ↓
[wave_extension]    ← optional plugin（このリポジトリには含まない）
  ↓
[interference_engine] ← optional plugin（このリポジトリには含まない）
```

---

## Files

```
cag_core/
├── README.md
├── cag_core.py          # Core engine（不変）
├── cag_schema.json      # JSON Schema（API入出力仕様）
└── normalizer_spec.md   # Normalizer I/O仕様（実装は外部）
```

---

## Quickstart

```python
from cag_core import CAGCore, SignalInput

core = CAGCore()

result = core.update(
    cognitive_signal=0.72,
    action_signal=0.44,
    timestamp="2026-04-06T16:00:00+09:00",
    market="ec",
    source="x_keepa",
    confidence=0.85
)

print(result.phase_label)   # "COG_LEAD"
print(result.cag_value)     # 0.28
print(result.verdict)       # "WAIT"
```

---

## Versioning

| Version | 内容 |
|---------|------|
| v1.15 | CAG定義・3層構造・EMA安定化（LoPAS-SEED） |
| v1.20 | カーネル化・Normalizer I/F分離・Wave外付け設計 |

---

## Theory

LoPAS-SEED v1.15 に基づく。  
真実は単一指標ではなく、**認知と行動のズレ**に現れる。

- 認知FieldVoice：SNS / Q&A / 検索トレンド
- 行動FieldVoice：Keepa / 在庫推移 / ランキング変動

CAGはこの2層の位相差を連続的に観測する装置である。

---

## Related

- `wave_extension`（別リポジトリ予定）：rolling FFT / dominant cycle / resonance
- `interference_engine`（別リポジトリ予定）：cross-market lag / propagation score
- LoPAS-SEED v1.20 仕様書（近日公開）
