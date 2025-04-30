---

## 🤔 `root` ディレクティブの本質とlocalhost:8080の関係

---

### ✅ 現状の構成を整理

- 昨日いじっていたHTMLは `/var/www/html/index.html`
- 今日は `/var/www/mysite/index.html` に変更して表示されるようにした
- `echo ... | sudo tee` は簡易的にHTMLを作成するための手段（ファイル出力）

---

### 🤔 今やっていることは「複数サイト運用」ではないの？

✅ **違う！**  
今の段階でやっているのはあくまで：

> **「http://localhost:8080 にアクセスしたとき、表示されるHTMLファイルを変えたい」**

そのために `root` ディレクティブを編集しているだけ。

```nginx
# 初期状態
root /var/www/html;

# 今回変更した状態
root /var/www/mysite;
```

これは「**Nginxが探しに行く先のディレクトリを変えている**」ということ。

---

### 🤔 「localhost:8080」って変えられないの？

✅ ホスト側から見ると「変えられないように見える」けど、実際はこう：

| 要素 | 実際の仕組み |
|:--|:--|
| `localhost` | VirtualBoxホストPC側の127.0.0.1 |
| `8080` | VirtualBoxがポートフォワーディングしている（ホスト8080 → ゲスト80） |
| `nginx` | ゲスト側のUbuntuでは常に `listen 80`（つまり通常のWebポート）で待機中 |

---

### 🤔 `/var/www/html` にしか置けないの？

❌ そんなことはない！

- 任意のPATHに `mkdir` して、そこにHTMLを置くことは可能
- ただし `root` を変えないと、Nginxは `/var/www/html` しか見に行かない

---

### ✅ 方法1 vs 方法2 の再整理

| 方法 | 内容 | 限界・自由度 |
|:--|:--|:--|
| 方法1：index.htmlだけ書き換える | `/var/www/html/index.html` を直接編集 | 簡単だけど、表示ディレクトリは `/html` に固定 |
| 方法2：root を変更する | `root /var/www/mysite` に変更してHTML配置場所を変える | ルーティング自由度高いが、Nginx設定編集が必要 |

---

### 🤔 じゃあ「複数サイト運用にrootが必要不可欠」はなんで出てきたの？

🧠 それは、「将来的にバーチャルホスト構成を作るとき、**各サイトごとに `root` を指定する必要がある**」から。

例：

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/example.com;
}

server {
    listen 80;
    server_name blog.example.com;
    root /var/www/blog;
}
```

- 今はまだ1サイト構成（`localhost:8080`）の中での練習
- でも **複数サイト運用時にも `root` は必須**
- だから「rootの使い方を先に理解しておく」＝将来必要不可欠なスキル！

---

### ✅ まとめ：今の学習ステップの意図

| 学習内容 | 意図 |
|:--|:--|
| `root` を編集してHTMLの表示場所を変える | 単サイト構成でのファイル配置を理解するため |
| `/var/www/html` → `/var/www/mysite` に変更 | 配置の自由度を体感するため |
| バーチャルホスト構成ではない | あくまで1サイト（localhost）での練習中 |
| 今後の応用でrootが活きる | サイトごとに表示先を分ける実務スキルの土台になる |

---
---

## 🚧 実践：mysite.local を使って独自ホスト名でアクセスする設定

---

### 🎯 目的

Nginxで複数サイト運用を想定し、  
**`http://localhost:8080` ではなく `http://mysite.local:8080` に切り替える**構成を試す。

---

### 🧱 実施内容（構成）

| 項目 | 内容 |
|:--|:--|
| アクセスURL | `http://mysite.local:8080` |
| 物理サーバー | VirtualBox内の Ubuntu Server |
| 表示HTML | `/var/www/mysite/index.html` |
| 使用Webサーバー | Nginx（仮想マシン内） |
| 名前解決の仕組み | `/etc/hosts` + `server_name` 設定 |

---

### 🛠️ 手順まとめ

#### ① 新しいルートディレクトリとHTML作成

```bash
sudo mkdir -p /var/www/mysite
echo '<h1>hello mysite</h1>' | sudo tee /var/www/mysite/index.html
```

#### ② Nginxの仮想ホスト設定を修正

```bash
sudo nano /etc/nginx/sites-available/default
```

👇 以下を追記または書き換え（他のserverブロックと重複しないように注意）：

```nginx
server {
    listen 80;
    server_name mysite.local;

    root /var/www/mysite;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

#### ③ Ubuntu側の `/etc/hosts` に名前登録

```bash
sudo nano /etc/hosts
```

```plaintext
127.0.0.1    mysite.local
```

#### ④ Nginx 設定チェック＆再起動

```bash
sudo nginx -t
sudo systemctl reload nginx
```

---

### 🔍 補足：この時点では「mysite.local」はまだWindows側で解決できない

→ 対応として、Windowsの `hosts` にも以下を追加：

```plaintext
127.0.0.1    mysite.local
```

---

### ✅ 確認手順

- ブラウザで `http://mysite.local:8080` にアクセス
- `<h1>hello mysite</h1>` が表示されればOK！

---

---

## ✅ 名前解決エラーと `hosts` ファイルの重要性

### ✋ 問題の発端

ローカル環境で `http://mysite.local:8080` にアクセスしても表示されない問題が発生。

---

### ❓ なぜ起きたのか？

| ステップ | 動作内容 | 結果 |
|:--|:--|:--|
| 1 | ブラウザが「mysite.local にアクセスしたい！」とOSに問い合わせる | Windowsが「そんな名前知らない！」と返す |
| 2 | OSがDNS（ルーターやISP）に問い合わせる | mysite.local はグローバルDNSに登録されていないので失敗 |
| 3 | → 名前解決できず、サイトが開けない | ❌ アクセス失敗（ERR_NAME_NOT_RESOLVEDなど） |

---

### 💡 補足：DNSの前にまず参照されるのが `hosts` ファイル！

Windowsは以下の順で名前解決を行う：

1. `C:\Windows\System32\drivers\etc\hosts`
2. ネットワークのDNS（ルーター／プロバイダ）
3. 最後にエラー

---

### ✅ 解決方法

#### 🛠️ Windows側の `hosts` に手動で追記！

```plaintext
127.0.0.1    mysite.local
```

📍これにより、ブラウザが `mysite.local` を使ってアクセスすると、
自動的に `127.0.0.1`（＝このPC）に送られるようになる。

---

### ✅ Ubuntu 側と合わせて整理

| 設定場所 | 意味 | 実行済み |
|:--|:--|:--:|
| `/etc/nginx/sites-available/default` | `server_name mysite.local` を追加 | ✅ |
| `/etc/hosts`（Ubuntu） | Nginxが `mysite.local` を自分で理解するため | ✅ |
| `/etc/hosts`（Windows） | ブラウザが `mysite.local` に行くときの地図 | ✅ |
| `nginx -t` ＆ `systemctl reload nginx` | 設定反映 | ✅ |

---

### 🔍 補足ポイントまとめ

- `mysite.local` のようなローカルドメインはインターネットには存在しない
- **ブラウザやVS Codeなどは、Windows側のDNSルートを使う**
- だから**自分のPCに「これは127.0.0.1だよ」と教えてあげる必要がある**

---

### 📦 イメージ図（名前解決）

```text
ブラウザ入力 → mysite.local:8080
   ↓
OSに問い合わせ
   ↓
[✓] hosts ファイルに登録 → 127.0.0.1 に変換
   ↓
Nginx が受け取って `/var/www/mysite/index.html` を表示！
```

---

