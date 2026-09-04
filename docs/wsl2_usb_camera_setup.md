# WSL2でUSBカメラ（PC標準/内蔵カメラ含む）を使う手順

> 対象: F-3-1（`multi_agent_fleet.md`、固定PCカメラでのフリート物体発見）で使用する
> デスクPC（WSL2環境）。Jetson実機コンテナとは無関係。
> 検証日: 2026-08-17。最終確認済み手順（`usb_cam`で`/image_raw`が32Hzで
> streamingすることを実機確認済み）。

## 問題

WSL2で`usbipd-win`を使いUSBカメラをアタッチしても、**カーネルに`uvcvideo`
（USB Video Classドライバ）が無いと `/dev/video*` が作られず一切使えない。**

```
$ sudo modprobe uvcvideo
modprobe: FATAL: Module uvcvideo not found in directory /lib/modules/<カーネルバージョン>
```

**2026-08-17時点、Microsoft配布のWSL2カーネルはバージョンによって明暗が分かれる**:

| カーネルバージョン | `uvcvideo` |
|---|---|
| `5.15.167.4-microsoft-standard-WSL2`（このPCの初期状態） | ❌ 一切含まれない |
| `6.18.33.2-microsoft-standard-WSL2`（`wsl --update` 後） | ✅ ロード可能なモジュールとして同梱済み |

古いカーネルのままだと**カスタムカーネルの自前ビルドが必要**（後述・付録）だが、
**まず `wsl --update` を試すだけで直る可能性が高い**。今回もこれで解決した。

## 解決手順（推奨・まずこれを試す）

### 1. WSLを最新化する

Windows PowerShellから:

```powershell
wsl --update
wsl --shutdown
```

`wsl --shutdown` は**実行中の全WSLディストロ・全プロセス（Claude Codeセッション含む）を
道連れにする**。ユーザー自身の判断で好きなタイミングで行うこと。

再度WSLターミナルを開き、バージョンを確認:

```bash
uname -r
find /lib/modules/$(uname -r) -iname "*uvc*"
# kernel/drivers/media/usb/uvc/uvcvideo.ko が出てくれば含まれている
```

### 2. `uvcvideo` をロードする（要sudo、毎回のWSL起動ごとに必要）

```bash
sudo modprobe uvcvideo
lsmod | grep uvc   # uvcvideo が出れば成功
```

### 3. usbipd-winでカメラをWSLへアタッチする

**usbipd-win自体が未インストールの場合**（Windows PowerShell、winget、管理者権限で
ドライバインストールのUACプロンプトが出る）:

```powershell
winget install --id dorssel.usbipd-win -e --accept-source-agreements --accept-package-agreements
```

インストール直後は同一PowerShellセッションのPATHに反映されないことがあるため、
フルパス（`C:\Program Files\usbipd-win\usbipd.exe`）で呼ぶのが確実。

```powershell
usbipd list
# 例: 4-1  30c9:00c5  Integrated Camera, Camera DFU Device   Not shared

usbipd bind --busid 4-1     # 初回のみ、要管理者権限（UACプロンプト）
usbipd attach --wsl --busid 4-1   # WSL起動のたびに必要（bindは永続、attachは都度）
```

`bind`は管理者権限が必要なため、非対話実行の場合は以下のように昇格させる:

```powershell
Start-Process -Verb RunAs -Wait -FilePath 'C:\Program Files\usbipd-win\usbipd.exe' -ArgumentList 'bind','--busid','4-1'
```

WSL側で確認:

```bash
lsusb                  # カメラが見えるか（例: ID 30c9:00c5 ... Integrated Camera）
ls -la /dev/video*      # /dev/video0 等が作られていれば成功
```

### 4. ROS2ノードで実ストリーミング確認

```bash
source /opt/ros/humble/setup.bash
ros2 run usb_cam usb_cam_node_exe --ros-args \
  -p video_device:=/dev/video0 -p image_width:=640 -p image_height:=480 &
ros2 topic hz /image_raw --window 5
# average rate: 30前後 が出れば成功
```

## 既知の注意点

- **`usbipd attach` はWSL起動のたびに再実行が必要。** `bind`（"Shared"状態）は
  Windows再起動を跨いで保持されるが、`attach`（WSL側で実際にUSBバスへ載せる操作）は
  都度必要。カメラを使うbringupスクリプトの手前で `usbipd list` を確認する運用にすること。
- **`uvcvideo` の `modprobe` もWSL起動のたびに必要**（永続化するには `/etc/modules-load.d/`
  に `uvcvideo` を1行追記する方法があるが、sudo操作でありまだ未設定）。
- 今回 `usbipd list` に `usbipd: warning: Unknown USB filter 'nxusbf' may be
  incompatible with this software; 'bind --force' may be required.` という警告が
  出た（このPC固有のUSBフィルタドライバ由来と思われる）。今回は無視して問題なく
  `bind`/`attach` とも成功した。
- `wsl --update` は**実行中の全ディストロを巻き込む再起動を最終的に要求する**
  （更新自体はバックグラウンドで進むが、新カーネルの適用には `wsl --shutdown` が必要）。
  今回の作業中に実際にWSL全体が自動的に再起動し、進行中のClaude Codeセッションが
  一度切断される事象が発生した（ビルド中のカスタムカーネル成果物が失われた）。
  **`wsl --update` を実行したら、それ単体で完結させず、早めに `wsl --shutdown` して
  意図したタイミングで反映させる方が安全**（放置すると予期しないタイミングで
  自動再起動されうる）。

## 付録: `uvcvideo` が全く同梱されていない古いカーネルの場合（カスタムビルド）

`wsl --update` 後も `uvcvideo.ko` が見つからない場合のみ、以下の自前ビルドが必要
（Microsoft公式の既知の回避策）。**今回は`wsl --update`だけで解決したため未検証**
（ビルド自体は開始し順調に進行していたが、WSL自動再起動により成果物を得る前に
中断・破棄した）。

1. ビルド依存: `sudo apt-get install -y build-essential flex bison libssl-dev libelf-dev bc dwarves libncurses-dev git`
2. `git clone --depth 1 --branch linux-msft-wsl-<uname -rの値> https://github.com/microsoft/WSL2-Linux-Kernel.git`
3. `zcat /proc/config.gz > .config` を土台に `./scripts/config --set-val CONFIG_MEDIA_SUPPORT y` 等でUVC関連を有効化し `make olddefconfig`
4. `make -j$(nproc) vmlinux`
5. `vmlinux` を `/mnt/c/Users/<ユーザー名>/wsl-kernel/vmlinux` へ配置
6. `%UserProfile%\.wslconfig` の `[wsl2]` セクションに `kernel=C:\\Users\\<ユーザー名>\\wsl-kernel\\vmlinux` を追記
7. `wsl --shutdown` して反映

## 関連

- 使用先: `todo/multi_agent_fleet.md` F-3-1f（`toyof_robot_subagent_vision`
  パッケージの実カメラE2E確認）
- 参考: usbipd-win公式（https://github.com/dorssel/usbipd-win）
