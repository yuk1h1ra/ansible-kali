# Ansible-Kali

Kali Linux VM の初期セットアップを自動化する Ansible Playbook。
Kali VM に XFCE デスクトップ上でログインし、そのデスクトップ上で開いた端末から、
**ローカル実行**（`connection: local`）で使うことを前提にしている。

## できること

- Kali標準ツール群はそのまま維持し、便利なCLIツール（ripgrep, fzf, bat, htop, tree, jq等）を追加インストール
- Docker（`docker.io`）のインストール、サービス有効化、現在ユーザーの docker グループ追加、docker compose プラグイン対応
- BloodHound Community Edition を公式 `bloodhound-cli` 経由でインストール（初期admin パスワードを指定値に設定）
- tmux / vim の実用的な設定ファイル配置
- XFCE のアイドル自動ロック（screensaver）を完全に無効化
- キーボード配列を日本語(JIS)に変更（コンソール・XFCE GUI セッション両方）

## 構成

```
README.md       # このファイル
playbook.yaml   # 変数・タスク・handlers をすべて記述した単一 playbook
config/         # ホームに配置する設定ファイル（vimrc → ~/.vimrc, tmux.conf → ~/.tmux.conf）
```

## 事前準備

```bash
sudo apt install ansible
ansible-galaxy collection install community.general
```

BloodHound の初期admin パスワードは、実行時にプロンプトで入力する（ファイルには保存しない）。

## 実行方法

**XFCEデスクトップ上で開いた端末（xfce4-terminal 等）から、通常ユーザーの権限で実行すること。**
SSHのみのセッションや、playbook全体を sudo で実行すると、`screenlock`/`keyboard` タグの
xfconf設定タスクが `DISPLAY`/`DBUS_SESSION_BUS_ADDRESS` 不在で失敗する。

```bash
ansible-playbook playbook.yaml --ask-become-pass
```

一部だけ実行したい場合はタグを指定する（`cli_tools` / `docker` / `bloodhound` / `tmux` / `vim` /
`screenlock` / `keyboard`。apt cache 更新は常に実行される）:

```bash
ansible-playbook playbook.yaml --ask-become-pass --tags tmux,vim
ansible-playbook playbook.yaml --ask-become-pass --skip-tags bloodhound \
  -e bloodhound_admin_password=unused   # パスワードのプロンプトを省略したい場合
```

dry-run（文法・論理チェックのみ。`command`モジュールやxfconfのcheck-mode対応には限界があるため
最終確認は必ず実機で行うこと）:

```bash
ansible-playbook playbook.yaml --syntax-check
ansible-playbook playbook.yaml --check --diff
```

## 既知の注意点

- **docker グループの反映**: `docker` タグのタスクで現在ユーザーを `docker` グループに追加するが、
  通常はログインセッション単位でキャッシュされるため、同一セッションでは反映されない。
  `bloodhound` タグのdockerコマンドは `become_user` 経由（sudo再評価）で回避しているが、
  手動で `docker` コマンドを叩く場合は一度ログアウト/再ログイン、または `newgrp docker` が必要。
- **docker-compose-v2 の有無**: Kaliのリポジトリに `docker-compose-v2` パッケージが無い場合は、
  旧来の `docker-compose`（v1）にフォールバックする。どちらが入ったかは
  `docker compose version` / `docker-compose --version` で確認すること。
- **BloodHound CEのデータ配置**: `bloodhound-cli` の実行ファイル自体は `~/Awesome/BloodHound`
  に置かれるが、実際の `docker-compose.yml` や `bloodhound.config.json` は常に
  `~/.config/bloodhound/`（`bloodhound-cli` 側の仕様で変更不可）に生成される。
  アクセスURLは `http://localhost:8080/ui/login`、ユーザー名は `admin`。
  初回ログイン時はパスワード変更を求められる。
- **xfconf のプロパティパス**: `screenlock`（channel: `xfce4-screensaver`）と
  `keyboard`（channel: `keyboard-layout`）で使っているプロパティパスは、Xfceの一般的な命名に
  基づく推定。実機で以下を実行して実際のプロパティ名と差異がないか確認し、ずれていた場合は
  `playbook.yaml` の該当タスクの `property:` の値だけ修正すればよい。
  ```bash
  xfconf-query -c xfce4-screensaver -lv
  xfconf-query -c keyboard-layout -lv
  ```

## 実機での確認手順

```bash
cat /etc/default/keyboard
setxkbmap -query
xfconf-query -c xfce4-screensaver -lv
xfconf-query -c keyboard-layout -lv
systemctl status docker
docker compose version   # もしくは docker-compose --version
cd ~/Awesome/BloodHound && ./bloodhound-cli config get default_admin.password
```

- 一定時間放置してロック画面が出ないことを目視確認する
- JIS特有キー（`@` `(` `)` `"` `_`）がターミナルとGUIアプリ両方で正しく入力できることを確認する
- 再ログイン後 `groups` に `docker` が含まれること、`docker run hello-world` が sudo なしで
  動くことを確認する
- ブラウザで `http://localhost:8080/ui/login` にアクセスし、`admin` / 設定したパスワードで
  ログインできることを確認する
