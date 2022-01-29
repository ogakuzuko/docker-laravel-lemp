## 前提
- VSCodeをインストールする（[公式](https://azure.microsoft.com/ja-jp/products/visual-studio-code/)）
- Docker Desktop for Mac(or Windows)をインストールする（[公式](https://www.docker.com/products/docker-desktop)）
  - Windows用参考：[【Docker Desktop】Windowsにインストール（WSL2）](https://chigusa-web.com/blog/windows%E3%81%ABdocker%E3%82%92%E3%82%A4%E3%83%B3%E3%82%B9%E3%83%88%E3%83%BC%E3%83%AB%E3%81%97%E3%81%A6python%E7%92%B0%E5%A2%83%E3%82%92%E6%A7%8B%E7%AF%89/)

## 自分のPC上にクローンするまで
```bash
# GitLabからプロジェクトをクローン
git clone {プロジェクトURL}

# クローンしたプロジェクトフォルダに移動
cd docker-laravel-lemp
```

## 環境構築手順
以下のコマンドを順に実行して実行環境を構築する

```bash
# srcディレクトリを作成
mkdir src

# 環境変数ファイルをコピー
cp .env.example .env

# Dockerfileからイメージをビルド
docker compose build --no-cache

# コンテナを起動
# → このコマンド実行後、3つのコンテナがちゃんと起動しているか確認する（GUIもしくは`docker compose ps`など）
docker compose up -d

# Laravelをインストールするため、appコンテナに入る（/docker-laravel-lempディレクトリで実行）
# → 以下のコマンドを打つと「root@~~~:/var/www/html」といったような表示に切り替わるはず（ちゃんと切り替わっていればコンテナ内部に入れている証拠）
docker compose exec app bash

=========================
# このタイミングでこれから使うものがちゃんとあるか一応確認
# Composerの確認 -> Composer version 2.0.14 2021-05-21 17:03:37
composer -v

# npmの確認 -> 6.14.6
npm -v
=========================

# Laravel最新版のインストール（appコンテナの中に入っている状態（=表示が「root@~~~:/var/www/html」となっている状態）で実行する）
composer create-project --prefer-dist laravel/laravel .

# Laravelバージョン6系をインストールしたい場合はこちら
composer create-project --prefer-dist "laravel/laravel=6.*" .
```

## 動作確認
### webサーバーへのアクセス
URL: http://localhost:80  
※現状`*.blade.php`ファイルはホットリロードには対応していないみたい（=コードを変更したら更新ボタン押さないと画面に変更内容が反映されない）

### phpMyAdminへのアクセス
URL: http://localhost:8080 


## ホットリロードを設定したい場合
Dockerのappコンテナ内で以下の作業を実行する  
（※ローカルにNodeが入っている場合はローカルから入れてしまってもコンテナ内に同期されるのでどちらもでよい）  
（※後で判明。恐らくこれローカル環境にNode.jsが入ってないと動かせないかも…？）
```bash
# 前提
現在ターミナルの表示が「root@~~~:/var/www/html」となっていて、ちゃんとappコンテナの中に入れていることを確認する
現状コンテナ内のこの場所がローカルの`/src`ディレクトリとリンクしている（Laravelがインストールしてある場所）

# 「root@~~~:/var/www/html（=/srcとリンクしている）」で以下のコマンドを打って必要パッケージをインストール
npm install browser-sync browser-sync-webpack-plugin


========== appコンテナ内の作業はここで終わり！これ以降は/srcディレクトリ内で作業！ ==========


# src/webpack.mix.jsを以下のように編集
mix.js('resources/js/app.js', 'public/js')
    .sass('resources/sass/app.scss', 'public/css')
    .browserSync({
        files: ["resources/**/*", "public/**/*"],
        proxy: {
            target: "http://127.0.0.1:80",
        },
    });

# ホットリロード起動（/srcディレクトリで実行）
# ※この時Dockerのwebコンテナも立ち上げ済みであること（docker compose up -d）
npm run watch

# ホットリロードに対応した画面を表示する（恐らく上記コマンドで勝手に開くはず）
http://localhost:3000

# ここまでで上手く行かなかった場合、Docker内のappコンテナ内に入って`npm run watch`コマンドを実行してみると直るかも？
→ Dockerコンテナ内からブラウザの localhost:3000 にアクセスできない？みたいでダメそう。
・Nodeインストール（Mac）：https://volta.sh/
・Nodeインストール（Windows）：https://qiita.com/echolimitless/items/83f8658cf855de04b9ce
```

## `php artisan 〇〇`コマンドを打つ時は…
ローカル環境(コンテナ外部)にはPHPを実行する環境がないため、今後頻出する`php artisan 〇〇`コマンドを打つ際は、一度`appコンテナ`に入ってからコマンドを実行するようにする。  
```bash
# `php artisan 〇〇`コマンドを打つ際は一度 appコンテナ に入ってから実行する
docker compose exec app bash

# `php artisan 〇〇`コマンドの具体例（一部）
php artisan make:model Todo --migration # マイグレーションファイル作成＆Todoモデルの作成
php artisan migrate # マイグレーション処理の実行
php artisan make:controller TodosController # Todoコントローラーの作成
php artisan make:seeder TodoTableSeeder # シーダーファイルの作成
php artisan db:seed # シーダーファイルをもとにシーディング処理を実行
php artisan tinker # Laravelを対話的に動かすことが出来るようになるコマンド（参考：https://inouelog.com/laravel-tinker/）
```

## 開発注意点
- PHPは文末のセミコロン（`;`）を省略するとエラーとなるので注意！

## その他
以下は必要に応じて使うコマンド
```bash
# コンテナを停止&破棄
docker compose down

# コンテナの実行状況を確認
docker compose ps

# appコンテナの中（=「root@~~~:/var/www/html」）から抜ける方法
`exit`と打つ。もしくは、`Control+D`を入力（Macの場合）

# 全てやり直したい場合
/srcディレクトリを丸ごと削除する → その後ルートディレクトリで手順1の `mkdir src` からやり直そう！
```

## 実践: LaravelでTodoリストを作ろう！
- [Laravelで超シンプルなToDoアプリを作成する](https://qiita.com/s_harada/items/39cfbd64a6df4f2ccf5d)
- [【Laravel】Todoリストアプリの作成](https://qiita.com/JUM22676603/items/17beb3d4d71f5774df64)
- [Laravel 6 で簡単なタスクリストを作ってみた](https://blog.maro.style/post-1115/)