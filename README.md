# mlpoc — Autoware ML モデル取得/配置/切替ツール（PoC）

**mlpoc** は、`webauto` CLI から ML モデルのリリースを取得し、
`/opt/autoware/model-store/<model>/<target>/<version>/` に配置、
運用参照を `/opt/autoware/mlmodels/<model>` の **シンボリックリンク**で切り替えるための PoC ツールです。

* 目的：モデルの **バージョン/ターゲット** を安全かつ素早く切替可能にする
* 前提：インフラ側の DL は既存の `webauto` CLI を利用（このツールは配置と参照切替に専念）
* 既定のプロジェクト ID は **`prd_jt`**（環境変数や引数で変更可能）

---

## Quick start

もとの`/opt/autoware/mlmodels` に内容物がある場合必要に応じてバックアップしてください。

```bash
# preparation
sudo chmod +x mlpoc
sudo ln -sf "$(pwd)/mlpoc" /usr/local/bin/mlpoc
# test
mlpoc --help
```


```bash
mlpoc models
mlpoc releases centerpoint
sudo mlpoc pull --model centerpoint --latest --target xx1 # or --pick
sudo mlpoc pull --model centerpoint --latest --target x2
mlpoc list centerpoint
sudo mlpoc switch centerpoint --latest --target xx1  # or --pick
mlpoc current centerpoint
```

完了すれば下記のような表示になるはずです。
```bash
❯ mlpoc current centerpoint
/opt/autoware/model-store/centerpoint/xx1/2.3.1
```

---

## 想定する動作フロー

1. **リモートの候補を調査**：`mlpoc models` / `mlpoc releases <model>`（webauto 経由）
2. **必要なリリースを取得（pull）**：

   * `mlpoc pull` で model/target/version に従い **原子的に配置**
   * `--pick` で webauto 上のリリース候補を対話選択（`--target` でフィルタ可能）
   * 配布物に余計なネスト（`<uuid>/centerpoint` など）があっても **payload ルート検出**で直下にフラット展開
3. **運用参照の切替（switch）**：

   * `/opt/autoware/mlmodels/<model>` → 対応する store パスへ **原子的にリンク置換**
   * 既存実体から **`--target/--version` 指定**または \*\*対話選択（`--pick`）\*\*で切替

---

## ディレクトリ構成

```
/opt/autoware/
  model-store/
    <model>/<target>/<version>/        # 実体（.onnx, *.yaml などが直下に）
  mlmodels/
    <model> -> /opt/autoware/model-store/<model>/<target>/<version>  # 運用リンク
```

> *PoC では配置時に `.onnx` が最低1つあることを検証。必要に応じて manifest/SHA 検証を拡張可能です。*

---

## `webauto` CLI の背景知識

本ツールは以下の `webauto` サブコマンドを裏で呼びます。

* **モデル一覧**：

  ```bash
  webauto ml package list --project-id <PID> --output json
  ```

  * 主に `id`（package\_id）と `name`（モデル名）を利用。

* **リリース検索**：

  ```bash
  webauto ml package-release search --project-id <PID> --package-name <MODEL> --output json
  ```

  * 返却 JSON の各要素から以下を解釈：

    * `package_id` → package id
    * `id` → release id
    * `name`（または `version`/`tag`）→ リリース名（例: `xx1/2.3.0`）。`target/version` の推定に使用
    * `created_at` → 最新判定（`--latest`）に利用

* **リリース取得（pull）**：

  ```bash
  webauto ml package-release pull \
    --project-id <PID> \
    --package-id <PACKAGE_ID> \
    --package-release-id <RELEASE_ID> \
    --target-dir <TMPDIR> \
    --output json --quiet
  ```

  * ダウンロード後、payload ルートを自動検出して **フラット展開**します。

> PoC では release 検索結果から **package\_id と release\_id を自動抽出**するため、通常 ID の手打ちは不要です。

---

## インストール

1. `mlpoc` スクリプトを配置：

   ```bash
   sudo install -m 0755 mlpoc /usr/local/bin/mlpoc
   ```
2. 依存：`bash`, `jq`, `webauto`（必要に応じて `fzf` があると選択 UI が快適）
3. 追加（任意）：Bash 補完を `/etc/bash_completion.d/mlpoc` に配置

---

## 環境変数

* `WEBAUTO_PROJECT_ID`：既定のプロジェクト ID（デフォルト: `prd_jt`）
* `STORE_ROOT`：モデル実体のルート（デフォルト: `/opt/autoware/model-store`）
* `LINK_ROOT`：運用リンクのルート（デフォルト: `/opt/autoware/mlmodels`）

> `TARGET_DEFAULT` はスクリプト内で定義されていますが、現行実装ではリリース名から target/version を推測するため利用されません。

> いずれもサブコマンドの引数で都度上書き可能です。

---

## 使い方（サマリ）

### リモート一覧（webauto 経由）

```bash
mlpoc models
mlpoc releases <model>
```

### ローカルのインストール済み候補

```bash
mlpoc list                 # モデル名一覧
mlpoc list <model>         # target/version/path を表形式で
mlpoc list <model> --json  # JSON で出力
```

### 取得のみ（配置するが参照は切替えない）

```bash
sudo mlpoc pull --model <name> \
  [--latest | --version <ver> [--target <tgt>] | --release-name <name> | --release-id <rid> | --pick] \
  [--project-id <pid>] [--json] [--payload-subdir <relpath>]
```

* `--latest`：`created_at` が最新のリリースを選択
* `--version` + `--target`：`name = target/version` を解決
* `--release-name`：`"xx1/2.3.1"` のように name を直接指定
* `--release-id`：release id を直接指定
* `--pick`：リモートのリリース候補を対話選択（`--target` で絞り込み、`fzf` があればファジー検索）
* `--payload-subdir`：自動検出が合わない特殊ケース向け
* `--json`：選択結果を JSON（model/target/version/path など）で出力

### 取得して参照も切替（原子的 symlink 切替）

```bash
sudo mlpoc pull-switch --model <name> \
  [--latest | --version <ver> [--target <tgt>] | --release-name <name> | --release-id <rid> | --pick] \
  [--project-id <pid>] [--json] [--payload-subdir <relpath>] [--no-switch]
```

* `--no-switch`：実体だけ pull し、参照は切替えず JSON をそのまま出力

### 参照の切替（ローカル既存から）

```bash
# target/version 指定
sudo mlpoc switch <model> --target <tgt> --version <ver>

# 指定 target の中で最新版
sudo mlpoc switch <model> --latest --target <tgt>

# 対話選択（fzf あれば fzf、無ければ番号選択）
sudo mlpoc switch <model> --pick [--target <tgt>]

# 互換（絶対パスで指定）
sudo mlpoc switch <model> /opt/autoware/model-store/<model>/<target>/<version>
```

### 現在の参照先

```bash
mlpoc current <model>
```

---

## 設計の要点

* **原子的な切替**：一時リンク作成 → `rename` による置換で中断時も壊れリンクを見せない
* **フラット展開**：payload ルート自動検出（最大数階層の空ディレクトリ剥離 → `<uuid>/<model>` 推測 → 重要ファイルを含む最浅ディレクトリ探索）
* **権限最小化**：検証は非特権でも可能／実ファイル配置とリンク置換は `sudo` 実行
* **ロギング**：`$HOME/.local/share/autoware-mlswitch/<model>.jsonl` に切替履歴を追記（ts/before/after）
* **寛容な JSON パース**：`webauto` の返却差異に備え、複数のキー候補を許容する `jq` 式

---

## 権限とセキュリティ

* `/opt/autoware` 配下への書換や symlink 切替は通常 **管理者権限**が必要
* 運用で sudo を避けたい場合：

  * **専用グループ**を作成し `mlmodels` に `g+ws` を付与
  * あるいは **Polkit** 経由の最小権限ヘルパーを用意

---

## トラブルシューティング

* `permission denied`：`sudo` で実行／グループ権限設定を確認
* `webauto: command not found`：CLI のインストールと認証状態を確認
* `pulled dir is empty`：該当 release の asset を確認（権限/ネットワーク）
* 配置後に `.onnx` が見つからない：配布物のパスが深い可能性 → `--payload-subdir` を指定
* `--latest` で想定外が選ばれる：`created_at` が並び替え基準。`--version/--target` を明示

---

## 既知の制限 / 今後の拡張

* ✅ PoC：pull / switch / list / 対話選択 / フラット展開 / 最小検証
* ⏳ 追加予定（要望に応じて）：

  * ロールバック（履歴から直近 N 件を選んで復元）
  * manifest / SHA256 検証と署名チェック
  * `update-alternatives` 連携
  * 複数モデルを束ねる **プロファイル切替**（一括適用）
  * ヘルスチェック（ONNXRuntime で軽ロード）→ 失敗時の自動ロールバック

---

## ライセンス / 免責

* 本リポジトリは社内 PoC として提供されます。実運用前に十分な検証を行ってください。

---

## クイックスタート（例）

```bash
# モデルとリリースの調査
mlpoc models
mlpoc releases centerpoint

# 最新を取得して切替（xx1 ターゲット）
sudo mlpoc pull-switch --model centerpoint --latest --target xx1

# 取得だけして後で切替
sudo mlpoc pull --model centerpoint --version 2.3.1 --target xx1
sudo mlpoc switch centerpoint --target xx1 --version 2.3.1

# ローカル候補から対話選択
mlpoc list centerpoint
sudo mlpoc switch centerpoint --pick
```
