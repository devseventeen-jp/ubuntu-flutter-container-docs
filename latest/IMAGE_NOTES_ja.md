# 利用の前提・仕様（内包パッケージ / UID / パス）

このドキュメントは、このリポジトリで作るFlutter開発用イメージの **前提・仕様**をまとめたものです。  
「pull/FROMして使う手順」は `USAGE.md`、ビルド/保守の詳細は `README.md` を参照してください。

## ベースイメージとユーザー

- ベース: `mcr.microsoft.com/devcontainers/base:ubuntu`
- 既定ユーザー: `vscode`
- 推奨運用: **任意UID**で実行（compose/runで `user: <uid>:<gid>` を指定）
  - ホストのUID/GIDとずれると、bind mount先で権限問題が起きやすいです

## 主要パスと権限

- Flutter SDK: `FLUTTER_HOME=/opt/flutter`
  - SDK本体は基本 **読み取り専用**（`root:root` / `a+rX`）
  - 例外: `/opt/flutter/bin/cache` は任意UIDでも更新できるよう `1777`
- Pub cache:
  - イメージ既定: `PUB_CACHE=/tmp/.pub-cache`（`1777`）
  - 推奨: 実運用では `PUB_CACHE` をプロジェクト配下の永続volumeへ（例：`/workspace/.cache/pub-cache`）
- HOME:
  - 推奨: `HOME` をプロジェクト配下の永続volumeへ（例：`/workspace/.home`）

## Android同梱イメージ（`Dockerfile.android`）

### 構成の比較

| 項目 | Webイメージ | Androidイメージ |
| --- | --- | --- |
| Flutter SDK | 含む | 含む |
| Webサポート | ビルド引数で有効化 | ビルド引数で有効化 |
| Android SDK | 含まない | 含む |
| Emulator | 含まない | 含む |
| 主な用途 | Flutter Web開発 | Flutter Web開発 + Androidエミュレータ/実機検証 |

### 実行時の挙動

| 項目 | 内容 |
| --- | --- |
| Android SDK配布元 | `/opt/android-sdk.dist` |
| ランタイムSDKキャッシュ | 起動時に `/opt/android-sdk.dist` を `$HOME/.cache/android-sdk` へ初期化 |
| 環境変数 | `ANDROID_SDK_ROOT` / `ANDROID_HOME` / `PATH` は `ENTRYPOINT` で上書きされる |
| 既定AVD | `dev` をビルド時に作成 |

## 内包パッケージ（詳細）

### Webイメージ（`Dockerfile`）

| パッケージ | 用途 |
| --- | --- |
| `curl` | Flutter取得や外部ダウンロード |
| `git` | Flutter SDKのclone、`git config --global` |
| `unzip` | Flutter周辺アーカイブ展開 |
| `xz-utils` | 圧縮アーカイブ関連の補助 |
| `zip` | 圧縮アーカイブ関連の補助 |
| `libglu1-mesa` | Flutter/GUI系の実行補助 |
| `openjdk-17-jdk` | Flutter/Android系ツールチェーンのJava実行環境 |

### Androidイメージ（`Dockerfile.android`）

| パッケージ | 用途 |
| --- | --- |
| `ca-certificates` | HTTPS証明書の検証 |
| `curl` | Flutter取得やAndroid cmdline-toolsのダウンロード |
| `git` | Flutter SDKのclone、`git config --global` |
| `unzip` | FlutterおよびAndroid cmdline-toolsの展開 |
| `xz-utils` | 圧縮アーカイブ関連の補助 |
| `zip` | 圧縮アーカイブ関連の補助 |
| `libglu1-mesa` | Flutter/GUI系の実行補助 |
| `openjdk-17-jdk` | Android SDK/FlutterのJava実行環境 |
| `libnss3` | Android Emulator実行時の依存 |
| `libx11-6` | Android Emulator実行時のX11依存 |
| `libxcomposite1` | Android Emulator実行時の描画依存 |
| `libxcursor1` | Android Emulator実行時の描画依存 |
| `libxdamage1` | Android Emulator実行時の描画依存 |
| `libxext6` | Android Emulator実行時のX11拡張依存 |
| `libxfixes3` | Android Emulator実行時の描画依存 |
| `libxi6` | Android Emulator実行時の入力系依存 |
| `libxrandr2` | Android Emulator実行時の画面調整依存 |
| `libxrender1` | Android Emulator実行時の描画依存 |
| `libxtst6` | Android Emulator実行時の入力/テスト系依存 |
| `libpulse0` | Android Emulator実行時の音声依存 |
| `libdrm2` | Android Emulator実行時の描画依存 |
| `libxkbcommon0` | Android Emulator実行時のキーボード依存 |
| `libasound2t64` または `libasound2` | Android Emulator実行時の音声依存。`libasound2t64` があればそちら、なければ `libasound2` を入れる |

### Android側の追加コンポーネント（パッケージ以外）

| コンポーネント | 内容 |
| --- | --- |
| Android cmdline-tools | `sdkmanager` / `avdmanager` を含むツール群 |
| Android platform-tools | `adb` など |
| Android platform | `platforms;android-34` |
| Android build-tools | `build-tools;34.0.0` |
| Android emulator | `emulator` パッケージ |
| Android system image | `system-images;android-34;google_apis;x86_64` |

## 互換性・制約メモ

- Rootless実行を前提にしています（Podman想定）。
- エミュレータのHWアクセラレーションはホスト依存です（KVMが必要）。
- 実端末USB直結はホスト依存が強いので、基本は「ホストadb＋無線デバッグ（TCP）」運用を推奨します。
