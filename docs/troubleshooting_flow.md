# 障害切り分けフロー — Robot SRE

OTel / Prometheus / Grafana で異常を検知したときの一次切り分け手順。
Prometheus アラート（`docker/observability/prod/alert_rules.yml`）の 4 パターン
（P1 メモリ枯渇 / P2a Collector プロセス異常 / P2b ROS パイプライン停滞 /
P3 LLM レイテンシ超過）に対応する。メトリクス名は M2-H（収集レイヤ最終化・2026-07-04）
確定の正典名。

関連:
- アラート定義: [`docker/observability/prod/alert_rules.yml`](../docker/observability/prod/alert_rules.yml)
- 収集レイヤ・正典メトリクス表: [`docs/observability_detail.md`](observability_detail.md) §収集メトリクス一覧
- スタック構成: [`docker/observability/README.md`](../docker/observability/README.md)
- 詳細ロードマップ: `todo/observability_otel.md` M2-F / M2-H

---

## 一次切り分けフロー

```mermaid
flowchart TD
    START([Grafana / Prometheus で異常検知]) --> Q0{どのアラートか?}

    Q0 -->|"OtelCollectorProcessAbnormal<br/>(プロセス数 != 1)"| P2a
    Q0 -->|"RosCameraStale<br/>(カメラ hz 低下)"| P2b
    Q0 -->|"JetsonMemoryLow<br/>(メモリ残量低下)"| P1
    Q0 -->|"LLMLatencyHigh<br/>(LLM 応答が遅い)"| P3

    %% ── P2a: Collector プロセス異常 ──────────────────────
    subgraph P2aG["P2a: Collector プロセス異常（0個 or 2個以上）"]
        P2a["process_memory_usage_bytes{executable_name=~otelcol}<br/>の本数を確認"] --> P2aA{"本数は?"}
        P2aA -->|"0個（停止）"| P2aB["otelcol-contrib / tegrastats_exporter<br/>のプロセス・ログを確認 → 再起動<br/>（systemd: systemctl status/restart）"]
        P2aA -->|"2個以上（2重起動）"| P2aC["重複プロセスを特定し片方を kill<br/>（既知障害: 2重起動でタレット暴走）"]
    end

    %% ── P2b: ROS パイプライン停滞 ────────────────────────
    subgraph P2bG["P2b: ROS パイプライン停滞（カメラ hz 低下）"]
        P2b["ros_topic_hz{topic=/image_raw/compressed}<br/>の推移を確認"] --> P2bA["ROS2 ノード生存確認<br/>work_event_node / metrics_node / camera node"]
        P2bA --> P2bB{"ノード生存?"}
        P2bB -->|No| P2bC["該当ノードを再起動<br/>bringup_observability.sh / bringup_hard.sh"]
        P2bB -->|Yes| P2bD["ネットワーク確認<br/>Jetson→PC:9090 疎通 / remote_write ログ"]
    end

    %% ── P1: メモリ枯渇 ─────────────────────────────────
    subgraph P1G["P1: メモリ枯渇"]
        P1["system_memory_usage_bytes{state=free}<br/>の推移を確認"] --> P1A{"AIモード切替直後?"}
        P1A -->|Yes| P1B["LLM↔YOLO の同時常駐を避ける<br/>ai_mode_manager が排他制御しているか確認"]
        P1A -->|No| P1C["リークプロセス特定<br/>process_memory_usage_bytes / nvidia-smi / ps"]
        P1B --> P1D["drop_caches が on_configure で<br/>効いているかログ確認"]
    end

    %% ── P3: 推論レイテンシ超過 ─────────────────────────
    subgraph P3G["P3: LLM レイテンシ超過"]
        P3["histogram_quantile(0.95, inference_latency_ms_bucket{model=llm})<br/>を確認"] --> P3A["llama-server ログ確認<br/>GPU offload / VRAM 断片化"]
        P3A --> P3B{"VRAM 逼迫?"}
        P3B -->|Yes| P3C["他 AI ノードを deactivate<br/>→ VRAM 解放後に再試行"]
        P3B -->|No| P3D["llama-server 再起動<br/>ウォームアップ再実行"]
    end

    P2aB --> DONE([復旧確認: メトリクス再開 / アラート解消])
    P2aC --> DONE
    P2bC --> DONE
    P2bD --> DONE
    P1D --> DONE
    P1C --> DONE
    P3C --> DONE
    P3D --> DONE
```

---

## パターン別チェックリスト

### P1: メモリ枯渇（`JetsonMemoryLow` / `system_memory_usage_bytes{state="free"} < 512000000` が 2 分継続）

Jetson Orin Nano 8GB の制約下で、重い AI ノード（LLM / YOLO / YOLO-World）が
同時常駐するとメモリが逼迫する。正典はホスト hostmetrics（M2-H）。

1. `system_memory_usage_bytes{state="free"}` の時系列を Grafana で確認し、落ち込みの起点を特定する。
2. 起点が AI モード切替と一致する場合、`ai_mode_manager_node` の排他制御
   （deactivate → cleanup → configure → activate）が正しく走ったかログで確認する。
3. 各 LifecycleNode の `on_configure` 内 `drop_caches` が効いているかを確認する。
4. モード切替と無関係な単調減少ならリークを疑い、`process_memory_usage_bytes{executable_name=~...}`
   （hostmetrics process スクレイパ）や `nvidia-smi` / `ps` でプロセス単位のメモリを確認する。

### P2a: Collector プロセス異常（`OtelCollectorProcessAbnormal` / プロセス数が 1 以外で 5 分継続）

ホスト hostmetrics の process スクレイパで otelcol-contrib の生存本数を監視する
（`--pid=host` 前提。M2-H）。0 個＝停止、2 個以上＝2重起動（既知障害: タレット暴走の原因になりうる）。

1. `count(process_memory_usage_bytes{executable_name=~".*otelcol.*"})` の値を確認する。
2. 0 個なら Collector プロセス・ログを確認して再起動する
   （systemd 化後: `systemctl status/restart otel-collector`）。
3. 2 個以上なら重複プロセスを `pgrep -fa otelcol` 等で特定し、片方を kill する。

### P2b: ROS パイプライン停滞（`RosCameraStale` / `ros_topic_hz{topic="image_raw/compressed"} < 1` が 2 分継続）

カメラはどの AI モードでも常時 publish される想定のため、hz 低下は perception 層の停止を示す。

1. `ros_topic_hz{topic="image_raw/compressed"}` の推移を確認する。
2. `work_event_node` / `metrics_node` / カメラノードの ROS2 生存を確認する。
   落ちていれば `bringup_observability.sh` / `bringup_hard.sh` で再起動。
3. どちらも生きていれば、Jetson → PC:9090 の疎通・`remote_write` の送信ログ
   （`Exporting failed. Will retry.` の有無）を確認する。

### P3: 推論レイテンシ超過（`LLMLatencyHigh` / `inference_latency_ms_bucket{model="llm"}` の p95 が 10 秒超で 1 分継続）

3A方式（各AIノード → metrics_node 集約）の Histogram。Prometheus では
`histogram_quantile(0.95, rate(inference_latency_ms_bucket{model="llm"}[5m]))` で見る。

1. Grafana でレイテンシ p95 のスパイクを確認する。
2. `llama-server` のログで GPU offload の状況・VRAM 断片化の兆候を確認する。
3. VRAM が逼迫していれば、他 AI ノードを `deactivate` して VRAM を解放後に再試行する。
4. VRAM に余裕があるのに遅い場合は `llama-server` を再起動し、ウォームアップを再実行する。

---

## 既知の環境トラブル（Jetson・開発 PC）

アラートでは捕まらない、環境側で起きる障害の記録。いずれも発生時点（2026年2月ごろ）の事実で、現行の対策状況は各項の「いま」に書く。

### E1: apt の自動更新のあと、Isaac ROS のノードが起動しなくなる

- **症状**: 前日まで動いていた `isaac_ros_yolov8` が起動せず、`NitrosContext: cudaErrorNotSupported` → `setCUDAMemoryPoolSize Error: GXF_FAILURE` → `terminate called after throwing an instance of 'std::runtime_error'`。コードは変えていない。
- **原因（2つ）**:
  1. ホストの TensorRT が 10.3.0.30 → 10.7.0.23 へ上がった。Isaac ROS のコンテナは 10.3 向けにビルドされている。
  2. `nvidia-container-toolkit` が 1.16.2 → 1.18.2 へ上がり、コンテナから見た `/dev/nvmap` のグループ権限と CUDA の初期化（`cuInit`）が変わった。
- **見分け方**: ホストとコンテナの `dpkg -l | grep -E 'tensorrt|nvinfer'` の版を並べて比べる。`/dev/nvmap` の所有グループもホストとコンテナで比べる。
- **復旧**: TensorRT 関連パッケージを 10.3.0.30-1+cuda12.5 へ、`nvidia-container-toolkit` 系を 1.16.2-1 へ `apt install --allow-downgrades` で戻し、`systemctl restart docker`。TensorRT エンジン（`.plan`）は作り直す（`models/` の書き込み権限を確認）。
- **予防**: `apt-daily.timer`・`apt-daily-upgrade.timer` を止める。`/etc/apt/preferences.d/` で `tensorrt` と `nvidia-container-toolkit` の版を固定する（`apt-cache policy` で Pinned 表示を確認）。
- **いま**: 固定が今の Jetson で効いているかは未確認。確認は `apt-cache policy tensorrt nvidia-container-toolkit`。新しいパッケージを入れる前に、上のパッケージが `apt list --upgradable` に出ないかを見る。
- 経緯の読み物: `docs/blog_drafts/article_03_jetson_autoupdate_incident.md`。

### E2: 画像トピックを流すと Jetson が固まり SSH まで切れる

- **症状**: 既定の設定で画像を流すと 2〜3 秒でフリーズし、SSH も切れる。画像サイズ・FPS・QoS を下げると続く。
- **仮説（原因は 1 つに確定していない）**: ①画像が Reliable で、受け手（PC・ネットワーク）が遅いと DDS の送り手側に未送達のメッセージが溜まりメモリ・CPU を食う ②CPU／softirq の飽和（変換・コピー・シリアライズ）③WSL・RViz の描画やネットワーク帯域がボトルネックで再送・滞留が上流へ波及 ④USB／V4L2 側。
- **見るもの**: `tegrastats`（全体）・`free`／`vmstat`（メモリ・スワップ）・`top -H`（スレッド）・`dmesg`（OOM・soft lockup）・`ss -tnp`（Send-Q/Recv-Q の肥大）・`ip -s link`（drop/overrun）。ROS 側は `ros2 topic hz／bw`。
- **対策（現行）**: 生画像を常時流さず、圧縮画像（`image_raw/compressed`）を配信し、購読も必要なときだけ（N-OBS-4、T-57）。生画像の Reliable 購読を増やさない。
- 詳細: 古い調査メモは gtd の `99_Archive/PJ-Jetson環境トラブルと復旧.md`。

> 注: このフローは異常の**一次切り分け**を目的とする。恒久対策や根本原因分析は
> 各作業領域の `todo/logs/<領域>.md` を参照すること。
