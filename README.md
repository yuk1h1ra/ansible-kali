# Ansible-Kali

Kali Linux VM の初期セットアップを自動化する Ansible Playbook。
Kali VM に XFCE デスクトップ上でログインし、そのデスクトップ上で開いた端末から、
**ローカル実行**（`connection: local`）で使うことを前提にしている。

## できること

- システムのタイムゾーンを `Asia/Tokyo` に変更
- Kali標準ツール群はそのまま維持し、便利なCLIツール（ripgrep, fzf, bat, htop, tree, jq等）を追加インストール
- Docker（`docker.io`）のインストール、サービス有効化、現在ユーザーの docker グループ追加、docker compose プラグイン対応
- BloodHound Community Edition を公式 `bloodhound-cli` 経由でインストール（初期admin パスワードを指定値に設定）
- tmux / vim の実用的な設定ファイル配置（tmux は個人 dotfiles の設定をプラグイン無しで再現）
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

一部だけ実行したい場合はタグを指定する（`timezone` / `cli_tools` / `docker` / `bloodhound` / `tmux` / `vim` /
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

## tmux の主なキーバインド

prefix は `C-t`（`C-t C-t` でアプリに `C-t` を送る）。TPM・プラグインは使わず、dotfiles で使っていた
tmux-sensible / tmux-pain-control / tmux-yank / tmux-logging / tmux-split-window-cross-plugin /
tmux-powerline(simple テーマ) 相当の設定を `config/tmux.conf` に直接書いている。

| キー | 動作 |
|---|---|
| `prefix r` / `prefix R` | 設定リロード |
| `prefix \|` / `prefix -` | 左右 / 上下に分割（カレントパス継承。`\` `_` は全幅分割） |
| `prefix +` | 十字に4分割 |
| `prefix h/j/k/l` | ペイン移動 |
| `prefix H/J/K/L` | ペインリサイズ（連打可） |
| `prefix <` / `prefix >` | ウィンドウの並べ替え |
| `prefix C-p` / `prefix C-n` | 前 / 次のウィンドウ |
| コピーモードで `y` | クリップボードへコピー（xclip。マウス選択は PRIMARY へ） |
| `prefix P` | ペインのロギング開始/停止（`~/tmux-<session>-<window>-<pane>-<日時>.log`） |
| `prefix M-p` / `prefix M-P` | 画面 / 全履歴をファイルに保存 |
| `prefix M-c` | ペインの履歴を消去 |

新しく開いたペイン/ウィンドウは `script` 経由でシェルが起動し、`~/tmux-logs/<日時>-pane<ペインID>.log`
（と `.timing`）に常時自動記録される。ログはANSIエスケープ込みなので `less -R` で見るか、
`scriptreplay -t X.timing X.log` で再生する。`new-window 'htop'` のようにコマンドを指定したペインは
記録対象外。ログは自動削除されないので定期的に整理すること。

ステータスバーは上部に表示し、セッション情報・LAN IP・VPN(`tun*`)の IP・曜日・日付・時刻を出す。
Powerline 用パッチフォントが無くても崩れない記号（▶ ❯ など）を使っている。

## 既知の注意点

- **docker グループの反映**: `docker` タグのタスクで現在ユーザーを `docker` グループに追加するが、
  通常はログインセッション単位でキャッシュされるため、同一セッションでは反映されない。
  `bloodhound` タグの `bloodhound-cli install` は `sg docker -c` で docker グループを付与して実行し回避しているが、
  手動で `docker` コマンドを叩く場合は一度ログアウト/再ログイン、または `newgrp docker` が必要。
- **docker compose**: Kali には `docker-compose-v2` パッケージが無いが、`docker-compose` パッケージ自体が
  v2 で `docker compose` プラグイン（`/usr/libexec/docker/cli-plugins/docker-compose`）も提供するため、
  これをインストールしている。`docker compose version` で確認できる。
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
docker compose version
cd ~/Awesome/BloodHound && ./bloodhound-cli config get default_admin.password
```

- 一定時間放置してロック画面が出ないことを目視確認する
- JIS特有キー（`@` `(` `)` `"` `_`）がターミナルとGUIアプリ両方で正しく入力できることを確認する
- 再ログイン後 `groups` に `docker` が含まれること、`docker run hello-world` が sudo なしで
  動くことを確認する
- ブラウザで `http://localhost:8080/ui/login` にアクセスし、`admin` / 設定したパスワードで
  ログインできることを確認する
