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