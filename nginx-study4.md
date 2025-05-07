## 🔐 Nginxで自己署名SSLを使ったHTTPS化（ローカル環境）

---

### ✅ 実施目的

- グローバルドメインがなくても、HTTPS通信の暗号化を体験
- 自己署名SSLを使って、NginxのSSL設定方法と挙動を学ぶ

---

### 🛠️ ステップ①：自己署名SSL証明書の作成

#### 🔧 秘密鍵と証明書を作成（有効期限365日）

```bash
sudo mkdir -p /etc/nginx/ssl
cd /etc/nginx/ssl

sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout selfsigned.key \
  -out selfsigned.crt
```

#### 🔍 できたファイルの確認

```bash
ls -l /etc/nginx/ssl
```

```plaintext
selfsigned.crt  selfsigned.key
```

---

### 🛠️ ステップ②：NginxにSSL設定を追加

#### ✏️ `/etc/nginx/sites-enabled/default` にSSL用のブロック追加

```nginx
server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate     /etc/nginx/ssl/selfsigned.crt;
    ssl_certificate_key /etc/nginx/ssl/selfsigned.key;

    root /var/www/mysite;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

#### 🔁 設定反映

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

### 🧪 ステップ③：`curl` でHTTPS動作確認

#### 🔍 証明書エラーを無視してアクセス（`-k`）

```bash
curl -k -I https://localhost
```

#### ✅ 期待されるレスポンス

```plaintext
HTTP/1.1 200 OK
Server: nginx/1.18.0 (Ubuntu)
...
```

---

### 💡 curlの補足

| オプション | 説明 |
|------------|------|
| `-I` | ヘッダだけ表示（`HEAD`リクエスト） |
| `-k` | 自己署名証明書の警告を無視して接続 |
| `https://localhost` | 接続先：自分のPC上のNginx（SSL対応済） |

---

### ✨ ポイントまとめ

- 自己署名SSLでもHTTPSの暗号化体験が可能
- `curl -k` で動作確認すれば、ブラウザなしでも検証OK
- 本番環境ではLet's Encryptなどの**信頼された証明書**が必要になる

---
## 🔐 自己署名SSL証明書でHTTPS通信＋HTTP→HTTPSリダイレクト設定（Nginx）

---

### ✅ 実施目的

- SSL通信の暗号化処理を体験
- HTTPアクセスをHTTPSへ自動転送する構成を作成
- `curl`での動作確認で、リダイレクトも含めた挙動を確認

---

### 🛠️ Nginx設定（HTTPSブロック）

```nginx
server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate     /etc/nginx/ssl/selfsigned.crt;
    ssl_certificate_key /etc/nginx/ssl/selfsigned.key;

    root /var/www/mysite;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

---

### 🔁 HTTPアクセス時にHTTPSへ転送（リダイレクト）

```nginx
server {
    listen 80;
    server_name localhost;

    return 301 https://$host$request_uri;
}
```

- `$host` → リクエストされたホスト名（localhostなど）
- `$request_uri` → パス部分（例：`/index.html`）

---

### 🔄 設定反映コマンド

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

### 🧪 動作確認：curlでHTTPSとリダイレクトをチェック

#### 🔍 HTTPSで直接アクセス（自己署名を無視）

```bash
curl -I -k https://localhost
```

#### 🔁 HTTPでアクセスし、HTTPSに転送されるか確認

```bash
curl -I -k http://localhost
```

#### ✅ 期待レスポンス（どちらにも出る）

```plaintext
HTTP/1.1 301 Moved Permanently
Location: https://localhost/
```

---

### 💡 補足：`-k`オプションの意味

| オプション | 内容 |
|------------|------|
| `-I` | HTTPヘッダのみ取得（HEADリクエスト） |
| `-k` | 自己署名証明書のエラーを無視して通信 |

---

### ✨ ポイントまとめ

- 自己署名SSLでもHTTPSの仕組みは完全に体験できる
- `curl -k`を使えばSSL警告を回避して通信確認が可能
- HTTPからHTTPSへの301リダイレクトもNginxでシンプルに設定可能
- 実サービスでは Let's Encrypt や CA認証の証明書を使うのがベスト

---
