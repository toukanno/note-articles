# GitHub Pagesで公開する方法

GitHub Pagesを使えば、無料で静的サイトを公開できます。

## 手順

### 1. リポジトリを作成する

GitHubで新しいリポジトリを作成します。公開サイトにする場合は **Public** に設定してください。

### 2. HTMLファイルを追加する

リポジトリのルートに `index.html` を配置します。

```html
<!DOCTYPE html>
<html>
<head><title>My Site</title></head>
<body><h1>Hello!</h1></body>
</html>
```

### 3. GitHub Pagesを有効にする

1. リポジトリの **Settings** に移動
2. 左メニューの **Pages** を選択
3. **Source** で公開するブランチを選択
4. **Save** をクリック

### 4. 公開を確認する

数分後、以下のURLでサイトにアクセスできます：

```
https://<ユーザー名>.github.io/<リポジトリ名>/
```

## Tips

> GitHub Pagesは静的サイトのみ対応しています。サーバーサイドの処理は実行できません。

| 項目 | 内容 |
|------|------|
| 料金 | 無料 |
| カスタムドメイン | 対応 |
| HTTPS | 自動対応 |
| ビルドツール | Jekyll（デフォルト） |

## まとめ

GitHub Pagesは手軽にサイトを公開できる便利なサービスです。Markdownと組み合わせれば、ブログのようなサイトも簡単に作れます。
