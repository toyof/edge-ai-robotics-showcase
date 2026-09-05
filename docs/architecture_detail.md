# Architecture Detail

ROS2ノードグラフ、レイヤード設計詳細、センサーフュージョン、LLMコマンド判定の詳細。

---

## レイヤードアーキテクチャ

このロボットは **レイヤードアーキテクチャ** で設計している。

```
Perception Layer (YOLO / LiDAR / IMU / Encoder / ToF)
│
▼
Control Layer (Nav2 / Follow Goal Generator / Turret Tracker)
│
▼
Actuation Layer (Pico Bridge → Motor / Servo)
```

詳細設計: [`docs/robot_architecture.md`](robot_architecture.md)

---

## ROS2 Node Graph

### 音声対話 + 排他制御（`ai_core_managers.launch.py`）

```
[マイク]
  → speech_io_node ─────────────────────────────────────→ [スピーカー]
        │ /speech/user_text            /speech/speak_command │
        ▼                                                     │
  llm_agent_node (LifecycleNode)  ←──────────────────────────┘
        │ /state_manager/command
        ▼
  ai_mode_manager_node ── ros2 lifecycle set ──→ llm_agent_node    (LLM モード)
                                             ├──→ yolo_follow_node  (YOLO モード)
                                             └──→ yoloworld_node    (YOLO-World 探索モード)
  yolo_follow_node ─── ストップワード → /state_manager/command ("activate:llm")
```

#### LLM コマンド判定 — キーワードとツール一覧

LLM ツール判定は 2 段階 JSON 方式（モデル非依存）。各ツールに `keywords` を設定することで STT 認識テキストとのマッチ精度を向上させている。`enabled: false` のツールは LLM プロンプトから除外され実行されない。

| キーワード | ツール | 状態 |
|---|---|---|
| ストップ | stop | 有効 |
| 人追従 | follow_person | 有効 |
| 物体検索 | search_object | 有効 |
| カメラ説明 | describe_camera | 有効 |
| 地図作成 | start_mapping | 有効 |
| カメラ保存 | save_objects | 有効 |
| 基本移動 | move | **無効** |
| 基本旋回 | turn | **無効** |

STT（faster-whisper）の `initial_prompt` にも各キーワードを含む発話例を設定し、音声認識段階での語彙ヒントとして機能させている。

**2026-07-13: `Maps_to_remembered` は `search_object` に統合され廃止。** 「地図記憶移動」「記憶している場所」
「覚えている場所」「〜のところに行って」等の記憶想起フレーズは全て `search_object` にルーティングされる。
`search_object` ハンドラ（`robot_agent.py`）は `target_object` について `memory_query()`（room単位の
`objects.yaml`）を先に確認し、ヒットすればカメラ探索（YOLO-World）を行わず
`ensure_localization_established(request_kind="map_required")` でNav2/AMCL確立を待ってから
記憶座標へ直接Nav2移動する。ヒットしなければ従来通り`activate:yoloworld`で新規に索敵する。
記憶か新規かの判定はLLMではなくランタイム側（`memory_query`）が行うため、プロンプトの
disambiguationルールも単一の`search_object`ルールに統合されている。
未実装のTODO: 記憶座標到着後にカメラで対象の存在を再確認し、無ければ周辺探索へフォールバックする
（現状は記憶座標を無条件に信頼）。詳細 → `todo/ai_core.md`。

### 視覚追従（`yolo_object_follow.launch.py` / `yolo_follow_node` 経由）

```
v4l2_camera
  ├─→ yolo_encoder → yolo_tensorrt → yolo_decoder → /detections_output
  └─→ depth_encoder → depth_tensorrt → ai_spatial_estimator → /ai/target_spatial

/detections_output
  → object_tracking_info_node → /object_tracking/info
        ├─→ follow_goal_generator_node ──→ NavigateToPose (Nav2) ──→ /cmd_vel
        └─→ turret_tracker_node ──→ /pico/servo_target

/cmd_vel + /pico/servo_target
  → pico_bridge_node → Pico W (UART) → Motor / Servo
```

### ベースシステム（`robot_bringup.launch.py`）

```
YDLIDAR ──→ /scan ──→ scan_filter ──┬─→ /scan_body_filtered ──→ 安全チェック(cmd_vel直接駆動)
                                     └─→ /scan_target_filtered ──→ SLAM / Nav2

IMU ──→ imu_filter ──→ imu_yaw_aligner ──→ /imu/data_aligned ──┐
Pico W ──→ pico_bridge_node ──→ /pico/rev_*                     │
                │                                                 │
                ▼                                                 │
         wheel_odom_node ──→ /wheel_odom ─────────────────────────┘
                                              │
                                              ▼
                                    EKF Node ──→ /odometry/filtered ──→ Nav2
```

---

## センサーフュージョン

**並進はホイールエンコーダ・回転はジャイロの完全分業**（Issue-21 / N21-2）。
IMU は `imu0_differential:false` の下では絶対方位を供給できない仕様のため、
姿勢を「測定」として融合するのをやめ、速度だけを入れて EKF に積分させる設計へ
切り替えた（詳細な構造欠陥の分析 → `docs/engineering_decisions.md` Issue-02
その後の変化、CLAUDE.md §6.9）。

```
Wheel Odometry (vx のみ)   ──┐
                              ├──→ EKF (robot_localization) ──→ /odometry/filtered
IMU (vyaw のみ)             ──┘
```

pose (x, y, yaw) はどちらのソースも融合しない。ホイール側の pose 出力・TF配信は
撤去済みで、値を持たないことは `UNKNOWN_COV=1e6` で明示的に宣言する
（0 を黙って出すと下流が「原点」と読んでしまう＝REP-105違反）。

### 速度依存の距離補正

4WD 構成による滑りを補正するため、速度帯に応じて2点を線形補間する
（`blend_dist_scale()`、N21-3b）。

```yaml
# wheel_odom.yaml（抜粋）
v_scale_low: 0.10        # 低速閾値 (m/s)
v_scale_high: 0.30       # 高速閾値 (m/s)
dist_scale_low: 0.988    # 低速時の距離補正係数
dist_scale_high: 1.144   # 高速時の距離補正係数
effective_tread: 0.523   # 車輪差→角速度（スリップ検知がジャイロと比較する専用、姿勢には使わない）
```

旋回方向の補正（旧 `yaw_scale_low/high`）は姿勢を積分しなくなったため撤去済み。

---

## ハードウェア構成

### システム全体

```
USB Camera (on Turret)
│
▼
Jetson Orin Nano
(Isaac ROS / YOLO / ROS2 / Nav2)
│
│ UART (USB Serial)
▼
Raspberry Pi Pico W
(Motor / Servo / ToF / Encoder)
│
├── PWM → BTS7960 Motor Driver → 4WD Motors
├── PWM → Pan-Tilt Servo (Turret)
├── I2C → VL53L0X ToF Sensor
└── GPIO → Quadrature Encoder ×4
```

### コンポーネント役割

| Component | Role |
|---|---|
| NVIDIA Jetson Orin Nano Developer Kit | AI inference / ROS2 control / Nav2 |
| Raspberry Pi Pico W | Motor / Servo / ToF / Encoder firmware |
| USB Camera | Vision sensor (on turret) |
| YDLIDAR Tmini-Plus | 2D LiDAR (SLAM / Navigation) |
| ICM-20948 IMU | 9-axis IMU (EKF fusion) |
| BTS7960 Motor Driver ×2 | High-current H-bridge |
| JGB37 Encoder Gear Motor ×4 | Robot drive (110RPM, 12V) |
| VL53L0X ToF Sensor | Distance measurement (turret mounted) |
| Pan-Tilt Servo | Camera turret |
| Yowoo 4S LiPo 14.8V 3000mAh | Power source |

### 電源構成

バッテリー（9〜16.8V）からメインヒューズ・メインスイッチを経て、系統別に保護・分配する。
Jetson系統は逆流防止ダイオード付きで直結、モータ系統は大型DCDCで12Vに安定化、ロジック・サーボ系統は5V、センサ・エンコーダ系統は3.4Vに降圧する。GNDは共通バスで結合する。

```mermaid
flowchart LR
    %% ==========================================
    %% 電源元・メイン保護
    %% ==========================================
    BAT[("バッテリー (9〜16.8V)")] --> F_MAIN["30A メインヒューズ"]
    F_MAIN --> SW["メインスイッチ"]

    %% ==========================================
    %% 電源分配・系統別保護
    %% ==========================================
    subgraph POWER_DIST ["電源分配・保護回路"]
        %% Jetson系統（直結＋ダイオードでノイズ・逆流から厳重保護）
        SW --> F_JET["インラインヒューズ (Jetson用 5A)"]
        F_JET --> DIODE["逆流防止ダイオード<br>(Jetson保護用 15A)"]
        DIODE -->|9〜16.8V 直結| JETSON_VIN["Jetson DCジャック"]

        %% モータ系統（独立した大型DCDCで電圧安定化）
        SW --> F_MOT["インラインヒューズ (モータ用 15~20A)"]
        F_MOT --> DCDC_12V_MOT["大型DCDC 12V<br>(モータ動力用: XL4016等)"]
        DCDC_12V_MOT -->|12V| VM_R["右ドライバ +12V"]
        DCDC_12V_MOT -->|12V| VM_L["左ドライバ +12V"]

        %% ロジック・サーボ系統 (5V)
        SW --> F_LOGIC["インラインヒューズ (5V系用 2A)"]
        F_LOGIC --> DCDC_5V["DCDC 5V (XL4015)"]
        DCDC_5V -->|5V| P_VIN["Pico VSYS (pin40)"]
        DCDC_5V -->|5V| LOGIC_5V["LOGIC_5V 共通ライン"]

        %% センサ・エンコーダ系統 (3.4V)
        SW --> F_SEN["インラインヒューズ (3.4V系用)"]
        F_SEN --> DCDC_34V["DCDC 3.4V (XL4015)"]
        DCDC_34V -->|3.4V| SENSOR_34V["3.4V センサ電源ライン"]
    end

    %% ==========================================
    %% 各コンポーネントの詳細定義
    %% ==========================================
    subgraph JETSON ["JETSON (メインPC)"]
        JETSON_VIN
        J_USB["USB Ports"]
        J6["pin6 GND"]
    end

    subgraph PICO ["Raspberry Pi Pico (ブリッジ)"]
        P_VIN
        P_GND["pin38 GND"]
        PUSB["USB Port"]

        P4["pin4 / GP2 (RPWM_R)"]
        P5["pin5 / GP3 (LPWM_R)"]
        P6["pin6 / GP4 (LPWM_L)"]
        P7["pin7 / GP5 (RPWM_L)"]

        P9["pin9 / GP6 (Servo PAN)"]
        P10["pin10 / GP7 (Servo TILT)"]

        P11["pin11 / GP8 (I2C SDA)"]
        P12["pin12 / GP9 (I2C SCL)"]

        P14["pin14 / GP10 (ENC_L A)"]
        P15["pin15 / GP11 (ENC_L B)"]
        P16["pin16 / GP12 (ENC_R A)"]
        P17["pin17 / GP13 (ENC_R B)"]
    end

    subgraph DRIVER_R ["BTS7960 Right"]
        VM_R
        R_GND["GND"]
        R_VCC["VCC (5V)"]
        R_RPWM["RPWM"]
        R_LPWM["LPWM"]
        R_REN["R_EN"]
        R_LEN["L_EN"]
    end

    subgraph DRIVER_L ["BTS7960 Left"]
        VM_L
        L_GND["GND"]
        L_VCC["VCC (5V)"]
        L_RPWM["RPWM"]
        L_LPWM["LPWM"]
        L_REN["R_EN"]
        L_LEN["L_EN"]
    end

    subgraph TURRET ["ターレット (上部機構)"]
        SV_PAN["サーボ Pan<br>(sx-101z)"]
        SV_TILT["サーボ Tilt<br>(sx-101z)"]
        VL53["距離センサ ToF<br>(VL53L0X)"]
    end

    subgraph ENCODERS ["モータエンコーダ"]
        ENC_R["右エンコーダ"]
        ENC_L["左エンコーダ"]
    end

    subgraph IMU ["IMU センサ"]
        I_VCC["VCC"]
        I_GND["GND"]
    end

    subgraph USB_DEV ["USBデバイス (Jetson給電)"]
        CAM["USB Camera"]
        LIDAR["YDLIDAR"]
    end

    %% ==========================================
    %% 配線・制御・通信の接続
    %% ==========================================

    %% --- 5Vロジック電源分配 ---
    LOGIC_5V --> R_VCC & L_VCC & R_REN & R_LEN & L_REN & L_LEN
    LOGIC_5V --> SV_PAN & SV_TILT

    %% --- 3.4V電源分配 ---
    SENSOR_34V --> ENC_R & ENC_L & VL53 & I_VCC

    %% --- USB通信 ---
    J_USB <--> PUSB
    J_USB --> CAM & LIDAR

    %% --- Pico モータ・サーボ制御信号 ---
    P4 --> R_RPWM
    P5 --> R_LPWM
    P6 --> L_LPWM
    P7 --> L_RPWM
    P9 --> SV_PAN
    P10 --> SV_TILT

    %% --- Pico I2C 信号線 (ToF用) ---
    P11 -->|SDA| VL53
    P12 -->|SCL| VL53

    %% --- Pico エンコーダ読み取り ---
    ENC_L --> P14 & P15
    ENC_R --> P16 & P17

    %% --- グランド結合（共通GNDバス） ---
    GND_BUS[("共通GND バス")]
    GND_BUS --- P_GND & J6 & R_GND & L_GND & I_GND
    GND_BUS --- SV_PAN & SV_TILT & VL53 & ENC_R & ENC_L
    GND_BUS --- DCDC_12V_MOT & DCDC_5V & DCDC_34V & BAT
```

### 安全設計

| Layer | Mechanism | Timeout |
|---|---|---|
| ROS2 (pico_bridge_node) | cmd_vel watchdog | 500 ms |
| MCU (Pico firmware) | UART command watchdog | 500 ms |
