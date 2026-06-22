# コンテナイメージを「使うだけ」の手順（FROM / pull向け）

このドキュメントは、`ubuntu-flutter-container` を **ベースイメージとして利用する**（`FROM` する / pullして使う）方向けの手順です。  
利用の前提（UID/パス/内包パッケージ）は `IMAGE_NOTES.md`、このリポジトリ自体のビルド方法や設計意図は `README.md` を参照してください。

## 想定する使い方

- 下流プロジェクトで `Dockerfile` を用意し、このイメージを `FROM` して使う
- 実行時のUID/GID・`HOME`・`PUB_CACHE` は下流側のcompose（または run オプション）で吸収する

## 下流プロジェクトの最小例（Web / 共通）

### Dockerfile（下流側）

イメージ名は環境に合わせて置き換えてください（ローカルなら `localhost/...`、レジストリなら `ghcr.io/<org>/...` など）。

```Dockerfile
FROM localhost/flutter-base-anyuid:latest
WORKDIR /workspace
CMD ["/bin/bash"]
```

### docker-compose.yml（下流側）

任意UID運用とキャッシュ永続化を、下流側のcomposeで吸収します。

```yaml
version: "3.8"

services:
  app:
    build:
      context: .
      dockerfile: ./Dockerfile
    working_dir: /workspace
    network_mode: host
    tty: true
    stdin_open: true

    user: "${UID:-1000}:${GID:-1000}"

    environment:
      - HOME=/workspace/.home
      - PUB_CACHE=/workspace/.cache/pub-cache

    volumes:
      - .:/workspace:cached
      - ./.cache/home:/workspace/.home
      - ./.cache/pub-cache:/workspace/.cache/pub-cache
```

起動例：

```bash
export UID
export GID="$(id -g)"
podman compose up --build
```

## Web + Android（エミュレータ実行まで）

このリポジトリには、Web向けの軽量イメージとは別に **Android SDK + Emulator を含む派生**として `Dockerfile.android` が入っています。

- Web向け: `Dockerfile`
- Web+Android向け: `Dockerfile.android`

下流プロジェクトで「どちらを `FROM` するか」を分けるのが基本です。

### エミュレータの前提（ホスト側）

- **KVMが使えるLinuxホスト**（`/dev/kvm` が存在すること）
- 実行ユーザーが **`/dev/kvm` にアクセス可能**（多くの環境では `kvm` グループ所属が必要）
- エミュレータGUIが必要なら **X11転送**（環境により設定は異なります）

### エミュレータ起動例（コンテナ内）

```bash
emulator -list-avds
emulator -avd dev
```

ヘッドレス：

```bash
emulator -avd dev -no-window -gpu swiftshader_indirect
```

補足：

- Android SDKは起動時に `/opt/android-sdk.dist` から **`$HOME/.cache/android-sdk` に初期化**されます（`HOME` を永続化していればキャッシュも残ります）。

## Android実端末で検証する（docker/composeを増やさない運用）

USBをコンテナに直結する運用（`/dev/bus/usb` / `privileged`）はホスト依存が強いため、実端末は **ホストの `adb` ＋無線デバッグ（TCP）**に寄せるのが軽いです。

前提：

- 端末とホストが同一ネットワーク（または到達可能）
- ホストで `adb` が使える（例：Linuxなら `android-tools-adb` など）

### 手順（Android 11+ の「ワイヤレス デバッグ」）

```bash
adb pair <phone-ip>:<pair-port>
adb connect <phone-ip>:<connect-port>
adb devices
```

コンテナ内（必要なら）：

```bash
adb devices
flutter devices
flutter run
```

### 旧方式（USBで一度つないでTCPに切替）

```bash
adb tcpip 5555
adb connect <phone-ip>:5555
```
