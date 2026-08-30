# GarTalkableDuck

`gar-talkable-duck` applicationと `microduck` 物理targetを固定した独立Product
repositoryです。Codespaces/devcontainer設定、Product固有hardware、build hook、
固定artifact契約を同じrepositoryで管理します。別の物理targetは別Product
repositoryとして作成します。

## Layout

```text
GarTalkableDuck/
  .devcontainer/
  config/
    common.env
    artifact.json
    product.env.example
  Makefile
  scripts/
    bootstrap.sh
    setup-common.sh
    setup-product-branch.sh
    product-sim-build.sh.example
    product-target-build.sh
  artifacts/             # generated output, ignored
```

## Setup

Codespaces 起動時は `.devcontainer/devcontainer.json` の `postCreateCommand` が
`scripts/post-create.sh` を実行します。実体は `scripts/bootstrap.sh` です。

起動時の流れ:

```text
scripts/setup-common.sh
scripts/setup-product-branch.sh
  config/product.env があれば読む
  .gitmodules があれば git submodule update --init --recursive
  scripts/product-install.sh が実行可能なら実行
```

手動で実行する場合:

```bash
make setup
```

製品ブランチ側で submodule を使っている場合に明示的に最新化するには:

```bash
make sync
```

`make setup` は製品ブランチに定義された設定を読み、必要な準備だけを実行します。
`.gitmodules` がある場合は、親リポジトリが記録している submodule commit を再現します。
`make sync` は branch checkout されている submodule だけ `git pull --ff-only` します。
`make build` と `make artifacts` はセットアップを自動実行しません。起動時セットアップは
Devcontainer の `postCreateCommand` に限定し、必要な場合だけ明示的に `make setup` を実行します。

## GAR Simulation Build Hook

`gar sim app build` の入口は Codespaces 固有ではなく、ローカルまたは Codespaces 上で動く
GaplessAgentRuntime です。製品 branch で simulation build が必要な場合は、
`scripts/product-sim-build.sh.example` を `scripts/product-sim-build.sh` にコピーして
アプリ固有の build コマンドを定義してください。GAR はその script を呼び出します。

通常、製品 branch はアプリを `sources/<app>`、共有 simulation asset を
`sources/gar-tools` に submodule として持ちます。template の `GAR_SIM_APP_DIR` と
`GAR_TOOLS_DIR` はその配置を参照し、アプリ側の command には
`GAR_TOOLS_ROOT` として後者を渡せます。

## GAR Target Build Hook

`gar target app build` は、選択したLocalまたはCodespaces build environmentで
`scripts/product-target-build.sh`を実行します。製品branchでは
`scripts/product-target-build.sh.example`をコピーし、`deploy.app`を持つ
`artifacts/from-codespace/artifact.json`を生成するbuild commandを定義してください。

## Product Branches

製品ブランチでは、共通シーケンスをなるべく触らず、個別定義だけを追加します。

```text
config/product.env
scripts/product-install.sh
scripts/product-build.sh
scripts/product-artifacts.sh
scripts/product-clean.sh
scripts/product-sim-build.sh
scripts/product-target-build.sh
sources/* submodules
AGENTS.md
```

関連リポジトリを submodule として持つ製品ブランチでは、子リポジトリを先に
commit/push し、そのあと親の submodule pointer を更新してください。

```bash
cd path/to/submodule
git add -A
git commit -m "Update product repo"
git push

cd path/to/GarTalkableDuck
git add path/to/submodule
git commit -m "Update product submodule pointer"
git push
```

## Product Build Hooks

Product固有のビルド手順はこのrepositoryのhookとして管理します。必要に応じて
次の hook を追加します。

```text
scripts/product-install.sh
scripts/product-build.sh
scripts/product-artifacts.sh
scripts/product-clean.sh
```

`make build` は `scripts/product-build.sh` があれば実行します。
`make artifacts` は `scripts/product-artifacts.sh` があれば実行します。
固定Targetのartifact契約は`config/artifact.json`で管理します。

PlatformIO は Python 仮想環境 `~/.venvs/platformio` にインストールされ、
`~/.bashrc` に PATH が追加されます。

## Fixed application / target

Application契約は`sources/gar-talkable-duck/app.json`、固定artifact契約は
`config/artifact.json`、将来の実機package実装は`scripts/target/`が所有します。
現在のtargetはplannedであり、
実装が追加されるまで実機packageは明示的に失敗します。

```bash
make check-target
```
