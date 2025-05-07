## 🛡️ Fail2banによるSSHアクセス監視と自動BAN設定

---

### ✅ 目的

- ログイン失敗を検知し、不正アクセスを自動で遮断する
- SSH対象の設定で fail2ban を活用
- `fail2ban-client` による詳細な動作確認方法も理解

---

### 🛠️ Step1：Fail2banの導入と起動

```bash
sudo apt update
sudo apt install fail2ban -y
sudo systemctl start fail2ban
sudo systemctl enable fail2ban
```

---

### ✅ 状態確認（systemctl）

```bash
sudo systemctl status fail2ban
```

```plaintext
● fail2ban.service - Fail2Ban Service
   Active: active (running)
```

---

### 🧰 fail2ban-client の使い方と役割

| コマンド | 説明 |
|----------|------|
| `sudo fail2ban-client status` | 現在有効な jail（監視ルール）の一覧 |
| `sudo fail2ban-client status sshd` | sshd jail の詳細ステータス |

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

---

### 🔧 Step2：SSH監視ルールの設定（jail.local）

```bash
sudo nano /etc/fail2ban/jail.local
```

```ini
[sshd]
enabled = true
port    = ssh
logpath = %(sshd_log)s
maxretry = 3
findtime = 600
bantime  = 600
```

| 項目 | 内容 |
|------|------|
| `maxretry` | 3回ログイン失敗で |
| `findtime` | 600秒以内に（＝10分） |
| `bantime` | 600秒間（＝10分）BANする |

---

### 🔁 設定反映

```bash
sudo systemctl restart fail2ban
```

---

### 🔍 動作確認：SSH監視が有効化されているか？

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

期待される表示例：

```plaintext
Jail list: sshd
Currently failed: 0
Currently banned: 0
File list: /var/log/auth.log
```

---

### 💡 `fail2ban-client`とは？

- fail2ban専用のCLIクライアント（状態取得・制御などに使う）
- 他のサービスでも `xxx-client` や `xxx-cli` は同様の構成が多い（例：`docker`, `redis-cli`, `etcdctl`）

---

### ✅ 現在の状態まとめ

| 状態 | 内容 |
|------|------|
| Fail2ban導入・起動 | ✅ 完了 |
| SSH監視ルール有効化 | ✅ `/etc/fail2ban/jail.local` 設定済み |
| SSHのログ監視 | ✅ `/var/log/auth.log` を参照中 |
| BAN状況 | ✅ 現在は攻撃がないため BANなし（正常） |

---

```bash
# 今後の確認用メモ：
sudo fail2ban-client set sshd status       # sshd jail 状態
sudo fail2ban-client set sshd unbanip IP   # 手動でBAN解除したいとき
```

---
## 🔐 Fail2ban実験：SSH失敗による自動BANの確認

---

### ✅ 実験の目的

- パスワード間違いによるSSHログイン失敗を故意に発生させる
- fail2banがそのIPアドレスを検知・BANするかを確認する

---

### 🧪 実験構成

| 項目 | 内容 |
|------|------|
| ホストOS | Windows（SSHクライアント） |
| ゲストOS | VirtualBox上のUbuntu Server（fail2banインストール済） |
| 接続方法 | ポートフォワーディングを使用（ホスト:2222 → ゲスト:22） |
| SSHコマンド | `ssh hogeuser@localhost -p 2222`（存在しないユーザーでログイン失敗） |

---

### ✅ SSH接続の仕組み（図式）

```
Windowsターミナル
  ↓ ssh @localhost -p 2222
VirtualBox NATポートフォワード
  ↓ （ホスト:2222 → ゲスト:22）
UbuntuゲストOS（fail2banがsshd監視中）
```

---

### ✅ 接続元・接続先の意味整理

| 要素 | 説明 |
|------|------|
| `hogeuser@` | 仮想マシンの存在しないユーザー → ログイン失敗要因 |
| `localhost` | ホスト自身を指すが、実際はポートフォワードでゲストに届く |
| `-p 2222` | ゲストのポート22に転送されるよう設定（VB NAT経由） |
| BAN対象IP | SSH接続をしたホストOS（例：10.0.2.2）のIP |

---

### 🔁 実行手順

1. Windowsから以下を複数回実行（パスワードは意図的にミス）
   ```bash
   ssh hogeuser@localhost -p 2222
   ```

2. パスワード3回失敗で `/var/log/auth.log` にログイン失敗が記録

3. Ubuntuゲスト上でfail2banの状態確認：

   ```bash
   sudo fail2ban-client status sshd
   ```

   出力例：

   ```plaintext
   Total failed: 3
   Currently banned: 1
   Banned IP list: 10.0.2.2
   ```

---

### ✅ BANの解除（手動）

```bash
sudo fail2ban-client set sshd unbanip 10.0.2.2
```

---

### 🧠 ポイントまとめ

- fail2banは `/var/log/auth.log` に記録された**失敗ログを元に自動BAN**を実施する
- ポートフォワーディングを使えば、**NAT環境でも外部接続テストが可能**
- 接続先が間違っていても、**接続元IPがBAN対象**になる
- `Currently banned: 1` と表示されれば**fail2banによるブロックが発動している証拠**

---

```bash
# 確認用コマンドまとめ
sudo fail2ban-client status
sudo fail2ban-client status sshd
sudo fail2ban-client set sshd unbanip [IP]
```

---
