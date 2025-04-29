# Ubuntu Server + Nginx構築練習まとめ

---

## ✅ ゴール
- 仮想環境（VirtualBox）にUbuntu Server 22.04.5 LTSをインストール
- Nginxサーバーを構築し、ブラウザからアクセス確認
- 自作HTMLページを公開
- 仮想マシンを安全にシャットダウン

---

## ✅ 実施した手順

### 1. VirtualBox仮想マシン作成
- メモリ：2048MB
- 仮想ディスク：20GB（SATA接続）
- ISOファイル：ubuntu-22.04.5-live-server-amd64.iso
- NAT接続＋ポートフォワーディング設定（ホスト8080→ゲスト80）

### 2. Ubuntu Serverインストール
- English選択
- キーボード：English(US)
- ネットワーク：自動取得（DHCP）
- ディスク設定：自動（Use an entire disk）
- OpenSSH server導入
- Snapアプリ選択なし（スキップ）

### 3. Nginxインストール
```bash
sudo apt update
sudo apt install nginx
```

### 4. Nginx動作確認
- 仮想マシン内でNginx起動確認
```bash
systemctl status nginx
```
- ブラウザでアクセス
```
http://localhost:8080
```
- 「Welcome to nginx!」表示確認

### 5. オリジナルページ作成
- `/var/www/html/`ディレクトリに移動
```bash
cd /var/www/html
```
- 新しい `index.html` 作成
```bash
sudo nano index.html
```
- HTML内容（例）
```html
<!DOCTYPE html>
<html>
<head>
    <title>My First Server</title>
</head>
<body>
    <h1>Hello World from My Ubuntu Server!</h1>
</body>
</html>
```
- ブラウザリロードでオリジナルページ表示確認！

### 6. 仮想マシンのシャットダウン
```bash
sudo shutdown now
```
または
```bash
sudo poweroff
```

---

## ✅ 理解ポイントまとめ

| 項目 | 理解したこと |
|:--|:--|
| localhostとは？ | 自分のPC自身へのアクセスを指す特別な名前 |
| ポートフォワーディングとは？ | ホストPCから仮想マシンへの通信中継設定 |
| Guest Additionsとは？ | コピペや画面サイズ調整などを可能にする仮想化ドライバ |
| 実務でHTMLを作る場所は？ | サーバー上ではなく、ローカルエディタ（VS Codeなど） |
| デプロイとは？ | 作ったコードを実際のサーバーに反映すること |

---
