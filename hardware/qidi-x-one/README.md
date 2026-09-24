# QIDI TECH X-One — エージェント向け運用資料

macOS + USB + OctoPrintで初代QIDI X-Oneを動かした実機確認記録です。他OSや別個体ではポート名・温度・向きが同じとは限りません。

## 実機で確認済みの構成

| 項目 | 値 |
|---|---|
| プリンタ | QIDI TECH X-One / X-One2系 |
| 造形範囲 | 150 × 150 × 140 mm、左手前原点 |
| ノズル / フィラメント | 0.4 mm / 1.75 mm |
| USBシリアル | Prolific PL2303、VID:PID `067b:2303` |
| このMacでのポート | `/dev/cu.usbserial-1140`（接続ごとに変わり得る） |
| 通信速度 | 115200 baud |
| ファームウェア応答 | `CBD make it.Date:Dec 27 2016 Time:17:41:12` |
| ホスト | macOS、Python 3.13、OctoPrint 1.11.8 |
| 必須プラグイン | Fix CBD Firmware 0.4.0 |
| スライサー | QIDI Print 6.5.4、X-One2プロファイル |
| OctoPrint URL | `http://127.0.0.1:5001`（ローカル限定） |

プロファイルは [`../../profiles/qidi-x-one-octoprint.profile`](../../profiles/qidi-x-one-octoprint.profile) に保存してある。

## macOSへ初回セットアップ

### 1. USBを確認

プリンタをUSB接続してから確認する。

```sh
system_profiler SPUSBDataType
ls /dev/cu.*
```

`Prolific` / `PL2303` と `/dev/cu.usbserial-*` が見えれば、追加USBドライバは入れない。この実機ではmacOS標準ドライバで動作した。ポート名を資料の値で決め打ちせず、そのPCで検出する。

### 2. OctoPrintを専用環境へ入れる

実機確認済みの組み合わせを使う。OctoPrint 1.11.8はPython 3.14ではなく3.13を使う。

```sh
brew install python@3.13
"$(brew --prefix python@3.13)/bin/python3.13" -m venv "$HOME/.venvs/octoprint"
"$HOME/.venvs/octoprint/bin/python" -m pip install --upgrade pip
"$HOME/.venvs/octoprint/bin/python" -m pip install \
  'OctoPrint==1.11.8' 'OctoPrint-FixCBDFirmware==0.4.0'
"$HOME/.venvs/octoprint/bin/octoprint" --version
```

Fix CBD Firmwareはファームウェアを書き換えない。CBD系ファームウェアの壊れた`M110`、軸指定の`G28`/`M84`、`wait`応答などをOctoPrint側で補正する。この個体には必要。

### 3. ローカル限定で起動

```sh
"$HOME/.venvs/octoprint/bin/octoprint" serve --host 127.0.0.1 --port 5001
```

`http://127.0.0.1:5001` を開き、初回ウィザードでアクセス制御と強い固有パスワードを有効にする。外部へ公開しない。自動接続・自動印刷は無効のままにする。APIキーを使う場合はローカルの権限600のファイルへ置き、Gitへ追加しない。

主な保存先:

- 設定: `~/Library/Application Support/OctoPrint/config.yaml`
- プロファイル: `~/Library/Application Support/OctoPrint/printerProfiles/`
- シリアルログ: `~/Library/Application Support/OctoPrint/logs/serial.log`

常駐させる場合はmacOSのユーザーLaunchAgentを使い、同じ`--host 127.0.0.1 --port 5001`で起動する。plist内のユーザー名と仮想環境の絶対パスは、そのPCに合わせる。

### 4. OctoPrintのプリンタープロファイル

- Name: `QIDI X-One`
- Form factor: Rectangular
- Origin: Lower left
- Width / Depth / Height: `150 / 150 / 140 mm`
- Heated bed: Yes
- Nozzle: `0.4 mm`
- Extruders: 1
- X/Y speed: 3600 mm/min以下
- Z speed: 200 mm/min以下
- E speed: 300 mm/min以下

接続は検出した`/dev/cu.usbserial-*`、115200 baudを使う。接続後にTerminalから`M115`を送り、`CBD make it...`系の応答とFix CBD Firmwareの動作をログで確認する。

### 5. QIDI Print

[QIDI公式Softwareページ](https://qidi3d.com/pages/software-firmware)から取得し、署名を確認して通常どおりApplicationsへ入れる。機種は **X-One2**、造形範囲150×150×140 mm、ノズル0.4 mm、フィラメント1.75 mmを選ぶ。

QIDI Printから直接シリアル送信せず、G-codeを書き出してOctoPrintへアップロードする。アップロードだけで印刷を開始せず、下記の検査後に選択・開始する。

## 初回の安全な動作確認

各段階の前後で現場の利用者が目視確認する。Webカメラがあればエージェントも補助確認できる。OctoPrint以外のプロセスがシリアルポートを開いていないことも確認する。

1. ヒーター目標がノズル/ベッドとも0℃、状態がOperationalであることを確認。
2. 手、工具、樹脂片を造形室外へ出す。
3. 障害物がない場合だけ全軸Homeする。CBD機では軸別Homeより`G28`を使う。
4. Home後、各軸を低速で2〜5mmずつ動かし、現場の利用者が方向を目視確認する。カメラがあればエージェントも補助確認する。
5. この個体ではZ正方向でベッドがノズルから離れる。緊急退避は低速でZを正へ動かす。
6. ノズル50℃、次に材料温度、ベッドは低温から材料温度へ段階的に上げ、実温度を監視する。
7. 材料温度でフィラメントを5mmずつ低速押出し、ノズル先端から新しい材料が安定して出ることを確認する。
8. 終了時は`M104 S0`, `M140 S0`, `M106 S0`, `M84`。安全温度まで監視する。

`M84`後は人が軸を動かせるため、以前の`M114`座標を信用しない。このCBDファームウェアの`M119`は詳細を返さず`ok`だけだったため、エンドストップ確認手段として頼らない。

## 材料の開始値

スプール表記を最優先する。材質不明なら印刷しない。

| 材料 | ノズル | ベッド | 冷却ファン | 備考 |
|---|---:|---:|---:|---|
| PLA | 200℃ | 60℃ | 有効 | 実機で加熱・押出確認済み |
| ABS | 240℃ | 100℃ | 停止から開始 | 小型ワッシャーを造形確認済み。換気必須 |

古いX-Oneではノズル250℃、ベッド110℃を運用上限とし、それ以上を含む新型QIDI向け高速材料プロファイルを使わない。ABSは臭気と微粒子が出るため換気し、印刷中はカバーを保つ。温度だけで定着を直そうとせず、造形面の清掃、Z高さ、初層速度、材料、反りも確認する。

## G-codeの必須検査

印刷前にファイル全体を解析する。

- X/Y/Zが`0..150 / 0..150 / 0..140 mm`内。
- 温度が材料と旧型X-Oneの上限内。
- `G90`/`G91`と`M82`/`M83`の切替を追跡し、相対移動を絶対座標と誤解しない。
- 障害物のない状態で全軸Homeしてから造形する。
- 押出量がゼロでも過大でもない。
- 終了時は通常ヒーター/ファン停止、Z退避、ヘッド退避、`M84`。
- `M500`, `M502`, `M303`など、依頼されていない永続設定・校正コマンドを拒否する。
- OctoPrintの解析結果でも造形範囲と推定フィラメント量を照合する。

OctoPrintの「100% / Success」はG-code送信完了を示すだけで、物理的な造形成功ではない。完了画像または目視で判定する。

## Webカメラ（任意）

Webカメラは不要。現場の利用者が目視確認できれば印刷できる。ある場合は、接続前、Home前、軸移動後、初層、完了時の状態をエージェントも確認しやすくなる。このMacではAnker PowerConf C300と`imagesnap`で1920×1080静止画を取得できた。

```sh
brew install imagesnap
imagesnap -l
mkdir -p "$HOME/Pictures/OctoPrint"
imagesnap -d 'Anker PowerConf C300' -w 2 \
  "$HOME/Pictures/OctoPrint/check.jpg"
```

使用する場合だけmacOSのカメラ権限を許可する。既定カメラにOBS Virtual Camera等が選ばれることがあるため`-d`で実機名を指定する。ノズル直下がヘッドで隠れない角度に固定し、古い画像ではなく移動直前の画像を見る。映像は補助情報に留め、手や工具の有無を断定できなければ現場の利用者へ確認する。

## gcoordinator（任意）

`gcoordinator`は座標列から直接G-codeを作るライブラリで、通常のSTLスライサーではない。複雑な一般モデルはQIDI Printを優先する。使うなら依存関係を分離するため公式推奨のPython 3.12専用環境にする。

```sh
brew install uv
uv init --python 3.12 my-print
cd my-print
uv add gcoordinator
```

生成後は上記の全検査が必要。今回、単純なABSワッシャーは成功したが、独自のねじれスター経路は途中で造形失敗した。直接経路生成では壁厚、経路順、トラベル、リトラクション、Z-hop、初層、オーバーハングをエージェント側で保証する必要がある。

## 今回得た注意点

- フィラメントが上から挿入口へ入って見えても、ノズルまでロード済みとは限らない。印刷前の短い押出確認を省略しない。
- PLA設定でABSを刷ると定着不良になる。色や外観で材質を推測しない。
- 糸状樹脂が絡んだ状態でHomeや長距離移動をしない。まず画像確認し、必要ならZを2〜5mm離してから短く退避する。
- カメラでヘッドが造形物を隠す場合、進捗率だけで定着成功と判断しない。
- 高温保持は有人・明示依頼時だけ。通常終了と異常終了は必ず冷却する。

## 参考

- [OctoPrint Download & Setup](https://octoprint.org/download/)
- [Fix CBD Firmware](https://plugins.octoprint.org/plugins/fixcbdfirmware/)
- [gcoordinator](https://github.com/tomohiron907/gcoordinator)
