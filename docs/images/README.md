# docs/images/

README（日本語版・英語版）が参照する画像を置くディレクトリ。ファイルを置くだけで、
リポジトリルートの `README.md` / `README.en.md` に自動的に表示される（README側の編集は不要）。

| ファイル名 | 内容 | 目安 |
|---|---|---|
| `robot_front.jpg` | ロボット実機・正面斜めからの写真 | 明るい場所、背景を片付けて撮影 |
| `robot_side.jpg` | ロボット実機・別角度（側面 or もう一方の斜め） | 上と違う角度にする |
| `grafana_dashboard.png` | Grafanaダッシュボード（Robot Overview） | 実際に稼働中のパネルが見えている状態 |
| `grafana_dashboard_detail.png` | Grafanaダッシュボード（Robot Detail） | 同上 |
| `guard_corridor_vs_sector.png` | GUARD安全ゲート：扇形方式とコリドー方式の幾何比較 | Engineering Challenges「発生源を問わない最終安全ゲート」 |
| `ai_mode_memory_budget.png` | LLM/YOLO排他制御のメモリ使用量概算 | Engineering Challenges「8GBメモリ制約下でのLLM/YOLO排他制御」 |
| `serial_buffer_backlog.png` | シリアル通信RXバッファ滞留の実測推移 | Engineering Challenges「シリアル通信デッドロックの特定と解消」 |
| `tof_blocking_timeline.png` | ToF I2Cブロッキングによるメインループ遅延の概念図 | Engineering Challenges「ToF I2Cブロッキング」 |
| `lidar_intensity_compare.png` | LiDARゴースト方向vs本物反射の反射強度比較 | Engineering Challenges「LiDAR強度によるゴースト解消」 |
| `lidar_scan_geometry.png` / `lidar_multipath_diagram.png` | LiDARスキャンジオメトリ／多重経路の概念図 | `docs/lidar_intensity_ghost_analysis.md` 詳細図 |
| `ekf_weight_occupancy.png` | laser_odomをEKFへ統合した場合の重み占有率 | Engineering Challenges「レーザーオドメトリをEKFへ統合しなかった判断」 |
| `ekf_z_distribution.png` | laser_odom較正のz分布ヒストグラム | `docs/engineering_decisions.md`（Issue-10）詳細図 |
| `llm_crosslingual_latency.png` | 日本語プロンプトと英語プロンプトのLLM判定時間比較 | Engineering Challenges「Crosslingual Prompting」 |

- 横幅 1200px 程度で十分（大きすぎるファイルはリポジトリを重くする）。
- ファイル名はこの表の通りに厳守（README側は固定パスで参照しているため）。
