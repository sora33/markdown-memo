# 要件定義書

## 我々の動き方
### 開発手法
- テスト駆動開発を採用します。開発前に、テストケースを作成し、それらを満たすアプリケーションを開発することで、設計段階で認識合わせた通りの仕様に実装できます。
  - テスト駆動開発のメリット:
    - 仕様の理解度が上がる
    - 未来のリファクタリングや拡張が容易
    - 高いコード品質を維持

### メンバー
- PM
  - 土屋
  - 平沼
- エンジニア
  - 福地（バックエンド・インフラ）
  - 小田（フロントエンド）

```mermaid
flowchart TD
  style Client fill:#f9dcd4,stroke:#f08a81,color:#f08a81
  style Engineering fill:#d4e4fc,stroke:#6b9ce8,color:#6b9ce8
  style ProjectManagement fill:#d7f2dc,stroke:#68bf71,color:#68bf71
  
  subgraph Client[クライアント]
    先方[fa:fa-user 胡本]
  end
  
  subgraph Engineering[エンジニアリングチーム]
    フロントエンド[fa:fa-user 小田] <--> バックエンド[fa:fa-user 福地]
  end
  
  subgraph ProjectManagement[プロジェクト管理]
    PM1[fa:fa-user 土屋]
    PM2[fa:fa-user 平沼]
  end

  Client <--> ProjectManagement
  Engineering <--> PM2
```

## スケージュール
### 各工程とスケジュール
1. 10/1~10/7：要件定義
2. 10/8~10/14：システム設計、shopifyの仕様調査
3. 10/15~10/21：開発
4. 10/22~10/28：開発〜テスト
5. 10/29~10/31：動作検証・修正、納品
```mermaid
gantt
    title 開発スケジュール
    dateFormat YYYY-MM-DD
    axisFormat %d
    todayMarker off

    section 計画
    要件定義      :2023-10-01, 7d
    システム設計 :2023-10-08, 7d
    開発             :2023-10-15, 7d
    開発〜テスト :2023-10-22, 7d
    納品 :2023-10-29, 3d
```

### スケジュール通り行かなかった時の対処方法
- リスク管理: プロジェクトの進捗に応じてリソースの調整や優先順位の見直しを行うことで、予期せぬ遅延に柔軟に対応します。
  - 週次ミーティングで進捗状況を確認
  - 外部要因や技術的な課題への対応
  - リソース不足時には、土屋と平沼で巻き取ります。

## システム要件
### 概要
- Shopifyでの商品管理は全てタグで管理されているため、商品のタグを一括で編集出来るアプリケーション
- Shopify登録の全商品のデータを取得し、情報の変更・更新、その後差分のみの一括更新を行う。

### アーキテクチャ 

```mermaid
graph TD
    Shopify[Shopify]
    User[ユーザー]
    App[アプリケーション]
    DB[自社サーバーDB]
    Interface[ユーザーインターフェース]

    Shopify -- 商品データ --> App
    User -- タグ編集 --> App
    App -- データ取得/更新 --> DB
    App -- 提供 --> Interface
```
### モジュール
```mermaid
graph TD
    Interface[ユーザーインターフェース]
    TagEditor[タグ編集機能]
    DataFetcher[データ取得機能]
    DataUpdater[データ更新機能]

    Interface --> TagEditor
    Interface --> DataFetcher
    DataFetcher --> DataUpdater
```
### 機能要件
- 商品管理: Shopifyに登録されている全商品のデータを一括で取得し、タグ情報を編- 集。
データ更新: 編集された商品のタグ情報の差分のみを一括で更新。
- データベース設計: 提供されるデータベースの設計図に従い、必要なデータ構造を実現。

#### 商品リクエストからデータ取り込みまでのフロー
```mermaid
sequenceDiagram
    User->>System: リクエストボタンを押下
    System->>Shopify: 商品リストリクエスト
    Shopify-->>System: 商品データ作成中
    System->>User: スピンで稼働中表示
    User->>System: リフレッシュボタン押下 or ポーリング
    Shopify-->>System: 商品データをJSON形式で作成完了
    System->>Database: リクエストIDとステータスを保存
    System->>User: 取り込みボタンの有効化
    User->>System: 取り込みボタンを押下
    System->>Database: 商品情報を登録
    Database-->>System: 登録完了 & 現在の商品数を返却
    System->>User: 商品数を表示
```
#### 商品情報の変更と一括反映
```mermaid
sequenceDiagram
    User->>System: タグ選択 & 条件設定
    System->>Database: 商品情報検索
    Database-->>System: 検索結果
    System->>User: 検索結果の表示
    User->>System: 商品選択 & タグの編集
    System->>Database: 商品情報の更新
    Database-->>System: 更新完了
    System->>User: 更新された情報の表示
    User->>System: 更新ボタン押下
    System->>Database: 更新対象の確定
    System->>Shopify: タグ情報の一括反映
    Shopify-->>System: 反映完了
    System->>Database: 反映ステータスの保存
    System->>User: 反映結果の通知
```

### 非機能要件
- 応答時間: 95%のリクエストが2秒以内に応答し、6秒を超えるリクエストは許容しない。
同時利用者数: アプリケーションの同時接続数は最大で100。
- 負荷テスト: 1時間にわたる1000リクエスト/秒の負荷を持続的に処理し、平均応答時間は3秒以内。
- データベースパフォーマンス: クエリの最大実行時間は1秒以内で、同時アクセス数は1000を超えない。
