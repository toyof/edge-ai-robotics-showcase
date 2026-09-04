# Engineering Decisions

本ドキュメントは **Human Following Robot** の開発中に直面した技術的課題と、
その意思決定プロセスを記録する。

各エピソードは以下のフォーマットで統一する。

```
事象 → 影響 → 仮説と検証 → 対策 → 結果 → 学び
```

---

# Issue-01: LiDAR 取り付け位置と Scan Filter

## 事象

YDLIDAR Tmini-Plus をロボットカー上部に設置した場合、
LiDAR の計測高度が障害物（テーブルの脚、椅子の脚など）より高くなり、
障害物を検知できない。

## 影響

* Nav2 の Costmap に障害物が反映されず、衝突のリスクが発生。
* 人追従中に低い障害物を回避できない。

## 仮説と検証

| # | 仮説 | 検証方法 | 結果 |
|---|------|----------|------|
| A | 上部設置のまま Costmap の inflation_radius を拡大 | パラメータ変更+走行テスト | 棄却: 低い障害物はそもそも /scan に現れない |
| B | LiDAR を車体下部（地面付近）に設置 | 物理的に再配置 | 採用: 障害物を検知可能に |

## 対策

1. LiDAR を車体下部に再配置。
2. 車体自体が `/scan` に映り込むため、`laser_filters` パッケージの `scan_to_scan_filter_chain` を導入。
3. 車体範囲の scan データを NaN に変換するフィルタを `scan_filter.yaml` で設定。

```yaml
# scan_filter.yaml (抜粋)
scan_to_scan_filter_chain:
  ros__parameters:
    filter1:
      type: laser_filters/LaserScanBoxFilter
      params:
        box_frame: base_link
        min_x: -0.20
        max_x: 0.20
        min_y: -0.18
        max_y: 0.18
        min_z: -0.10
        max_z: 0.50
```

## 結果

| 指標 | フィルタ前 | フィルタ後 |
|------|-----------|-----------|
| 1周あたりの点群数 | [DATA: 例 450 点] | [DATA: 例 380 点] |
| 障害物検知率 | 不可 (上部設置時) | 正常検知 |
| Nav2 走破率 | [DATA: 例 3/3 完走] | [DATA: 例 3/3 完走] |

> **[TODO]** 実測データを計測して上記 [DATA] を埋める。
> `ros2 topic echo /scan --once` と `ros2 topic echo /scan_target_filtered --once` で点群数を比較できる。

## 学び

センサの取り付け位置は「最適な計測」と「自機干渉」のトレードオフであり、
ソフトウェア（scan filter）で干渉を除去できるなら、計測優先の配置が正解。

---

# Issue-02: Scan オドメトリ (rf2o) からホイールエンコーダへの切替

## 事象

4WD 構成（将来のアーム装備を想定）のため車輪の滑りが大きく、
当初はモーターエンコーダを使わず rf2o_laser_odometry（Scan + IMU ベース）でオドメトリを生成していた。

しかし、以下の問題が発生:

* **静止時:** オドメトリが微小にドリフトし、EKF の位置推定が不安定化。
* **移動時:** 前後方向・旋回方向の誤差が大きく、SLAM の地図品質が低下。

## 影響

* SLAM Toolbox のマップにゴースト壁が発生。
* Nav2 のゴール到達精度が低下（目標地点を通り過ぎる）。

## 仮説と検証

| # | 仮説 | 検証方法 | 結果 |
|---|------|----------|------|
| A | rf2o のパラメータチューニングで改善可能 | freq, laser_scan_topic 調整 | 棄却: 根本的な精度不足 |
| B | ホイールエンコーダ + 速度依存補正で改善 | wheel_odom_node 実装 + 実測キャリブレーション | 採用 |

### 検証データ

```
[DATA: rf2o の 1m 直進時オドメトリ推定値]
例: 実測 1.00m → rf2o 推定 0.72m (誤差 -28%)

[DATA: wheel_odom の 1m 直進時オドメトリ推定値]
例: 実測 1.00m → wheel_odom 推定 0.95m (補正後誤差 -5%)
```

## 対策

1. バックアッププランとして用意していたモーターエンコーダオドメトリ (`wheel_odom_node.py`) に切替。
2. 速度依存の距離補正を実装。低速時と高速時で補正係数が異なることを実測で確認。

```yaml
# wheel_odom.yaml (抜粋)
enable_speed_scaling: true
v_scale_low: 0.11        # 低速閾値 (m/s)
v_scale_high: 0.36       # 高速閾値 (m/s)
dist_scale_low: 1.156    # 低速時の距離補正係数
dist_scale_high: 1.462   # 高速時の距離補正係数
yaw_scale_low: 0.505     # 低速時の旋回補正係数
yaw_scale_high: 0.800    # 高速時の旋回補正係数
```

3. EKF (`robot_localization`) でホイールオドメトリ (pose: x, y, yaw) + IMU (yaw, vyaw) を統合。

## 結果

| 指標 | rf2o | wheel_odom (補正後) |
|------|------|-------------------|
| 1m 直進誤差 | [DATA] | [DATA] |
| 90° 旋回誤差 | [DATA] | [DATA] |
| 静止時ドリフト (10秒) | [DATA] | [DATA] |
| SLAM 地図品質 | ゴースト壁発生 | 正常 |

> **[TODO]** 定量データを計測して埋める。

## 学び

4WD 構成で滑りがあっても、速度依存の補正係数を実測キャリブレーションすれば
エンコーダオドメトリは十分に使える。バックアッププランを常に用意しておくことの重要性を再確認。

---

# Issue-03: YOLO 人追従と Nav2 の統合（3段階の進化）

## 事象

人追従機能を段階的に進化させた。各フェーズで異なる問題に直面。

## Phase 1: YOLO + P制御（視覚サーボ直接制御）

### 構成

```
Camera → YOLO → BoundingBox → P制御 → cmd_vel → Motor
```

### 問題

* **障害物を一切回避できない。** LiDAR データを参照していないため、壁や家具に直進する。

---

## Phase 2: Nav2 ローカルコストマップのみ

### 構成

```
Camera → YOLO → Follow Logic → cmd_vel ←→ Nav2 Local Costmap (障害物チェック)
```

### 問題

* **障害物で停止するが、迂回経路を生成できない。** グローバルプランナーが存在しないため「止まるだけ」。

---

## Phase 3: Nav2 フル統合（現在の構成）

### 構成

```
Camera → YOLO → PersonTrackingInfo → FollowGoalGenerator → NavigateToPose (Nav2)
  Nav2: Global Planner (SmacPlanner2D) + Local Planner → cmd_vel → Motor
```

### 結果

* 障害物を迂回しつつ人を追従できるようになった。

## 学び

段階的なアーキテクチャ進化が有効。Phase 1 → 2 → 3 のそれぞれで
「何ができて何ができないか」を明確にしたことで、最終構成の設計判断が合理的になった。

---

# Issue-04: 砲塔 (Turret) と TF ツリー統合の課題

## 事象

Nav2 フル統合により障害物回避が可能になったが、ロボットの旋回角度が大きい場合に
**カメラの視野から人をロスト** する問題が発生。
車体の向きとカメラの向きが一体であるため、Nav2 が大きく旋回すると追従対象を見失う。

## 影響

* 旋回時に人をロスト → 追従停止 → 再検出待ち → リアルタイム性が損なわれる。

## 対策

### 砲塔（Pan-Tilt サーボ）の導入

カメラを砲塔に搭載し、車体の旋回とカメラの向きを切り離した。

```
turret_tracker_node
  ├ PersonTrackingInfo を購読
  ├ PID 制御でサーボ角度を計算
  ├ /pico/servo_target を publish → PicoBridgeNode → Pico → サーボ
  └ TF broadcast: turret_base_link → turret_link
```

### TF 統合の問題

砲塔の向き（Pan 角度）を TF ツリーに統合した (`turret_base_link → turret_link`)。
しかし `follow_goal_generator_node` でゴール生成時に **TF から砲塔の角度を正確に取得できない** 問題が発生:

* TF のタイミングズレにより、取得した角度が古い。
* `lookup_transform` の timeout を調整しても安定しない。

### 妥協案: servo_state の直接購読

```python
# follow_goal_generator_node.py
self.servo_sub = self.create_subscription(
    Vector3, '/pico/servo_state', self.servo_cb, 10
)

def servo_cb(self, msg: Vector3):
    self.current_pan_deg = msg.x  # Pico から直接 Pan 角度を取得
```

`turret_tracker_node` が publish する `/pico/servo_state` を直接購読し、
数学的な座標回転（2D 回転行列）でゴール座標を base_link 基準に変換。

```python
# ターレット座標系 → base_link 座標系への回転
pan_rad = math.radians(self.current_pan_deg)
x_base = x_turret * math.cos(pan_rad) - y_turret * math.sin(pan_rad)
y_base = x_turret * math.sin(pan_rad) + y_turret * math.cos(pan_rad)
```

## 結果

* 砲塔導入により、旋回中も人を追従可能になった。
* TF 経由ではなく直接購読による角度取得は、理想的ではないが実用上安定動作している。

## 学び

TF ツリーは ROS2 のベストプラクティスだが、高頻度で更新される動的フレーム（サーボ角度）は
TF のタイミング同期が難しい場合がある。実用上は直接購読 + 数学的変換で十分なケースもある。
理想と現実の折り合いを付ける判断力が重要。

---

# Issue-05: 砲塔導入後のエンコーダ精度劣化（ToF ブロッキング問題）

## 事象

砲塔（サーボ + ToF センサー）導入後、ホイールエンコーダの精度が **約 50%** に劣化。

## 詳細分析

本件は独立した障害分析ドキュメントに記録。

→ [`docs/tof_blocking_analysis.md`](tof_blocking_analysis.md)

## 学び

マイコンのメインループにブロッキング処理を入れると、I/O スケジューリング全体に波及する。
センサー I/O は別スレッドに分離すべき。

---

# Issue-06: 8GB共有メモリ制約下でのLLM/YOLO排他制御

## 事象

Jetson Orin Nano は CPU・GPU でメモリを共有する（Unified Memory）。
LLM（Qwen2.5 1.5B via llama.cpp）と YOLO パイプライン（Isaac ROS DNN Inference + Depth Anything V2）を
同時に起動すると、合計メモリ消費が 8GB を超え OOM が発生。

* LLM（llama-server）: 約 3GB（モデルウェイト + KVキャッシュ）
* YOLO + Depth パイプライン（TensorRT）: 約 2GB

## 影響

* 同時起動すると推論速度の低下またはプロセス強制終了。
* 「音声で指示を受けて追従する」ユースケースが成立しない。

## 仮説と検証

| # | 仮説 | 検証方法 | 結果 |
|---|------|----------|------|
| A | 両プロセスを常駐させ、非アクティブ時はアイドルにする | メモリ使用量を `jtop` で計測 | 棄却: アイドル中もモデルウェイトがVRAMを占有し続ける |
| B | ROS2 Lifecycle でプロセスを on_cleanup まで落とせば解放される | `cleanup` 後の `/proc/meminfo` 確認 | 採用候補: 解放されるが OS のページキャッシュが残存 |
| C | `cleanup` + `echo 3 > /proc/sys/vm/drop_caches` でキャッシュも解放 | `cleanup` → drop_caches → `configure` のシーケンスを実装 | 採用: 次モデルが必要なメモリを確保できることを確認 |

## 対策

`ai_mode_manager_node` が以下のシーケンスで AI モードを排他切り替えする:

```
現在ノード: deactivate → cleanup
  → sudo drop_caches（ページキャッシュ解放）
  → 次ノード: configure → activate
```

ROS2 Lifecycle の各フックに責務を割り当てた:

| Lifecycle フック | 処理 |
|---|---|
| `on_configure` | モデルロード / llama-server 起動 |
| `on_activate` | 推論開始 |
| `on_deactivate` | 推論停止 |
| `on_cleanup` | プロセス終了・GPU メモリ解放 |

configure タイムアウトは 120s（llama-server の起動時間を考慮）、
activate タイムアウトは 60s（TensorRT エンジン初回ロードを考慮）。

## 結果

* LLM モード（音声対話）↔ YOLO モード（人追従）↔ YOLO-World モード（物体探索）の排他切り替えが安定稼働。
* モード切り替え所要時間: 約 30〜60s（モデルのアンロード→ドロップキャッシュ→次モデルロード）。

## 学び

サーバ運用における「プロセスの graceful shutdown → リソース解放 → 次サービス起動」というオペレーションを
ロボットの AI モード管理に適用した設計。OS レベルのページキャッシュ挙動を把握していないと
「プロセスを落としたのにメモリが足りない」という状況に陥る。

---

# Issue-07: エッジLLMのコマンド分類精度 — Crosslingual Prompting で劇的改善

## 事象

Jetson Orin Nano 上で動作するエッジ LLM（Qwen2.5 1.5B, llama.cpp）に
**日本語のシステムプロンプト**を与えてロボットコマンドを分類させると、
以下のような誤分類が頻発した。

```
入力: 「ヘイロボ、物体検索開始」
期待: search_object
実際: start_mapping  ← 誤分類
```

「開始（かいし）」という語が意味的に「start（開始する）」→ `start_mapping` に引きずられる。
日本語プロンプトでは `search_object` と `start_mapping` の disambiguation が機能しない。

## 影響

* 意図しないモードが起動する（SLAM 地図作成 vs 物体探索）。
* 音声コマンドの信頼性がゼロに近くなり、デモとして成立しない。
* LLM の判定処理時間が **~15 秒** かかり、コマンド→動作開始までのレイテンシが非実用的。

## 仮説と検証

| # | 仮説 | 検証方法 | 結果 |
|---|------|----------|------|
| A | 日本語プロンプトに明示的な禁止ルールを追記（「『物体検索』は絶対に start_mapping を選ぶな」等） | プロンプト改修 + テスト | 棄却: 改善するが一貫性がなく不安定。根本解決にならない |
| B | **英語プロンプト（Crosslingual Prompting）**: 日本語入力のまま、システムプロンプトとツール定義を英語に変更 | `_TOOL_JUDGE_PROMPT_EN_TMPL` を実装し A/B 比較 | **採用: 誤分類ゼロ・判定時間 ~0.8s に激減** |
| C | Python 側キーワードマッチングを LLM 判定の前段フィルタとして追加 | 設計検討のみ | 保留: 最終手段として記録。英語プロンプトで十分なため現時点では未実装 |

### 検証データ（B 案）

```
# 日本語プロンプト（改善前）
入力: 「ヘイロボ、物体検索開始」 → start_mapping  ❌  (~15s)
入力: 「ヘイロボ、ついてきて」   → follow_person  ✅  (~15s)

# 英語プロンプト（改善後）
入力: 「ヘイロボ、物体検索開始」 → search_object  ✅  (~0.8s)
入力: 「ヘイロボ、ついてきて」   → follow_person  ✅  (~0.8s)
入力: 「ヘイロボ、物体検索開始」 → search_object  ✅  (start_mapping への誤爆なし)
```

## 対策

### 英語プロンプトへの切り替え

システムプロンプトとツール定義をすべて英語に変更。入力（日本語音声認識テキスト）は
そのまま渡す。出力は JSON のみ指定。

```
# 変更前（日本語プロンプト）
あなたはロボット音声コマンドの判定役です。...
- search_object: 物体を探す。「探して」「見つけて」などに反応。...

# 変更後（英語プロンプト）
You are a robot voice command classifier.
Analyze the user's Japanese speech and select the ONE correct command.
DISAMBIGUATION RULES (strictly follow):
- "物体検索", "探して", "を探", "見つけて" -> ALWAYS search_object, NEVER start_mapping
- "地図作成", "地図を作", "マッピング" -> start_mapping, NEVER search_object
```

### `prompt_lang` パラメータによる実行時切り替え

```bash
# 英語プロンプト（デフォルト・推奨）
ros2 param set /llm_agent_node prompt_lang en

# 日本語プロンプト（比較デモ用）
ros2 param set /llm_agent_node prompt_lang ja
```

### `search_object` の聞き返しフロー

物体名が特定されなかった場合（`target_object = "unknown"`）に
インタラクティブに聞き返すフローも同時実装:

```
入力: 「ヘイロボ、物体検索開始」
  → LLM: search_object, target_object="unknown"
  → TTS: 「何を探しますか？」
  → 次の発話: 「ペットボトル」
  → activate:yoloworld, target_object="ペットボトル"
```

## 結果

| 指標 | 日本語プロンプト | 英語プロンプト |
|------|----------------|---------------|
| 「物体検索開始」→ 正しく search_object | ❌ start_mapping に誤爆 | ✅ |
| 「止まって」→ stop | ✅ | ✅ |
| 「ついてきて」→ follow_person | ✅ | ✅ |
| LLM 判定時間 | ~15s | ~0.8s |

判定時間の大幅短縮は想定外の副次効果。プロンプトのトークン長・構造の簡潔さが
llama.cpp の推論効率に影響した可能性がある。

## 学び

**Crosslingual Prompting（多言語入力 × 英語プロンプト）は小規模エッジ LLM に有効。**

1.5B パラメータ程度のモデルは学習データの大半が英語であるため、
英語での指示の方が意図を正確に理解できる。
日本語入力 → 英語プロンプト → JSON 出力という分離が、
セマンティック干渉（日本語の「開始」が start_mapping を引く）を排除する。

処理速度の向上は、英語プロンプトがトークナイザに対して効率的（英語の 1 単語 = 少ないトークン）
であることも一因と考えられる。

また、「精度が上がった理由を言語化できないまま使う」ではなく、
仮説（A/B/C）の比較と定量データで判断したことで、今後の言語設計の基準ができた。

---

# Issue-08: 見失った人をどう取り戻すか — 2段リカバリ設計

## 事象

使用している検出器（YOLO/YOLO-World + 自前 `IouTracker`）は ReID（人物再同定）を
持たない。一度追従対象を見失う（LOST）と、トラッカーは再出現した人物を「別人」として
新しい ID を割り振るため、単純に「ロストしたら周囲を見渡して再検出を待つ」だけでは
対象が視野の外（曲がり角の先など）へ移動した場合に再捕捉できない。

## 影響

* 人追従デモの中核シナリオ（人についてきて→物陰に消える→再追従）が成立しない。
* 「見失ったら停止して終わり」では実運用に耐えない。

## 仮説と検証

| # | 仮説 | 検証方法 | 結果 |
|---|------|----------|------|
| A | ロスト後は最後に見えた方位でその場サーチ（砲塔掃引）のみ | 実装・実機検証 | 部分採用: 近距離・視野内ロストには有効だが、対象が移動し続けるケースを取りこぼす |
| B | ロスト直後に人の軌跡から「これから行く場所」を予測し先回りする（Tier1） | breadcrumb記録＋外挿ロジックを実装・pytest | 採用: 曲がり角の先で再捕捉できる確率が上がった |
| C | Tier1 が空振りした場合、地図の未探索領域を自律探索する（Tier2） | GVD（一般化ボロノイ図）/フロンティア探索、部屋アンカー巡回を実装 | 採用: 「諦めない」を担保する最終手段 |

## 対策

3段階の状態機械として設計した。

1. **見渡し（`look_pause_sec` 既定3.0秒）** — ロスト確定直後にその場で停止し、最後に見えた
   方位を優先して砲塔（turret）を掃引する。その方位が LiDAR で塞がっていれば無駄なのでスキップ。
2. **Tier1: 軌跡先回り** — 追従中に記録した人の実測位置の軌跡（breadcrumb）から進行方向を
   外挿し、「人が居た場所」ではなく「これから行く場所」へ Nav2 ゴールを置く。
3. **Tier2: 自律探索** — Tier1 が空振りしたら探索へ移行する。地図がある部屋では AprilTag で
   登録済みのアンカーを巡回し（`RoomPatrolExplorer`）、未知環境では GVD の骨格から
   「まだ見ていない方向」を選ぶ（`GvdExplorer`）。

最も難しかったのは **「諦めない」と「無限ループしない」の両立** だった。探索し尽くしても
即終了せず巡回モードへ自動切替する一方、(a) 半径2.0m以内に居座って新規面積が増えない
**ローカル停滞**と (b) 直近15アクションの移動平均で見る**グローバル停滞**という独立した
2種類の判定で最終的に打ち切る（`search_giveup_timeout_sec`）。

## 結果

状態機械は ROS2 非依存の純ロジック（`follow_recovery_logic.py` / `gvd_explorer_logic.py` /
`frontier_explorer_logic.py` 等）に切り出してあり、実機なしで pytest による回帰テストを
回せる。「修正を無効化すると実際に落ちる」ことまで確認したテストで主要な不変条件
（ローカル/グローバル停滞判定・GATE_THROUGH の結果評価など）を固定している。

## 学び

ReID非搭載の検出器を前提にする場合、「ロスト後にどう振る舞うか」を単一のヒューリスティックに
頼らず、**手元にある情報の多さ（直近の軌跡 > 地図の未探索領域）に応じて段階的に手を打つ
状態機械**として設計すると、個々の対策の限界（Tier1は対象の移動を予測できる範囲でしか効かない、
Tier2は時間がかかる）を補い合える。「諦めない」ロジックは停止条件を明示的に設計しないと
無限ループのリスクを常に抱える。

→ 詳細設計・パラメータ → [`docs/mode_details.md`](mode_details.md)、CLAUDE.md §6.3/§6.4

---

# Issue-09: 発生源を問わない最終安全ゲート（GUARD）と3層の安全ゲート

## 事象

`/cmd_vel`（速度指令）を publish するノードは9箇所ある
（follow_goal_generator / yolo_follow / yoloworld / robot_agent / person_follower /
mapping_lifecycle / apriltag_servo / tag_localization_manager ＋ Nav2 velocity_smoother /
teleop）。個々の呼び出し箇所へ安全チェックを足していく方式では「どれか1つが暴走すれば
同じ事故が再発する」構造が残る。実際に2026-07-21、自己位置推定（AMCL）の共分散発散を
引き金に、壁への押し付け→ホイールスリップ→オドメトリ誤差蓄積→Nav2の連続REJECT→
waypoint高速空回りという事故連鎖が発生した。

## 影響

* 個別ノードへの対策では再発を防げない構造的リスク。
* 2026-08-13には、GUARD（後述）自体の幾何欠陥（扇形コーン方式）により、柱に左前方が
  接触したまま約91秒間前進し続ける事故が発生（GUARDは49回発火しており仕組み自体は
  正常に動いていたが、特定の衝突コースだけ幾何的に見落とす欠陥だった）。

## 仮説と検証

| # | 仮説 | 検証方法 | 結果 |
|---|------|----------|------|
| A | 各 publish 元に個別の安全チェックを追加 | コード調査 | 棄却: 呼び出し箇所が9つに分散し、将来の追加ノードでも同じ穴が再発する |
| B | モーターへの唯一の出口（`pico_bridge_node`）で発生源を問わず前進成分をゲートする | 純ロジック `cmd_vel_guard_logic.py` を新設・pytest | 採用: publisher側の変更ゼロで全経路に効く |
| C | ゲートの前方判定を扇形（コーン）方式にする | 実装・実機検証 | 棄却→是正: 扇形は原点で1点に収束し、衝突コースの障害物が停止判定前にコーンから外れる幾何欠陥があった（2026-08-13実機事故）。footprintをそのまま前方へ掃いた矩形（コリドー）方式へ置き換え |

## 対策

**GUARD（コリドー方式）**: モーターへの唯一の出口である `pico_bridge_node` の `cb_cmd_vel` で、
発生源を問わず前進成分（`linear.x > 0`）のみをゲートする。footprintをそのまま前方へ掃いた
矩形（半幅0.15m）内に障害物があれば減速帯で速度を絞り、停止帯で `linear.x=0` に落とす。
後退・その場旋回は常に通す（全成分を止めると壁の前で脱出不能になるため）。

**3層の安全ゲート（自己位置推定関連）**: GUARDより上位で、`follow_goal_generator_node` に
(A) AMCL共分散発散検知、(B) cmd_vel直接駆動時のNav2明示キャンセル＋スリップ検知、
(C) Nav2連続REJECTスパム停止、の3層を集約した。いずれもゲート発火後は「止めっぱなし」に
せず自力復帰（様子見→nomotion update→低速旋回）を試みる。

さらに「壁にどこまで近づいてよいか」は設定ファイル1箇所（`robot_safety_clearance_m`）を
マスターとし、各機能はそこからの差分計算（`マスター + 機能差分 − 原点オフセット`）で値を
導出するよう統一した。**数値をコピーして値を揃えるのを禁止**したのは、以前それをやって
値がずれ、11日間気付かずに壊れていた実例があるため。この不変条件は専用テスト
（`test_clearance_ladder.py`）が機械的に固定している。

## 結果

回帰テストで正面障害物の停止距離が旧方式と厳密に等価であることを固定したうえで、
実機で衝突コース上の障害物を最後まで視野に残せることを確認した。安全ゲート関連の
ロジックはすべて ROS2 非依存の純ロジックとして切り出され、pytest で検証可能。

## 学び

「安全装置を1つ持っている」ことと「その安全装置に幾何的な死角がない」ことは別問題。
装置自体は正常に発火していても、特定の軌道だけ判定ロジックの前提（扇形の収束点など）から
すり抜けることがあり、実機での様々な衝突コースでの検証が欠かせない。また、複数の安全機構
（GUARD・3層ゲート・離隔設定）が同じ「壁との距離」という概念を別々の基準原点で扱うと、
値を揃えたつもりでも静かにずれる。マスター値からの導出に統一し、不変条件をテストで
固定することが再発防止になる。

→ 詳細 → [`docs/safety_architecture.md`](safety_architecture.md)、CLAUDE.md §6.7

---

# 参考: 障害分析ドキュメント一覧

| ドキュメント | 内容 |
|-------------|------|
| [`docs/serial_deadlock_analysis.md`](serial_deadlock_analysis.md) | シリアル通信バッファ飽和によるデッドロック |
| [`docs/tof_blocking_analysis.md`](tof_blocking_analysis.md) | ToF ブロッキングによるエンコーダ精度劣化 |
| [`docs/observability_detail.md`](observability_detail.md) | 可観測性設計（OTel + Prometheus + Grafana）の詳細 |
