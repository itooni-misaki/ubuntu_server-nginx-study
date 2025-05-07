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
## 🔁 rewriteによる動的301リダイレクト設定（Nginx）

---

### ✅ 目的

`/old/*` でアクセスされたURLを、同じ構造を保ったまま `/new/*` に恒久的リダイレクト（301）させる。

---

### 🛠️ 設定手順

#### 📁 対象ファイル

```bash
sudo nano /etc/nginx/sites-enabled/default
```

#### 📄 記述内容（serverブロック内）

```nginx
location /old/ {
    rewrite ^/old/(.*)$ /new/$1 permanent;
}
```

---

### 🔄 設定の反映

```bash
sudo nginx -t                  # 設定構文チェック
sudo systemctl reload nginx    # 設定反映
```

---

### 🔍 動作確認コマンド

```bash
curl -I http://localhost:8080/old/sample.html
```

#### ✅ 期待されるレスポンス

```plaintext
HTTP/1.1 301 Moved Permanently
Location: http://localhost:8080/new/sample.html
```

---

### 💡 解説メモ

| 項目 | 内容 |
|------|------|
| `rewrite` | 正規表現で `/old/〜` を `/new/〜` に変換 |
| `$1` | `(.*)` のキャプチャ部分（元のパス）を保持 |
| `permanent` | ステータスコード `301` を指定するキーワード |
| `/old/` に `/` がついている理由 | `/oldabc` のような誤マッチを防ぐため（ディレクトリ限定） |

---

### ✨ ポイントまとめ

- rewriteはURL構造をそのまま引き継ぎたいときに便利
- `return`は固定先に飛ばしたいときに使い分ける
- `curl -I` を使えばヘッダのみでリダイレクトの有無を確認できる

---
## 🔁 Nginxにおける `return 301` と `rewrite` の違いと使い分け

---

### ✅ 基本の目的

どちらも **リダイレクト（恒久的な転送）** を行う機能だが、  
「**URL全体を固定先に転送する**」か「**構造を保って別パスに転送する**」かに違いがある。

---

### 🅰️ `return 301 /new;` の特徴

```nginx
location /old {
    return 301 /new;
}
```

| 項目 | 内容 |
|------|------|
| 転送先 | 常に `/new`（固定） |
| パス保持 | なし（元のURLは無視される） |
| 処理性能 | 軽くて高速 |
| 管理しやすさ | 単純でミスが少ない |
| 向いている場面 | ページ単位の完全移転、案内所的なリダイレクト |

#### 🏠 たとえ話：

> 「旧建物 `/old` の各部屋から来た人を、**新しい1つの案内所 `/new` に集める**」

---

### 🅱️ `rewrite ^/old/(.*)$ /new/$1 permanent;` の特徴

```nginx
location /old/ {
    rewrite ^/old/(.*)$ /new/$1 permanent;
}
```

| 項目 | 内容 |
|------|------|
| 転送先 | 動的（元のパスを保持） |
| パス保持 | あり（柔軟なURL対応） |
| 処理性能 | 少し重め（正規表現あり） |
| 管理しやすさ | 大規模構成に強い |
| 向いている場面 | ディレクトリ単位の引っ越し、構造ごとの維持が必要な移転 |

#### 🏠 たとえ話：

> 「旧建物 `/old/301号室` の住人を、**新建物 `/new/301号室` に案内する**」

---

### ✨ 選び分けのポイント

| 用途 | 使うべきディレクティブ |
|------|------------------------|
| 単発URLのリダイレクト | `return 301` |
| URL構造の一括移転 | `rewrite` |
| シンプルな案内所的動作 | `return` |
| 旧URLの履歴を保ちたいとき | `rewrite` |

---

### 💡 補足：どちらも SEO 効果はある？

- **Yes！** どちらも `301`（恒久リダイレクト）であれば検索エンジンには「新しいURLに評価を引き継いでほしい」という意味になる。
- 違いはあくまで**構造と管理しやすさ、性能面**。

---

### 📌 まとめ

- `return` は「どんなアクセスでも1か所に転送」＝簡単明快
- `rewrite` は「構造を保った転送」＝柔軟で拡張性あり
- 実運用では「併用」もアリ！（例：全体はrewrite、一部だけreturn）

---
## 📊 journalctl と tail / cat の違いまとめ（ログ監視・分析）

---

### ✅ 基本の考え方

| ツール | 説明 |
|--------|------|
| `tail`, `cat` | テキストファイルとしてログを直接読み取るCLIツール |
| `journalctl` | systemdが保持する**バイナリ形式のログ**を整形して表示する専用ツール |

---

### 🛠️ それぞれのログ元

| 比較項目 | tail / cat | journalctl |
|----------|------------|------------|
| ログの実体 | `.log`ファイル（プレーンテキスト） | systemdの**バイナリ形式ログ** |
| 書き出し先 | `/var/log/nginx/access.log` など | `/var/log/journal/` 配下のバイナリ |
| 読み方 | 直接ファイルを読む | `journalctl` コマンドが内部処理で整形表示 |
| 主な用途 | リアルタイム監視 / grepで抽出 | サービスエラー / systemdログ調査 |

---

### 💡 たとえで理解する

| ツール | たとえ話 |
|--------|-----------|
| `tail` / `cat` | 「机の上に置かれたメモを直接読む」 |
| `journalctl` | 「司書に“nginxの今日の履歴出して”と頼んで、本棚から時系列で出してもらう」 |

---

### 🧪 使用例

```bash
# Nginxログをリアルタイムで監視（tail）
sudo tail -f /var/log/nginx/access.log

# Nginxのサービスログを過去も含めて確認（journalctl）
sudo journalctl -u nginx

# 今日のNginxログだけ抽出
sudo journalctl -u nginx --since today

# エラー調査：詳細モードで追跡
sudo journalctl -xe
```

---

### ✨ ポイントまとめ

- `tail`はファイルログ（nginxなど個別アプリ）に強い
- `journalctl`はsystemd管理のサービス全体の履歴を対象にできる
- **ログ形式の違い：テキスト vs バイナリ**
- `journalctl`の方が検索・フィルタが強力で、起動トラブル時の調査に最適！

---
