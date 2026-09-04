# Observability

> **Deprecated:** 本ドキュメントが説明する `perf/`（CSV ログベースの Telemetry Collector）は
> OpenTelemetry + Prometheus + Grafana スタックに置き換えられ廃止済み。現行の可観測性設計は
> [`docs/observability_detail.md`](observability_detail.md) を参照。本ファイルは経緯の記録として残置する。

本ドキュメントは **Human Following Robot の可観測性設計とツール**を説明する。

対象

* システム状態の観測
* パフォーマンス分析
* 障害調査

ロボットは **AI / ROS / OS / Hardware** の複数レイヤで構成されるため
各レイヤの状態を継続的に観測する必要がある。

---

# 1 Observability Architecture

ロボットの観測構造。

```
Robot System
 ├ Perception (YOLO)
 ├ Control (ROS nodes)
 ├ Actuation (Motor Control)
 ├ OS (Linux)
 └ Hardware (Jetson / Pico)

        │
        ▼

Telemetry Collector
        │
        ▼

CSV Logs
        │
        ▼

Investigation / Visualization
```

観測データは **Telemetry Collector によって自動収集**される。

---

# 2 Observability Layers

可観測性は **4つのレイヤ**で構成される。

| Layer      | Observability     |
| ---------- | ----------------- |
| Perception | detection rate    |
| Control    | cmd_vel frequency |
| System     | CPU / memory      |
| Hardware   | Jetson GPU stats  |

---

# 3 Telemetry Sources

収集対象。

| Category | Data              |
| -------- | ----------------- |
| OS       | CPU / memory      |
| Process  | running processes |
| Network  | NIC statistics    |
| ROS      | topic hz / bw     |
| Jetson   | GPU / power       |

---

# 4 Telemetry Collector

本プロジェクトでは **軽量なテレメトリ収集ツール**を実装している。

構成

```
perf/
 ├ perf_collector.sh
 ├ perf_collector.conf
 └ perf_logs/
```

特徴

* 設定駆動
* 並列収集
* ログローテーション
* OS起動時自動実行

---

# 5 Collector Configuration

設定ファイル

```
perf_collector.conf
```

フォーマット

```
name||interval||command
```

例

```
os_cpu||10||sar -u 1 1
os_mem||10||free -m
```

意味

| Field    | Description        |
| -------- | ------------------ |
| name     | metric name        |
| interval | execution interval |
| command  | shell command      |

---

# 6 Execution Modes

Collectorは **2つの実行モード**を持つ。

---

## Periodic Execution

一定周期で実行。

例

```
os_cpu||10||sar -u 1 1
```

用途

* CPU usage
* memory monitoring

---

## Streaming Execution

ストリーミングデータを取得。

設定

```
interval = 0
```

例

```
vmstat
tegrastats
ros2 topic hz
```

---

# 7 AI モード状態の観測

ROS2 の AI モード排他制御（`ai_mode_manager_node`）は状態をトピックとブラックボードに書き出す。

```bash
# 現在の AI モード確認（LLM_ACTIVE / YOLO_ACTIVE / YOLOWORLD_ACTIVE / IDLE）
ros2 topic echo /state_manager/status

# 詳細コンテキスト確認（ブラックボード）
cat /tmp/robot_context.json | python3 -m json.tool
```

```json
{
  "current_state": "LLM_ACTIVE",
  "active_node": "llm",
  "target_object": null,
  "target_coords": null,
  "scene_description": null,
  "last_updated": "2026-06-04T..."
}
```

`target_object` や `target_coords` の書き込みタイミングを観察することで、
YOLO-World 探索フローの進捗をリアルタイムに追跡できる。

| 観測ポイント | コマンド |
|-------------|---------|
| AI モード遷移 | `ros2 topic echo /state_manager/status` |
| Lifecycle 状態確認 | `ros2 lifecycle list /llm_agent_node` |
| ブラックボード状態 | `cat /tmp/robot_context.json` |
| llama-server ログ | `tail -f /tmp/llama-server.log` |

---

# 9 ROS Observability

ROS2のパイプラインを監視する。

収集例

```
ros_hz||0||ros2 topic hz /image_raw/compressed
ros_bw||0||ros2 topic bw /image_raw/compressed
ros_echo||0||ros2 topic echo /image_raw/compressed
```

観測内容

| Metric       | Purpose        |
| ------------ | -------------- |
| topic rate   | pipeline speed |
| bandwidth    | camera load    |
| data content | debugging      |

---

# 10 Jetson Observability

JetsonのGPU状態。

```
tegrastats
```

取得できる情報

| Metric      |
| ----------- |
| GPU load    |
| CPU load    |
| RAM         |
| power usage |

---

# 11 Process Monitoring

プロセス観測。

```
ps -eo user,pid,ppid,pcpu,pmem,etime,cmd
```

用途

* runaway process
* memory leak detection

---

# 12 Log Format

ログ形式

```
timestamp,host,duration_ms,exit_code,metric,data
```

例

```
20250310-120001,jetson,10,0,os_cpu,CPU,12.5
```

---

# 13 Log Rotation

ログは **時間ベースでローテーション**される。

設定

```
ROTATE_INTERVAL_SEC=3600
GENERATION=7
```

保存構造

```
perf_logs
 ├ current
 ├ 202503101200
 ├ 202503101300
```

---

# 14 Automatic Startup

Collectorは **OS起動時に自動起動**する。

起動モード

```
force-start
```

理由

* 前回クラッシュ後の復旧
* zombieプロセス排除

---

# 15 Fault Investigation

ロボットの異常調査。

例: ロボットが遅い

調査手順

```
1 CPU load
2 ROS topic rate
3 GPU load
4 memory usage
```

---

# 16 Observability Tools

使用ツール。

| Tool                | Purpose          |
| ------------------- | ---------------- |
| Telemetry Collector | metrics logging  |
| ros2 topic hz       | topic frequency  |
| ros2 topic bw       | bandwidth        |
| ros2 topic echo     | data inspection  |
| vmstat              | system load      |
| tegrastats          | Jetson GPU stats |
| /tmp/robot_context.json | AI モード・ブラックボード状態 |

---

# 17 Design Rationale

設計方針

* 軽量
* 外部依存なし
* リアルタイム安全

理由

ロボットでは

* CPU負荷
* GPU負荷
* レイテンシ

が重要なため。

---

# 18 Future Extensions

将来的拡張。

* Prometheus exporter
* Grafana dashboard
* ROS2 tracing
* fleet monitoring

---
