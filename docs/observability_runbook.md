# Observability Runbook

アラート発火時に「何を確認し、何をするか」を1本にまとめた運用手順書。
各アラートの `annotations.runbook_url` からここへ飛んでくる想定。

設計・SLI/SLO定義・ダッシュボードIA自体は [`docs/observability_detail.md`](observability_detail.md) と
`todo/robot_sre.md` §2 を参照。本書は**トリアージの手順**だけを扱う。

---

## 共通の初動

1. **`robot-overview`（Tier1）を開く** — 総合GO/NO-GOと業務サマリを一目で確認。
2. NO-GOまたは業務サマリが赤/黄なら **`robot-detail`（Tier2）へドリルダウン**し、
   該当時間帯のモードタイムライン・トレンド・下部の生ログ（Tier3）を確認。
3. 実機の安全に関わる可能性がある場合は、まず**安全な停止・退避**を優先する
   （CLAUDE.md「Jetson実機セッションの運用ルール」参照）。原因究明は実機を止めてから。

---

## S1FastBurn / S1SlowBurn（追従成功率のError Budget消費）

**意味**: `object_tracking/info` の `target_visible=True` 割合（追従維持率）が
SLO目標95%に対して速い/緩やかなペースでError Budgetを消費している。

**確認**:
- `robot-detail` の「追従維持率（時系列）」パネルで低下区間の開始時刻を特定。
- 同時刻の「モードタイムライン」でYOLO追従中だったか確認（追従していない時間は
  この指標の対象外＝NaNになるため、低下は必ず追従中に発生している）。
- 「ロスト & リカバリ結果」パネルでロスト頻発と相関しているか確認。

**対処**:
- カメラ露光（`camera_exposure_guard.py`、CLAUDE.md既知不具合）や照明条件を疑う。
- タレット追従遅延（T-35/T-36）・GVDサーチの帯（P4-6/P4-7）等、既知のリカバリー
  ロジックの限界に該当しないか `todo/object_tracking_recovery.md` を確認。
- FastBurn（2分継続）は実機で即対応、SlowBurn（15分継続）は次回セッションで
  傾向として起票（`todo/object_tracking_recovery.md` へ `- [ ]` 追記）。

---

## S2FastBurn / S2SlowBurn（ロスト復帰率のError Budget消費）

**意味**: ロスト後20秒以内に再捕捉できた割合（≒p95<20s目標のratio化）が
SLO目標95%に対して速い/緩やかなペースでError Budgetを消費している。
**諦め（giveup）はすべてSLO違反として計上される**（20秒以内に再捕捉できなかった、
という点で本質的に同じ失敗のため）。

**確認**:
- `robot-detail` の「再捕捉所要時間 p50/p95（時系列）」で悪化区間を特定。
- 「ロスト & リカバリ結果」で「諦め」が増えていないか確認（Tier1/Tier2探索が
  機能していない可能性）。
- 「自律介入 内訳」で安全停止（AMCL発散等）が同時に増えていないか確認——
  安全停止がリカバリーを中断させ、間接的にS2を悪化させることがある。

**対処**:
- Tier1軌跡先回り・Tier2探索の既知の限界（`todo/object_tracking_recovery.md`
  P4-6/P4-7/P7-5等）に該当しないか確認。
- 安全停止が同時多発している場合は `MissionSafetyStopSpike`（下記）を先に見る。

---

## MissionSafetyStopSpike（自律安全停止の頻発）

**意味**: AMCL共分散発散・GUARD持続ブロック・知覚途絶・Nav2不健全等の安全停止
（`mission_safety_stop_total`）が10分間に3回以上発火。単発は正常な防御動作
（T-AT-6-21/T-AT-6-23等）だが、頻発は環境またはソフトウェアの異常を示唆する。

**確認**:
- `robot-detail` の「自律介入 内訳（理由別）」で `reason`
  （`PERCEPTION_LOST`/`LOCALIZATION_LOST`/`NAV_UNHEALTHY`）を特定。
- Tier3の生ログで `[SAFETY]` タグのログを時系列で確認し、同一箇所で
  繰り返し発火していないか（＝特定の地点・状況に起因する再現性のある問題か）確認。

**対処**:
- `LOCALIZATION_LOST` 頻発 → AMCL復帰シーケンス（T-AT-6-23）の`amcl_recovery_*`
  パラメータ調整、または地図・環境要因（狭所・反射面等）を疑う。
- `NAV_UNHEALTHY` 頻発 → Nav2連続REJECTの原因（コストマップ・ゴール到達不能地点）
  を`todo/navigation.md`の既知debtと突き合わせる。
- `PERCEPTION_LOST` 頻発 → カメラ/知覚パイプラインの死活を`robot-overview`の
  Go/No-Goタイルで確認。
- 実機セッション中に頻発を発見した場合、CLAUDE.mdの運用ルール通り
  **その場では直さず**、再現手順とログを`todo/logs/object_tracking_recovery.md`
  等へ記録し起票する。

---

## S4FastBurn / S4SlowBurn（autonomy稼働率）・S5FastBurn / S5SlowBurn（パイプライン鮮度）

既存アラート（`robot_sre.md` Phase1で実装済み）。個別ノードの死活は
`CoreNode*Abnormal`系アラート（`alert_rules.yml`）で理由が特定できるので、
まずそちらのfiring状況を確認する。

---

## 既知の限界（このrunbook自体の未検証事項）

- 本書に記載した閾値・対処は**実インシデントでの検証を経ていない**（2026-08-16時点、
  全てdev環境の合成データでの動作確認のみ）。実機での誤検知・見逃しがあれば
  閾値ごとチューニングし、本書に反映すること（`todo/robot_sre.md` 参照）。
