# Robotics as MCP — 設計書

> ステータス: **Phase 1（案A・読み取り専用4 tool）実装済み**（2026-10-07、`local/fleet_mcp/`）。
> **Phase 2/3（ロボ群をまとめて操作する tool）は設計のみ → §10**（2026-10-07、未実装・承認待ち）。`todo/multi_agent_fleet.md` F-S1 / F-S1-P1。
> 位置づけ: `career_strategy.md` C-2-2（設計1枚として外向けに示す資産）/
> C-2-3（技術ブログ・GitHub公開の素材）に接続する。実装は F-3（ロボB追加）
> 完了後、余力があれば着手する「棚」であり、本ロボットの背骨
> （人追従→ロスト→リカバリ）には寄与しない。

---

# 1 Prior Art（着手前の事実確認）

CLAUDE.md の指摘どおり、「ROS を MCP でAIに公開する」こと自体はまだ確立されていない
領域ではない。2026-07時点で確認できた事例:

- 単体 ROS/ROS2 を MCP 化するサーバーは既に50以上存在する（`ros-mcp-server` /
  `rosbridge-mcp-server` / `ros2_mcp`（LCAS）/ `ros2-mcp-server`（kakimochi）等）。
  いずれも「自然言語 → MCP tool call → 特定トピックへの publish/subscribe や
  アクション呼び出し」という同型のパターンで、rosbridge WebSocket 経由か
  ROS2 クライアントライブラリ直結かの実装差はあるが、**単一ロボットの
  遠隔操作・状態照会を LLM エージェントに開放する**という点では収束している。
- 一方、**複数の実機ロボット（フリート）が同じ地図上の物体情報を共有し、
  その集約結果を MCP 越しに1つの窓口として公開する**という事例は、
  上記調査の範囲では見当たらなかった。一般的な「マルチエージェント
  オーケストレーション × MCP」の議論はソフトウェアエージェント間の
  タスク委譲パターンとして多数あるが、物理ロボット・実センサ由来の
  情報連携を主題にしたものではない。

**結論: 「MCPでROSを叩く」は確立領域、「複数の実機ロボットの知覚結果を
mapベースで集約しMCPで1窓口化する」はニッチ。** 本設計書は後者の立ち位置を取り、
README・外部発信では「MCP自体の新規性」ではなく「フリート集約層としての
適用」を主張する（誠実さのため、この事実確認を明文化しておく）。

---

# 2 なぜやるか（career_strategy.md との接続）

このロボットの技術的な背骨（追従→ロスト→リカバリ）は `object_tracking_recovery.md`
が担い、`career_strategy.md` C-2-1 のキラーデモ動画で証明する。本設計書はそれとは
別軸で、**「作ったものを外部の標準プロトコルに接続できる形に一段抽象化できる」**
ことを示す資産である。

- **K1（外向け資産の素材）**: MCP は Anthropic が策定したオープン標準であり、
  「エッジAIロボットを標準プロトコル越しに外部のAIエージェントへ開放する」という
  設計は、`career_strategy.md` の主軸ストーリー（運用・信頼性 × AI実務 × physical AI
  の交差点）に「標準プロトコルへの接続力」という4点目の実演材料を足せる。
- 実装は要らない。**設計書1枚でこの主張は成立する**（CLAUDE.md「設計を描き切った
  時点で価値の半分は取れている」）。
- 既存実装（後述 §4）は F-1（マルチエージェント情報連携）としてロジック層
  （ROS2非依存）まで完成済みであり、MCP化は「新しい機能を作る」のではなく
  「既にある能力の見せ方を変える」だけで済む低コストな抽象化である。

---

# 3 スコープ（今回の設計対象/対象外）

**対象**: ロボットフリートが共有する「誰が・どこで・何を見つけたか」という
知覚結果（`ObjectFound` / `room_object_memory_logic.py` が保持する情報）を、
MCP tool として外部の LLM エージェントに公開する層の設計。

**対象外**（本設計書では扱わない）:

- ロボットの直接遠隔操作（`navigate_to_pose` 等の実行系 tool 化）。
  安全設計（`localization_safety_logic.py` の各種ガード）を経由しない
  外部からの直接駆動は事故リスクが高く、後続フェーズの検討課題とする。
- MCP サーバーの認証・権限設計（`arxiv:2605.22333` 等が指摘するとおり
  実運用の MCP サーバー認証は未成熟な領域であり、本ロボットは個人開発の
  ネットワーク内利用が前提のため、公開時は別途検討する）。
- F-2a/F-2b（`frame_prefix`/`ns` 対称化）・F-3（ロボB実機調達）そのもの。
  本設計は F-1 の情報連携層の上に乗るため、これらの完了を待たずに設計だけは
  先行できるが、**実装・実機検証は F-3 完了後**（`multi_agent_fleet.md` の
  既存方針どおり）。

---

# 4 既存資産の棚卸し（F-1、実装済み・ROS2非依存）

MCP化の土台となる既存実装（`multi_agent_fleet.md` F-1-1〜F-1-6、2026-07-15完了）:

| コンポーネント | 役割 | ROS2 依存 |
|---|---|---|
| `room_object_memory_logic.py` | `record_object(agent_id, name, x, y, confidence, timestamp)` / `query_latest_object(name)`（timestamp最大優先で競合解決） | なし（pytest対象） |
| `map/rooms/<name_id>/objects.yaml` | room（`name_id`）単位の物体観測履歴。map座標のみ保持しframe名を持たないため prefix 化の影響を受けない | — |
| `toyof_robot_interfaces/msg/ObjectFound.msg` | `agent_id`/`name`/`x`/`y`/`confidence`/`timestamp`/`name_id` + `Header` | あり |
| `/fleet/object_found`（グローバルトピック、transient_local） | 各エージェントが物体を確定するたび publish、他エージェントが subscribe してマージ | あり |
| `ai_mode_manager_node._save_object_memory()` | 物体確定時に record_object() 保存 → publish の起点 | あり |
| `robot_agent.py` (`memory_query`/`_navigate_to_coords`) | LLM 経由で「〜はどこ？」を受けて記憶座標へナビゲーション | あり |

**この表の右端2列（ROS2依存あり）が MCP サーバーの実装対象になる。** 左の2つ
（ロジック・データスキーマ）はそのまま MCP サーバー側でも再利用できる
（ROS2 非依存のため、MCP サーバープロセスに直接 import するか、同じ YAML を
共有マウントで読む形になる。詳細は §6 アーキテクチャ選択 A/B）。

---

# 5 MCP tool 設計（案）

外部の LLM エージェント（Claude Desktop 等）が呼び出す想定の tool 一覧。
いずれも「フリート全体の知覚結果への読み取りアクセス」に限定し、駆動系は含まない
（§3 スコープ対象外）。

```jsonc
// tool: fleet_find_object
// 「フリートのどれかのロボットが最後に見た <name> はどこか」を返す
{
  "name": "fleet_find_object",
  "description": "Query the latest known location of an object across the robot fleet, resolved by most-recent timestamp regardless of which agent observed it.",
  "input_schema": {
    "type": "object",
    "properties": {
      "name": { "type": "string", "description": "Object name, e.g. 'mug'" },
      "room": { "type": "string", "description": "Optional name_id to scope the search to a single room" }
    },
    "required": ["name"]
  }
}

// tool: fleet_list_agents
// 現在稼働中のエージェント一覧とその最終ハートビート時刻
{
  "name": "fleet_list_agents",
  "description": "List known agent_ids in the fleet and when each last reported an observation.",
  "input_schema": { "type": "object", "properties": {} }
}

// tool: fleet_room_summary
// 部屋（name_id）ごとに既知の物体一覧を返す
{
  "name": "fleet_room_summary",
  "description": "Summarize all objects known in a given room (name_id), with which agent observed each and when.",
  "input_schema": {
    "type": "object",
    "properties": { "room": { "type": "string" } },
    "required": ["room"]
  }
}
```

**設計判断: 駆動系 tool（`fleet_navigate_agent_to` 等）は Phase 2 以降に先送りする。**
理由は §3 のとおり安全ガードとの整合が未検討のため。Phase 1 は読み取り専用の
「フリート知覚のクエリ窓口」に限定することで、MCP サーバー自体の実装・検証
コストを小さく保つ（CLAUDE.md の「一足飛びの実装をしない」方針と整合）。

---

# 6 アーキテクチャ選択

MCP サーバープロセスがロボット側のデータにどうアクセスするかで2案ある。

**案A: MCP サーバーが `objects.yaml` を直接読む（ROS2非依存・ファイル共有）**

```
[LLM Client] --MCP(stdio/HTTP)--> [MCP Server (Python)] --read--> map/rooms/*/objects.yaml
```

- 長所: `room_object_memory_logic.py` をそのまま import するだけで済み、
  ROS2 環境が無いマシン（開発PC等）でも MCP サーバーを動かせる。実装が最も軽い。
  Prior Art の多くのROS MCPサーバーと違い ROS2 ランタイム自体を必要としない
  ため、依存が薄いのが差別化点にもなる。
- 短所: ファイルシステム共有（NFS等）かポーリングが前提になり、
  「今まさに見つけた」というリアルタイム性は `/fleet/object_found` 経由に劣る。

**案B: MCP サーバーが ROS2 ノードとして `/fleet/object_found` を subscribe する**

```
[LLM Client] --MCP--> [MCP Server = rclpy Node] --subscribe--> /fleet/object_found (DDS)
```

- 長所: リアルタイム性が高く、Prior Art の一般的な ROS MCP サーバーと同型の
  実装パターンに乗れる（保守性・他事例との比較がしやすい）。
- 短所: MCP サーバーが ROS2 依存になり、Isaac ROS コンテナ内 or 同一
  `ROS_DOMAIN_ID` のマシンでしか動かせない。外部公開時のポータビリティが落ちる。

**採用案（暫定）: 案A を Phase 1、案B を Phase 2 で検討。**
まず「フリートの知覚結果を外部に見せる」という価値を最小実装で証明し
（案Aはロジック層が既に完成しているため実装コストが最小）、リアルタイム性が
要る用途（例: LLMに「今探して」と言わせて数秒待たせる)が出てきた段階で
案Bへの切り替えを検討する。

---

# 7 実装フェーズ（設計のみ・未着手）

```
Phase 0（本設計書）: 設計のみ。実装なし。
Phase 1: 案A（ファイル読み取り）で fleet_find_object / fleet_room_summary を実装。 ← 実装済み（2026-10-07）
         実装では fleet_list_agents（最終報告時刻で代用）と fleet_list_rooms（部屋名の入口）も追加。
         → local/fleet_mcp/README.md
         F-1 実装（objects.yaml）が既にあるため、実機ロボB無しでも sim 2プロセス
         （F-1-6 と同じ構成）で検証可能。
Phase 2: F-3（ロボB実機追加）完了後、リアルタイム性の要求を見て案Bへの
         移行を検討。fleet_list_agents のハートビート判定を DDS discovery
         ベースに置き換える。
Phase 3（対象外・要別途設計）: 駆動系 tool・認証設計。
```

---

# 8 リスク・オープンクエスチョン

- **Q1**: MCP サーバーの認証をどうするか（個人LAN内利用の前提が崩れ外部公開する場合）。
  Phase 1（LAN内・読み取り専用）では据え置いてよいが、Phase 3 着手前に必須で検討する。
- **Q2**: `objects.yaml` はロボットのローカルディスクにある。案Aのファイル共有には
  NFS/rsync等の同期手段が要り、`multi_agent_fleet.md` Q1（ロボB実体: 移動ロボ or
  固定PCカメラ）の結論次第で構成が変わる。
- **Q3**: 本設計書の外部公開（ブログ・README掲載）タイミングは `career_strategy.md`
  C-2-3 の媒体選定と合わせてユーザー判断待ち。

---

# 9 まとめ

「ROSをMCPで叩く」は既に確立領域だが、「複数実機ロボットの知覚結果をmapベースで
集約しMCPで1窓口化する」動きは薄い。本ロボットは F-1 でその集約層（ロジック・
スキーマ・トピック配線）を既に実装済みであり、MCP化はゼロから作るのではなく
「既存資産の見せ方を変える」だけで到達できる。実装は F-3（ロボB追加）後の
「余力があれば」枠だが、この設計書自体が `career_strategy.md` C-2-2/C-2-3 に
接続できる資産として、実装前の時点で価値を持つ。

---

# 10 Phase 2/3 設計: ロボ群をまとめて tool で操作する（2026-10-07、設計のみ）

> ステータス: **設計のみ・実装未着手**。新規ノード追加にあたるため、実装は本章の承認後
> （CLAUDE.md §1.3）。台帳 → `todo/multi_agent_fleet.md` F-S1-P2。
> 位置づけ: ユーザーの「夢」枠（背骨・キャリアの K 軸には接続しない＝無印の棚）。
> §3 で「対象外」としていた駆動系を、**既存の安全経路を一切迂回しない形に限って**取り込む。

## 10.1 ゴール

外部の LLM エージェント（Claude Desktop 等）から、**フリート全体を1つの道具箱として**扱えるようにする。

```
「家じゅうで赤いマグカップを探して、見つけたら一番近いロボを行かせて」
  → fleet_search("red mug")        … 全カメラ・全ロボへ一斉に問い合わせ（F-4）
  → fleet_status()                  … 誰が空いているか
  → fleet_tell("jetson01", "赤いマグカップのところへ行って")  … ロボ自身の LLM に任せる
```

## 10.2 安全の原則（本章で一番重要。tool を足すたびにここへ照らす）

| # | 原則 | 理由 |
|---|---|---|
| S1 | **`cmd_vel`・`navigate_to_pose` を tool にしない。** 渡すのは既存の高水準命令（モード切替・停止・ロボ自身の LLM への依頼）だけ | 高水準命令は ai_mode_manager の排他制御（T-55 single-flight）→ 各モードの安全ゲート（§6.7）→ GUARD（モーター直前の前進ゲート）を必ず通る。低水準を直接開けると、走行中の追従モードと Nav2 ゴールを奪い合う |
| S2 | **受け付ける命令はホワイトリスト。** `SwitchMode.srv` の書式（`activate:<mode>` / `stop` / `mode:*` / `status`）以外は MCP 層で弾く | LLM の生成文字列をそのまま流さない |
| S3 | **停止は常に通す。** `fleet_stop` は承認不要・ホワイトリスト検査のみ・全台一括可 | 止める操作を止める理由は無い |
| S4 | **動かす tool には `destructiveHint: true`** を付け、クライアント側で人間の承認を挟む（Claude Desktop/Code は実行前に確認を出す） | 「AI が勝手に動かした」を構造的に起こさない |
| S5 | **結果は「受け付けたか」まで。** 遷移や移動の完了は待たない（`SwitchMode` と同じ、configure は最大200秒） | tool 呼び出しを長時間ブロックしない。進行は `fleet_status` で見る |
| S6 | **ネットワークに開かない。** トランスポートは stdio のみ（クライアントが PC 上で子プロセスとして起動） | MCP の認証は未成熟（§3）。クラウドの Claude から LAN のロボへ届かせる経路は本設計の対象外 |

**GUARD は MCP 経由の命令にも等しく効く**（モーターへの唯一の出口 `pico_bridge_node` にあるため）。
ただし GUARD は物理的な非常停止の代わりにはならない（§6.13 の foxglove_bridge と同じ注意）。

## 10.3 tool 一覧

| tool | 種別 | 中身（既存のどこに乗るか） | annotations |
|---|---|---|---|
| `fleet_status(robot?)` | 読み | 各ロボの `status/lifecycle`（`IDLE`/`TRANSITIONING`/`ACTIVE:<MODE>`）と `status/activity`（`<MODE>:<PHASE>`）の最新値。どちらも transient_local なので購読直後に最新値が届く（§3.6） | readOnly |
| `fleet_find_object` ほか Phase 1 の4本 | 読み | 変更なし（案A を残す。案B のノード内でも同じロジックを呼ぶ） | readOnly |
| `fleet_search(object_en, object?, room?)` | 読み（推論を起こす） | `/fleet/search_request` へ publish し、`/fleet/search_status`・`/fleet/object_found` を `FleetSearchCoordinator`（F-4-2、ACK窓2.0s→結果締切30s）で集計して返す。**ロボは動かない**（固定カメラと各ロボのカメラが見るだけ） | readOnly, openWorld |
| `fleet_switch_mode(robot, command)` | 動かす | `/<robot>/mode/request`（`SwitchMode.srv`）を呼び、`accepted`/`code`（`BUSY` 等）/`message` をそのまま返す | **destructive** |
| `fleet_stop(robot \| "all")` | 止める | `/<robot>/mode/stop`（`std_srvs/Trigger`）。`all` は既知の全ロボへ並列に呼ぶ | — （S3） |
| `fleet_tell(robot, text)` | 動かしうる | `/<robot>/speech/typed_hint` → `/<robot>/speech/user_text` の順に publish（＝ロボの LLM エージェントへの「話しかけ」。ダッシュボード入力欄 N-VIZ-26d と同じ手順でウェイクワード不要）。返答は `/<robot>/speech/speak_command` を数秒集めて返す。**`status/lifecycle` が `ACTIVE:LLM` でなければ送らず**、`fleet_switch_mode(robot, "activate:llm")` を促すエラーを返す（LLM モード以外では誰も聞いていない） | **destructive** |

### `fleet_tell` を入れる理由（エージェント同士の会話）

「マグカップのところへ行って」のような**目的レベルの依頼**は、ロボ側の `robot_agent.py` が既に持っている
（記憶検索 `memory_query` → Nav2、ミスなら `activate:yoloworld`、F-4-5 でフリート照会も挟む）。
MCP 側にナビゲーションを再実装すると S1 に反するうえ、ロボ側の判断（地図確立ゲート・メモリ予算ゲート等）を
二重に持つことになる。**外の Claude は「何をしてほしいか」をロボの LLM に話しかけるだけ**にし、
「どう動くか」はロボ自身が決める。マイク無しテスト（CLAUDE.md §5.3 の `speech/user_text` pub）と同じ経路で、
新しい命令口を作らない。

- 制約: 返答は「ロボが喋った文」で構造化されていない。合否は `fleet_status` の遷移で確認する。
- ウェイクワード: `llm_agent_node` はウェイクワード（「ヘイロボ」等）を要求するが、直前に `speech/typed_hint` を送るとアクティブ時間が始まり不要になる（N-VIZ-26d、入力欄と同じ）。MCP 層は typed_hint → user_text の順で送る。

## 10.4 アーキテクチャ（案B を採用）

```
Claude Desktop / Claude Code（PC）
   │ MCP (stdio)
   ▼
fleet_mcp_node（rclpy ノード、PC 上・ROS2 Humble）
   ├─ subscribe  /<robot>/status/lifecycle, /<robot>/status/activity     （各ロボ）
   ├─ pub/sub    /fleet/search_request, /fleet/search_status, /fleet/object_found （グローバル）
   ├─ client     /<robot>/mode/request, /<robot>/mode/stop               （各ロボ）
   └─ pub/sub    /<robot>/speech/user_text, /<robot>/speech/speak_command （各ロボ）
   │ DDS（同じ ROS_DOMAIN_ID）
   ▼
jetson01 / jetson02 / PC カメラ（subagent_vision）…
```

| 決定 | 内容 | 理由 |
|---|---|---|
| D1 置き場所 | `local/fleet_mcp/` に `ros_bridge.py`（rclpy 配線）を足す。純ロジックは `fleet_mcp_logic.py` 系に置き pytest 対象 | Phase 1 と同居させ、rclpy が無い環境では Phase 1 の4本だけで起動する（import 失敗で案B 部分を無効化） |
| D2 ロボの列挙 | `local/robot_ops/fleet.json` の `robots[].id` を正とする（`fleet_config.py` を再利用） | 運用画面（N-VIZ-9）とフリート構成表を1つに保つ。DDS discovery で「見えているか」は `fleet_status` に `reachable` として添える |
| D3 トピック名 | `/<robot_id>/` + 相対名で組み立てる。`/fleet/*` だけグローバル | §3.5 の namespace 運用そのもの。`robot_id` が空（無印運用）なら前置しない（`robot_ns.sh` と同じ規則） |
| D4 executor | `MultiThreadedExecutor` を別スレッドで spin し、MCP の非同期ハンドラからは Future で待つ | MCP サーバーのイベントループと rclpy の spin を同じスレッドで回さない（F-4-4 の ACK 窓問題と同じ構造） |
| D5 F-4 の集計 | `FleetSearchCoordinator` を**ファイル単位で読み込んで再利用**する（`toyof_robot_ai_control` パッケージには依存しない） | ROS2 非依存の純ロジックなので PC でも動く。パッケージ依存にすると Isaac ROS 依存で PC ビルドが落ちる（F-3-1 で踏んだ） |
| D6 承認 | 動かす tool は `ToolAnnotations(destructiveHint=True)`。サーバー側でも `--allow-motion` 起動フラグが無ければ `fleet_switch_mode`/`fleet_tell` を**登録しない** | クライアントの承認 UI に頼り切らない二重の鍵。既定は「見る・探す・止める」だけ |

## 10.5 前提と順序

| 順 | 内容 | 状態 |
|---|---|---|
| 0 | F-3-2 RMW 統一（Jetson=Cyclone / PC=Fast DDS の混在解消）と `ROS_DOMAIN_ID` 分離 | 設計確定・未実装（`docs/fleet_dds_design.md`） |
| 1 | F-4-6b（Jetson・PC の `toyof_robot_interfaces` 一致確認） | 手順書完了・実施待ち |
| 2 | **P2a**: `fleet_status` / `fleet_stop` / `fleet_search`（動かさない3本＋停止） | 未着手 |
| 3 | **P2b**: `fleet_switch_mode`（`--allow-motion`） | 未着手 |
| 4 | **P3**: `fleet_tell`（ロボ LLM への依頼） | 未着手。F-4-5 の後が望ましい |

**ロボ1台でも成立する。** 「群」は jetson01＋PC カメラ（subagent_vision）で形になり、2台目は sim
（`docker/sim/`、`robot_id:=sim01`）で代替できる。

## 10.6 検証方針

- 純ロジック（命令のホワイトリスト検査・トピック名の組み立て・`fleet_status` の畳み込み・`all` の展開）は pytest。
  **ホワイトリストを外したら落ちる**ことをミューテーションで確認する（§3.4）。
- 配線は sim で E2E: sim01 を起動し、MCP クライアントから `fleet_switch_mode("sim01","activate:llm")` →
  `fleet_status` に `ACTIVE:LLM` が出る → `fleet_stop("all")` で `IDLE` に戻る。
- 実機は「止める」から確認する（`fleet_stop` が効くことを先に確かめてから `--allow-motion` を試す）。

## 10.7 オープンクエスチョン

| # | 論点 |
|---|---|
| Q-P1 | `fleet_tell` の返答をいつまで集めるか（固定秒 vs 発話が途切れたら）。LLM の応答は数秒〜十数秒 |
| Q-P2 | `fleet_search` の後に「見つけた場所へ行かせる」を MCP 側で組むか、`fleet_tell` に任せるか（S1 に照らすと後者） |
| Q-P3 | 複数のクライアント（Claude と人間の Lichtblick 操作）が同時に命令したとき。ロボ側 single-flight（T-55）が `BUSY` で守るので MCP 側では調停しない、で足りるか |
| Q-P4 | ~~ウェイクワードが必要か~~ → 解決: `speech/typed_hint` を先に送れば不要（§10.3） |
| Q-P5 | 外部（クラウドの Claude）から呼べるようにするか。やるなら認証付きトンネル＋読み取り tool 限定から（S6 の例外として別設計） |

## 10.8 承認依頼の要点（F-S1-P2b、2026-11-14 追記）

> 目的: ユーザーが §10 全体を読み直さず、**決める点だけ**で承認できるようにする。承認が出るまで実装には着手しない（CLAUDE.md §1.3）。

### 承認してほしいこと

**新規ノード `fleet_mcp_node`（rclpy、PC 上）を `local/fleet_mcp/` に足す。** ロボ（Jetson）側のコードは変えない。既存の `mode/request`・`mode/stop`・`speech/user_text` を外から呼ぶだけで、新しい命令口は作らない（§10.3）。

### 決める 6 点（推奨つき）

| # | 決めること | 推奨 | 理由・代替 |
|---|---|---|---|
| 1 | 実装の段階 | **P2a（見る・探す・止める）だけ先に**。`fleet_status` / `fleet_search` / `fleet_stop` | 動かす tool（P2b/P3）は `--allow-motion` が要る。まず「止める」が効くことを確認してから（§10.6）。代替: P2a〜P2b をまとめて承認 |
| 2 | 動かす tool の鍵 | **二重の鍵を維持**（クライアントの承認 UI＋サーバーの `--allow-motion`） | クライアント側の確認だけに頼ると、設定ミスで無確認になる（D6） |
| 3 | ロボ構成表 | **`local/robot_ops/fleet.json` を正**（運用画面と共用） | 構成表を 2 つ持たない（D2） |
| 4 | 前提の F-3-2（RMW 統一） | **P2a の前に実施は不要、ただし実機検証の前には必須** | 現状は Jetson=Cyclone / PC=Fast DDS の混在（`docs/fleet_dds_design.md`）。純ロジックと sim 検証は混在でも進められる |
| 5 | 外部（クラウドの Claude）への公開 | **しない（stdio のみ）**（S6、Q-P5） | MCP の認証は未成熟。LAN のロボをネットへ開く設計は別承認。**クライアントはローカル LLM が主、Claude は任意のおまけ**（§10.9、2026-11-14 ユーザー指示） |
| 6 | 位置づけ | **無印の棚（K0）のまま**。3 本柱（Physical AI／可観測性／ダッシュボード）を止めない | CLAUDE.md §1.1。やるなら設計を描き切った時点で価値の半分は取れている |

### 先にわかっているリスク

- **安全**: GUARD はこの経路の命令にも等しく効く（モーター直前にあるため）が、物理的な非常停止の代わりにはならない。`cmd_vel` / `navigate_to_pose` は tool にしない（S1）。
- **同時操作**: Claude と人間（Lichtblick）が同時に命令したら、ロボ側 single-flight（T-55）が `BUSY` で断る。MCP 側では調停しない（Q-P3）。
- **ROS2 依存**: `rclpy` は PC の Humble 前提。無い環境では Phase 1 の 4 本だけで起動する（D1）。
- **検証の限界**: 実機（Jetson）との接続は F-4-6b（`toyof_robot_interfaces` の一致確認）が先。クラウドデスクでは純ロジック＋stdio E2E までしか確認できない。

### 承認後に起票するもの

| ID | 内容 | 環境 |
|---|---|---|
| F-S1-P2a-1 | 純ロジック（命令ホワイトリスト検査・トピック名組み立て・`fleet_status` の畳み込み・`all` 展開）と pytest。ホワイトリストを外すと落ちることをミューテーションで確認 | @any |
| F-S1-P2a-2 | `ros_bridge.py`（rclpy 配線、`MultiThreadedExecutor` を別スレッド） | @any |
| F-S1-P2a-3 | sim01 で `fleet_stop("all")`・`fleet_status` の E2E | @sim |
| F-S1-P2a-4 | 実機で `fleet_stop` が効くことの確認（F-3-2・F-4-6b の後） | @jetson |

## 10.9 前提の訂正: クライアントはローカル LLM が主（2026-11-14、ユーザー指示）

**インターネット無しで動くロボが目標。** MCP を呼ぶ側（クライアント LLM）はローカルで動くものを主とし、Claude Desktop / Claude Code は「つなげば使える」任意のおまけとする。§10.1・§10.4 の図にある「Claude Desktop」は例であって必須ではない。

| 観点 | 影響 | 対応 |
|---|---|---|
| ネット接続 | サーバーは PC 上の stdio 子プロセスで、LAN の DDS だけを使う。**外部通信は発生しない**（S6 と整合） | 変更なし。オフライン動作を受け入れ条件に加える（下記） |
| 小さいモデルは道具の呼び出しが不安定 | Jetson の Qwen2.5 1.5B は tool calling の安定性が未検証（台帳 F-S3）。PC 側なら大きいモデルを載せられる | **tool を少なく・引数を単純に**: P2a の 3 本（`fleet_status` / `fleet_search` / `fleet_stop`）から。`robot` と `command` は列挙型（enum）で渡し、自由文字列を避ける |
| 安全 | クライアントが賢くないほど誤呼び出しが増える | **S1〜S6 とホワイトリスト検査は不変**。サーバー側で弾くので、モデルが何を出しても安全側に倒れる。`fleet_stop` は常に通す（S3） |
| `fleet_tell` | ロボ自身の LLM（Jetson）へ話しかける経路は、もともとローカル | 変更なし。クライアント LLM と Jetson の LLM は別物 |
| クラウド API を使う既存機能 | `describe_camera`（`robot_agent.py`）は `GEMINI_API_KEY` を要する | MCP の範囲外。オフライン目標の観点では別途の課題として認識しておく |

### 新しく決める点

| # | 決めること | 状態 |
|---|---|---|
| 7 | **MCP クライアントの置き場所とモデル** | **決定（2026-11-14、ユーザー）: Jetson の Qwen2.5 1.5B**。詳細 → §10.10 |
| 8 | 呼び出しの安定性の合格基準 | 案: 用意した 20 件の依頼文で、正しい tool・引数が選ばれる率を測る（誤りがあっても安全側＝弾かれること自体は別途ミューテーションで確認） |

### 追加の受け入れ条件

- ネットワークを切った状態（WAN 遮断）で、クライアント → `fleet_status` → `fleet_stop("all")` が通る。
- モデルが範囲外の命令・存在しないロボ名を出しても、サーバーが拒否して何も送らない。

**優先順位への影響**: 「ネット無し」が目標なので、F-S3（ローカル LLM の tool calling 検証）は本機能の前提条件へ格上げされる。P2a の実装より先に、候補モデルで 8 の測定を行うのが順序として自然。

## 10.10 決定: クライアントは Jetson の Qwen2.5 1.5B（2026-11-14、ユーザー決定）

### 決まったこと

| 項目 | 決定 |
|---|---|
| クライアント LLM | Jetson 上の Qwen2.5 1.5B（既存の `llm` モード＝`llama-server`、OpenAI 互換 API） |
| ライブラリ | **新規パッケージは不要の見込み**。サーバー側で使っている `mcp` SDK に、クライアント側（`ClientSession` / `stdio_client`）が同梱されている（x86 開発環境の mcp 2.3.0 で import を確認。**Jetson の Python での導入は未確認**）。ユーザーは「入れる」を承認済み |
| サーバーの置き場所 | 主構成は **Jetson 上**（旧 §10.4 の「PC 上」から変更）。Isaac ROS コンテナ内の ROS2 Humble の rclpy を使う。PC 上で動かす構成も引き続き可能（同じコード） |

### 既存実装から分かったこと（設計に効く）

- ロボの LLM エージェント（`robot_agent.py`）は、いわゆる tool calling を使っていない。**「2段判定」**で、JSON モードの1回目の問い合わせで `{"tool": "...", 引数...}` を出させ、既知の tool 名表（`TOOL_BY_NAME`）に無ければ通常会話へ落とす。1.5B で実運用してきた方式。
- **コンテキスト長は 2048 トークン**（`LLAMA_CONTEXT_LEN`）。tool の説明を詰め込めない。

→ MCP クライアントも**同じ2段判定を踏襲**する。MCP の `list_tools` で得た tool を、短いプロンプトに畳んで JSON で選ばせ、サーバー側のスキーマ検査を通ったものだけ `call_tool` する。モデルがスキーマ外を出したら呼ばずに通常会話へ落とす（安全側）。

### 設計上の制約と対応

| 論点 | 内容 | 対応 |
|---|---|---|
| 排他制御 | LLM が載るのは `ACTIVE:LLM` の間だけ。**自分自身を `activate:yolo` 等へ切り替えるとクライアント LLM 自体が解放される**（結果を喋れない） | 既存の `robot_agent.py` と同じ挙動。S5（受付までを返す）に合致。結果の確認は人が `status/activity` を見る |
| `fleet_tell` の宛先 | クライアントが Jetson の LLM なので、自分自身への `fleet_tell` は無意味 | 自分の `robot_id` は宛先から除外。他ロボ・PC カメラが相手 |
| コンテキスト 2048 | tool 説明が長いと本文が入らない | P2a は 3 本。説明は 1 行、引数は列挙型。プロンプトの予算（目安 600 トークン以内）を測定項目に含める |
| メモリ | llama-server に加え、MCP クライアントとサーバーの Python プロセスが増える | メモリ予算ゲート（T-26-1a）への影響を実測する（未測定） |
| ロボ間通信 | 他ロボ・PC への DDS は RMW 混在のまま | F-3-2 が実機検証の前提（§10.5） |

### 新しく決める点

| # | 決めること | 推奨 |
|---|---|---|
| 9 | クライアントの実装場所 | **まず単体スクリプト（`local/fleet_mcp/` 配下）で測定**する。`llm_agent_node` / `RobotAgent` への組み込みは、測定で合格してから。既存ノードの大幅改修は別承認（CLAUDE.md §1.3） |

### 測定（#8 の具体化）

20 件の依頼文（例: 「全部止めて」「ロボ1の状態は？」「マグカップを探して」＋範囲外・曖昧・存在しないロボ名）で、実機のモデルに次を測る。**合格ラインは提案値で、実測の前に決めておく**。

| 指標 | 提案ライン |
|---|---|
| JSON として解釈できる率 | 95% 以上 |
| 正しい tool が選ばれる率 | 90% 以上 |
| 引数がスキーマに合う率 | 90% 以上 |
| 範囲外・曖昧な依頼で**呼ばない**率 | 100%（誤呼び出しは安全側で弾かれるが、そもそも選ばないことを確認） |
| 1 回の判定時間（p95） | 実測してから決める（`status/activity` の応答感に影響） |

### 起票するもの

| ID | 内容 | 環境 |
|---|---|---|
| F-S1-P2a-0a | 測定の道具: プロンプト生成・応答検証・採点の純ロジック＋ 20 件の評価セット＋ pytest（モデル不要の部分） | @any |
| F-S1-P2a-0b | Jetson で `llm` モードを上げ、実モデルで測定。結果で P2a の tool 数・説明を調整 | @jetson |
