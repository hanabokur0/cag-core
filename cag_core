"""
cag_core.py
===========
CAG（Cognitive-Action Gap）Core Engine
LoPAS-SEED v1.20 — カーネル化リリース

責務：
  - 認知スコアと行動スコアの差分を測定する
  - EMA安定化
  - delta / delta2 算出
  - phase_label / verdict 出力

責務外（Waveモジュール担当）：
  - フーリエ分解
  - 周期抽出
  - 市場間干渉
"""

from __future__ import annotations

import math
from collections import deque
from dataclasses import dataclass, field
from typing import Deque, Literal, Optional
from datetime import datetime


# ─────────────────────────────────────────
# Types
# ─────────────────────────────────────────

PhaseLabel = Literal["COG_LEAD", "SYNC", "ACT_LEAD"]
Verdict    = Literal["GO", "WAIT", "STOP"]
TrendLabel = Literal["WIDENING", "STABLE", "CONVERGING"]


# ─────────────────────────────────────────
# Input
# ─────────────────────────────────────────

@dataclass
class SignalInput:
    """
    Normalizerから受け取る正規化済みシグナル。
    cognitive_signal, action_signal はともに [0.0, 1.0]。
    """
    cognitive_signal: float          # 認知層スコア（0〜1）
    action_signal:    float          # 行動層スコア（0〜1）
    timestamp:        str            # ISO 8601
    market:           str            # "ec" / "geopolitics" / etc.
    source:           str            # "x_keepa" / "sns_only" / etc.
    confidence:       float = 1.0   # Normalizer由来の信頼度（0〜1）

    def __post_init__(self):
        assert 0.0 <= self.cognitive_signal <= 1.0, "cognitive_signal out of range"
        assert 0.0 <= self.action_signal    <= 1.0, "action_signal out of range"
        assert 0.0 <= self.confidence       <= 1.0, "confidence out of range"


# ─────────────────────────────────────────
# Output
# ─────────────────────────────────────────

@dataclass
class CAGResult:
    """
    CAG Core の出力。Waveモジュールへの入力でもある。
    """
    timestamp:    str
    market:       str
    source:       str

    cag_value:    float          # C − A（EMA安定化後）
    delta_cag:    float          # Δ(cag_value)
    delta2_cag:   float          # Δ²(cag_value)

    phase_label:  PhaseLabel     # COG_LEAD / SYNC / ACT_LEAD
    verdict:      Verdict        # GO / WAIT / STOP
    confidence:   float          # 入力信頼度を引き継ぐ

    trend:        Optional[TrendLabel] = None   # 系列が短い場合は None
    zero_cross:   bool = False                  # 今回ゼロクロスしたか

    def to_dict(self) -> dict:
        return {
            "timestamp":   self.timestamp,
            "market":      self.market,
            "source":      self.source,
            "cag_value":   round(self.cag_value,  4),
            "delta_cag":   round(self.delta_cag,  4),
            "delta2_cag":  round(self.delta2_cag, 4),
            "phase_label": self.phase_label,
            "verdict":     self.verdict,
            "confidence":  round(self.confidence, 4),
            "trend":       self.trend,
            "zero_cross":  self.zero_cross,
        }


# ─────────────────────────────────────────
# EMA helper
# ─────────────────────────────────────────

class EMA:
    """指数移動平均（初回はシードとして入力値をそのまま使う）"""

    def __init__(self, alpha: float = 0.3):
        assert 0 < alpha <= 1.0
        self.alpha = alpha
        self._value: Optional[float] = None

    def update(self, x: float) -> float:
        if self._value is None:
            self._value = x
        else:
            self._value = self.alpha * x + (1 - self.alpha) * self._value
        return self._value

    @property
    def value(self) -> Optional[float]:
        return self._value


# ─────────────────────────────────────────
# Core Engine
# ─────────────────────────────────────────

class CAGCore:
    """
    CAG（Cognitive-Action Gap）カーネル。

    Parameters
    ----------
    ema_alpha : float
        EMAの平滑化係数。小さいほど過去重視。デフォルト 0.3。
    sync_threshold : float
        |CAG| がこの値以下のとき SYNC と判定。デフォルト 0.1。
    history_maxlen : int
        内部保持する過去CAG値の最大数。delta2 算出に使用。
    """

    def __init__(
        self,
        ema_alpha:       float = 0.3,
        sync_threshold:  float = 0.1,
        history_maxlen:  int   = 64,
    ):
        self.sync_threshold = sync_threshold
        self._cog_ema = EMA(alpha=ema_alpha)
        self._act_ema = EMA(alpha=ema_alpha)
        self._history: Deque[float] = deque(maxlen=history_maxlen)

    # --------------------------------------------------
    def update(self, inp: SignalInput) -> CAGResult:
        """シグナルを1件受け取り、CAGResultを返す。"""

        # EMA安定化
        c = self._cog_ema.update(inp.cognitive_signal)
        a = self._act_ema.update(inp.action_signal)
        cag = c - a

        # delta / delta2
        if len(self._history) >= 1:
            delta  = cag - self._history[-1]
        else:
            delta  = 0.0

        if len(self._history) >= 2:
            prev_delta = self._history[-1] - self._history[-2]
            delta2     = delta - prev_delta
        else:
            delta2 = 0.0

        # ゼロクロス検出
        zero_cross = (
            len(self._history) >= 1
            and self._history[-1] * cag < 0  # 符号反転
        )

        self._history.append(cag)

        # phase_label
        phase = self._classify_phase(cag)

        # verdict
        verdict = self._classify_verdict(cag, phase, delta)

        # trend（3点以上で判定）
        trend = self._classify_trend() if len(self._history) >= 3 else None

        return CAGResult(
            timestamp   = inp.timestamp,
            market      = inp.market,
            source      = inp.source,
            cag_value   = cag,
            delta_cag   = delta,
            delta2_cag  = delta2,
            phase_label = phase,
            verdict     = verdict,
            confidence  = inp.confidence,
            trend       = trend,
            zero_cross  = zero_cross,
        )

    # --------------------------------------------------
    def _classify_phase(self, cag: float) -> PhaseLabel:
        if abs(cag) <= self.sync_threshold:
            return "SYNC"
        return "COG_LEAD" if cag > 0 else "ACT_LEAD"

    def _classify_verdict(
        self, cag: float, phase: PhaseLabel, delta: float
    ) -> Verdict:
        """
        判定ロジック（v1.15仕様）：
          ACT_LEAD               → GO
          SYNC + delta < 0       → GO（収束途中：行動が認知に追いつきつつある）
          SYNC + delta >= 0      → WAIT
          COG_LEAD + delta < 0   → WAIT（認知先行だがギャップ縮小中）
          COG_LEAD + delta >= 0  → STOP
        """
        if phase == "ACT_LEAD":
            return "GO"
        if phase == "SYNC":
            return "GO" if delta < 0 else "WAIT"
        # COG_LEAD
        return "WAIT" if delta < 0 else "STOP"

    def _classify_trend(self) -> TrendLabel:
        """直近3点のCAG絶対値でトレンドを判定。"""
        recent = list(self._history)[-3:]
        abs_vals = [abs(v) for v in recent]
        if abs_vals[-1] > abs_vals[0] * 1.05:
            return "WIDENING"
        if abs_vals[-1] < abs_vals[0] * 0.95:
            return "CONVERGING"
        return "STABLE"

    # --------------------------------------------------
    def reset(self):
        """状態をリセット（市場切り替え時など）。"""
        self._cog_ema = EMA(alpha=self._cog_ema.alpha)
        self._act_ema = EMA(alpha=self._act_ema.alpha)
        self._history.clear()


# ─────────────────────────────────────────
# Minimal smoke test
# ─────────────────────────────────────────

if __name__ == "__main__":
    core = CAGCore()

    samples = [
        # (cognitive, action, label)
        (0.80, 0.30, "バズ開始"),
        (0.75, 0.40, "バズ継続"),
        (0.60, 0.55, "収束中"),
        (0.45, 0.60, "行動先行へ"),
        (0.35, 0.72, "行動先行・最適ゾーン"),
        (0.50, 0.50, "同期"),
    ]

    print(f"{'label':<20} {'cag':>7} {'delta':>7} {'delta2':>8} {'phase':<12} {'verdict':<6} {'trend'}")
    print("-" * 80)

    for i, (cog, act, label) in enumerate(samples):
        inp = SignalInput(
            cognitive_signal=cog,
            action_signal=act,
            timestamp=f"2026-04-06T{10+i:02d}:00:00+09:00",
            market="ec",
            source="x_keepa",
            confidence=0.85,
        )
        r = core.update(inp)
        print(
            f"{label:<20} "
            f"{r.cag_value:>7.3f} "
            f"{r.delta_cag:>7.3f} "
            f"{r.delta2_cag:>8.3f} "
            f"{r.phase_label:<12} "
            f"{r.verdict:<6} "
            f"{r.trend or '-'}"
            f"{'  ← ZERO CROSS' if r.zero_cross else ''}"
        )
