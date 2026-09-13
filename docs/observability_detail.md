# Observability Detail

OpenTelemetry + Prometheus + Grafana による Robot SRE 基盤の詳細設計。

---

## 設計思想

AI ロボットは AI 推論・ROS 通信・OS 負荷・GPU 利用率が複合的に絡み合うため、**「どの入力がどの負荷を引き起こしたか」を説明できる可観測基盤** が不可欠になる。

### 可視化の2軸

```
[仕事量の軸]  音声コマンド・AI指示・YOLO検出 ─→ 何が来たか・いつ来たか
                       ↓ 相関分析（同一タイムライン）
[負荷の軸]    CPU / GPU / メモリ / ROS hz ────→ どのくらい消費したか
```

---

## スタック全体像

**M2-H（2026-07-04）でレイヤ分離を確定。** CPU/メモリ/ロードと Jetson GPU/温度/電力は
「ホスト側」、ROS 系（仕事量イベント・トピック hz・推論レイテンシ）は「コンテナ側」で
収集する。判断根拠は下記「収集レイヤの設計判断」を参照。

```
Jetson ホスト（常駐プロセス。Docker 停止時も生存）
  otelcol-contrib
    ├─ hostmetrics(cpu/memory/load) ─→ system_cpu_* / system_memory_usage_bytes
    ├─ hostmetrics(process)         ─→ process_*{executable_name}（死活・2重起動検知）
    └─ otlp receiver :4317 ←─────────┐
  tegrastats_exporter.py ───OTLP────┘  jetson_gpu_util_pct / jetson_cpu_temp_c /
                                        jetson_gpu_temp_c / jetson_power_mw

Docker コンテナ（Jetson、ROS 2 Humble）
  work_event_node  ─┐  軸1: 音声コマンド・YOLO検出・状態遷移 → OTel Counter
  metrics_node     ─┤  軸2: ROS topic hz/bw・推論レイテンシ(3A集約) → OTel Gauge/Histogram
                   └─ gRPC (OTLP) → otelcol-contrib :4317（上記と同一 Collector）
                                           │
                                           │ prometheusremotewrite
                                           │ ※ PC 未起動時は file_storage にバッファ
                                           │   復旧後に自動再送
                                           ▼
                                  Prometheus（PC / WSL Docker）+ alert_rules.yml
                                           │
                                        Grafana
                                  ├── Canvas パネル: Jetson ハード層（温度・電力・メモリ）
                                  └── Node Graph パネル: ROS2 ノード・トピックフロー
```

### 収集レイヤの設計判断（M2-H）

- 標準 Docker（`--pid` 未指定＝コンテナ独自 PID 名前空間、`--network host` のみ）では
  `/proc/{meminfo,stat,loadavg}` は非名前空間化のため、コンテナ内 vmstat とホストの値は
  **同一**。二重取得は無駄なので一本化する。
- 正典をホストにする理由: **コンテナが落ちても CPU/メモリ/ロードの監視を継続**でき、
  「コンテナ落ち → OS レイヤから原因調査」ができる。tegrastats も同じ理由でホスト側
  `local/observability/tegrastats_exporter.py`（`parse_tegrastats_line` を再利用）に移設。
- ROS 系（仕事量イベント・トピック hz/bw・推論レイテンシ）はコンテナ内でしか取得できない
  （ROS グラフへの subscribe が必要）ためコンテナ側のまま。
- 死活監視は「プロセス数の異常検知（0個=停止／2個以上=2重起動）」＋「トピック hz 閾値」の
  2本立て（`docker/observability/prod/alert_rules.yml` 参照）。前者は ROS コンテナが
  `--pid=host`（`docker/jetson/scripts/run_detached.sh`）で起動されていることが前提で、
  ホストの PID 名前空間からコンテナ内プロセスも同様に見える。

---

## ファイル構成

| 場所 | 用途 |
|---|---|
| `local/observability/` | Jetson ホスト向け（OTel Collector 設定・起動スクリプト） |
| `docker/observability/` | PC（WSL）向け（Prometheus + Grafana Docker Compose）。`prod/`＝本番(Jetson受信・:9090/:3000)と`dev/`＝ローカル検証(:9091/:3001)を完全分離。データは `data/` bind mount・git管理外 |
| `src/toyof_robot_observability/` | ROS 2 ノード（work_event_node / metrics_node） |
| `src/toyof_robot_bringup/launch/observability.launch.py` | launch ファイル |

---

## フリート拡張性

OTel Collector の `service.instance_id` ラベルにより、複数台のロボットからのメトリクスを同一 Prometheus インスタンスに集約できる。

```yaml
# otelcol-contrib config (Jetson ホスト)
resource:
  attributes:
    - key: service.instance_id
      value: "robot-01"   # 台ごとに変更するだけで済む
      action: upsert
```

Grafana ダッシュボードの変数に `instance` フィルタを追加するだけで、2台目以降のロボットを横並びで比較できる設計。

---

## 収集メトリクス一覧（正典・M2-H 確定）

### HOST レイヤ

| メトリクス名 | 種別 | 収集元 | 内容 |
|---|---|---|---|
| `system_cpu_time_seconds_total` | Counter | hostmetrics(cpu) | CPU 時間（state ラベルで idle/user/system 等） |
| `system_memory_usage_bytes{state}` | Gauge | hostmetrics(memory) | メモリ使用量（state="free"\|"used"\|"cached" 等） |
| `system_cpu_load_average_1m/5m/15m` | Gauge | hostmetrics(load) | ロードアベレージ |
| `process_*{executable_name,pid}` | Gauge/Counter | hostmetrics(process) | 全プロセスの CPU/メモリ等（死活・2重起動検知に使用） |
| `jetson_gpu_util_pct` | Gauge | tegrastats_exporter（ホスト） | GPU 使用率 (%) |
| `jetson_cpu_temp_c` | Gauge | 同上 | CPU 温度 (℃) |
| `jetson_gpu_temp_c` | Gauge | 同上 | GPU 温度 (℃) |
| `jetson_power_mw` | Gauge | 同上 | 総消費電力 (mW) |

### CONTAINER レイヤ（work_event_node — 軸1: 仕事量）

| メトリクス名 | 種別 | 内容 |
|---|---|---|
| `work_event_count{topic,...}` | Counter | トリガーイベント数（音声・状態遷移・検出・追従） |
| `work_event_chars{topic}` | Counter | テキスト系イベントの文字数 |
| `work_event_detections{topic}` | Counter | YOLO 検出オブジェクト数 |

### CONTAINER レイヤ（metrics_node — 軸2: 負荷・レイテンシ）

| メトリクス名 | 種別 | 内容 |
|---|---|---|
| `ros_topic_hz{topic}` | Gauge | 各 ROS トピックの publish 周波数 |
| `ros_topic_bps{topic}` | Gauge | 各 ROS トピックの帯域 (B/s) |
| `inference_latency_ms{model}` | Histogram | 推論レイテンシ（3A方式・`/metrics/inference_latency` 集約。model="yolo"\|"depth"\|"llm"\|"stt"） |

> `os_*` / コンテナ内 `jetson_*` ゲージ（`metrics_node.py` 内、`enable_vmstat` /
> `enable_tegrastats` パラメータ）は汎用環境向けフォールバックとして残しているが、
> 本プロジェクトの既定運用では **無効**（ホスト側収集が正典）。

### 推論レイテンシの計測（3A方式・producer 側の実装レシピ）

`inference_latency_ms{model}` は **各AIノードが計測して publish → metrics_node が集約**
する 3A 方式で収集する（SDK を各ノードで直接叩かない）。producer 側の定型実装は
`toyof_robot_observability` の再利用ヘルパ `LatencyReporter` に集約した。各AIノードは
以下の数行を足すだけで計測区間を `/metrics/inference_latency` に流せる。

```python
from toyof_robot_observability.latency_reporter import LatencyReporter

# __init__ で1回だけ生成（model は "yolo" | "depth" | "llm" | "stt"）
self._latency = LatencyReporter(self, 'yolo')

# 計測したい推論区間を with で包む
with self._latency.measure():
    result = self._infer(image)
```

- **fail-open**: `toyof_robot_interfaces` 未導入・publisher 生成失敗・publish 失敗の
  いずれでも例外を外へ投げず warn ログのみ（`report` が no-op 化）。可観測性の計測が
  推論本体を止めないための設計。
- **時刻源**: `Stopwatch` は既定で `time.monotonic`。逆転（クロックスキュー）は 0.0 に
  クランプし、負のレイテンシを Histogram に混入させない。
- **純ロジック分離**: 計測本体（`elapsed_ms` / `normalize_model` / `Stopwatch`）は
  ROS2 非依存の `latency_reporter_logic.py` に分離し、`test_latency_reporter_logic.py`
  でホスト単体テスト（コンテナ不要）。
- 計測ポイント（どの区間を測るか）の各ノード結線は M2-H-7（要事前相談）。ヘルパ導入で
  ノード側の追加はボイラープレート数行に収まる。

## Canvas パネル化構想（M2-C・設計のみ、実装は次回 UI 操作セッション待ち）

`todo/observability_otel.md` M2-C の「ロボット構成図レイアウト」を、実際に Grafana
Canvas パネルへ落とし込むための配置案。**Canvas パネルの要素追加・データバインドは
Grafana UI 上の手作業が必須**なため、このセッション（docker/ROS とも利用不可）では
実装できない。次に Grafana UI を触れるセッションで、この配置案どおりに要素を作る。

電力・データの流れ方向（上→下）に3段のブロックを積み、各ブロックの下に主要メトリクスを
`metric value` 要素としてバインドする。

```
┌────────────────────────────────────────────┐
│ Jetson Orin Nano（ホスト）                    │
│  CPU load 1m : system_cpu_load_average_1m    │ 閾値 黄2 / 赤4
│  GPU 使用率  : jetson_gpu_util_pct           │ 閾値 黄60 / 赤80
│  GPU/CPU 温度: jetson_gpu_temp_c / jetson_cpu_temp_c │ 閾値 黄65 / 赤75
│  消費電力    : jetson_power_mw               │ 閾値 黄5500 / 赤7000
│  メモリ空き  : system_memory_usage{state="free"} │ 閾値 赤<512MB / 黄<1GB
└──────────────────┬───────────────────────────┘
                    │ （矢印アイコン要素・ラベル "Docker"）
┌──────────────────▼───────────────────────────┐
│ ROS2 コンテナ                                 │
│  仕事量      : rate(work_event_count[1m]) * 60 （/min換算）│
│  推論レイテンシ: rate(inference_latency_ms_sum[5m])       │
│               / rate(inference_latency_ms_count[5m])      │ model別、robot_detail.json
│               （id:38 のBarチャートと同一クエリ・相互参照）│  パネルへリンク
│  カメラ hz   : ros_topic_hz{topic="/image_raw/compressed"} │ 閾値 赤<1
│  プロセス別メモリ: sum by (executable_name) (process_memory_usage) │ robot_detail.json id:39 と同一
└──────────────────┬───────────────────────────┘
                    │
┌──────────────────▼───────────────────────────┐
│ 4WD シャーシ（将来拡張・プレースホルダ）        │
│  エンコーダ・モーター電流は未計測                │ テキスト要素のみ、メトリクス未バインド
└────────────────────────────────────────────┘
```

- 各ブロックは `rectangle` 要素、メトリクス値は `metric value` 要素（datasource
  Prometheus・上記 PromQL をそのままクエリに設定）、矢印は `icon` 要素でよい。
- 閾値の色分けは `robot_detail.json` の対応する stat パネル（id 3-8）・M2-E で追加した
  timeseries パネル（id 38-39）と同じ値に揃える（ドリフト防止）。
- 「ROS2コンテナ」ブロックの推論レイテンシ・プロセス別メモリの2項目は、今回 M2-E で
  `robot_detail.json` に追加した id:38（推論レイテンシ比較）・id:39（モード組み合わせ別
  メモリ使用量）と同一クエリにしてあるため、Canvas 側は「今の値」・timeseries 側は
  「推移」という役割分担になる。
- 現状の Stat + TimeSeries 構成（`robot_detail.json`）で情報としては同等のものが既に
  見えているため、Canvas 化自体の優先度は高くない（`todo/observability_otel.md` M2-C
  の位置づけどおり、ポートフォリオ向けの見た目改善が主目的）。
