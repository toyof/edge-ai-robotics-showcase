# ログ地図（logging_map）— 事象からログの在り処を引く索引

**目的は網羅的なログ仕様書ではなく、「この事象を調べたいときはどのファイルを grep するか」を
1分で引ける索引を作ること。** 調査のたびに grep 先を思い出す作業を無くす。

起票の経緯（N-OBS-3）: 2026-08-15、GVD 探索の不調を調べた際に `mapping_standalone.log` だけを見て
**「SLIP 0件」と誤断定**した。実際には `hw_bringup.log` に SLIP 19件・GUARD 8件・DEADBAND 87件・
KICK 39件があり、**N23-5 のスリップ棄却も3件発火していた**（＝取れていた検証結果を取りこぼしかけた）。
同種の誤りは過去にも複数回起きている。**記憶ではなく本ファイルで運用する。**

> **鉄則: ログは1ファイルだけ見て結論を出さない。**
> ロボットの1回の走行は最低3プロセスツリー（ハードウェア層・Nav2/SLAM 層・AI頭脳層）に分かれており、
> **同じ時刻の出来事が3ファイルに分散して書かれる。**
> 走行後は `bash scripts/mapping_diag/collect_logs.sh <dest>` でまとめて回収すること
> （コンテナ内 `/tmp` はコンテナを止めると消える）。

**更新（N-OBS-6、2026-09-02）**: 手作業での突き合わせを構造的に不要にする2つの仕組みを追加した。

1. **run_id によるログ集約**（`bringups/run_context.sh`）: `bringup_hardware.sh` を起点に
   `runs/<run_id>/` へ3層のログが自動で集まる（`runs/current` が最新走行を指す）。以前は
   操作者が起動時に `> /tmp/xxx.log 2>&1` を付け忘れると本節の `/tmp/*.log` が丸ごと消えた
   （§4(a) 参照）。**§2 の `/tmp/*.log` パスは互換シンボリックリンクとして残っており、
   実体は `runs/<run_id>/` 側にある。** 新しい手順は `runs/current` を直接見ればよい。
2. **走行後スタットサマリー**（`scripts/run_summary.py runs/current`）: 3ファイルを人手で
   grep する前に、まずこれを実行する。レベル別・タグ別の件数、背骨イベント（GUARD/SLIP/
   ESCAPE等）の回数、WARN/ERROR全文を1画面にまとめる。**どの層が欠けているか
   （`absent`/`missing_layers`）も明示するため、本節が起票された「SLIP 0件」の
   誤断定はこの一手で防げる。**

ログのタグは `src/toyof_robot_observability/toyof_robot_observability/log_vocabulary.py`
の閉じた語彙で管理する（本ファイル §1 の「grep するタグ」列と対応）。新しいタグを
書くときはそちらへ登録すること（`test_log_vocabulary.py` が未登録タグを検出する）。

---

## 1. 逆引き表（事象 → 見る場所）

| 調べたい事象 | 見るファイル | grep するタグ／文字列 |
|---|---|---|
| 前進が止まる・障害物手前で動かない | `hw_bringup.log` | `[GUARD]` |
| 車輪が空転している | `hw_bringup.log` | `SLIP DETECTED` / `Slip cleared` / `/wheel_odom/slip` |
| 発進・回頭がガクつく、モーターが反応しない | `hw_bringup.log` | `[DEADBAND]` / `KICK` |
| ジャイロのバイアス・静止判定 | `hw_bringup.log` | `[ZUPT]` |
| Pico との通信・テレメトリ破損 | `hw_bringup.log` | `[pico]` / `Serial` |
| 経路が引けない・ゴールが ABORT する | `autonomy_launch.log` | `[planner_server]` / `[bt_navigator]` / `no valid path` |
| 速度が出ない・回頭が遅い（Nav2 側の指令） | `autonomy_launch.log` | `[controller_server]` / `[velocity_smoother]` |
| 通路幅プロファイル（NARROW/WIDE）の切替 | `autonomy_launch.log` | `[corridor_width]` |
| SLAM の地図更新・保存 | `autonomy_launch.log` | `slam_toolbox` / `serialize` |
| 探索が進まない・ゲートが選ばれない | `mapping_standalone.log` | `[gvd]` / `[gvd_debug]` |
| 地図保存の成否・部屋の確定 | `mapping_standalone.log` | `[mapping]` |
| 地図確立（タグ前スキャン→Nav2/SLAM 起動）の判断 | `mapping_standalone.log` / AI頭脳層 | `[localization_session]` |
| タグ前スキャンの旋回・AMCL initialpose 投入 | `tag_localization.log` | `[localization]` |
| タグが登録されない・`tag_db.yaml` が更新されない | `tag_registration.log`（N-OBS-5、2026-08-27対処済み） | `[tag_recorder]` |
| 人追従のロスト・Tier1/Tier2 リカバリ | AI頭脳層のログ（例 `yolo_recovery.log`） | `[FOLLOW]` / `[RECOVERY]` / `[STOP]` |
| 安全停止（AMCL発散・知覚途絶・GUARD持続） | AI頭脳層のログ | `[SAFETY]` |
| AIモード切替・メモリ予算ゲート | AI頭脳層のログ | `[ai_mode_manager]` / `[lifecycle]` |
| LLM（llama-server）の起動・GPU offload | `/tmp/llama-server.log` | `[llama]` / `offload` |
| 物体探索（yoloworld）の検出・色検証 | AI頭脳層のログ | `[yoloworld]` / `[worker]` |
| 音声 STT/TTS | AI頭脳層のログ | `[STT]` / `[TTS]` |
| 上記で見つからない／ノード単体のクラッシュ | `~/.ros/log/latest/` | ノード名 |

---

## 2. ログファイル一覧

| ファイル | 中身（プロセスツリー） | 誰が作るか |
|---|---|---|
| `/tmp/hw_bringup.log` | 第1層 ハードウェア層（`hardware_bringup.launch.py` 配下すべて） | **操作者のリダイレクト**（§4 の注意） |
| `/tmp/autonomy_launch.log` | 第2層 Nav2 / SLAM / 通路幅監視（`autonomy.launch.py` 配下） | `mapping_lifecycle_node` と `nav_launch_selector` が**追記**で開く |
| `/tmp/mapping_standalone.log` | 第3層のうち standalone mapping セッション | **操作者のリダイレクト** |
| `/tmp/yolo_recovery.log` | 第3層のうち standalone yolo（`recovery_mode:=on`）セッション | **操作者のリダイレクト**（`docs/mode_details.md` の手順） |
| `/tmp/tag_localization.log` | タグ前スキャン／タグ監視の `tag_localization_manager_node` | `nav_launch_selector` が追記で開く |
| `/tmp/tag_registration.log` | mapping中のタグ登録3プロセス（`apriltag_node`/`tag_map_recorder_node`/turret静的TF） | `mapping_lifecycle_node` が追記で開く（N-OBS-5） |
| `/tmp/llama-server.log` | llama-server（LLM 本体、ROS 外） | `llm_agent_node` が追記で開く |
| `~/.ros/log/latest/` | rcl 標準のノード別ログ（全ノード） | ROS 2 が自動 |

**回収**: `bash scripts/mapping_diag/collect_logs.sh <dest>` が上記のうち `/tmp` の主要4ファイルと
`~/.ros/log/latest` をワークスペースへコピーする。存在しないファイルは `absent:` と表示されるだけで
失敗しないので、**`absent:` が出たら「その層のログを取り損ねている」と読むこと。**

---

## 3. ノード一覧

### 第1層: ハードウェア層 — `hardware_bringup.launch.py` → `hw_bringup.log`

| ノード | 担当機能 | パッケージ | 代表ログタグ | 主要トピック |
|---|---|---|---|---|
| `pico_bridge_node` | **モーターへの唯一の出口。** cmd_vel ゲート（GUARD）・不感帯補償・Pico シリアル | `toyof_robot_vehicle` | `[GUARD]` `[DEADBAND]` `[pico]` | SUB `cmd_vel`, `wheel_odom/slip` / PUB `cmd_vel_guard/blocked`, `cmd_vel_guard/blocked_kind`, `pico/*` |
| `wheel_odom_node` | 車輪オドメトリ（vx のみ）・スリップ検知 | `toyof_robot_vehicle` | `SLIP DETECTED` `Slip cleared` | SUB `pico/rev_*`, `pico/enc_ts_ms` / PUB `wheel_odom`, `wheel_odom/slip`, `wheel_odom/slip_debug` |
| `imu_node` (`imu.py`) | IMU 生値・ZUPT バイアス再推定・アンチエイリアス標本化 | `toyof_robot_sensor` | `[ZUPT]` `Calibration` | PUB `imu/data_raw`, `imu/mag`, `imu/temp` |
| `imu_filter_node` | Madgwick 姿勢フィルタ（外部パッケージ） | `imu_filter_madgwick` | — | SUB `imu/data_raw` / PUB `imu/data` |
| `imu_yaw_aligner_node` | IMU ヨーの基準合わせ | `toyof_robot_vehicle` | `Input IMU` | SUB `/imu/data` / PUB `/imu/data_aligned` |
| `ekf_filter_node` | EKF（**回転＝ジャイロ / 並進＝車輪**、Issue-21） | `robot_localization` | — | PUB `odometry/filtered` |
| `scan_filter_node` | LiDAR の自車体・追従対象マスク | `toyof_robot_vehicle` | `BoxFilter` | SUB `scan` / PUB `scan_body_filtered`, `scan_target_filtered` |
| `ydlidar_ros2_driver_node` | LiDAR ドライバ | `ydlidar_ros2_driver` | — | PUB `scan` |
| `robot_state_publisher` | URDF → TF（`frame_prefix` で namespace 対応） | `robot_state_publisher` | — | PUB `/tf`, `/tf_static` |
| `work_event_node` | 可観測性: 作業イベント→OTel | `toyof_robot_observability` | `[work_event]` `[mission]` | SUB `follow/recovery_state`, `observability/log_event` |
| `metrics_node` | 可観測性: tegrastats/vmstat/トピックレート→OTel | `toyof_robot_observability` | `[latency]` `OTel` | SUB `object_tracking/info` ほか |
| `pico_stub_node` | （sim のみ）Pico の代替。GUARD は実機と同一ロジック | `toyof_robot_vehicle` | `[GUARD]` `[PicoStub]` | 実機の `pico_bridge_node` と同じ |
| `imu_sim_node` | （sim のみ）IMU 代替 | `toyof_robot_vehicle` | `[imu_sim]` | PUB `imu/data_raw` |
| `laser_odom_node`（N24-68、既定は起動しない） | 並進限定レーザーオドメトリ。EKF未接続・観測専用 | `toyof_robot_vehicle` | `[laser_odom]` | SUB `scan_target_filtered`, `imu/data_aligned`, `wheel_odom` / PUB `laser_odom`, `laser_odom/debug` |

> カメラ系（`camera_perception.launch.py`: `v4l2_camera` / `RectifyNode` /
> `ImageFormatConverterNode`）も第1層から起動されうるが、**AI頭脳層が自前で起動し直す経路もある**
> ため、出力先は起動元による（§4）。NITROS のログは `[NitrosNode]` `[NitrosPublisher]`。

### 第2層: Nav2 / SLAM — `autonomy.launch.py` → `autonomy_launch.log`

| ノード | 担当機能 | パッケージ | 代表ログタグ |
|---|---|---|---|
| `controller_server` | 経路追従（RPP）。速度・回頭の実指令 | `nav2_controller` | `[controller_server]` |
| `planner_server` | 経路計画（SmacPlanner2D） | `nav2_planner` | `[planner_server]` / `no valid path` |
| `bt_navigator` | BehaviorTree（`navigate_w_recovery.xml` 等） | `nav2_bt_navigator` | `[bt_navigator]` |
| `behavior_server` | リカバリ動作（spin/backup/wait） | `nav2_behaviors` | `[behavior_server]` |
| `velocity_smoother` | 加減速制限。**最後にここが指令を握る** | `nav2_velocity_smoother` | `[velocity_smoother]` |
| `amcl` | 既存マップでの自己位置推定 | `nav2_amcl` | `[amcl]` / PUB `amcl_pose` |
| `async_slam_toolbox_node` | SLAM（地図生成・posegraph） | `slam_toolbox` | `slam_toolbox` |
| `corridor_width_monitor_node` | 通路幅を推定し NARROW/WIDE プロファイルを注入 | `toyof_robot_navigation` | `[corridor_width]` |
| `map_harden_node`（N24-54/55/57/59） | `/map` の狭所未知セルを占有化 or ソフトコスト化し `/map_hardened` を発行。costmap は `map_harden:=false` で `/map` に戻せる | `toyof_robot_navigation` | `[map_harden]` |
| `lifecycle_manager` | Nav2 ノード群の lifecycle 管理 | `nav2_lifecycle_manager` | `[lifecycle_manager]` |

### 第3層: AI頭脳層 — `ai_core_managers.launch.py` → 起動元のログ

| ノード | 担当機能 | パッケージ | 代表ログタグ | 主要トピック |
|---|---|---|---|---|
| `ai_mode_manager_node` | AIモードの排他ローテーション・地図確立ゲート・メモリ予算ゲート | `toyof_robot_ai_control` | `[ai_mode_manager]` `[lifecycle]` `[warmup]` | SUB `state_manager/command` / PUB `state_manager/status` |
| `localization_session_manager_node` | 地図確立（タグ前スキャン→既存マップ or SLAM）の一元管理 | `toyof_robot_ai_control` | `[localization_session]` | PUB `localization_session/status` |
| `follow_goal_generator_node` | **追従ゴール生成とロスト時リカバリ（Tier1/Tier2）・安全ゲート群** | `toyof_robot_ai_control` | `[SAFETY]` `[FOLLOW]` `[RECOVERY]` `[STOP]` `[TRAIL]` `[CAPTURE]` | SUB `amcl_pose`, `cmd_vel_guard/blocked`, `wheel_odom/slip`, `scan_body_filtered` / PUB `cmd_vel`, `follow/person_*` |
| `object_tracking_info_node` | 検出のトラッキング（IoU）とターゲット選択 | `toyof_robot_ai_control` | `[T-29-6]` `Target` `LOST` | SUB `detections_output`, `target_id` / PUB `object_tracking/info` |
| `mapping_node` (`mapping_lifecycle_node`) | SLAM 探索（GVD/Frontier）・地図保存・タグ登録の起動 | `toyof_robot_navigation` | `[mapping]` `[gvd]` `[gvd_debug]` | SUB `/map`, `cmd_vel_guard/blocked`, `odometry/filtered` / PUB `cmd_vel`, `exploration/*` |
| `yolo_follow_node` | 人追従モードの LifecycleNode（Nav2・YOLO パイプラインを起動） | `toyof_robot_ai_control` | `[yolo_follow]` | — |
| `yoloworld_node` | 物体探索モードの LifecycleNode | `toyof_robot_ai_control` | `[yoloworld]` | — |
| （yoloworld worker） | YOLO-World 推論・色検証（別プロセス） | `toyof_robot_ai_control` | `[worker]` `[calibration]` | — |
| `llm_agent_node` | LLM（llama-server 管理・意図解釈） | `toyof_robot_ai_control` | `[llm_agent]` `[llama]` `[LLM]` | — |
| `speech_io_node` | 音声 STT/TTS | `toyof_robot_ai_speech` | `[STT]` `[TTS]` | SUB `speech/speak_command` / PUB `speech/user_text` |
| `turret_tracker_node` | タレット（パン/チルト）PID 追従・サーチスイープ | `toyof_robot_vehicle` | `[T-22-11]` | SUB `object_tracking/info`, `follow/person_estimate` / PUB `pico/servo_target` |
| `camera_snapshot_service_node` | カメラ静止画取得サービス | `toyof_robot_ai_control` | `[camera_snapshot]` | — |

### タグ／自己位置（AprilTag 系）

| ノード | 担当機能 | パッケージ | 代表ログタグ | 出力先 |
|---|---|---|---|---|
| `tag_localization_manager_node` | タグ前スキャン旋回・AMCL `initialpose` 投入 | `toyof_robot_localization` | `[localization]` | `tag_localization.log` |
| `tag_map_recorder_node` | タグ座標をステージングへ記録（T-AT-6-29） | `toyof_robot_localization` | `[tag_recorder]` | `tag_registration.log`（N-OBS-5） |
| `apriltag_node` | AprilTag 検出（Isaac ROS） | `isaac_ros_apriltag` | — | `tag_registration.log`（N-OBS-5） |
| `apriltag_servo_node` | タグ追尾サーボ | `toyof_robot_localization` | `[apriltag_servo]` | 起動元による |

---

## 4. ログの出力先が決まる仕組み（ここを外すと誤断定する）

ノードのログがどのファイルに入るかは、**そのノード自身ではなく「誰が起動したか」で決まる。**

**(a) 〜2026-09-01: 操作者のリダイレクトが起点だった（現在は解消済み、N-OBS-6）。**
以前の `bringups/*.sh` は**リダイレクトを一切しなかった**（`bringup_hardware.sh` は
`exec ros2 launch ...` のみ）。付け忘れるとその層のログは端末にしか残らず、走行後には
回収できなかった。**現在は `bringups/run_context.sh` の `robot_run_init` /
`robot_run_logged` が各 `bringups/*.sh` の中で自動的にリダイレクトする**
（`runs/<run_id>/` へ、`/tmp/xxx.log` はそこへの互換シンボリックリンク）。
操作者が明示的に何かを付ける必要はもう無い。

**(b) 子プロセスは3通り。** ノードが別の launch を `subprocess` で起動するとき、
`process_utils.start_shell_process()` の引数で出力先が決まる。

| 呼び方 | 出力先 | 使われている例 |
|---|---|---|
| `devnull=True` | **破棄** | §5 参照 |
| `log_path=...` | そのファイルへ**追記** | `autonomy.launch.py` → `autonomy_launch.log`、タグ前スキャン → `tag_localization.log` |
| どちらも指定なし | **親の stdout を継承** | YOLO パイプライン・`camera_perception` → AI頭脳層のログへ合流 |

`autonomy.launch.py` を**追記**にしているのは、mapping 経路（`mapping_lifecycle_node`）と
yolo 経路（`nav_launch_selector`）のどちらから起動しても **grep 先を1つに保つ**ため
（N19-11 / N22-7 の対策）。

**(c) 「ログに0件」は「起きていない」の証拠にならない。**
その層のログを取り損ねている（a）か、破棄されている（b）可能性を先に潰すこと。
N22-7 はまさにこれで、「切替ログが0件だから適用されていない」と読んだが、実際は
**書かれていても読む先が無かった**。

---

## 5. 既知欠陥: いま破棄されているログ（DEVNULL 監査）

`grep -rn "DEVNULL" src/` の結果を精査した。大半は CLI の stderr 抑止で無害。
**ROS ノード本体の出力を丸ごと捨てている箇所が1件あったが、2026-08-27 に対処済み。**

### ✅ 対処済み（N-OBS-5, 2026-08-27）: タグ登録サブプロセスのログ破棄

`mapping_lifecycle_node.py` の `_start_tag_registration()` が
**`apriltag_node` / `tag_map_recorder_node` / turret 静的TF** の3プロセスを
`stdout=DEVNULL, stderr=DEVNULL` で起動しており、`tag_map_recorder_node` が持つ
`[tag_recorder]` タグのログ（7箇所）が**どのファイルにも出ない**状態だった。

実測で裏取り済み（`mapping_diag/run_20260826_1241/logs/`）:

```
mapping_standalone.log:18  [mapping] タグ登録サブプロセス起動完了（並行、ベストエフォート）
$ grep -c '\[tag_recorder\]' hw_bringup.log autonomy_launch.log mapping_standalone.log
0   0   0
```

**起動したことだけが記録され、その後タグを1枚も見なかったのか・見たが弾いたのか・
ステージングへ書いたのかが一切分からない。** これは N19-11（yolo 経路）・N22-7（mapping 経路の
Nav2）で2回潰したのと**同型の欠陥の3例目**。

**対処内容**: 既存の `autonomy_launch.log`（N22-7）と同じ open-then-Popen パターンで
`/tmp/tag_registration.log` へ追記するよう変更（`_TAG_REGISTRATION_LOG` 定数、
`stderr=subprocess.STDOUT` で標準エラーも合流）。`scripts/mapping_diag/collect_logs.sh` の
回収リストにも追加した。**`colcon build`/`colcon test` は未実施**（本セッションは docker/ROS
とも利用不可のクラウド環境のため）。次回コンテナのあるセッションでビルド・pep257/flake8確認と、
実機で `[tag_recorder]` ログが実際に出力されることの検証（T-AT-6-30/T-AT-6-29b と合わせて）が
残る。

### 無害と判断したもの（対処不要）

| 箇所 | 内容 | 判断 |
|---|---|---|
| `mapping_lifecycle_node.py:1908` | `ros2 action send_goal` の stderr | stdout は PIPE で解析済み。CLI の警告のみ |
| `ai_mode_manager_node.py:823` / `tag_localization_manager_node.py:425` | `ros2 action send_goal` の CLI 出力 | 成否は別途トピック／サービスで判定 |
| `metrics_node.py:304,333` / `tegrastats_exporter.py:71` | `tegrastats` / `vmstat` の stderr | stdout をパースする用途。stderr は不要 |
| `tts_engine.py` / `audio_device_detector.py` | `aplay` 等の音声 CLI | 定常的に出る ALSA 警告の抑止 |
| `robot_agent.py:543` | 外部 CLI 呼び出し | 結果は戻り値で判定 |

---

## 6. このファイルの使い方・更新ルール

- **調査を始める前にまず §1 を引く。** 「たぶんこのログ」で1ファイルだけ開かない。
- **ノードを追加・改名したとき、ログタグを変えたときは §3 を更新する。**
- **新しく `subprocess` でノードを起動するコードを書くときは §4(b) の3通りを必ず意識する。**
  `devnull=True` を選ぶなら「このノードのログは永久に読めなくなる」ことを受け入れた場合だけ。
- ログ形式の統一・構造化ログ（レベル定義・タグ語彙・`RobotLog`・`MetricReporter`・
  `run_id` 集約・`run_summary.py`）は N-OBS-6（2026-09-02）で実装済み。詳細は
  `docs/design_notes.md` N-OBS-6 と `todo/observability_otel.md` N-OBS-6 を参照。

### 棚卸しの再実行コマンド

```bash
# ログ出力箇所とタグ（ノード別）
grep -rn "get_logger()" src/ --include='*.py' | grep -v test

# 出力を捨てている箇所
grep -rn "DEVNULL" src/ --include='*.py' | grep -v test

# 実ログでのタグ分布（回収済みログに対して）
for f in <dir>/*.log; do echo "--- $f"; \
  grep -oE '\[[A-Za-z0-9_-]+\]' "$f" | sort | uniq -c | sort -rn | head -15; done
```
