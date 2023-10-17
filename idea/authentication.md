# 認証方法について
本サービスは、エンジニアのみに登録して頂けるように、ログインは、Github認証のみにしよう考えました。

Deviseのomniauthを使った認証、NextAuth.jsを使った認証　の２つで悩みました。
結論、NextAuth.jsの方がシンプルで効率的と思ったので、そちらを採用しました。

## devise-token-auth, omniauth-githubなどを使った認証
### フロー
```mermaid
sequenceDiagram
    ユーザー->>Frontend: ログインボタンをクリック
    Frontend->>Rails (Backend): GitHub認証ページへのリダイレクトを要求
    Rails (Backend)->>GitHub認証: ユーザーをGitHub認証ページへリダイレクト
    ユーザー->>GitHub認証: GitHubでの認証情報入力
    GitHub認証->>Rails (Backend): 認証情報および承認コードを返す
    Rails (Backend)->>Frontend: Tokenを生成して返す (devise-token-auth)
    Frontend->>ユーザー: ログイン成功とともにTokenを保持
    ユーザー->>Frontend: 保護されたページにアクセス
    Frontend->>Rails (Backend): Tokenを付与してリソース要求
    Rails (Backend)->>Frontend: Tokenの検証とリソースの返却
```

## NextAuth.jsとGitHubでの認証、NextAuthで生成されたJWEを使用して、RailsでJWTを検証する認証:
### フロー
```mermaid
sequenceDiagram
    ユーザー->>Next.js (Frontend): ログインボタンをクリック
    Next.js (Frontend)->>NextAuth.js: GitHub認証の開始リクエスト
    NextAuth.js->>GitHub: GitHubでの認証をリクエスト
    GitHub->>NextAuth.js: 認証情報を返す
    NextAuth.js->>NextAuth.js: 認証情報をもとにJWEを生成
    NextAuth.js->>Next.js (Frontend): 生成されたJWEを返す
    Next.js (Frontend)->>ユーザー: ログイン成功とともにJWEを保持
    ユーザー->>Next.js (Frontend): 保護されたページにアクセス
    Next.js (Frontend)->>Rails (Backend): JWEを付与してリソース要求
    Rails (Backend)->>Rails (Backend): JWEの復号化とJWTの検証
    Rails (Backend)->>Next.js (Frontend): 検証の結果とリソースの返却
```

## 採用理由
deviseを採用すると、omniauth系のgemも複数必要で、内部的な処理を把握しづらいが、
NextAuth.jsはわかりやすい。
### ユーザーデータの柔軟性:
Deviseを使用しないため、必要に応じてユーザーデータの管理やカスタマイズが行いやすくなる。
### シームレスな統合:
NextAuth.jsはNext.jsのための認証ライブラリであり、Next.jsとの相性が非常に良い。これにより、シームレスな統合が可能となる。
### セキュリティ:
JWTは正しく実装されている場合、セキュアな認証方法となります。
### スケーラビリティ:
JWTはステートレスな認証方法であり、分散環境やマイクロサービスアーキテクチャにも適応しやすい。