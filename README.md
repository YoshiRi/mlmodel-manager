# mlpoc — Autoware ML モデル取得/配置/切替ツール（PoC）

**mlpoc** は、`webauto` CLI から ML モデルのリリースを取得し、
`/opt/autoware/model-store/<model>/<target>/<version>/` に配置、
運用参照を `/opt/autoware/mlmodels/<model>` の **シンボリックリンク**で切り替えるための PoC ツールです。

* 目的：モデルの **バージョン/ターゲット** を安全かつ素早く切替可能にする
* 前提：インフラ側の DL は既存の `webauto` CLI を利用（このツールは配置と参照切替に専念）
* 既定のプロジェクト ID は **`prd_jt`**（環境変数や引数で変更可能）

---

## Quick start

初回利用時は既存の `/opt/autoware/mlmodels` を必要に応じてバックアップしてください。

```bash
# setup
sudo chmod +x mlpoc
sudo ln -sf "$(pwd)/mlpoc" /usr/local/bin/mlpoc
mlpoc --help
```

Switch まで一括で行う Demo：

```bash
mlpoc models
mlpoc releases centerpoint
mlpoc pull-switch centerpoint # 対話選択後、自動で切替
ll /opt/autoware/mlmodels/centerpoint
```

Pull のみ → あとで Switch：

```bash
mlpoc pull --model centerpoint --latest --target xx1 # or --pick
mlpoc pull --model centerpoint --latest --target x2
mlpoc switch centerpoint --latest --target xx1  # or --pick
mlpoc current centerpoint
```

---

## 動作フロー

1. **リモート候補を調査**：`mlpoc models` / `mlpoc releases <model>`
2. **必要なリリースを取得（pull）**：

   * `mlpoc pull` で target/version に従い **原子的に配置**
   * `<uuid>/centerpoint` のような余計なネストは **payload 自動検出**で解消
3. **参照切替（switch）**：

   * `/opt/autoware/mlmodels/<model>` → store パスに **原子的 symlink 切替**
   * `--target/--version` または `--pick` で操作

---

## ディレクトリ構成

```
/opt/autoware/
  model-store/
    <model>/<target>/<version>/        # 実体配置
  mlmodels/
    <model> -> .../<model>/<target>/<version>  # 運用リンク
```

---

## webauto CLI 背景

* **モデル一覧**：`webauto ml package list`
* **リリース検索**：`webauto ml package-release search`
* **リリース取得**：`webauto ml package-release pull`

`mlpoc` は ID を自動解釈するため、通常 ID を手打ちする必要はありません。

---

## 使い方

### リモート一覧

```bash
mlpoc models
mlpoc releases <model>
```

### ローカル一覧

```bash
mlpoc list                 # モデル名のみ
mlpoc list <model>         # target/version/path を表形式
mlpoc list <model> --json  # JSON 出力
```

出力例：

```bash
❯ mlpoc list centerpoint
model: centerpoint
used    target  version path
        x2      2.2.0   /opt/autoware/model-store/centerpoint/x2/2.2.0
        x2      2.3.0   /opt/autoware/model-store/centerpoint/x2/2.3.0
✔       x2      2.3.1   /opt/autoware/model-store/centerpoint/x2/2.3.1
```

### Pull のみ

```bash
sudo mlpoc pull --model <name> [--latest|--version <ver> --target <tgt>|--release-name <name>|--release-id <rid>]
```

### Pull + Switch

```bash
sudo mlpoc pull-switch --model <name> [options] [--no-switch]
```

### Switch（ローカル候補から）

```bash
sudo mlpoc switch <model> --target <tgt> --version <ver>
sudo mlpoc switch <model> --latest --target <tgt>
sudo mlpoc switch <model> --pick
```

### 現在の参照

```bash
mlpoc current <model>
```

---

## 環境変数

* `WEBAUTO_PROJECT_ID`：既定プロジェクト ID（デフォルト: prd\_jt）
* `STORE_ROOT`：実体ルート（デフォルト: `/opt/autoware/model-store`）
* `LINK_ROOT`：リンクルート（デフォルト: `/opt/autoware/mlmodels`）
* `TARGET_DEFAULT`：target 未指定時の仮の既定（デフォルト: default）

---

## 設計の要点

* **原子的切替**：rename により壊れリンクを避ける
* **フラット展開**：payload ルートを自動検出
* **権限最小化**：配置とリンク更新のみ sudo
* **選択 UI**：fzf があれば fzf、無ければ番号選択

---

## 既知の制限 / Known Limitations

* `name` が `x2/2.0.0_hoge` や `xx1/1.0.0/hogefuga` の場合：

  * 現在は **suffix を無視**して `x2/2.0.0` として扱う
  * 結果、異なる flavor が同じディレクトリに上書きされる可能性あり
  * 必要なら suffix を別フォルダに保存する拡張が必要
* `--latest` は `created_at` ベースのソートに依存
* SHA 検証や manifest チェックは未実装

---

## 今後の拡張

* Suffix 区別保存 (`--preserve-suffix`)
* manifest / SHA256 検証
* ロールバック機能
* プロファイル切替（一括適用）
* ヘルスチェック（ONNXRuntime）

---

## 免責

* 本リポジトリは社内 PoC として提供。実運用前に十分に検証してください。
