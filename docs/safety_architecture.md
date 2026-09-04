# Safety Architecture

本ドキュメントはロボットの **安全設計**を説明する。

---

# 1 Safety Philosophy

設計原則

```
Fail Safe
```

通信やソフト異常時は

```
robot stop
```

を行う。

---

# 2 Safety Layers

本ロボットは **多層安全設計**。

```
AI Layer
Control Layer
Motor Layer
```

---

# 3 Layer 1 (ROS Safety)

ノード

```
pico_bridge_node
```

機能

```
cmd_vel watchdog
cmd_vel forward guard（N13-3、2026-07-22）
```

設定

```
watchdog_sec = 0.5
cmd_vel_guard_stop_distance_m = 0.30
cmd_vel_guard_half_angle_rad  = 0.35
cmd_vel_guard_scan_timeout_sec = 1.0
```

処理

```
timeout → L:0,R:0 (UART送信)
```

## 3.1 cmd_vel forward guard（発生源非依存の最終安全層）

`/cmd_vel` へ直接 publish するノードは9箇所（follow_goal_generator / yolo_follow /
yoloworld / robot_agent / person_follower / mapping_lifecycle / apriltag_servo /
tag_localization_manager ＋ Nav2 velocity_smoother ＋ teleop）あり、個々の呼び出し
箇所へ安全チェックを足すだけでは「どこか1つが暴走すれば同じ事故が再発する」構造が
残る（実例: T-AT-6-21 AMCL共分散発散インシデント）。

モーターへの唯一の出口である `pico_bridge_node`（sim では `pico_stub_node`）の
`cb_cmd_vel` で、発生源を問わず前進成分（`linear.x > 0`）のみをゲートする。
`/scan_body_filtered`（自車体のみ除去・追従対象は残すスキャン、詳細 →
`docs/robot_architecture.md` §3）の前方コーンに
障害物があれば `linear.x=0` に落とす。**後退・その場旋回は常に通す**
（全成分を止めると壁の前で脱出不能になるため）。ロジックは `cmd_vel_guard_logic.py`
（ROS2非依存、pytest 17件）。配線は publisher 側の変更ゼロで、`pico_bridge_node`が
1トピック購読するだけ。詳細 → `todo/navigation.md` N13-3。

---

# 4 Layer 2 (MCU Safety)

Pico側

```
WD_MS = 500
```

条件

```
command timeout
```

処理

```
stop_all()
```

---

# 5 Hardware Stop

PWM停止

```
duty = 0
```

結果

```
motor stop
```

---

# 6 Failure Scenarios

| Failure           | 対応            |
| ----------------- | ------------- |
| ROS crash         | watchdog stop |
| serial disconnect | pico watchdog |
| invalid command   | ignore        |

---

# 7 Redundant Stop Mechanism

停止は **2系統**

```
ROS watchdog
+
MCU watchdog
```

---
