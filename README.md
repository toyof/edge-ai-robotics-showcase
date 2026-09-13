# Edge AI Robotics Platform

[🇯🇵 日本語](README.md) | [🇺🇸 English](README.en.md)

> **AIで賢くなるロボットを、現場で安定稼働させる。**
>
> Jetson Orin Nano 上で **人追従 → 見失う → 軌跡から先回り → 自律探索 → 再捕捉** までを
> 自律で回す実機ロボット。「動くデモ」で終わらせず、可観測性・安全設計・インシデント記録まで
> 通した **運用視点の physical AI** です。

<!-- ロボット実機の写真（正面 + 斜めの2枚構成）。 -->
<p align="center">
  <img src="docs/images/robot_front.jpg" alt="ロボット実機（正面）" width="45%">
  <img src="docs/images/robot_side.jpg" alt="ロボット実機（斜め）" width="45%">
</p>

---

## Why This Project

ロボットそのものが目的ではありません。**「運用・信頼性」「AI の実務適用」「physical AI の実機実装」の
3つが1人に揃っていること**を示すための証拠として作っています。

| よく居る人 | 私の裏付け |
|---|---|
| **運用・信頼性が分かる**（SRE / DBA / インフラ） | Oracle DBA・インフラ運用 10年以上。20名規模の DBA チームリーダー。障害対応・性能管理・運用自動化が主戦場 |
| **AI を実務で使える** | 本業で IT 保守運用の生産性向上を AI で PoC 〜 実導入 |
| **physical AI を実機で組める** | 本プロジェクト。Jetson + ROS2 + Nav2 + YOLO + LLM をゼロから統合し、実機で走らせている |

3つそれぞれの専門家は大勢いますが、**交差点に立つ人はほとんどいません。**
「AI を試す人」でも「ロボットを作る人」でもなく、**エッジ AI を "本番運用" まで持っていける** ——
このリポジトリはその立ち位置の証拠です。

だから以下も「何ができるか」だけでなく、**どう壊れ、どう検知し、どう直したか**を同じ比重で書いています。

---

## Demo

<p align="center">
  <a href="https://www.youtube.com/shorts/wOj_O3pmqlM">
    <img src="https://img.youtube.com/vi/wOj_O3pmqlM/hqdefault.jpg" alt="人追従デモ" width="30%">
  </a>
  <a href="https://www.youtube.com/shorts/eDrZ-v9ciB8">
    <img src="https://img.youtube.com/vi/eDrZ-v9ciB8/hqdefault.jpg" alt="地図なし人追従デモ" width="30%">
  </a>
  <a href="https://www.youtube.com/shorts/eqMXYGakyOI">
    <img src="https://img.youtube.com/vi/eqMXYGakyOI/hqdefault.jpg" alt="音声対話・状況説明デモ" width="30%">
  </a>
</p>

<p align="center">
  <sub><b>左:</b> 人追従（YOLO + 深度推定 + Nav2）　｜　<b>中:</b> 地図なし人追従（<code>local</code> モード / odom フレーム、事前地図なしで動作）　｜　<b>右:</b> 呼びかけ〜状況説明（STT + LLM + Cloud VLM）</sub>
</p>

<p align="center">▶ その他の動画は <a href="https://www.youtube.com/@toyof-robo">YouTubeチャンネル</a> で公開中</p>

### デモシナリオ

以下は**全機能を1本に通したときのシナリオ**です。上の3本の動画のうち左・中が「承」の追従パートを
別々の条件——地図あり／地図なし——で撮ったもの、右が「起」〜「結」（呼びかけから状況説明まで）を
1本で撮ったものです。「転」（自然言語での物体探索）はまだ動画化できていません。

| シーン | 操作 | ロボットの動作 | 使用技術 |
|---|---|---|---|
| 起 | 「ヘイロボ」 | 「はい。何の処理をしましょうか？」 | STT + LLM |
| 承 | 「人についてきて」 | 追従開始 → 障害物を自律回避 → 追従継続 | YOLO + Depth + Nav2 |
| 転 | 「くまのぬいぐるみを探して」 | 自律探索 →「見つかりました」 | YOLO-World |
| 結 | 「何が見える？」 | 部屋にいる人物を検出し、状況を自然文で説明 | Cloud VLM (Gemini) |

> 全処理は Jetson Orin Nano (8GB) 上でリアルタイム実行（VLMのみクラウド）

---

## What This Robot Can Do

| できること | どうやって | 主要技術 |
|---|---|---|
| 音声で指示を受ける | 「ヘイロボ」→ STT → LLM判定 → モード切替 | faster-whisper + Qwen2.5 1.5B |
| 指定物体を追いかける | YOLO検出 → 深度推定 → Nav2ゴール生成（人がデフォルト、音声でペットボトルなど任意のCOCO物体に変更可） | YOLOv8 + Depth Anything V2 + Nav2 |
| 見失っても諦めない | 軌跡から先回り（Tier1）→ 自律探索（Tier2）→ 再捕捉 | breadcrumb 予測 + GVD / フロンティア探索 |
| 障害物を自律回避する | LiDAR → コストマップ → 経路再計画 | SLAM Toolbox + Nav2 DWB |
| 物体を探す | 自然言語指定 → ゼロショット検出 → 自律探索 | YOLO-World |
| タグでルームを記憶し自己位置を復元する | AprilTag検出 → room解決 → 対応マップでAMCL初期位置投入（RViz手動操作不要） | AprilTag + AMCL + Nav2 |
| 見えるものを説明する | カメラ画像 → Cloud VLM → 自然言語応答 | Gemini API |
| 運用状態を可視化する | OTel収集 → 時系列DB → ダッシュボード | OpenTelemetry + Prometheus + Grafana |
| 障害を検知・切り分けする | 仕事量×負荷の相関 → 異常検知 | Grafana アラート + 相関分析 |

---

## Key Numbers

**性能**

| Metric | Value | Condition |
|---|---|---|
| YOLO推論レイテンシ | 70〜128 ms（平均 約10Hz） | YOLOv8s, TensorRT FP16, 640x480, Depth Anything V2 と同時稼働 |
| Depth推論レイテンシ | 90 ms〜1.3 s（平均 2〜3.6Hz） | Depth Anything V2 ViT-S, TensorRT, YOLOv8s と同時稼働（GPU競合時にばらつき大） |
| LLM初回応答（コールドスタート） | 約 51.0 秒 | Qwen2.5 1.5B, llama.cpp GPU offload, context_len=1024, バックグラウンドウォームアップ未完了時（サーバ起動〜ヘルスチェック成功までは別途62.5秒） |
| LLM応答（ウォームアップ済） | 約 1.16 秒 | 同上 |

**規模・検証可能性**

| Metric | Value | 備考 |
|---|---|---|
| カスタムROS2パッケージ数 | 11 | `src/toyof_robot_*` |
| ROS2 非依存の純ロジックモジュール | 43 | `*_logic.py`。実機・コンテナなしで `pytest` 可能（→ [Show Me the Code](#show-me-the-code)） |
| 自動テスト | 約1,870 件 / 70 ファイル | 上記の純ロジック中心。lint 自動テスト（flake8 / pep257 / copyright、27ファイル）は除く |
| CI | GitHub Actions | 上記 pytest ＋ flake8 / pep257（docstring 規約）を push ごとに実行 |

> 計測環境: Jetson Orin Nano 8GB, JetPack 6, Isaac ROS Dev Container
> 未計測の指標（追従時の目標ロスト率・音声コマンド認識→動作開始レイテンシ・連続稼働時間）は実測でき次第追記する。

---

## Engineering Challenges I Solved

実機開発で直面した技術課題と、その調査・解決プロセスの記録です。
**前半は「設計として解いた問題」、後半は「実機で踏んだ障害の調査記録」** に分けています。

### 設計として解いた問題

この3本がプロジェクトの中核です。いずれも「動くものを足す」より
**「壊れ方を先に決める」「作らない判断をする」** ほうに時間を使っています。

#### 見失った人をどう取り戻すか — 2段リカバリ設計

ReID（人物再同定）を持たない検出器では、一度見失った時点で追跡が終わる。
これを「失った瞬間に手元にある情報の多さ」に応じて段階的に手を打つ状態機械として設計した。

1. **見渡し（3秒）** — ロスト確定直後にその場で停止し、最後に見えた方位を優先して砲塔を掃引する。
   その方位が LiDAR で塞がっていれば無駄なので、スキップして次段へ。
2. **Tier1: 軌跡先回り** — 追従中に記録した人の実測位置の軌跡（breadcrumb）から進行方向を外挿し、
   「人が居た場所」ではなく「これから行く場所」へ Nav2 ゴールを置く。
3. **Tier2: 自律探索** — Tier1 が空振りしたら探索へ移行する。地図がある部屋では
   AprilTag で登録済みのアンカーを巡回し、未知環境では **GVD（一般化ボロノイ図 — 通路の中心線を
   抽出する手法）** の骨格から「まだ見ていない方向」を選ぶ。

設計で最も難しかったのは **「諦めない」と「無限ループしない」の両立** だった。
探索し尽くしても即終了せず巡回モードへ自動で切り替える一方、
「同じ場所に居座っていて新規に地図が増えない」ローカル停滞と、直近15アクションの
移動平均で見るグローバル停滞という独立した2つの判定で打ち切る。
状態機械は ROS2 非依存の純ロジック（`follow_recovery_logic.py`）に切り出してあり、
実機なしで pytest による回帰テストを回せる。

```mermaid
stateDiagram-v2
    [*] --> FOLLOWING
    FOLLOWING --> LOST: 検出途絶
    LOST --> LOOK_PAUSE: 見渡し(3秒)
    LOOK_PAUSE --> FOLLOWING: 再検出
    LOOK_PAUSE --> TIER1: 見渡し失敗
    TIER1: Tier1 軌跡先回り
    TIER1 --> FOLLOWING: 再検出
    TIER1 --> TIER2: タイムアウト(8s)
    TIER2: Tier2 自律探索
    TIER2 --> FOLLOWING: 再検出
    TIER2 --> TIER2: 探索完了でも巡回継続(諦めない)
    TIER2 --> GIVEUP: search_giveup_timeout_sec(60s)
    GIVEUP --> [*]
```

> 追従ロスト率・再捕捉成功率は現時点で未計測（本README冒頭「未計測の指標」参照）。上図はロジック構造の設計図。

→ 詳細設計・全パラメータ・実機検証ログ → [docs/mode_details.md](docs/mode_details.md)

#### 発生源を問わない最終安全ゲート

`/cmd_vel`（速度指令）を publish するノードは 9 箇所ある。個々の呼び出し箇所に安全チェックを
足していく方式では「どれか1つが暴走すれば同じ事故が再発する」構造が残る
（実際に自己位置推定の発散で踏んだ）。

そこで **モーターへの唯一の出口** である `pico_bridge_node` に、発生源を問わず
前進成分だけを止めるゲートを置いた。車体幅ぶんのコリドー（走行帯）に障害物があれば
減速帯で速度を絞り、停止帯に入れば `linear.x=0` に落とす。
**後退とその場旋回は常に通す**（全成分を止めると壁の前で脱出不能になるため）。
publisher 側の変更はゼロで、1トピックを購読するだけで全経路に効く。

さらに「壁にどこまで近づいてよいか」は設定ファイル 1 箇所（`robot_safety_clearance_m`）を
マスターとし、各機能はそこからの差分計算で値を導出する。**数値をコピーして揃えるのを禁止** したのは、
以前それをやって値がずれ、11日間気付かずに壊れていたためで、この不変条件は専用テストが
機械的に固定している。

<p align="center">
  <img src="docs/images/guard_corridor_vs_sector.png" alt="扇形方式とコリドー方式の幾何比較" width="80%">
</p>

→ 幾何欠陥の詳細・シーケンス図・3層ゲートの設計 → [docs/safety_architecture.md](docs/safety_architecture.md)

#### レーザーオドメトリを較正した結果、あえてEKFへ統合しなかった判断

車輪スリップ時に汚染される並進速度（vx）を補うため、LiDARスキャンから独立に
並進を推定する `laser_odom_node` を新設した。回転はジャイロから既知として
外部注入し、探索を並進2自由度だけに縮退させることで、一般的なICP系レーザー
オドメトリが抱える「回転と並進の同時推定による不安定化（アパーチャ問題）」を
そもそも避ける設計にした。

実機3セッション・約100万行のログをオフラインで較正した結果は単純な合否では
なかった。申告した共分散は実誤差の**約3倍の過大申告**（`std(z)=0.33`、理想1.0）
で安全側ではあったが、EKFへ統合した場合の平常時の重み占有率は中央値18〜22%に
留まり、**入れてもほとんど何も変わらない**ことが分かった。意味のある効果が出る
スリップ時（重み81〜93%）は、車輪の異常を検出して速度を0へ落とす**別のスリップ
検知の第3軸**として既に回収済みだったため、EKFへの統合はあえて見送った——1つの
センサが「異常の検出根拠」と「検出後の主測定」を兼ねると単一障害点になるという、
精度とは別軸の設計判断による。

<p align="center">
  <img src="docs/images/ekf_weight_occupancy.png" alt="EKF統合を想定した場合のlaser_odom重み占有率" width="65%">
</p>

→ z分布ヒストグラム・較正の全過程 → [docs/engineering_decisions.md](docs/engineering_decisions.md) (Issue-10)

---

### 実機で踏んだ障害の調査記録

いずれも **症状で止まらず真因まで降りた** 記録です。
最初に目立つログやもっともらしい仮説は、たいてい真因ではありませんでした。

#### 8GBメモリ制約下でのLLM/YOLO排他制御

Jetson Orin Nanoの8GB共有メモリでLLM（~3GB）とYOLOパイプライン（~2GB）を
同時に載せられない問題に対し、ROS2 Lifecycle + OS drop_cachesによる
排他的メモリ管理を設計・実装した。

<p align="center">
  <img src="docs/images/ai_mode_memory_budget.png" alt="LLM/YOLO排他制御のメモリ使用量概算" width="70%">
</p>

→ Lifecycle遷移シーケンス図の詳細 → [docs/engineering_decisions.md](docs/engineering_decisions.md) (Issue-06)

#### シリアル通信デッドロックの特定と解消

Jetson ↔ Pico間のUART通信が不定期にハングする事象が発生。
カーネルのシリアルバッファ上限（4095 bytes）への到達が原因と特定し、
送受信プロトコルの再設計で解消した。

<p align="center">
  <img src="docs/images/serial_buffer_backlog.png" alt="シリアル通信RXバッファ滞留の実測推移" width="75%">
</p>

→ 実測ログ全文・通信シーケンス図 → [docs/serial_deadlock_analysis.md](docs/serial_deadlock_analysis.md)

#### ToF I2Cブロッキングによるエンコーダ精度劣化の特定と解消

ToFセンサのI2C読み取りがMCUのメインループをブロックし、
エンコーダ割り込みの取りこぼしが発生。タスク分離により解消。

<p align="center">
  <img src="docs/images/tof_blocking_timeline.png" alt="ToF I2Cブロッキングによるメインループ遅延の概念図" width="80%">
</p>

→ [docs/tof_blocking_analysis.md](docs/tof_blocking_analysis.md)

#### LiDAR強度(intensity)による障害物ゴーストの解消

床の段差が原因だと仮定して距離ベースの対策を試したが、現地確認で物理的な段差は
存在しないと判明。反射強度を直接調べたところ、ゴースト方向は本物の反射より
桁違いに弱い値（強度2〜3 vs 7〜60台）であることを発見し、intensityフィルタで解消。
床面の鏡面反射による多重経路(マルチパス)が原因という仮説を実測データで裏付けた。

<p align="center">
  <img src="docs/images/lidar_intensity_compare.png" alt="ゴースト方向vs本物反射の反射強度比較" width="60%">
</p>

→ スキャンジオメトリ図・多重経路概念図・閾値探索の全過程 → [docs/lidar_intensity_ghost_analysis.md](docs/lidar_intensity_ghost_analysis.md)

#### エッジLLMのコマンド分類精度 — Crosslingual Prompting

Qwen2.5 1.5B（日本語プロンプト）では「物体検索開始」が `start_mapping` に誤分類されていた。
日本語入力のまま**英語プロンプト**に切り替えたところ（Crosslingual Prompting）、
誤分類がゼロになり、LLM 判定時間も **~15s → ~0.8s** に大幅短縮した。
小規模エッジ LLM では、英語プロンプトが日本語のセマンティック干渉を排除する。

<p align="center">
  <img src="docs/images/llm_crosslingual_latency.png" alt="日本語プロンプトと英語プロンプトのLLM判定時間比較" width="55%">
</p>

→ [docs/engineering_decisions.md](docs/engineering_decisions.md)

---

---

## Show Me the Code

> **コード本体は非公開です。** 設計と、その設計が実在することを示す抜粋のみを公開しています。

このプロジェクトの規約は **「ROS2 通信とビジネスロジックを別ファイルに分ける」** こと
（`xxx_node.py` は pub/sub と lifecycle だけ、`xxx_logic.py` は ROS2 を import しない純 Python）。
おかげでロジック側は **実機もコンテナも無しに `pytest` で回せます**（40 モジュール / 1,617 件）。

例として、上の[最終安全ゲート](#発生源を問わない最終安全ゲート)で触れた
「壁にどこまで近づいてよいか」の導出関数と、その不変条件を守るテストを挙げます。

**① 純ロジック** — `src/toyof_robot_navigation/toyof_robot_navigation/clearance_logic.py`

```python
def derive_clearance(master_m: float, delta_m: float,
                     origin_offset_m: float) -> float:
    """マスター値から、呼び出し側の基準における閾値を導出する.

    Args:
        master_m: `robot_safety_clearance_m`。base_link 中心から正面障害物
            までの最小距離 [m]。
        delta_m: その機能の差分 [m]。0.0 でマスターと同一線、正で緩く
            （＝より手前で反応）、負で厳しく振る舞う。
        origin_offset_m: 呼び出し側の測定原点が base_link からどれだけ前に
            あるか [m]。footprint前端基準なら `robot_footprint_front_m`、
            LiDAR の生レンジ基準なら `robot_lidar_offset_x_m`、地図EDT の
            ように中心基準ならば 0.0.

    Returns:
        呼び出し側の基準で比較に使える閾値 [m]。原点が閾値より前にある
        （＝計算結果が負になる）場合は 0.0 にクランプする——負の距離は
        「どんな観測値も閾値を下回らない」＝ゲートが常に無効という意味に
        なってしまい、安全機構としては最悪の壊れ方をするため.
    """
    return max(0.0, float(master_m) + float(delta_m) - float(origin_offset_m))
```

**② その不変条件を機械的に固定するテスト** — `src/toyof_robot_navigation/test/test_clearance_ladder.py`

```python
def test_recovery_layer_is_not_stricter_than_guard(geo):
    """リカバリ層が実行安全層(GUARD)より厳しくないこと（今回壊れていた条件）.

    厳しいと「GUARD は前進を許すのに GATE_THROUGH / ESCAPE / ナッジが
    自ら諦める」帯ができ、狭所で一歩も踏み出せなくなる。途中で止まっても
    GUARD が安全に止めるので、開始判定を GUARD より厳しくする理由は無い.
    """
    guard = _guard_stop_center(geo)
    master = geo['robot_safety_clearance_m']
    for key in ('through_safe_clearance_delta_m',
                'guard_escape_clear_delta_m',
                'nudge_safe_clearance_delta_m'):
        recovery = master + geo[key]
        assert recovery <= guard + 1e-9, (
            f'{key} により リカバリ層 {recovery:.3f}m が '
            f'GUARD 停止 {guard:.3f}m より厳しい（中心基準）。'
            f'GUARD が通す場所でリカバリが諦める帯ができる'
        )
```

このテストが守っているのは**値そのものではなく、値どうしの順序関係**です。
「リカバリ層が実行安全層より厳しい」状態はログにもエラーにも出ず、各ノードは指令どおりに動き、
どこにも例外は出ません。体感でしか分からないため、実際に **11日間気付かれませんでした**。
だから数値ではなく関係をテストに固定しています
（同ファイルには、設定値が `robot.urdf` / `nav2_params.yaml` の実体とズレたら落ちるテストも置いています）。

テストが通ることではなく、**「その修正を無効化したら実際に落ちるか」まで確認する**のを規約にしています。
落ちないテストは何も守っていないためです。

---

## Observability — Robot SRE

> ロボットは「動く」だけでは足りない。
> 「なぜ止まったか」「どこが劣化しているか」を説明できなければ、現場では使えない。

<!-- Grafanaダッシュボードのスクリーンショット（概要 + 詳細の2枚）。 -->
<p align="center">
  <img src="docs/images/grafana_dashboard.png" alt="Grafanaダッシュボード（概要）" width="90%">
</p>
<p align="center">
  <img src="docs/images/grafana_dashboard_detail.png" alt="Grafanaダッシュボード（詳細）" width="90%">
</p>

### 設計思想: 仕事量 × 負荷の相関

従来のサーバ監視と同じく、「何が来たか」と「どのくらい消費したか」を
同一タイムラインで可視化し、障害時の一次切り分けを可能にする。

| 軸 | 収集対象 | 例 |
|---|---|---|
| 仕事量 | 音声コマンド / YOLO検出 / 状態遷移 | 「14:03に人追従開始」 |
| 負荷 | CPU / GPU / メモリ / ROS topic hz | 「14:03からGPU 95%に張り付き」 |

### スタック

```
ROS2ノード (OTel SDK)
  → OpenTelemetry Collector (Jetson)
  → Prometheus (PC/クラウド)
  → Grafana
```

メトリクスとログは**閉じた語彙**の下に置いています。数えたい事象は登録済みの名前でしか
出せず、未登録のメトリクス名は実行時に破棄され、未登録のログタグは CI が検出します。監視項目が場当たりに増えて
「誰も見ないダッシュボード」になるのを構造的に防ぐためです。

### 障害をインシデントとして扱う

2026-07-21、探索中のロボットが壁に押し付けられ、車輪の空転から自己位置推定（AMCL）の共分散が
平常時（0.05〜0.3）の一桁以上——`yaw_var=6.63`——まで発散しました。

このとき大量に出ていたログは `REJECTED (lifecycle likely not active)` でしたが、これは
**症状であって真因ではありません**。一次データ（`/amcl_pose`）まで遡って初めて発散が見えました。

対策は「壊れた動きを直す」ではなく、**「壊れたことを検知して安全に止める層を独立に3つ足す」**。
2026-07-24 に3層すべてが実機で発火し、人手を介さず自動復帰することまで確認しています。

> それらしいエラーメッセージを鵜呑みにせず一次データまで降りるのは、DBA として10年やってきた
> 障害対応の作法そのものです。ロボットでも型は変わりませんでした。

→ 3層ゲートの設計 → [docs/safety_architecture.md](docs/safety_architecture.md) ／
一次切り分けフロー → [docs/troubleshooting_flow.md](docs/troubleshooting_flow.md) ／
アラート別 Runbook → [docs/observability_runbook.md](docs/observability_runbook.md)

### 実装済みと設計段階

「作った」と「設計した」は混ぜません。現時点の線引きは次のとおりです。

| ステータス | 内容 |
|---|---|
| **実装済み・実機稼働** | OTel Collector → Prometheus → Grafana（Jetson 上で systemd 常駐）／3層ドリルダウンのダッシュボード／ログ・メトリクスの閉じた語彙と CI 検証／3層の安全ゲート（実機で発火・自動復帰まで確認） |
| **実装済み・stub 環境で検証** | SLO / Error Budget（multi-window バーンレートの2段アラート）。現時点では S4（稼働率）・S5（パイプライン鮮度）の2指標のみで、実機メトリクスへの接続は未了／アラート別 Runbook（閾値・対処は dev 環境の合成データで確認したのみで、実インシデントでの検証は未了） |
| **設計段階（未実装）** | N台フリート管制（`kubectl scale`）／Azure AKS + GitOps ／ Terraform IaC ／ アラート→自動修復のループ |

**フリート運用への拡張性（設計）**: OTel Collector の `service.instance_id` により、複数台の
ロボットからのメトリクスを同一 Prometheus へ集約できる。2台目の追加時はダッシュボードの
横展開で対応する設計。

→ [docs/observability_detail.md](docs/observability_detail.md)

---

## System Architecture

```mermaid
flowchart TB
    subgraph MCU["Raspberry Pi Pico W (MicroPython)"]
        MOT["Motor Driver / Servo"]
        ENC["Encoder / IMU"]
    end

    subgraph JETSON["Jetson Orin Nano — JetPack 6 / Isaac ROS Dev Container (ROS2 Humble)"]
        subgraph PERC["Perception"]
            CAM["USB Camera"]
            LID["YDLIDAR"]
            SENS["IMU + Wheel Encoder"]
        end
        subgraph AI["AI Inference — Isaac ROS NITROS (zero-copy)"]
            YOLO["YOLOv8 / YOLO-World"]
            DEPTH["Depth Anything V2"]
            LLM["Qwen2.5 1.5B (llama.cpp)"]
            STT["faster-whisper STT / Open JTalk TTS"]
        end
        subgraph NAV["Navigation / Localization"]
            SLAM["SLAM Toolbox"]
            APRILTAG["apriltag_ros (別プロセス)"]
            AMCL["AMCL"]
            EKF["EKF (wheel_odom + IMU)"]
            NAV2["Nav2 (SmacPlanner2D + DWB)"]
        end
        subgraph CTRL["Control / Brain"]
            MODE["ai_mode_manager (Lifecycle 排他制御)"]
            LOCMGR["localization_session_manager"]
            FOLLOW["follow_goal_generator"]
            AGENT["LLM Agent"]
        end
    end

    subgraph OBS["Observability"]
        OTEL["OTel Collector"]
        PROM["Prometheus"]
        GRAF["Grafana"]
    end

    CAM --> YOLO & DEPTH & APRILTAG
    LID --> SLAM
    SENS --> EKF
    SENS -. UART .- ENC

    STT --> AGENT
    AGENT --> MODE
    MODE --> YOLO & LLM & SLAM
    MODE -. ensure .-> LOCMGR
    APRILTAG --> LOCMGR
    LOCMGR -. Nav2/AMCL起動管理 .-> AMCL & NAV2
    YOLO --> FOLLOW
    DEPTH --> FOLLOW
    EKF --> NAV2
    SLAM --> NAV2
    AMCL --> NAV2
    FOLLOW --> NAV2
    NAV2 -->|cmd_vel| MOT
    MODE -. cmd_vel/UART .-> MOT

    MODE -.metrics.-> OTEL
    NAV2 -.metrics.-> OTEL
    OTEL --> PROM --> GRAF
```

<sub>Jetson Orin Nano (AI/ROS2) と Raspberry Pi Pico W (モーター/センサー) の 2 層分離構成。UART で通信。</sub>

| Layer | Component | Role |
|---|---|---|
| Perception | USB Camera + YDLIDAR + IMU + Encoder | 環境認識 |
| AI Inference | YOLOv8 + Depth Anything V2 + Qwen2.5 | 検出・深度・言語 |
| Localization | AprilTag + AMCL + localization_session_manager | ルーム記憶・自己位置復元・Nav2/AMCL一元管理 |
| Control | Nav2 + Follow Goal Generator + LLM Agent | 経路計画・追従・判断 |
| Actuation | Pico W → Motor Driver / Servo | 物理駆動 |
| Observability | OTel + Prometheus + Grafana | 運用監視 |

→ [docs/architecture_detail.md](docs/architecture_detail.md)

---

## Tech Stack

| Category | Technology |
|---|---|
| Edge Device | NVIDIA Jetson Orin Nano (JetPack 6) |
| Framework | ROS2 Humble |
| AI (Vision) | YOLOv8 / Depth Anything V2 / YOLO-World / TensorRT |
| AI (Language) | Qwen2.5 1.5B (llama.cpp) / Gemini API |
| AI (Speech) | faster-whisper (STT) / Open JTalk (TTS) |
| Navigation | Nav2 (SLAM Toolbox + AMCL + EKF) |
| GPU Pipeline | Isaac ROS NITROS (zero-copy) |
| Observability | OpenTelemetry + Prometheus + Grafana |
| MCU | Raspberry Pi Pico W (MicroPython) |
| Auxiliary Sensing | Raspberry Pi 3（固定カメラ、死角補完サブエージェント、Isaac ROS非依存） |
| Container | Docker (Isaac ROS Dev Container) |
| CI/Dev | GitHub Actions (pytest + flake8/pep257) / x86 Gazebo sim (`robotcar-sim`) + stub nodes for hardware-free testing |

---

## How I Built It — AIエージェント駆動開発

このプロジェクトは **AI コーディングエージェント（Claude Code）と共同で開発しています。**
ただし価値があるのは「AI に書かせたこと」ではなく、**AI を長期プロジェクトの運用に載せるための
ルールを設計したこと** です。本業のテーマ（IT 保守運用の生産性を AI で上げる）と同じ問題構造を、
個人プロジェクトで実践しています。

| 仕組み | 解いている問題 |
|---|---|
| **エージェント向け指示書を「ルールとインデックス」に限定** | 決定の本文まで書くと肥大して読まれなくなる。決定は設計ノートへ委譲し、指示書には1〜3行の索引だけ置く |
| **重要度ラベルを2軸で合成**（目的への接続 × リスク → P0〜P2 / hold） | 「技術的に面白い方」へ流れるドリフトを構造的に止める。技術的な正しさではなく、目的への寄与だけで優先度が決まる |
| **タスクキューを実行環境ごとに二重化**（実機用 / デスク用） | 貴重な実機セッションが実機不要な作業に食われるのを防ぐ。1セッション丸ごと失った事故から作ったルール |
| **セッション冒頭の台帳検証 `grep`** | ポインタが腐る（実タグが消えて説明文だけ残る）ことを毎回機械的に検出する。人間の記憶を前提にしない |
| **実機セッション中は不具合を直さない** | 再現手順とログの採取だけ行い、原因究明と実装は起票してデスクセッションへ回す。実機時間を保護するための意図的な制約 |
| **ROS2 通信とロジックの分離を規約化** | エージェントが書いたコードを実機なしで `pytest` 検証できる状態を保つ（→ [Show Me the Code](#show-me-the-code)） |

要するに、**エージェントに対しても「運用設計」をしています。** 目的を定義し、ドリフトを検知し、
高コストなリソース（実機時間）を保護する——DBA / SRE でやってきたことと同じ型です。

---

## Documentation

**まず動かしたい方へ**: ビルド・起動手順は [docs/development_guide.md](docs/development_guide.md) にあります。
Jetson 実機が無くても、x86 の Gazebo シミュレーション（`robotcar-sim`）で動作を再現できます。

| Document | Content |
|---|---|
| [docs/architecture_detail.md](docs/architecture_detail.md) | ROS2ノードグラフ、レイヤード設計詳細、センサーフュージョン |
| [docs/development_guide.md](docs/development_guide.md) | ビルド手順、テスト手順 |
| [docs/observability_detail.md](docs/observability_detail.md) | OTelスタック構成、ファイル配置 |
| [docs/serial_deadlock_analysis.md](docs/serial_deadlock_analysis.md) | UART通信障害の調査記録 |
| [docs/tof_blocking_analysis.md](docs/tof_blocking_analysis.md) | I2Cブロッキング障害の調査記録 |
| [docs/lidar_intensity_ghost_analysis.md](docs/lidar_intensity_ghost_analysis.md) | LiDAR障害物ゴースト誤検出の調査記録（intensityフィルタ導入） |
| [docs/engineering_decisions.md](docs/engineering_decisions.md) | 設計判断の記録 |
| [docs/robot_architecture.md](docs/robot_architecture.md) | ロボットアーキテクチャ詳細 |
| [docs/mode_details.md](docs/mode_details.md) | 各AIモードの内部ロジック・起動シーケンス・パラメータ |
| [docs/safety_architecture.md](docs/safety_architecture.md) | 安全設計（多層フェイルセーフ・cmd_velガード） |
| [docs/troubleshooting_flow.md](docs/troubleshooting_flow.md) | 障害切り分けフロー（Robot SRE） |
| [docs/observability_runbook.md](docs/observability_runbook.md) | アラート別 Runbook（SLO バーンレート / 安全停止頻発 等の初動） |
| [docs/logging_map.md](docs/logging_map.md) | どのノードがどのログへ何を出すかの一覧（調査の入口） |
| [docs/robotics_as_mcp_design.md](docs/robotics_as_mcp_design.md) | マルチロボット連携の設計書（Robotics as MCP、未実装） |

---

## License

MIT License — 詳細は [LICENSE](LICENSE) を参照。
