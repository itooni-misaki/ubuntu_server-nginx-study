## 🔍 Nginxログの確認方法と分析

### 📁 ログファイルの場所

| ファイル | 説明 |
|----------|------|
| `/var/log/nginx/access.log` | アクセスログ（全リクエスト） |
| `/var/log/nginx/error.log` | エラーログ（設定ミス、拒否、ファイル未発見など） |

---

### 📌 基本コマンドと用途

```bash
sudo tail -n 10 /var/log/nginx/access.log      # アクセスログの末尾10行を表示
sudo tail -n 10 /var/log/nginx/error.log       # エラーログの末尾10行を表示
sudo tail -f /var/log/nginx/access.log         # アクセスログをリアルタイム監視
```

✋ `tail -f` は設定変更やアクセス制限テストのときに超便利！

---

### 📊 ログ活用テクニック

```bash
# ステータスコード403（アクセス拒否）を抽出
grep '403' /var/log/nginx/access.log

# 特定のIPアドレスからのアクセスを調査
grep '192.168.0.1' /var/log/nginx/access.log

# アクセスの多いIPをランキング表示
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

🧠 `awk` は1列目（IP）を抽出して頻度を集計 → 攻撃やボット判定に活用！

---

### 💡 補足：ログの構造イメージ

```plaintext
192.168.0.1 - - [06/May/2025:10:12:34 +0000] "GET /index.html HTTP/1.1" 200 512 "-" "Mozilla/5.0"
```

| 項目 | 内容 |
|------|------|
| IP | アクセス元（クライアント） |
| 日時 | リクエストが届いたタイミング |
| メソッド＋パス | リクエスト内容（GET /index.htmlなど） |
| ステータスコード | 200, 403, 404など |
| バイト数 | レスポンスのサイズ |
| User-Agent | ブラウザやクローラーの種類 |

---

### ✨ 理解のポイント

- **403ログ＝アクセス拒否の証拠**（deny/allow設定が効いてる）
- **404ログ＝存在しないファイルのアクセス**（try_filesの確認にも◎）
- `error.log`にはNginx自身のエラーも出るので設定ミス時に確認！

---

## 📦 Nginxのgzip圧縮設定と403アクセス拒否の対応ログ

---

### ✅ gzip圧縮の目的と有効化

#### 📌 設定ファイルの編集

```bash
sudo nano /etc/nginx/nginx.conf
```

`http { ... }` ブロック内に以下を追加：

```nginx
gzip on;
gzip_disable "msie6";
gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
```

---

#### 🔁 設定の反映手順

```bash
sudo nginx -t                  # 構文チェック
sudo systemctl reload nginx    # 設定を反映
```

✋ `nginx -t` はあくまでチェック！リロードしないと反映されない！

---

#### 🔍 圧縮確認コマンド

```bash
curl -H "Accept-Encoding: gzip" -I http://localhost:8080
```

期待されるレスポンス例：

```plaintext
HTTP/1.1 200 OK
Content-Encoding: gzip
```

---

### ❗ トラブル：403 Forbiddenが返る

#### 🧠 原因と分析

アクセス制御（`allow / deny`）設定が有効で、許可されていないIPからのアクセスだったため。

`access.log`を確認：

```bash
sudo tail -n 20 /var/log/nginx/access.log
```

例：

```plaintext
127.0.0.1 - - [日付] "GET / HTTP/1.1" 403 ...
```

---

#### 🛠️ 対応内容（アクセス制御の修正）

```nginx
location / {
    allow 127.0.0.1;
    deny all;

    try_files $uri $uri/ =404;
}
```

→ `127.0.0.1`（ゲストOS自身）からのアクセスのみ許可

---

### ✅ 最終動作確認結果

```bash
curl -H "Accept-Encoding: gzip" -I http://localhost:8080
```

```plaintext
HTTP/1.1 200 OK
Content-Type: text/html
Content-Encoding: gzip
```

🟢 gzip圧縮＋アクセス許可の両方が成功！

---

### ✨ ポイントまとめ

- **nginx.conf**と**サイト設定ファイル**の両方を編集する可能性あり
- `nginx -t` で確認 → `systemctl reload` で反映が鉄則
- `403`が出たら `access.log` でIPを確認するクセをつけよう！

---
## 🔁 Nginxで301リダイレクトを設定する方法

---

### ✅ 目的

`/old` にアクセスされた場合に `/new` へ恒久的にリダイレクト（301）させる。

---

### 🛠️ 設定手順

#### 🔧 対象ファイル：

```bash
sudo nano /etc/nginx/sites-enabled/default
```

#### 🔧 serverブロック内に以下を追加：

```nginx
server {
    listen 8080 default_server;
    listen [::]:8080 default_server;

    server_name localhost;

    # 🔁 リダイレクト設定
    location /old {
        return 301 /new;
    }

    # 🔍 通常ルート（必要に応じて）
    location / {
        try_files $uri $uri/ =404;
    }
}
```

---

### 🔁 動作確認コマンド

```bash
curl -I http://localhost:8080/old
```

#### ✅ 期待される出力：

```plaintext
HTTP/1.1 301 Moved Permanently
Location: http://localhost:8080/new
```

---

### 📚 解説メモ

| 項目 | 内容 |
|------|------|
| `301` | 恒久的リダイレクト。検索エンジンに新URLとして認識される |
| `return 301 /new;` | 指定されたパスに即時リダイレクト |
| `Location`ヘッダ | クライアント側がリダイレクト先を判断するための情報 |

---

### ✨ ポイントまとめ

- `location /old { return 301 /new; }` は**serverブロック内**に記述
- `server_name` は `server {}` 直下に書くこと（`location` の外）
- `curl -I` を使うとヘッダだけ取得でき、リダイレクトの確認に最適

---
