# Robotics as MCP — 設計書

> ステータス: 設計のみ（未実装）。`todo/multi_agent_fleet.md` F-S1。
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
Phase 0（本設計書）: 設計のみ。実装なし。 ← イマココ
Phase 1: 案A（ファイル読み取り）で fleet_find_object / fleet_room_summary を実装。
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
