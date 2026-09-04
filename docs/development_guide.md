# Development Guide

ビルド手順、**各モードの動作確認コマンド**、カスタムパッケージ一覧、リポジトリ構成。

> 内部ロジック・起動シーケンス図は [`mode_details.md`](mode_details.md) を参照。

---

## Custom ROS2 Packages

| Package | Type | Role |
|---|---|---|
| `toyof_robot_interfaces` | ament_cmake | カスタムメッセージ定義（ObjectTrackingInfo） |
| `toyof_robot_bringup` | ament_cmake | launch ファイル・config・URDF の一元管理 |
| `toyof_robot_vehicle` | ament_python | 車両HW制御（Pico W通信・オドメトリ・サーボ・IMU） |
| `toyof_robot_sensor` | ament_python | ICM-20948 IMUドライバ |
| `toyof_robot_ai_vision` | ament_cmake (C++/Python) | 視覚AIパイプライン（Depth推定・点群生成 C++コンポーネント + 可視化Python） |
| `toyof_robot_navigation` | ament_python | SLAM・自律探索（フロンティア/GVD探索ロジック） |
| `toyof_robot_localization` | ament_python | AprilTag 自己位置推定 |
| `toyof_robot_ai_control` | ament_python | 統合制御（追従対象選択・Nav2ゴール生成・LLMエージェント・状態管理） |
| `toyof_robot_ai_speech` | ament_python | 音声AI（STT/TTS） — `speech_io_node` |
| `toyof_robot_observability` | ament_python | 可観測性（OTel Counter/Gauge） — `work_event_node` / `metrics_node` |

---

## Repository Structure

```
src/
├── toyof_robot_interfaces/     # カスタムメッセージ定義（ament_cmake）
│   └── msg/ObjectTrackingInfo.msg
│
├── toyof_robot_bringup/        # 起動・設定一元管理（ament_cmake）
│   ├── config/                 # 全パラメータ yaml
│   │   ├── follow_v0.yaml, nav2_params*.yaml, ekf.yaml
│   │   ├── wheel_odom.yaml, pico_bridge_node.yaml, scan_filter.yaml
│   │   └── imu_calibration.yaml, Tmini-Plus-SH.yaml ...
│   ├── launch/
│   │   ├── robot_full_system.launch.py     # メタ起動（3層一括）
│   │   ├── hardware_bringup.launch.py      # 第1層: センサー（camera_perception.launch.py をネスト）
│   │   ├── camera_perception.launch.py     # ai_perception_container + v4l2_camera + format_converter
│   │   ├── autonomy.launch.py              # 第2層: SLAM / Nav2
│   │   ├── ai_core_managers.launch.py      # 第3層: AI頭脳（音声・LLM・YOLO管理）
│   │   ├── yolo_object_follow.launch.py    # YOLO人追従パイプライン（LoadComposableNodes）
│   │   ├── robot_bringup.launch.py         # 後方互換（hardware+autonomy相当）
│   │   ├── bringup-localmode.launch.py
│   │   ├── observability.launch.py         # 可観測性ノード（work_event/metrics）単体起動（復旧用）
│   │   ├── sim_hardware_bringup.launch.py  # hardware_bringup の Gazebo Classic 版（x86 sim）
│   │   ├── devenv.launch.py                # スタブ構成（pico_stub + detection_stub）
│   │   └── state_transition_test.launch.py # Lifecycle 排他制御テスト（ダミーノード）
│   ├── config/
│   │   ├── ai_core_params.yaml              # AI頭脳層全ノードパラメータ（STT/TTS/LLM/YOLO/Mapping等）
│   │   └── ... (nav2_params*, ekf, wheel_odom, etc.)
│   └── urdf/robot.urdf
│
├── toyof_robot_vehicle/        # 車両ハードウェア制御（ament_python）
│   └── toyof_robot_vehicle/
│       ├── pico_bridge_node.py       # Jetson↔Pico W シリアルブリッジ
│       ├── wheel_odom_node.py        # エンコーダ→ホイールオドメトリ
│       ├── turret_tracker_node.py    # PIDカメラ砲塔制御
│       ├── motor_driver_node.py      # モーター直接制御（旧実装）
│       ├── pico_stub_node.py         # Picoスタブ（ハードウェアなし確認用）
│       ├── imu_yaw_aligner_node.py   # IMUヨー補正
│       ├── scan_filter_node.py       # LiDARフィルタ（車体除外 + 追従対象除外）
│       └── scan_filter_logic.py      # フィルタロジック（ROS2非依存・pytest対象）
│
├── toyof_robot_sensor/            # ICM-20948 IMUドライバ（ament_python）
│   └── toyof_robot_sensor/imu.py
│
├── toyof_robot_ai_vision/      # 視覚AI（ament_cmake: C++ + Python）
│   ├── src/
│   │   ├── ai_spatial_estimator.cpp       # Depth Anything→空間座標 (ComposableNode)
│   │   └── tensor_to_depth_image_node.cpp # テンソル→深度画像+点群 (ComposableNode)
│   └── toyof_robot_ai_vision/
│       ├── bbox_visualizer_node.py        # 検出可視化（デバッグ用）
│       ├── yolo_cpu_detector_node.py      # CPU YOLO（カメラあり確認用）
│       └── detection_stub_node.py         # 検出スタブ（ハードウェアなし確認用）
│
├── toyof_robot_navigation/     # SLAM・自律探索（ament_python）
│   └── toyof_robot_navigation/
│       ├── mapping_lifecycle_node.py          # SLAM + フロンティア探索自律地図作成（Step 5）
│       ├── frontier_explorer_logic.py         # フロンティア探索ロジック（ROS2非依存）
│       ├── gvd_explorer_logic.py              # GVD 探索ロジック（ROS2非依存）
│       ├── gvd_explorer_strategy.py           # GVD 探索戦略
│       ├── explorer_factory.py                # 探索戦略ファクトリ
│       ├── explorer_interface.py              # 探索インタフェース定義
│       ├── map_harden_node.py                 # 壁の隙間の未知セルを占有化して再発行（N24-54）
│       └── map_harden_logic.py                # 硬化ロジック（ROS2非依存・汎用）
│
├── toyof_robot_localization/   # AprilTag 自己位置推定（ament_python）
│   └── toyof_robot_localization/
│       ├── tag_localization_manager_node.py   # AprilTag 自己位置推定マネージャー
│       ├── tag_localization_logic.py          # AprilTag ローカライゼーションロジック（ROS2非依存）
│       └── tag_map_recorder_node.py           # AprilTag マップ記録ノード
│
├── toyof_robot_ai_control/     # 統合制御・LLM・AI モード管理（ament_python）
│   └── toyof_robot_ai_control/
│       ├── object_tracking_info_node.py       # YOLO検出→追従対象選択・制御誤差算出
│       ├── follow_goal_generator_node.py      # Nav2ゴール生成・追従
│       ├── follow_path_generator_node.py      # Nav2 FollowPath制御
│       ├── person_follower.py                 # シンプル追従（旧実装）
│       ├── llm_agent_node.py                  # LLMエージェント（脳）
│       ├── robot_agent.py                     # ROS2コマンドディスパッチ・Gemini VLM
│       ├── ai_mode_manager_node.py            # AI モード排他制御マネージャー（旧 state_manager）
│       ├── yolo_follow_lifecycle_node.py      # YOLOv8 物体追従 Lifecycle ノード（Step 3）
│       ├── yoloworld_lifecycle_node.py        # YOLO-World + Depth Anything 物体探索（Step 4）
│       ├── yoloworld_worker_main.py           # yoloworld サブプロセスワーカー
│       ├── camera_snapshot_service_node.py    # カメラスナップショットサービス（Step 4）
│       ├── localization_session_manager_node.py # Nav2/AMCL地図確立の一元管理（常駐Lifecycle、排他制御対象外）
│       └── dummy_lifecycle_node.py            # テスト用ダミー Lifecycle ノード（Step 1）
│
├── toyof_robot_ai_speech/      # 音声AI（ament_python）
│   └── toyof_robot_ai_speech/
│       ├── speech_io_node.py             # STT/TTS（耳・口）
│       └── audio_recorder.py, stt_engine.py, tts_engine.py
│
├── toyof_robot_observability/  # 可観測性（ament_python）
│   └── toyof_robot_observability/
│       ├── work_event_node.py       # 軸1: 仕事量・トリガーイベント収集（OTel Counter）
│       └── metrics_node.py          # 軸2: OS / ROS 2 負荷メトリクス収集（vmstat・tegrastats）
│
├── image_pipeline/             # ROS2 image_pipeline（外部）
└── isaac_ros_compression/      # Isaac ROS画像圧縮（外部）
```

リポジトリ直下（`src/` 以外）:

```
isaac_ros-dev/
├── docker/
│   ├── Dockerfile.robotcar         # Isaac ROS custom image (Jetson)
│   ├── jetson/                     # Jetson 用カスタム Dockerfile・起動スクリプト
│   ├── scripts/entrypoint_additions/
│   └── sim/                        # x86 Gazebo シミュレーション環境
│       ├── Dockerfile / docker-compose.yml / entrypoint.sh / run_mapping_test.sh
├── models/                         # TensorRT engines / ONNX models (.gitignore 対象)
├── map/                            # SLAM maps
├── local/
│   ├── observability/              # Jetson ホスト向け OTel Collector 設定
│   └── monitoring/                 # PC（WSL）向け Prometheus + Grafana
├── bringups/                       # Shell bringup scripts
├── docs/
├── tools/
└── pico_firmware/
```

### AIモデルファイル

| ファイル | 用途 |
|---|---|
| `models/yolov8s.engine` | YOLOv8 TensorRTエンジン（人検出） |
| `models/yolov8s.onnx` | YOLOv8 ONNXモデル |
| `models/yolov8s-worldv2.pt` | YOLO-World PyTorch モデル（Zero-shot 物体探索） |
| `models/depth_anything_v2_metric_hypersim_vits.engine` | Depth Anything V2 TensorRTエンジン |
| `models/depth_anything_v2_metric_hypersim_vits_single.onnx` | Depth Anything V2 ONNXモデル |
| `models/camera_info.yaml` | カメラキャリブレーション情報 |

---

## Development (x86 sim 環境)

Jetson が手元にないときは `robotcar-sim:latest`（`docker/sim/`）で開発できる。
Gazebo シミュレーション・pytest・スタブノード起動をこれ一本でカバーする。

詳細な起動手順 → [`docker/sim/README.md`](../docker/sim/README.md)

---

## Testing

各機能は独立してテストできる。ハードウェアが不要なテストは `robotcar-sim` コンテナで実行可能。

### 0. ビルド確認（前提）

```bash
# コンテナ内で実行
cd /workspaces/isaac_ros-dev
source /opt/ros/humble/setup.bash
colcon build --symlink-install --cmake-args -DBUILD_TESTING=OFF \
  --packages-select toyof_robot_interfaces toyof_robot_sensor toyof_robot_vehicle \
    toyof_robot_ai_vision toyof_robot_ai_control toyof_robot_ai_speech toyof_robot_bringup
```

**確認:** `Summary: N packages finished` と表示され ERROR がないこと。

---

### 1. Lifecycle 状態遷移（ハードウェア不要）

ROS2 Lifecycle ノードの排他制御ロジックをダミーノードで検証する。

```bash
# ターミナル1: 起動
source install/setup.bash
ros2 launch toyof_robot_bringup state_transition_test.launch.py

# ターミナル2: A → B と切り替え
ros2 topic pub --once /state_manager/command std_msgs/String "{data: 'activate:dummy_a'}"
sleep 5
ros2 topic pub --once /state_manager/command std_msgs/String "{data: 'activate:dummy_b'}"

# 状態確認
ros2 topic echo /state_manager/status --once
cat /tmp/robot_context.json
```

**確認:** `status` が `IDLE`、`active_node` が `"dummy_b"` になっていること。

---

### 2. 音声対話 AI（マイクなし・実機またはコンテナ）

LLM コマンド判定・TTS 発話・状態遷移のエンドツーエンド確認。

```bash
# ターミナル1: システム起動（LLM が自動 activate）
source install/setup.bash
ros2 launch toyof_robot_bringup ai_core_managers.launch.py

# ターミナル2: テキストを直接注入
# 人追従（キーワード: 人追従）
ros2 topic pub --once /speech/user_text std_msgs/String "{data: 'ヘイロボ、人追従を開始して'}"

# 物体検索（キーワード: 物体検索）
ros2 topic pub --once /speech/user_text std_msgs/String "{data: 'ヘイロボ、物体検索。赤いマグカップを探して'}"

# カメラ説明（キーワード: カメラ説明 / Gemini API が必要）
ros2 topic pub --once /speech/user_text std_msgs/String "{data: 'ヘイロボ、カメラ説明をして'}"

# 停止（キーワード: ストップ）
ros2 topic pub --once /speech/user_text std_msgs/String "{data: 'ヘイロボ、ストップ'}"

# TTS 発話内容を確認
ros2 topic echo /speech/speak_command

# コンテキスト確認
cat /tmp/robot_context.json
```

**確認:**
- `/speech/speak_command` に応答テキストが publish される
- 人追従コマンドで `status` が `LLM_ACTIVE` → `YOLO_ACTIVE` に遷移する
- 「止まって」で `YOLO_ACTIVE` → `LLM_ACTIVE` に戻る

---

### 3. カメラスナップショット（カメラ接続時）

```bash
# ターミナル1: カメラ起動（640×480）
source install/setup.bash
ros2 run v4l2_camera v4l2_camera_node \
  --ros-args -p image_size:="[640,480]" -p frame_rate:=10

# ターミナル2: スナップショットサービス
ros2 run toyof_robot_ai_control camera_snapshot_service_node

# ターミナル3: テスト
ros2 service call /snapshot/camera/take std_srvs/srv/Trigger '{}'
ls -la /tmp/snapshot.jpg
```

**確認:** `success: True` が返り `/tmp/snapshot.jpg` が更新されること。

---

### 4. 物体探索 — YOLO-World + Depth Anything V2（実機）

```bash
# ai_core_managers.launch.py + カメラ起動済みの状態で
# 探索対象をカメラ前に置いてから実行
ros2 topic pub --once /speech/user_text std_msgs/String "{data: 'ヘイロボ、赤いマグカップを探して'}"

# 進捗確認（1秒間隔で更新）
watch -n 1 'cat /tmp/robot_context.json | python3 -m json.tool'
```

**確認:**
1. `target_object` が `"赤いマグカップ"` に書き込まれる
2. `status` が `YOLOWORLD_ACTIVE` に遷移する
3. 検出後 `target_coords` に `{x, y, z, frame}` が書き込まれる
4. `status` が `IDLE` に戻り TTS で「発見しました」と発話する

---

### 5. 視覚追従パイプライン（x86 スタブ）

実機なしで Nav2 ゴール生成・追従ロジックを確認する（`robotcar-sim` コンテナ内）。

```bash
# robotcar-sim コンテナ内
colcon build --packages-select toyof_robot_interfaces toyof_robot_vehicle \
  toyof_robot_ai_vision toyof_robot_ai_control toyof_robot_bringup --symlink-install
source install/setup.bash

ros2 launch toyof_robot_bringup devenv.launch.py

# 別ターミナルで確認
ros2 topic echo /cmd_vel             # 追従コマンド速度
ros2 topic echo /detections_output   # スタブ検出データ
ros2 topic echo /object_tracking/info
```

---

### 6. YOLO 人追従 — 単体起動（実機）

AI コアスタック（LLM 等）を使わず、人追従パイプラインのみを直接起動する場合。

**前提:** `hardware_bringup.launch.py` が起動済みであること（LiDAR・カメラ・Pico）

```bash
# コンテナ内で実行

# local モード（デフォルト・マップ不要）
# → Nav2 ローカルコストマップ + YOLO (goal_frame:=odom)
./bringups/bringup_humanflowing.sh

# SLAM モード（地図を作りながら追従）
# → SLAM Toolbox + YOLO (goal_frame:=map)
./bringups/bringup_humanflowing.sh slam

# global モード（既存マップで全域ナビ追従）
# → Nav2 full (AMCL) + YOLO (goal_frame:=map)
./bringups/bringup_humanflowing.sh global
./bringups/bringup_humanflowing.sh global /workspaces/isaac_ros-dev/map/myroom.yaml
```

> **Ctrl+C** で Nav2 と YOLO パイプラインが連動して終了する。

**確認:**
```bash
ros2 node list | grep -E "controller_server|yolo|follow"
ros2 topic echo /object_tracking/info  # 検出・追従状態
ros2 topic echo /cmd_vel               # 速度指令
```

---

### 7. SLAM + フロンティア探索による自律地図作成（実機）

**前提:** `hardware_bringup.launch.py` が起動済みであること（LiDAR・カメラ・Pico）

```bash
# ターミナル1: 第3層（mapping_node も登録される）
source install/setup.bash
ros2 launch toyof_robot_bringup ai_core_managers.launch.py

# ターミナル2: フロンティア探索開始
ros2 topic pub --once /state_manager/command std_msgs/String "{data: 'activate:mapping'}"

# 途中停止（ストップワードで地図保存 → LLM モードへ復帰）
ros2 topic pub --once /speech/user_text std_msgs/String "{data: '止まって'}"
```

**確認:**
```bash
ros2 topic echo /state_manager/status          # MAPPING_ACTIVE → LLM_ACTIVE に遷移
ros2 topic echo /map --no-arr                  # SLAM マップが更新されること
ls -la /workspaces/isaac_ros-dev/map/          # explored_map.pgm / .yaml / .posegraph が保存される
```

> ロジック詳細（探索ループ・フロンティア検出アルゴリズム）→ [`mode_details.md`](mode_details.md)

---

### Lifecycle 状態確認コマンド（共通）

```bash
ros2 lifecycle list /llm_agent_node
ros2 lifecycle list /yolo_follow_node
ros2 lifecycle list /yoloworld_node
ros2 lifecycle list /mapping_node

# 手動でモードを切り替える
ros2 topic pub --once /state_manager/command std_msgs/String "{data: 'activate:llm'}"
ros2 topic pub --once /state_manager/command std_msgs/String "{data: 'activate:yolo'}"
ros2 topic pub --once /state_manager/command std_msgs/String "{data: 'activate:mapping'}"
ros2 topic pub --once /state_manager/command std_msgs/String "{data: 'deactivate:yolo'}"
```


---

## 初回セットアップ（新規マシン）

> `CLAUDE.md` から移設（2026-08-29）。新しい Jetson / PC で1回だけ実行する手順。

### USBスピーカー音量設定

USB Speaker (card 0) はデフォルトで PCM ボリュームが 0% になっている。ホスト側で一度設定して永続化する。

```bash
# ボリュームを80%に設定
amixer -c 0 sset PCM 80%

# 設定を永続化（/var/lib/alsa/asound.state に保存）
sudo alsactl store 0
```

> **注意:** コンテナ内で `amixer` を実行しても再起動で消える。必ずホスト側で実行すること。

### カスタムDockerfileの検索パス設定

`docker/jetson/` にあるカスタムDockerfileを `run_dev.sh` のビルドで参照させるため、ホームディレクトリに config ファイルを作成する。

```bash
echo "CONFIG_DOCKER_SEARCH_DIRS=(\"$HOME/workspaces/isaac_ros-dev/docker/jetson\")" \
  > ~/.isaac_ros_common-config
```

これにより `run_dev.sh -i ros2_humble.robotcar` 実行時に:
- `docker/jetson/Dockerfile.robotcar` など → このリポジトリの `docker/jetson/` を優先参照
- それ以外 → `src/isaac_ros_common/docker/`（Nvidia）にフォールバック

### ローカル（ホスト側）セットアップ一括実行

`local/` 配下（現状は可観測性基盤）のホスト側セットアップは、コンテナの外・Jetson/PCホスト上で1回実行する。

```bash
bash local/setup.sh
```

OTel Collectorバイナリのダウンロード・ホスト側Python依存（`opentelemetry-sdk`等）のインストールまでを行う。
systemd自動起動（otel-collector/tegrastats-exporter/observability-watchdog）はsudoが必要な別ステップ:

```bash
bash local/observability/install_systemd.sh
```

---
