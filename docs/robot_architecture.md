# Robot Architecture

本ドキュメントは **Human Following Robot Car の詳細アーキテクチャ**を説明する。

対象範囲

* ROS2 ノード構成（3層アーキテクチャ）
* AI モード排他制御（Lifecycle ノード管理）
* 知覚・制御パイプライン
* MCU 通信プロトコル
* センサーフュージョン
* 安全設計

---

# 1 System Overview

ロボットは **3層アーキテクチャ + AI 頭脳層**で設計されている。

```
┌─────────────────────────────────────────────────┐
│  第3層: AI 頭脳層（ai_core_managers.launch.py）  │
│  speech_io / LLM / YOLO追従 / YOLO-World        │
└───────────────────┬─────────────────────────────┘
                    │ /state_manager/command
┌───────────────────▼─────────────────────────────┐
│  第2層: 自律移動層（autonomy.launch.py）          │
│  SLAM Toolbox / Nav2 / AMCL                     │
└───────────────────┬─────────────────────────────┘
                    │ /cmd_vel / /scan_*_filtered
┌───────────────────▼─────────────────────────────┐
│  第1層: ハードウェア層（hardware_bringup.launch.py）│
│  カメラ / LiDAR / IMU / Pico W 通信 / オドメトリ  │
└─────────────────────────────────────────────────┘
```

---

# 2 Hardware Architecture

```
Jetson Orin Nano (8GB)
      │
      │ USB Serial (UART)
      ▼
Raspberry Pi Pico W
      ├── PWM → BTS7960 Motor Driver → 4WD Motors
      ├── PWM → Pan-Tilt Servo (Turret)
      ├── I2C → VL53L0X ToF Sensor
      └── GPIO → Quadrature Encoder ×4

USB Camera (on Turret) → Jetson (UVC)
YDLIDAR Tmini-Plus      → Jetson (USB)
ICM-20948 IMU           → Jetson (I2C)
```

### Jetson 役割

* TensorRT AI 推論（YOLOv8 / Depth Anything V2 / YOLO-World）
* ROS2 ノード実行
* Nav2 ナビゲーション
* LLM 推論（llama-server: Qwen2.5 1.5B）
* STT / TTS（faster-whisper / Open JTalk）

### Pico W 役割

* PWM 生成（モーター・サーボ）
* エンコーダ読み取り
* ToF 距離測定
* UART ウォッチドッグ安全制御

---

# 3 第1層: ハードウェア層（hardware_bringup.launch.py）

```
YDLIDAR Tmini-Plus ──→ /scan ──→ scan_filter_node ──┬─→ /scan_body_filtered
                                                     │    (自車体のみ除去。cmd_vel
                                                     │     直接駆動の安全チェック用)
                                                     └─→ /scan_target_filtered
                                                          (自車体+追従対象を除去。
                                                           SLAM/AMCL/コストマップ用)

ICM-20948 ──→ imu_node ──→ /imu/data
                   │
                   ▼
           imu_filter_madgwick ──→ imu_yaw_aligner ──→ /imu/data_aligned

pico_bridge_node ──→ /pico/rev_*        (エンコーダ生値)
                 ──→ /pico/tof_m        (ToF距離 [m])
                 ──→ /pico/servo_state  (サーボ角度)
                 ←── /cmd_vel          (速度指令)
                 ←── /pico/servo_target (サーボ目標)

wheel_odom_node (/pico/rev_* + 速度依存補正) ──→ /wheel_odom

ekf_filter_node (/wheel_odom + /imu/data_aligned) ──→ /odometry/filtered → Nav2

ai_perception_container（常駐 ComposableNodeContainer）
  └── v4l2_camera ──→ /image_raw + /camera_info
  └── format_converter ──→ /image (rgb8)
  └── format_converter_bgr ──→ /apriltag/image_bgr (bgr8, AprilTag向け系統)
  └── rectify_node ──→ /apriltag/image_rect + /apriltag/camera_info_rect
  └── [YOLO/Depth ノードは yolo_follow_node 起動時に動的注入]
```

`format_converter_bgr` + `rectify_node` は AprilTag自己位置推定（`todo/apriltag_localization.md`）向けに追加。
YOLO用の `format_converter`（rgb8）とは別系統にして影響を分離している。RectifyNode の入力は
`nitros_image_bgr8` を要求するため bgr8 変換が必要。`apriltag_ros` 自体はこのコンテナには含めず、
`/apriltag/image_rect` + `/apriltag/camera_info_rect` を購読する別プロセスとしてセッション単位で起動する
想定（フェーズ5 `apriltag_slam_session_node` / `apriltag_nav_session_node`）。GPU版の `isaac_ros_apriltag`
は未導入（ワークスペースには `isaac_ros_apriltag_interfaces` のみ存在）で、低頻度用途のため当面CPU版
`ros-humble-apriltag-ros`（インストール済み）を使う方針（2026-07-07）。

### C++ コンポーネント（toyof_robot_ai_vision）

- `AISpatialEstimatorNode` と `TensorToDepthImageNode` は `ComposableNode` として `ai_perception_container` に内包される
- ゼロコピー通信（NITROS）を使用するため、`use_intra_process_comms: True` が必須
- 深度スケール変換: `dist_m = scale_a * raw_val + scale_b`（現在: a=0.765, b=-1.667）
- `TensorToDepthImageNode` は1秒間引き（点群生成のCPU負荷軽減）

### ai_perception_container の動的注入設計

第1層の `ai_perception_container` に v4l2_camera が常駐する。
YOLO モード起動時は `yolo_object_follow.launch.py` が `LoadComposableNodes` で
YOLO/Depth ノードをコンテナに動的注入し、NITROS ゼロコピー通信を維持する。
YOLO cleanup 時に LoadComposableNodes が SIGINT を受け、コンテナからノードを解放。
v4l2_camera は生き続けるため DDS ゴーストパブリッシャー問題が発生しない。

---

# 4 第2層: 自律移動層（autonomy.launch.py）

```
/scan_target_filtered ──→ SLAM Toolbox (Online Async) ──→ /map + TF(map→odom)
/scan_target_filtered ──→ AMCL ──→ /amcl_pose (Nav モード)

/map ──→ map_harden_node ──→ /map_hardened（N24-54/55/57/59/60。狭所の未知セルを
                                占有へ変換 or ソフトコスト付与。costmap 側は
                                map_topic: /map_hardened を参照。
                                map_harden:=false でロールバック）

/scan_target_filtered ──→ corridor_width_monitor_node ──→ 通路幅プロファイル
                                切替（NARROW/WIDE、N22-5/N24-30。Nav2パラメータを
                                動的 set_parameters）

/odometry/filtered + /scan_target_filtered + /map(_hardened)
  → Nav2 (SmacPlanner2D + DWBLocalPlanner) → /cmd_vel
```

| mode | 起動内容 |
|------|----------|
| `slam` | SLAM Toolbox による地図作成（+ `mapping_lifecycle_node` によるGVD/Frontier自律探索、下記） |
| `nav`  | AMCL による自己位置推定 + ナビゲーション |

**GVD/Frontier 自律探索・人追従ロストリカバリ（`mapping_lifecycle_node`, `toyof_robot_navigation`）**:
mapping モードの地図拡張、および人追従ロスト時の Tier2 探索（→ CLAUDE.md §6.3）の両方を
`GvdExplorer` 共有クラスが担う。GVD（一般化ボロノイ図）の骨格からフロンティア/ゲートを検出し、
GATE_THROUGH（2レグ駆動）・エリアcommit/退役・スタック脱出などの状態機械で探索を進める。
内部ロジックの詳細（パラメータ・状態遷移・ロールバック手段）→ `docs/mode_details.md`、
CLAUDE.md §6.4。

### Nav2 パラメータファイル

用途に応じて複数の yaml が存在する。

| ファイル | 用途 |
|---|---|
| `nav2_params.yaml` | 現用（`autonomy.launch.py` が参照） |
| `nav2_params-standard.yaml` | `nav2_params.yaml` と同内容のバックアップ |
| `nav2_params-smooth-run.yaml` | 滑らか走行チューニング版 |
| `nav2_params_local-only.yaml` | 人追従モード用（マップ不要・`global_frame=odom`・`bt_navigator`+`planner_server`+ローリングウィンドウ global_costmap を含む） |

- ローカルコストマップは2D（`use_3d_world: false`）
- Nav2は `/odometry/filtered` を使用（生ホイールオドメトリではない）。EKFをスキップするとナビゲーション不可
- カスタム Behavior Tree: `behavior_trees/navigate_w_recovery.xml`（壁際スタック対策・T-23）。リカバリ順序は **BackUp（0.15m/0.10m/s）優先 → Spin（spin_dist=0.4）→ Wait → Clear**。`nav2_params.yaml` の `bt_navigator > default_nav_to_pose_bt_xml` で絶対パス参照（install 先の share パス）
- **N24-43（2026-08-31）: `allow_unknown` はモードごとに出し分ける。** `localization=="amcl"`（既存マップでのnavのみ）は `planner_server.GridBased.allow_unknown` を `autonomy.launch.py` が実行時に `False` へ上書きする。`slam`/`local` は従来どおり `True`（探索は未知セルへゴールを置く前提のため）。汎用の `overrides`（ドット区切りキー→値）を `robot_identity.rewrite_params_file()` に追加した実装で、今後のモード別パラメータ切替もここを再利用する想定。

---

# 5 第3層: AI 頭脳層（ai_core_managers.launch.py）

## 5.1 ノード構成

| ノード | 種別 | 役割 |
|--------|------|------|
| `ai_mode_manager_node` | 常駐（Node） | AI モード排他制御・ブラックボード管理・Nav2 移動 |
| `speech_io_node` | 常駐（Node） | STT（faster-whisper）/ TTS（Open JTalk）|
| `camera_snapshot_service_node` | 常駐（Node） | `/snapshot/camera/take` サービス（VLM 用）|
| `llm_agent_node` | **非常駐（LifecycleNode）** | Qwen2.5 1.5B / llama.cpp — 対話・コマンド判定 |
| `yolo_follow_node` | **非常駐（LifecycleNode）** | YOLOv8 人追従パイプライン |
| `yoloworld_node` | **非常駐（LifecycleNode）** | YOLO-World + Depth Anything V2 ゼロショット探索 |
| `localization_session_manager_node` | 常駐（Lifecycle、**排他制御対象外**） | Nav2/AMCL 地図確立の一元管理。AIモード切替を跨いでセッションを保持（詳細 → `mode_details.md`） |

## 5.2 AI モード排他制御（ai_mode_manager_node）

Jetson Orin Nano の 8GB メモリ制約のため、LLM・YOLO・YOLO-World は同時起動できない。
`ai_mode_manager_node` が以下の順序で排他切り替えを行う。

```
deactivate(current)
  → cleanup(current)          ← ROS2 Lifecycle: モデルをメモリから解放
  → echo 3 > /proc/sys/vm/drop_caches  ← OSレベルで強制的にページキャッシュを破棄（泥臭い）
  → configure(next)           ← 次のモデルをメモリにロード
  → activate(next)
```

> **なぜ drop_caches が必要か:**  
> ROS2 の `on_cleanup` でプロセスを終了しても、Linux のページキャッシュに
> モデルデータが残り続けることがある。LLM（~3GB）と YOLO（~1GB）を切り替える際、
> キャッシュが残っていると次モデルのロードで OOM が発生する。
> そのため `echo 3 > /proc/sys/vm/drop_caches` で OS に強制的にキャッシュを捨てさせている。

- configure タイムアウト: 120s（llama-server 起動考慮）、activate タイムアウト: 60s（初回モデル初期化考慮）
- deactivate/cleanup タイムアウト: 15s
- 遷移中: TTS が「モード変更中です」を非同期発話（UX 維持）
- 失敗時: IDLE に戻り「起動に失敗しました」を TTS
- drop_caches の実施タイミングはノードごとに異なる（重いモデルロード直前）: yolo_follow は `ai_perception_container` kill+sleep 直後、yoloworld は worker `Popen` 直前、llm は llama-server 起動直前（`process_utils.drop_page_cache()`）
- yolo / yoloworld が `deactivate:X` で停止した後は `ai_mode_manager_node` が自動で `activate:llm` を発行し LLM モードへ復帰する。ノード自身が終了する場合（yolo ストップワード・yoloworld タイムアウト/発見）はノード側が直接 `activate:llm` を publish するため二重起動にはならない

切り替えコマンド: `/state_manager/command` トピックに `"activate:llm"` 等を publish。

| トピック | 型 | 役割 |
|---|---|---|
| `/state_manager/command` | `std_msgs/String` | 遷移コマンド（`"activate:llm"` 等） |
| `/state_manager/status` | `std_msgs/String` | 現在のシステム状態 |
| `/speech/user_text` | `std_msgs/String` | speech_io_node → llm_agent_node（STT 認識テキスト） |
| `/speech/speak_command` | `std_msgs/String` | llm_agent_node → speech_io_node（TTS 発話） |
| `/speech/interrupt` | `std_msgs/String` | speech_io_node → llm_agent_node |
| `/speech/status` | `std_msgs/String` | speech_io_node の状態 |
| `/speech/last_response` | `std_msgs/String` | llm_agent_node の最終応答 |

**CUDA メモリ断片化の注意:** Jetson Orin Nano は CPU/GPU でメモリを共有。`ai_perception_container`
（NITROS/GXF）が先に起動して CUDA メモリを断片化させると、後から起動する llama-server が
compute buffer（~300MB）用の連続ブロックを確保できず OOM で失敗することがある。
LLM を `ai_perception_container` より先に起動するか、STT（~960MB）等が多くのメモリを
保持している状態で LLM を起動しない。コンテナ起動直後（メモリクリーンな状態）で
LLM を先に起動するのが最も確実。

**GPU ウォームアップ知見（2026-06-08 実証）:** cuDNN autotuning はプロセスメモリ内のみで、
プロセス間で引き継がれない。yoloworld の初回推論は常に ~8-15s かかる（2回目以降は 0.05s）。
サブプロセス方式・実 Lifecycle 方式いずれもウォームアップ効果なし。

## 5.3 ブラックボード（/tmp/robot_context.json）

AI ノード間の状態共有に使うファイルベースのブラックボード。

```json
{
  "current_state": "LLM_ACTIVE",
  "active_node": "llm",
  "task": null,
  "task_args": {},
  "target_object": null,
  "target_room": null,
  "target_coords": null,
  "scene_description": null,
  "last_updated": "2026-06-04T..."
}
```

## 5.4 音声対話フロー

```
[マイク]
  → speech_io_node (VAD → STT) → /speech/user_text
        ↓
  llm_agent_node (ウェイクワード検出 → LLM 推論 → コマンド判定)
        ├── /speech/speak_command → speech_io_node (TTS) → [スピーカー]
        └── /state_manager/command → ai_mode_manager_node (Lifecycle 制御)
```

ウェイクワード `ヘイロボ` 検出後 20s 間はウェイクワード不要。

---

# 6 知覚・追従パイプライン（YOLO モード）

`yolo_follow_node` の `on_configure` で `yolo_object_follow.launch.py` が起動する。

```
ai_perception_container へ LoadComposableNodes で動的注入:
  /image (rgb8)
    ├──→ yolo_encoder → yolo_tensorrt → yolo_decoder → /detections_output
    └──→ depth_encoder → depth_tensorrt (depth_tensor)
                             ├──→ ai_spatial_estimator → /ai/target_spatial (Point)
                             └──→ tensor_to_depth_image_node
                                     → /camera/depth/image_raw
                                     → point_cloud_xyz → /camera/depth/points

/detections_output
  → object_tracking_info_node → /object_tracking/info (PersonTrackingInfo)
        ├──→ follow_goal_generator_node
        │       + /ai/target_spatial (最優先)
        │       + /pico/tof_m (第2優先、use_tof 既定 false)
        │       → NavigateToPose (Nav2) → /cmd_vel
        └──→ turret_tracker_node → /pico/servo_target (PID サーボ制御)
```

### 距離推定優先順位（N19-9a）

| 優先度 | ソース | 説明 |
|--------|--------|------|
| 1位 | AI深度 (`/ai/target_spatial`) | Depth Anything V2 メトリック深度。`ai_depth_min_m` 未満の生値は信頼しない |
| 2位 | ToF (`/pico/tof_m`) | 砲塔搭載 VL53L0X（ターゲット方向のみ、`use_tof` 既定 **false**） |
| 近距離クランプ | AI深度が `ai_depth_min_m` 未満・ToF も無効 | `ai_depth_min_m` にクランプした値を返す（正確な距離は不明だが「非常に近い」ことは確実） |
| フォールバック | いずれも無効 | `None` を返し呼び出し側で安全に停止させる |

**BBox高さ推定・固定値2mフォールバックはいずれも撤去済み**（2026-07-05）。前者は
対象物体のサイズが多様で「実物の高さは一定」という前提が成立せず大幅な過大推定を
起こし（実測: 1.9m先のぬいぐるみを15.25mと誤推定）、後者は対象を認識していても
常に2m先を狙って近づかない不具合の原因だった。

### サーボ角度考慮

砲塔（Pan-Tilt）の Pan 角度を `/pico/servo_state` から取得し、
2D 回転行列でゴール座標を base_link 基準に変換してから Nav2 へ送信する。

```python
pan_rad = math.radians(current_pan_deg)
x_base = x_turret * math.cos(pan_rad) - y_turret * math.sin(pan_rad)
y_base = x_turret * math.sin(pan_rad) + y_turret * math.cos(pan_rad)
```

### カスタムメッセージ: `ObjectTrackingInfo`

| フィールド | 型 | 内容 |
|---|---|---|
| `target_visible` | bool | ターゲット検出フラグ |
| `center_error` | float32 | 画像X中心からのズレ（正=左） |
| `center_y_error` | float32 | 画像Y方向のズレ（正=上） |
| `bbox_area_norm` | float32 | バウンディングボックス面積（正規化） |
| `confidence` | float32 | YOLO信頼度 |
| `tracking_id` | string | トラッキングID（ロックオン用） |

- `object_tracking_info_node` の追従モード: `/target_id` トピックでIDロックオン、空文字で最大面積モードに戻る。追従クラスは `target_class_ids` パラメータで指定（デフォルト: `["0", "person"]`）
- `follow_goal_generator_node` は `/object_tracking/goal_pose`（PoseStamped, goal_frame=odom）を `goal_update_rate_hz` ごとに publish する。`scan_filter_node` の Object filter がこれを購読して追従対象をスキャンから除外する
- ロスト時リカバリー（`recovery_mode` パラメータ）詳細 → [`../todo/object_tracking_recovery.md`](../todo/object_tracking_recovery.md)

---

# 7 YOLO-World 物体探索パイプライン（Step 4）

`yoloworld_node` の `on_configure` で YOLO-World + Depth Anything V2 をロード。
バックグラウンドスレッドで検出ループを実行（ROS スピンスレッドをブロックしない）。

```
1. ai_mode_manager_node がコンテキストから target_object を読み取る
2. yoloworld_node が YOLO-World でゼロショット検出
3. 検出 BBox 中心の深度 → 3D カメラ座標 → TF2 で map 座標変換
4. target_coords を context に保存
5. deactivate:yoloworld を publish → ai_mode_manager が Nav2 ゴール送信
```

---

# 8 センサーフュージョン

**Issue-21 / N21-2（2026-08-08）でヨー源を車輪からジャイロへ移した（gyro-odometry）。**
`wheel_odom_node` はもう姿勢（x/y/yaw）を積分・出力しない。EKF へは
「並進＝wheel_odom の vx のみ／回転＝IMU の vyaw のみ」を測定として渡し、
フィルタ側に積分させる（大域補正は map→odom＝AMCL/SLAM の仕事、REP-105の役割分担）。

```
wheel_odom_node (N21-3, 2スカラーへ減量)
  /pico/rev_* (エンコーダ生値) → dist_scale補正 → /wheel_odom
  ※ x/y/yaw は出力しない。pose は UNKNOWN_COV=1e6 で「値が無い」と宣言する
     （0を黙って出すと下流が原点と読む＝REP-105違反）

EKF (robot_localization, ekf.yaml)
  /wheel_odom       — vx のみ使用（imu0_differential: false, pose は全 false）
  /imu/data_aligned — vyaw のみ使用
  → /odometry/filtered → Nav2

laser_odom_node（N24-68, toyof_robot_vehicle。既定 true で起動する）
  並進限定レーザーオドメトリ。回転Δθはジャイロから既知として与え、
  scan-to-scanを並進2自由度の格子探索へ縮退。EKFへは繋がない（N24-68cで確定・観測専用）。
  wheel_odomのスリップ検知（下記）の第3軸として利用。未起動時は当該軸がフェイルオープンで無害。
  ロールバック: hardware_bringup.launch.py の laser_odom:=false
  （2026-09-05、起動していないとスリップ検知translation軸が証言者を持たず常に無効になるため
  既定falseから変更。代償はCPU 1コア約31%＝全体約5%を常時負担）
```

### wheel_odom の距離スケール補正（wheel_odom.yaml 抜粋、N21-3）

エンコーダ counts→m の系統誤差を吸収するスカラーを速度で2点blendする。
`effective_tread` はスリップ検知でジャイロと比較するためだけに使い、姿勢には使わない。
**実測値はドライブトレインごとの校正値。別のロボットにそのままコピーしないこと。**

```yaml
v_scale_low: 0.10
v_scale_high: 0.30
dist_scale_low: 0.988
dist_scale_high: 1.144
effective_tread: 0.523
cov_vx: 0.001
```

### スリップ検出・修復

車輪空転（壁ドン空転＝ホイールは回るが車体は静止）を独立した2〜3軸で検知し、
検知したらその軸の測定を棄却（EMA も更新しない）。ヨーではなく**並進(vx)を汚す**（N23-5）。

| 軸 | 判定 | 対応 |
|---|---|---|
| 回転（ホイール vs IMU） | 校正済みホイール角速度が `slip_w_wheel_high`(0.40 rad/s) 超過 かつ IMU角速度が `slip_w_imu_low`(0.15) 未満 | 回転のみ `self.theta` を IMU絶対yaw差分で置換 |
| 並進（報告速度） | 報告速度が `slip_reject_speed_mps`(0.15 m/s＝指令上限の1.5倍) 超過 | vx を棄却・EMA凍結（N23-5） |
| 並進（wheel vs laser_odom、既定無効） | `slip_translation_enable: true` かつ wheel(`slip_v_wheel_high` 0.15) と laser_odom(`slip_v_laser_low` 0.05) が乖離 | vx を棄却（N24-68d。IMU可否と無関係に評価できる独立軸。laser_odom未起動時はフェイルオープンで無害） |

| パラメータ | デフォルト | 意味 |
|---|---|---|
| `slip_detect_enable` | `true` | スリップ検出の有効/無効 |
| `slip_cov_vx_boost` | `20.0` | スリップ時の vx 共分散膨張倍率 |
| `slip_hold_duration` | `0.5` s | スリップ判定ヒステリシス時間 |
| `slip_imu_timeout` | `0.2` s | IMU データが古いとみなすタイムアウト |
| `slip_reject_speed_mps` | `0.15` m/s | 並進棄却の閾値（`0.0` でロールバック） |
| `slip_translation_enable` | `false`（`wheel_odom.yaml` で `true` に上書き） | laser_odom併用の第3軸を有効化 |

詳細 → CLAUDE.md §6.9、`docs/design_notes.md` N21-2/N21-3/N23-5/N24-68。

---

# 9 Pico W 通信プロトコル

`pico_bridge_node.py` ↔ `pico_firmware/src/main.py` が共有する UART プロトコル。
**プロトコル変更時は必ず両方を同時に更新すること。**

## Jetson → Pico W（コマンド）

```
L:<value>,R:<value>\n
S:<pan_deg>,<tilt_deg>\n
```

| フィールド | 範囲 | 説明 |
|-----------|------|------|
| L / R | -100 〜 100 | 左右モーター出力 |
| S pan | -90 〜 90 | パン角度 (deg) |
| S tilt | 0 〜 90 | チルト角度 (deg) |

## Pico W → Jetson（テレメトリ）

エンコーダパルス差分・ToF 距離・サーボ状態をシリアルで返送。

**注意:** UART バッファは 4095 バイト。高頻度パブリッシュでオーバーフローのリスクあり。
詳細: [`docs/serial_deadlock_analysis.md`](serial_deadlock_analysis.md)

---

# 10 安全設計

| レイヤ | ノード | メカニズム | タイムアウト |
|--------|--------|------------|------------|
| ROS2 | `pico_bridge_node` | `/cmd_vel` ウォッチドッグ | 500 ms |
| MCU | Pico W ファームウェア | UART コマンドウォッチドッグ | 500 ms |

どちらのウォッチドッグもタイムアウトで `L:0,R:0` を送信してモーターを停止する。
ROS と MCU の二重停止機構により、片方が死んでもロボットが暴走しない。

---

# 11 制御周波数

| モジュール | 周波数 |
|-----------|--------|
| v4l2_camera | 10 Hz（v4l2-ctl で強制設定）|
| YOLO 推論 | ~10 Hz |
| Depth Anything V2 推論 | ~2-3 Hz（処理遅延 300〜500ms）|
| object_tracking_info_node | カメラ検出に同期 |
| follow_goal_generator_node | 追従トピックに同期 |
| pico_bridge_node（cmd_vel → UART） | 最大 10 Hz |
| Pico W メインループ | ~100 Hz |
| TF → SLAM | リアルタイム |

---

# 12 TensorRT エンジン

| ファイル | 用途 | 備考 |
|---------|------|------|
| `models/yolov8s.engine` | YOLOv8 人検出 | デバイス固有 |
| `models/depth_anything_v2_metric_hypersim_vits.engine` | Depth Anything V2 深度推定 | デバイス固有 |

`.engine` ファイルは GPU アーキテクチャ固有のため別の Jetson では再生成が必要。
`force_engine_update: False` で既存エンジンを再利用し初回のみ自動生成する。

---

# 13 マルチエージェント連携（フリート、F-3-1）

上記1〜12は単一ロボット（Jetson Orin Nano）内で完結する構成。これとは独立に、
**固定PCカメラで死角を補完するサブエージェント**（`toyof_robot_subagent_vision`、
Isaac ROS非依存・x86/WSL2で動作）が同じ map 座標系を共有する別プロセスとして存在する。

```
固定PCカメラ（x86/WSL2） → subagent_vision_node
  ├─ YOLO-World でゼロショット検出
  ├─ 較正済み外部パラメータ（AprilTagで較正）で画像座標 → map座標へ変換
  └─ /fleet/object_found を publish（agent_id 付き）
```

Jetson側とはトピック直書きではなく `map/rooms/<name_id>/objects.yaml` 経由の座標共有を基本とし、
`frame_prefix`/`ns` パラメータ化で複数ロボットへの拡張にも対応する設計。詳細（トピック衝突回避・
`agent_id` 設計・実装状況）→ [`todo/multi_agent_fleet.md`](../todo/multi_agent_fleet.md)。

---

# 14 参考ドキュメント

| ドキュメント | 内容 |
|-------------|------|
| [`docs/serial_deadlock_analysis.md`](serial_deadlock_analysis.md) | シリアル通信バッファ飽和によるデッドロック |
| [`docs/tof_blocking_analysis.md`](tof_blocking_analysis.md) | ToF ブロッキングによるエンコーダ精度劣化 |
| [`docs/engineering_decisions.md`](engineering_decisions.md) | LiDAR 配置・オドメトリ選択・Nav2 統合・砲塔設計 |
| [`docs/observability_detail.md`](observability_detail.md) | 可観測性設計（OTel + Prometheus + Grafana）の詳細 |
| [`docs/safety_architecture.md`](safety_architecture.md) | 安全設計の詳細 |
