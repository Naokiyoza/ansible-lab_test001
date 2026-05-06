# Ansible学習記録 - 2026-05-06

明日の自分のための振り返りノート。本日の学習内容を網羅的にまとめた。

---

## 1. 本日のゴールと達成状況

**ゴール:** 実務で使えるAnsibleプロジェクトの土台を一通り構築する。

| カテゴリ | 達成内容 |
|---|---|
| 環境構築 | Docker Composeでコントローラ + 管理ノード3台を起動 |
| Ansible基礎 | プレイブック、ロール、変数、ハンドラ、テンプレート |
| 構造化 | ロール化（common/nginx_https/mysql_server）+ group_vars階層化 |
| セキュリティ | Ansible Vaultでパスワードを暗号化 |
| テスト自動化 | Moleculeでロールのcreate/converge/idempotence/verify |
| CI/CD | GitHub Actionsでマトリクス並列ロールテスト |
| バージョン管理 | GitHubに公開（https://github.com/Naokiyoza/ansible-lab_test001） |

---

## 2. 構築したアーキテクチャ

### コンテナ構成

```
[Windows Docker Desktop]
        |
        +-- ansible-controller (192.168.1.70)
        |     |
        |     +-- Ansible / Molecule / Docker CLI
        |     +-- /var/run/docker.sock マウント (Molecule用)
        |
        +-- ansible-node1 (192.168.1.71) [webservers]
        |     +-- nginx (HTTPS/443のみ、自己署名証明書)
        |     +-- ホスト 8443 -> コンテナ 443
        |
        +-- ansible-node2 (192.168.1.72) [webservers]
        |     +-- nginx (HTTPS/443のみ、自己署名証明書)
        |     +-- ホスト 8444 -> コンテナ 443
        |
        +-- ansible-node3 (192.168.1.73) [dbservers]
              +-- MySQL 8.0 (mysql_native_password)
              +-- testdb / appuser
```

### 重要なDocker設定ポイント

| 設定 | 目的 |
|---|---|
| `privileged: true` | systemdをコンテナで動かすため |
| `command: /sbin/init` | initプロセスとしてsystemdを起動 |
| `tmpfs: [/run, /tmp]` | systemdが書き込む一時領域 |
| `cgroupns_mode: host` | systemdのcgroup管理 |
| `/var/run/docker.sock` マウント | コントローラから兄弟コンテナを起動可能に |

---

## 3. ディレクトリ構成

```
ansible-lab/
├── .github/
│   └── workflows/
│       └── molecule.yml             # GitHub Actions: 自動テスト
├── .gitattributes                   # 改行コードLF統一
├── .gitignore                       # .vault_pass.txt等を除外
├── .vault_pass.txt                  # Vault複合鍵(GitignoreでGit対象外)
├── ansible.cfg                      # roles_path、interpreter等の設定
├── docker-compose.yml               # コンテナ定義
├── controller/
│   └── Dockerfile                   # Ansible+Molecule+Docker CLI
├── managed-nodes/
│   └── Dockerfile                   # systemd対応AlmaLinux 9
├── inventory/
│   ├── hosts                        # webservers/dbserversグループ定義
│   └── group_vars/                  # インベントリ隣に置くのが標準
│       ├── all/main.yml             # 全ホスト共通変数
│       ├── webservers/main.yml      # nginx関連変数
│       └── dbservers/
│           ├── main.yml             # DB関連変数(参照のみ)
│           └── vault.yml            # 暗号化済パスワード
├── playbooks/
│   ├── site.yml                     # メインプレイブック(全構成)
│   └── 01-05_*.yml                  # 学習過程のプレイブック
└── roles/
    ├── common/                      # 共通パッケージ
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml
    │   └── molecule/default/        # ロールテストシナリオ
    │       ├── molecule.yml
    │       ├── prepare.yml
    │       ├── converge.yml
    │       └── verify.yml
    ├── nginx_https/                 # HTTPS構築
    │   ├── defaults/main.yml
    │   ├── handlers/main.yml
    │   ├── tasks/main.yml
    │   ├── templates/ssl.conf.j2
    │   └── molecule/default/
    └── mysql_server/                # MySQL 8.0構築
        ├── defaults/main.yml
        ├── handlers/main.yml
        ├── tasks/
        │   ├── main.yml             # importでサブタスク呼び出し
        │   ├── install.yml
        │   ├── configure.yml
        │   ├── secure.yml
        │   └── databases.yml
        ├── templates/custom.cnf.j2
        └── molecule/default/
```

---

## 4. 学習の旅程（時系列）

### フェーズ1: Ansible基礎

1. **`01_hello.yml`** - ping接続確認
2. **`02_install_nginx.yml`** - nginxインストール（webserversグループ対象）
3. **`03_install_packages.yml`** - 共通パッケージ一括インストール
4. **`04_rhel_basics.yml`** - SELinux確認、httpdインストール → nginxとのポート競合を解決
5. **`05_mysql_setup.yml`** - MySQL 8.0構築、認証プラグイン制御

### フェーズ2: ロール化とVault

6. **ロール化** - `common`/`nginx_https`/`mysql_server`に分割
7. **group_vars階層化** - all/webservers/dbservers
8. **Vault暗号化** - DBパスワードをAES256で暗号化

### フェーズ3: テスト自動化

9. **Molecule導入** - controllerにDocker CLI追加、socketマウント
10. **3ロール分のシナリオ作成** - create/converge/idempotence/verify
11. **GitHub Actions連携** - マトリクス戦略で3ロール並列テスト

---

## 5. 重要な技術概念

### 5-1. 変数の優先順位（弱 → 強）

```
1. role defaults     (roles/<role>/defaults/main.yml)
2. group_vars/all    (全ホスト共通)
3. group_vars/<group>(グループ単位)
4. host_vars/<host>  (ホスト単位)
5. play vars         (プレイブック内 vars:)
6. -e (extra vars)   (コマンドラインで渡す変数、最強)
```

### 5-2. Vaultの命名パターン

機密情報を `vault_xxx` 命名で暗号化ファイルに格納し、通常変数からラップ参照する。

```yaml
# group_vars/dbservers/vault.yml (暗号化)
vault_mysql_root_password: "RootPass123!"

# group_vars/dbservers/main.yml (平文)
mysql_root_password: "{{ vault_mysql_root_password }}"
```

メリット: コードレビュー時に「どの変数が機密か」がgrepで追跡しやすい。

### 5-3. MySQL 8.0の認証プラグイン

| プラグイン | デフォルトのバージョン | クライアント互換性 |
|---|---|---|
| `caching_sha2_password` | MySQL 8.0以降 | PyMySQL 0.10.0+, mysqlclient 1.4.0+ |
| `mysql_native_password` | MySQL 5.7まで | ほぼ全クライアント |

`my.cnf` を **サービス起動「前」** に配置することで、初期化時から認証プラグインを反映させる。

### 5-4. RHEL系の初期rootパスワード仕様

| 提供元 | 初期rootパスワード |
|---|---|
| MySQL Community Server (公式) | 一時パスワードを `/var/log/mysqld.log` に出力 |
| RHEL 9 / AlmaLinux 9 標準パッケージ | **空文字（パスワードなし）** |

冪等性のために `mysql --no-defaults -u root -e "SELECT 1;"` で空パスワード接続できるか確認してから設定するロジックを書いた。`--no-defaults` で `~/.my.cnf` を無視する。

### 5-5. Moleculeのテストライフサイクル

```
dependency -> lint -> destroy -> syntax -> create -> prepare ->
converge -> idempotence -> side_effect -> verify -> cleanup -> destroy
```

| フェーズ | 役割 |
|---|---|
| `create` | テスト用コンテナ起動 |
| `converge` | ロール適用（メイン） |
| **`idempotence`** | **2回目実行でchanged=0を検証（冪等性）** |
| **`verify`** | **assert等で状態検証** |
| `destroy` | クリーンアップ |

冪等性とverifyが本番リリース判断の基準になる。

### 5-6. FQCN（Fully Qualified Collection Name）

ansible-lintのベストプラクティス。`dnf:` ではなく `ansible.builtin.dnf:` と書く。

```yaml
# Good
- name: パッケージインストール
  ansible.builtin.dnf:
    name: nginx

# Bad (lintで警告)
- name: パッケージインストール
  dnf:
    name: nginx
```

メリット: コレクション衝突の防止、IDE補完精度向上。

---

## 6. よく使うコマンド集

### Docker Compose

```powershell
# 起動
docker-compose up -d --build

# 完全停止＆ボリューム保持
docker-compose down

# 個別コンテナ再作成
docker-compose stop node3
docker-compose rm -f node3
docker-compose up -d node3

# コントローラに入る
docker exec -it ansible-controller bash
```

### Ansible

```bash
# プレイブック実行
ansible-playbook /ansible/playbooks/site.yml

# タグ指定で部分実行
ansible-playbook /ansible/playbooks/site.yml --tags mysql

# アドホック実行
ansible -m ping all
ansible -m debug -a "var=mysql_root_password" dbservers

# 詳細ログ
ansible-playbook xxx.yml -vvv
```

### Ansible Vault

```bash
# 暗号化
ansible-vault encrypt /path/to/vault.yml

# 編集（一時複合→保存時に再暗号化）
ansible-vault edit /path/to/vault.yml

# 閲覧のみ
ansible-vault view /path/to/vault.yml

# 復号（平文に戻す）
ansible-vault decrypt /path/to/vault.yml

# パスワード変更
ansible-vault rekey /path/to/vault.yml
```

### Molecule

```bash
# 段階実行（学習・デバッグ向き）
cd /ansible/roles/common
molecule create        # コンテナ起動
molecule converge       # ロール適用
molecule idempotence    # 冪等性チェック
molecule verify         # 検証
molecule destroy        # クリーンアップ

# 一気に全工程実行（CI向き）
molecule test

# テストコンテナに入ってデバッグ
molecule login

# シナリオ確認
molecule list
```

### Git

```powershell
# 初期化と設定
git init
git branch -M main
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# .gitignoreチェック
git check-ignore -v .vault_pass.txt

# リモート追加（HTTPS）
git remote add origin https://github.com/<user>/<repo>.git

# 認証（PATの workflow スコープが必要）
git push -u origin main
# Username: <github-username>
# Password: <PAT (ghp_xxx...)>
```

---

## 7. 本日遭遇したエラーと解決法

### 7-1. nginx起動失敗（ポート80競合）

**症状:** httpdとnginxが同じport 80を使おうとして失敗
**解決:** nginxをHTTPS/443専用に移行（自己署名証明書を生成）

### 7-2. 「The vault password file ... was not found」

**症状:** GitHub Actionsで `.vault_pass.txt` が無くて失敗
**原因:** `.gitignore` で除外しているのでGitHubに無い
**解決:** `ansible.cfg` から `vault_password_file` を外し、コントローラ環境変数（`ANSIBLE_VAULT_PASSWORD_FILE`）で指定

### 7-3. 「the role 'common' was not found」（Molecule on CI）

**症状:** Moleculeがロール本体を見つけられない
**原因:** `roles_path` が CI 環境で正しいパスを指していない
**解決:** `molecule.yml` の `provisioner.env` に `ANSIBLE_ROLES_PATH: "${MOLECULE_PROJECT_DIRECTORY}/.."` を設定

### 7-4. 「Not supported URL scheme http+docker」（Molecule初回）

**症状:** Pythonの docker SDK が requests と非互換
**解決:** `pip3 install --upgrade 'docker>=7.1.0' 'requests<2.32'`

### 7-5. 「rsync not found」（Molecule create時）

**症状:** Moleculeのcontextシンク失敗
**解決:** controllerに `rsync` パッケージ追加

### 7-6. 「Failed to create temporary directory」（初回ping時）

**症状:** managed-nodesにansibleで接続不可
**原因:** `~/.ansible/tmp` の作成権限問題
**解決:** `ansible.cfg` に `remote_tmp = /tmp/.ansible/tmp` を設定

### 7-7. 「!: event not found」（手動でmysqlコマンド実行時）

**症状:** bashのヒストリ展開が `!` を解釈
**解決:** 対話シェルでは `set +H`、Ansibleは `command` モジュールでシェルを介さないので問題なし

### 7-8. group_varsが ansible-playbook で読めない

**症状:** `ansible -m debug` では変数が見えるが `ansible-playbook` では undefined
**原因:** `group_vars/` を inventory ディレクトリ隣に置いていなかった
**解決:** `/ansible/inventory/group_vars/` に配置

### 7-9. GitHub push 失敗（PATの workflow スコープ不足）

**症状:** `.github/workflows/` 配下のファイルが push できない
**解決:** PATに `workflow` スコープを追加

---

## 8. 設計上の判断ポイント（実務に活きる）

### 8-1. なぜ`include_role`ではなく`roles:`ディレクティブを使うか

Moleculeのロール自動解決は `roles:` ディレクティブのみ機能する。`include_role`/`import_role` では `roles_path` を明示的に設定する必要があり、CI/ローカル両環境での互換性が低下する。

### 8-2. なぜ vault.yml を main.yml と分けるか

Vaultは値そのものが暗号化されるため、ファイル全体が暗号化されると平文部分（変数名・コメント）も読めなくなる。**分離することで暗号化対象を最小化**し、コードレビュー性を担保。

### 8-3. なぜ `tasks/main.yml` を `import_tasks` で分割するか

mysql_serverロールは数十タスクに及ぶため、1ファイルだと可読性が低下する。`install/configure/secure/databases` のように責務分離することで、タグ指定（例: `--tags secure`）で部分実行も可能になる。

### 8-4. なぜテスト用コンテナで systemd を動かすか

production環境がsystemdベース（RHEL系）なので、コンテナでも同じ条件にしないと「ローカルでは動いたが本番で失敗」のパターンが発生する。`privileged: true` + `/sbin/init` でコンテナ内systemdを実現。

---

## 9. 次のステップ候補

| トピック | 学習価値 |
|---|---|
| **handlers の活用** | `notify`+`listen` の連鎖、`force_handlers` の使い所 |
| **block / rescue / always** | エラーハンドリング、ロールバック処理 |
| **roles の依存関係** | `meta/main.yml` の `dependencies` |
| **動的インベントリ** | AWS/GCP/Azureの動的取得 |
| **AWX / Ansible Automation Platform** | GUI管理、RBAC、ログ集約 |
| **HashiCorp Vault連携** | より本格的なシークレット管理 |
| **Goss / Testinfra** | より宣言的なverifier |
| **複数OSターゲットマトリクス** | AlmaLinux9/Ubuntu22/Debian12 等で同時テスト |

---

## 10. 振り返り：実務でそのまま使える土台

本日の構成は **実務で使える Ansible プロジェクトの骨格** が揃っている。

- ローカル開発: docker-compose で隔離環境
- 構造化: ロール + 変数階層
- セキュリティ: Vault
- 品質保証: Molecule
- CI/CD: GitHub Actions
- バージョン管理: Git

新しいロールを追加する際の標準フロー:

1. `roles/<new_role>/{tasks,defaults,handlers,templates,molecule/default}` を作成
2. `tasks/main.yml` を書く
3. `molecule/default/{molecule.yml,converge.yml,verify.yml}` を書く
4. ローカルで `molecule test` を実行
5. PASSしたらGitにpush
6. GitHub Actionsで自動テスト
7. 緑なら本番（site.yml）に追加してdeploy

このフローを回せるようになった、というのが本日の最大の成果。
