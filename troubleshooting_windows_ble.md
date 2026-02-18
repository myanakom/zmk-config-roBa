# roBa キーボード Windows Bluetooth 接続トラブルシューティング

## 作成日: 2026-02-17

## 目次

1. [問題の概要](#問題の概要)
2. [環境情報](#環境情報)
3. [症状の詳細](#症状の詳細)
4. [調査で判明した原因候補](#調査で判明した原因候補)
5. [設定ファイルの変更履歴と影響分析](#設定ファイルの変更履歴と影響分析)
6. [解決手順](#解決手順)
7. [各設定項目の解説](#各設定項目の解説)
8. [関連する ZMK GitHub Issues](#関連する-zmk-github-issues)
9. [参考ドキュメント](#参考ドキュメント)

---

## 問題の概要

roBa 分割キーボードを Windows 11 PC に Bluetooth で接続しようとすると、デバイスは認識されるが「もう一度デバイスを接続してください」エラーが表示され、ペアリングが完了しない。

---

## 環境情報

### ハードウェア

| 項目 | 内容 |
|------|------|
| キーボード | roBa（分割キーボード） |
| MCU | Seeed XIAO BLE (nRF52840) × 2 |
| 右手側 | Central（BLE 接続管理側）+ PMW3610 トラックボール |
| 左手側 | Peripheral（右手側に BLE 接続する従属側） |
| トラックボール | PMW3610（SPI 接続、400 CPI） |
| エンコーダ | EC11（左手側） |

### ファームウェア

| 項目 | 内容 |
|------|------|
| ファームウェア | ZMK (main ブランチ) |
| ビルド方式 | GitHub Actions (`zmkfirmware/zmk/.github/workflows/build-user-config.yml@main`) |
| ビルドターゲット | `xiao_ble` + `roBa_R`（右）、`xiao_ble` + `roBa_L`（左） |
| ZMK Studio | 有効（`CONFIG_ZMK_STUDIO=y`） |
| Pointing | 有効（`CONFIG_ZMK_POINTING=y`） |

### 接続先デバイス

| デバイス | OS | 接続結果 |
|----------|-----|---------|
| Windows PC (1台目) | Windows 11 | **失敗** - BT1 割り当て |
| Windows PC (2台目) | Windows 11 | **失敗** - BT0 割り当て |
| Raspberry Pi | Linux | 成功 |
| Android スマートフォン | Android | 成功 |

---

## 症状の詳細

1. Windows の Bluetooth 設定画面でキーボード「roBa」がデバイスとして認識・表示される
2. 接続ボタンを押すと「もう一度デバイスを接続してください」というエラーが出る
3. キーボード側で BT_CLR（ボンドクリア）を実行済みだが改善しない
4. 複数の Windows PC で同じ症状が再現する
5. Linux（Raspberry Pi）や Android には問題なく接続できる
6. **過去には Windows に接続できていたが、キー配列変更後から繋がらなくなった**

---

## 調査で判明した原因候補

### 原因1（最有力）: HID ディスクリプタのキャッシュ不整合

#### 概要

ZMK で `CONFIG_ZMK_POINTING=y` を有効にすると、キーボードの HID レポートディスクリプタにマウス HID レポート（Report ID 0x03）が追加される。このディスクリプタには以下が含まれる:

- マウスボタン 5個
- 16bit X/Y 相対移動
- 16bit 垂直スクロールホイール
- 16bit 水平パン（AC_PAN）

**Windows は BLE デバイスの HID ディスクリプタを積極的にキャッシュする。** ファームウェアを変更しても Windows 側のキャッシュが古い状態のまま残り、新しいディスクリプタとの不一致で接続が失敗する。Linux や Android はディスクリプタの変更に柔軟に対応するため問題が起きない。

#### 本件での具体的な変更点

PMW3610 ドライバを旧カスタムモジュールから公式 ZMK ドライバに切り替えたことで、HID ディスクリプタの構造が変わった:

**旧構成（接続できていた時）:**
- `kumamuk-git/zmk-pmw3610-driver` カスタムモジュール使用
- `CONFIG_PMW3610=y` + 多数の `CONFIG_PMW3610_*` Kconfig 設定
- Device tree: `irq-gpios`, `automouse-layer = <4>`, `scroll-layers = <5>`

**新構成（接続できなくなった後）:**
- ZMK 公式組み込みドライバ使用
- `CONFIG_INPUT_PMW3610=y`
- Device tree: `motion-gpios`, `zephyr,axis-x`, `zephyr,axis-y`, `smart-mode`, `res-cpi`, `invert-x`

#### ZMK 公式ドキュメントの記載

> "Enabling certain features or behaviors in ZMK changes the data structure that ZMK sends over USB or BLE to host devices. This in turn requires HID report descriptors to be modified for the reports to be parsed correctly."
>
> "While the descriptor refresh happens on boot for USB, hosts will frequently cache this descriptor for BLE devices. In order to refresh this cache, you need to remove the keyboard from the host device, clear the profile associated with the host on the keyboard, then pair again."

出典: https://zmk.dev/docs/features/bluetooth

---

### 原因2（中程度の可能性）: ZMK Studio BLE GATT サービスの干渉

#### 概要

`CONFIG_ZMK_STUDIO=y` と `CONFIG_ZMK_BLE=y` が同時に有効な場合、ZMK は自動的に `CONFIG_ZMK_STUDIO_TRANSPORT_BLE=y` を有効化し、カスタム BLE GATT サービスを追加する。

これにより、キーボードは以下の BLE サービスをアドバタイズする:

1. HID サービス（キーボード + マウスのコンポジットデバイス）
2. バッテリーサービス
3. デバイス情報サービス
4. **ZMK Studio カスタム GATT サービス**（暗号化要求あり）

Windows は BLE の HID ペリフェラルに予期しない GATT サービスがある場合にサービス探索やペアリングで問題を起こすことが知られている。

#### 該当する設定箇所

```
# config/boards/shields/roBa/roBa_R.conf
CONFIG_ZMK_STUDIO=y            # Studio 有効化
CONFIG_ZMK_STUDIO_LOCKING=n    # ロック無効
# CONFIG_ZMK_STUDIO_TRANSPORT_BLE は暗黙的に y になる
```

`build.yaml` の `snippet: studio-rpc-usb-uart` は USB UART トランスポートの設定のみで、BLE トランスポートは別途自動的に有効化されている点に注意。

---

### 原因3（中程度の可能性）: BLE PHY 2Mbps の互換性問題

#### 概要

nRF52840 はデフォルトで BLE 2Mbps PHY をサポートしている。しかし、Windows PC に搭載されている一部の Bluetooth チップ（特に Realtek や Intel 製）は、2Mbps PHY モードとの互換性に問題がある。

ZMK のトラブルシューティングドキュメントには以下の記載がある:

> "Disabling PHY 2Mbps (CONFIG_BT_CTLR_PHY_2M=n) helps to pair and connect for certain wireless chipset firmware versions, particularly on Windows (Realtek and Intel chips)."

出典: https://zmk.dev/docs/troubleshooting/connection-issues

---

### 原因4（参考）: ZMK main ブランチの BLE リグレッション

ZMK の `main` ブランチは Zephyr 4.1.0 アップグレード後に BLE 関連のリグレッションが報告されている（[Issue #3207](https://github.com/zmkfirmware/zmk/issues/3207)）。スプリットキーボードのセントラル側がディープスリープから復帰できない問題などが確認されている。

現在の `west.yml` では `revision: main` を使用しており、不安定なコミットが含まれている可能性がある。

---

## 設定ファイルの変更履歴と影響分析

### Git コミット履歴（新しい順）

| コミット | 日付 | 内容 | BLE への影響 |
|---------|------|------|-------------|
| `b6986a7` | 2026-02-08 | UF2 ファイル削除 | なし |
| `dd74f75` | 2026-02-07 | トラックボール垂直方向反転（`invert-x` 追加） | なし |
| `dc1c81f` | 2026-02-07 | コード構造リファクタリング | なし |
| `8433842` | 2026-02-07 | JP レイアウトの `< >` 出力修正 | なし |
| `e4a2437` | 2026-02-07 | PMW3610 CPI とスマートモード調整 | 間接的 |
| `1bf61ad` | - | トラックボール軸方向復元（X/Y スワップ） | 間接的 |
| `716734e` | - | PMW3610 設定を Zephyr バインディングに統一 | **あり** - HID ディスクリプタ変更 |
| `3f8f5a7` | - | PMW3610 DT プロパティ修正 | **あり** |
| `b21cb9f` | - | PMW3610 カスタムモジュール削除 | **あり** - ドライバ変更 |
| `878c6c9` | - | デフォルトレイアウトのキーラベル追加、エンコーダ設定更新 | なし |
| `cd07069` | - | ボード名を `seeed_xiao_ble` → `xiao_ble` に変更 | 間接的 |

### 主要な設定変更の差分

#### `roBa_R.conf` の変更

```diff
# 削除された設定（旧カスタムドライバ）
- CONFIG_PMW3610=y
- CONFIG_PMW3610_CPI=400
- CONFIG_PMW3610_CPI_DIVIDOR=1
- CONFIG_PMW3610_ORIENTATION_180=y
- CONFIG_PMW3610_SCROLL_TICK=16
- CONFIG_PMW3610_INVERT_X=n
- CONFIG_PMW3610_INVERT_SCROLL_X=y
- CONFIG_PMW3610_RUN_DOWNSHIFT_TIME_MS=3264
- CONFIG_PMW3610_REST1_SAMPLE_TIME_MS=20
- CONFIG_PMW3610_POLLING_RATE_125_SW=y
- CONFIG_PMW3610_AUTOMOUSE_TIMEOUT_MS=700
- CONFIG_PMW3610_MOVEMENT_THRESHOLD=0
- CONFIG_PMW3610_SMART_ALGORITHM=y

# 追加された設定（公式ドライバ）
+ CONFIG_INPUT_PMW3610=y
```

#### `roBa_R.overlay` の変更

```diff
  trackball: trackball@0 {
      compatible = "pixart,pmw3610";
      reg = <0>;
      spi-max-frequency = <2000000>;
-     irq-gpios = <&gpio0 2 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;
-     automouse-layer = <4>;
-     scroll-layers = <5>;
+     motion-gpios = <&gpio0 2 (GPIO_ACTIVE_LOW | GPIO_PULL_UP)>;
+     zephyr,axis-x = <INPUT_REL_Y>;
+     zephyr,axis-y = <INPUT_REL_X>;
+     res-cpi = <400>;
+     smart-mode;
+     invert-x;
  };
```

#### `west.yml` の変更

```diff
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: main
      import: app/west.yml
-   - name: zmk-pmw3610-driver
-     remote: kumamuk-git
-     revision: main
```

---

## 解決手順

### ステップ1: ボンド情報の完全クリア（必須・最初に実施）

#### キーボード側の操作

1. Layer3（`mo 3` で到達）で対象の BT プロファイルに切り替える
   - `BT_SEL 0`（BT0）または `BT_SEL 1`（BT1）
2. `BT_CLR` を押してそのプロファイルのボンド情報をクリアする
3. 全プロファイルをクリアしたい場合は `BT_CLR_ALL` を使用

#### Windows 側の操作

1. **設定 > Bluetooth とデバイス** で「roBa」デバイスを削除
2. **デバイスマネージャー** を開く
   - 「表示」メニュー > 「非表示のデバイスの表示」を有効にする
   - 「Bluetooth」カテゴリ配下に残っている「roBa」関連のエントリをすべて削除
3. Windows の Bluetooth をオフにして、数秒待ってからオンに戻す
4. キーボードの対象プロファイルを選択した状態で、Windows から再ペアリングする

> **重要:** キーボード側だけでなく、Windows 側のデバイス削除とデバイスマネージャーのクリーンアップも必ず行うこと。Windows は HID ディスクリプタのキャッシュをデバイスマネージャーレベルで保持している。

---

### ステップ2: BLE 実験的接続設定の追加

`config/boards/shields/roBa/roBa_R.conf` に以下を追加:

```kconfig
CONFIG_ZMK_BLE_EXPERIMENTAL_CONN=y
```

この設定により以下が有効になる:
- `CONFIG_BT_CTLR_PHY_2M=n`（2Mbps PHY の無効化）

ファームウェアを再ビルドして書き込み、ステップ1のボンドクリアを再度実施してからペアリング。

---

### ステップ3: ZMK Studio の BLE トランスポート無効化

ステップ2で解決しない場合、`roBa_R.conf` にさらに追加:

```kconfig
CONFIG_ZMK_STUDIO_TRANSPORT_BLE=n
```

これにより:
- ZMK Studio のカスタム GATT サービスが BLE アドバタイズから除去される
- USB 経由の Studio 接続（`studio-rpc-usb-uart` スニペット）は引き続き使用可能
- キーボードの BLE サービスがシンプルになり、Windows との互換性が向上する可能性がある

---

### ステップ4: 追加の Windows 互換性設定

ステップ3まででも解決しない場合、`roBa_R.conf` にさらに追加:

```kconfig
CONFIG_BT_GATT_ENFORCE_SUBSCRIPTION=n
CONFIG_BT_CTLR_TX_PWR_PLUS_8=y
```

- `CONFIG_BT_GATT_ENFORCE_SUBSCRIPTION=n`: Windows の GATT サブスクリプション処理のバグを回避
- `CONFIG_BT_CTLR_TX_PWR_PLUS_8=y`: 無線送信出力を最大（+8dBm）に設定

---

### ステップ5: Pointing 機能の一時無効化（診断用）

問題の切り分けとして、`roBa_R.conf` から一時的に以下を変更:

```kconfig
# CONFIG_ZMK_POINTING=y    ← コメントアウト
# CONFIG_INPUT_PMW3610=y   ← コメントアウト
```

これでキーボードのみ（マウス HID レポートなし）のファームウェアをビルドし、Windows に接続できるか確認する。接続できた場合、マウスのコンポジット HID ディスクリプタが Windows に拒否されていることが確定する。

---

### ステップ6: settings_reset ファームウェアの書き込み

上記すべてで解決しない場合:

1. GitHub Actions でビルドされた `settings_reset-xiao_ble-zmk.uf2` を入手
2. **左右両方**の XIAO BLE にこの UF2 を書き込む（ブートローダーモードで接続してコピー）
3. すべての保存済み BLE ボンド情報とデバイス設定が消去される
4. 通常のファームウェア（`roBa_R` / `roBa_L`）を再度書き込む
5. Windows 側もデバイスを完全に削除してから新規ペアリング

---

### ステップ7: ZMK のリビジョン固定

`main` ブランチの不安定性が原因の可能性がある場合、`config/west.yml` で安定したコミットにピン止め:

```yaml
manifest:
  remotes:
    - name: zmkfirmware
      url-base: https://github.com/zmkfirmware
  projects:
    - name: zmk
      remote: zmkfirmware
      revision: <安定したコミットハッシュ>
      import: app/west.yml
  self:
    path: config
```

> Issue #2957（スムーズスクロールの BLE バグ）の修正が PR #2998 で 2025年7月20日にマージされているため、それ以降かつ Zephyr 4.1.0 アップグレード（2025年12月頃）より前のコミットが安定している可能性が高い。

---

## 各設定項目の解説

### 現在の `roBa_R.conf` の全項目

```kconfig
CONFIG_ZMK_KEYBOARD_NAME="roBa"
# キーボードの BLE アドバタイズ名。Windows のデバイス一覧に表示される名前。

CONFIG_ZMK_POINTING=y
# ZMK の Pointing デバイスサポートを有効化。
# マウス HID レポートディスクリプタが追加される。

CONFIG_NFCT_PINS_AS_GPIOS=y
# nRF52840 の NFC ピンを通常の GPIO として使用。
# NFC 機能を使わない場合に GPIO ピンを確保するための設定。

CONFIG_ZMK_BLE=y
# BLE 機能を有効化。スプリットキーボードの通信とホストへの接続に必須。

CONFIG_SPI=y
# SPI バスを有効化。PMW3610 トラックボールとの通信に使用。

CONFIG_INPUT=y
# Zephyr の入力サブシステムを有効化。ポインティングデバイスに必要。

CONFIG_INPUT_PMW3610=y
# ZMK 公式の PMW3610 ドライバを有効化。
# 旧 kumamuk-git/zmk-pmw3610-driver に代わる公式実装。

CONFIG_EC11=y
# EC11 ロータリーエンコーダのドライバを有効化。

CONFIG_EC11_TRIGGER_GLOBAL_THREAD=y
# エンコーダの割り込みをグローバルスレッドで処理。

CONFIG_ZMK_BATTERY_REPORTING=y
# バッテリー残量レポートを有効化。BLE 経由でホストにバッテリー情報を送信。

CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_PROXY=y
# セントラル（右手）がペリフェラル（左手）のバッテリー残量を代理送信。

CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING=y
# セントラルがペリフェラルのバッテリー残量を取得する機能を有効化。

CONFIG_ZMK_STUDIO=y
# ZMK Studio（ランタイムキーマップ変更ツール）を有効化。
# 注意: CONFIG_ZMK_BLE=y と同時に有効の場合、
# CONFIG_ZMK_STUDIO_TRANSPORT_BLE=y が暗黙的に有効になる。

CONFIG_ZMK_STUDIO_LOCKING=n
# Studio のロック機能を無効化。物理キーによるロック解除が不要になる。
```

### トラブルシューティング用に追加可能な設定

```kconfig
CONFIG_ZMK_BLE_EXPERIMENTAL_CONN=y
# BLE の実験的接続パラメータを使用。
# CONFIG_BT_CTLR_PHY_2M=n を含み、2Mbps PHY を無効化する。
# Realtek / Intel チップとの互換性問題を解決することが多い。

CONFIG_ZMK_BLE_EXPERIMENTAL_FEATURES=y
# さらに踏み込んだ実験的 BLE 機能を有効化。
# Secure Connection パスキーエントリやキー上書きを含む。
# CONFIG_ZMK_BLE_EXPERIMENTAL_CONN の上位互換。

CONFIG_ZMK_STUDIO_TRANSPORT_BLE=n
# Studio の BLE トランスポートを明示的に無効化。
# カスタム GATT サービスが BLE から除去される。

CONFIG_BT_GATT_ENFORCE_SUBSCRIPTION=n
# GATT サブスクリプション強制を無効化。
# Windows の BLE GATT 実装のバグを回避。

CONFIG_BT_CTLR_TX_PWR_PLUS_8=y
# BLE の送信出力を +8dBm（最大）に設定。
# 接続安定性の向上に寄与。
```

---

## 関連する ZMK GitHub Issues

| Issue | タイトル | ステータス | 関連度 | 概要 |
|-------|---------|-----------|--------|------|
| [#805](https://github.com/zmkfirmware/zmk/issues/805) | Bluetooth security failing on Windows | Open | 高 | Windows で BLE セキュリティが失敗、他 OS では動作 |
| [#2063](https://github.com/zmkfirmware/zmk/issues/2063) | Realtek RTL8852AE Bluetooth connectivity | Open | 高 | `EXPERIMENTAL_CONN` で解決 |
| [#2061](https://github.com/zmkfirmware/zmk/issues/2061) | BT 5.3 adapter connection failure | Open | 高 | `EXPERIMENTAL_CONN` で解決 |
| [#2245](https://github.com/zmkfirmware/zmk/issues/2245) | Reconnect between multiple Windows PCs | Open | 高 | `EXPERIMENTAL_FEATURES` で解決 |
| [#2826](https://github.com/zmkfirmware/zmk/issues/2826) | Keyboard not working after deep sleeps on Win11 | Open | 中 | Windows 11 固有の BLE 問題 |
| [#2941](https://github.com/zmkfirmware/zmk/issues/2941) | Mouse not working with XIAO BLE | Closed | 中 | HID ディスクリプタのリフレッシュが必要 |
| [#2957](https://github.com/zmkfirmware/zmk/issues/2957) | Smooth scrolling broken over BLE | Closed | 中 | HID フィーチャーレポートバグ修正済み |
| [#2798](https://github.com/zmkfirmware/zmk/issues/2798) | Mouse clicks stopped working | Closed | 中 | リビルド後に再ペアリングで解決 |
| [#392](https://github.com/zmkfirmware/zmk/issues/392) | Battery level not updating on Windows | Closed | 低 | `GATT_ENFORCE_SUBSCRIPTION=n` で解決 |
| [#3207](https://github.com/zmkfirmware/zmk/issues/3207) | Central fails to wake from deep sleep | Open | 参考 | main ブランチの BLE リグレッション |

---

## 参考ドキュメント

- ZMK Bluetooth トラブルシューティング: https://zmk.dev/docs/troubleshooting/connection-issues
- ZMK Bluetooth 設定リファレンス: https://zmk.dev/docs/config/bluetooth
- ZMK Pointing デバイスドキュメント: https://zmk.dev/docs/features/pointing
- ZMK Pointing 設定リファレンス: https://zmk.dev/docs/config/pointing
- ZMK Studio ドキュメント: https://zmk.dev/docs/features/studio
- ZMK HID ディスクリプタリフレッシュ: https://zmk.dev/docs/features/bluetooth （「Refreshing the HID Descriptor」セクション）
- ZMK Bluetooth 機能概要: https://zmk.dev/docs/features/bluetooth

---

## キーボードのレイヤー構成（BT 操作の参考）

BT 操作は **Layer3**（左手の `mo 3` キーを押しながら）で行う。

### Layer3 のキー配置（BT 関連部分）

```
行2: BT_CLR | BT_NXT | BT_PRV | BT_CLR_ALL    BT_SEL 0 | BT_SEL 1 | BT_SEL 2 | BT_SEL 3 | BT_SEL 4
行3:                                             BT_DISC 0| BT_DISC 1| BT_DISC 2| BT_DISC 3| BT_DISC 4
```

### Layer6（NUM レイヤー）の BT 関連キー

```
行1:                                             BT_SEL 0 | BT_SEL 1 | BT_SEL 2 | BT_SEL 3 | BT_SEL 4
行3右端:                                                                                       BT_CLR
行4右端:                                                                                       BT_CLR_ALL
```

---

## ファイル構成

```
zmk-config-roBa/
├── config/
│   ├── boards/shields/roBa/
│   │   ├── Kconfig.defconfig      # スプリット設定（右=Central, 左=Peripheral）
│   │   ├── Kconfig.shield         # シールド定義
│   │   ├── roBa.dtsi              # 共通デバイスツリー（キーマトリクス、エンコーダ）
│   │   ├── roBa.zmk.yml           # シールドメタデータ
│   │   ├── roBa_L.conf            # 左手側設定
│   │   ├── roBa_L.overlay         # 左手側デバイスツリー
│   │   ├── roBa_L.zmk.yml         # 左手側メタデータ
│   │   ├── roBa_R.conf            # 右手側設定（★BLE 設定の中心）
│   │   ├── roBa_R.overlay         # 右手側デバイスツリー（★トラックボール設定）
│   │   └── roBa_R.zmk.yml         # 右手側メタデータ
│   ├── roBa.keymap                # キーマップ定義（7レイヤー）
│   ├── roBa.json                  # レイアウト JSON
│   └── west.yml                   # 依存関係管理（ZMK リビジョン指定）
├── build.yaml                     # GitHub Actions ビルドマトリクス
└── zephyr/
    └── module.yml                 # Zephyr モジュール設定
```
