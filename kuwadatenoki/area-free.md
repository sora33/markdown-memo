# エリアフリーの工数と見積り

## アプリケーションの概要
### 機能一覧
- 一般的なデータベースの操作
  - ユーザー登録、編集、一覧、削除
    - ガイド
    - 一般
    - 管理者
  - 通知機能
  - メッセージ機能
  - レビュー機能
- その他
  - 課金システム
  - googleマップapiで地図にピンを立てる
  - ビデオ通話、自動翻訳
  - 広告掲載（アフィ？）

## 決めること
- ①Webアプリ想定で、ビデオ通話を開発する際の想定工数
- ②ビデオ通話の際にリアルタイムで同時翻訳で字幕を付ける際の、実装は可能か、またその想定工数・費用感
- ③リアルタイム翻訳はどのくらいの時差でできるものか（話した内容が字幕で出るまでの時間）
- ④海外の人の検索に引っかかるようにする、海外の人にも利用してほしいとなったら、その国のサーバーをそれぞれ契約する必要があるか

### ①ビデオ通話を開発する際の想定工数
１.5人月
- SAAS使う
  - twilio？
  
#### サービス選定
- Twilio
- Skyway
- 

### ②ビデオ通話の際にリアルタイムで同時翻訳
１人月

### ③リアルタイム翻訳はどのくらいの時差？
５秒くらい？

### ④海外の人の検索に引っかかるようにする、海外の人にも利用してほしいとなったら、その国のサーバーをそれぞれ契約する必要があるか




## 工数 & 費用見積り
|項目|工数（人月）|費用（円）|補足|
|--|--|--|--|
|ビデオ機能|1.5|0~（従量課金）|twilio (他のSaaSでもOK)|
|翻訳機能|1|0~（従量課金）|DeepLかGoogle translate|
|広告|0.3|--|--|
|Push通知|0.5|--|--|
|課金|0.5|--|フロントとバックエンド|
|その他アプリケーション機能|2|10,000円（従量課金）|フロントとバックエンド|
|合計|--|--|--|

## 質問
- 通知どうしますか？めーる or Push通知的なの
- 広告どうしますか？アフィ？