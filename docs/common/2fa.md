# 多要素認証の設定方法

!> 2024年12月01日より、Shinonome Education Cloudでは多要素認証が必須になります

## 概要

Shinonome Education Cloudでは、セキュリティを最重要事項として位置づけております。
お客様のアカウントをお守りする観点から、2024年12月01日より多要素認証による追加認証が必要になります。

本項目では、多要素認証の設定方法を説明します。

## 多要素認証とは

「多要素認証(2FA / 二段階認証)」とは、パスワードとは別にワンタイムコード(60秒ごとに変わる認証用コード)を追加入力する認証方式で、パスワードが外部に漏洩した場合でも不正アクセスを防ぐことが出来ます※。

※ 100%の不正アクセス防止を保証するものではありません

## 設定方法

### (初回のみ) 多要素認証用のスマートフォンアプリを設定

#### Google Authenticatorのインストール

?> 既にGoogle Authenticatorをインストールしている場合は、この手順は不要です。

多要素認証における、ワンタイムコードを生成するスマートフォンアプリをダウンロードします。
様々なメーカーからワンタイムコードを生成するアプリが提供されていますが、ここでは「Google Authenticator」による設定方法を紹介します。

まず、App StoreもしくはGoogle Play Storeから「Google Authenticator」をダウンロードします。

##### ダウンロードURL

| OS | URL |
| --- | --- |
| iOS | https://apps.apple.com/jp/app/google-authenticator/id388497605 |
| Android | https://play.google.com/store/apps/details?id=com.google.android.apps.authenticator2&hl=ja |

#### アプリケーション設定

多要素認証が設定されていない状態で、Shinonome Education Cloudにログインすると、以下の画面(QRコード)が表示されます。

![alt text](image-1.png ':size=500')

Google Authenticatorを起動し、「コードを追加」ボタンをタップします。

![alt text](IMG_0547.PNG ':size=500')

「QRコードをスキャン」をタップすると、QRコードをスキャンする画面が表示されますので、画面に表示されているQRコードをスキャンしてください。

![alt text](IMG_0548.PNG ':size=500')

スキャンすると、6桁のワンタイムコードが表示されます。

![alt text](IMG_0549.PNG ':size=500')

このコードをShinonome Education Cloudの画面に入力し、「続ける」ボタンをクリックしてください。
その後、表示されているコードをメモ帳などに記録し、安全な場所に保管してください。(多要素認証で使用しているスマートフォンが故障した場合などに必要になります)
記録後に、「続ける」ボタンをクリックしてください。

![alt text](image-2.png ':size=500')


以上で、多要素認証の設定が完了です。