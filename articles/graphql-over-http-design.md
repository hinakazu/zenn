---
title: "GraphQL over HTTP: 5つの設計判断とNestJSでの実装"
type: "tech"
topics: ["graphql", "nestjs", "apollo", "http"]
published: false
---

# はじめに

GraphQL APIを実装する際、「単一エンドポイントだから簡単」と思われがちですが、実際にはHTTPレイヤーで多くの設計判断が必要です。本記事では、GraphQL公式ドキュメント（https://graphql.org/learn/serving-over-http/）の内容をもとに、実務で直面する5つの設計ポイントとNestJS + Apollo Serverでの実装例を解説します。

# 1. 単一エンドポイント `/graphql` の設計

## RESTとの違い

RESTではリソースごとに複数のエンドポイントを持ちますが、GraphQLは単一エンドポイント（通常 `/graphql`）ですべてのクエリとミューテーションを処理します。

```
REST:
  GET    /users/123
  POST   /users
  PATCH  /users/123
  DELETE /users/123

GraphQL:
  POST   /graphql  (すべての操作)
```

## NestJSでの実装

Apollo Serverを使用すると、デフォルトで `/graphql` エンドポイントが設定されます。

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      autoSchemaFile: true,
      // デフォルトで /graphql に設定される
      path: '/graphql',
    }),
  ],
})
export class AppModule {}
```

## 複数エンドポイントが必要なケース

公開API（`/graphql`）と管理画面API（`/admin/graphql`）を分ける場合、NestJSではモジュールを分割します。

```typescript
// app.module.ts
@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      autoSchemaFile: 'schema.public.gql',
      path: '/graphql',
    }),
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      autoSchemaFile: 'schema.admin.gql',
      path: '/admin/graphql',
      include: [AdminModule], // 管理画面用のリゾルバのみ
    }),
  ],
})
export class AppModule {}
```

# 2. POST vs GET: キャッシュとセキュリティのトレードオフ

## POST（推奨）

すべてのGraphQL操作（Query/Mutation）はPOSTで送信するのが標準です。

```bash
POST /graphql HTTP/1.1
Content-Type: application/json

{
  "query": "query GetUser($id: ID!) { user(id: $id) { name email } }",
  "variables": { "id": "123" },
  "operationName": "GetUser"
}
```

**メリット:**
- リクエストボディに機密情報を含められる
- URLの長さ制限を回避
- サーバーログにクエリが残らない

**デメリット:**
- HTTPキャッシュ（CDN/ブラウザ）が効かない

## GET（Query限定）

Queryに限り、GETリクエストを許可することでCDNキャッシュを活用できます。

```bash
GET /graphql?query=query%20GetUser%28%24id%3AID%21%29%7Buser%28id%3A%24id%29%7Bname%20email%7D%7D&variables=%7B%22id%22%3A%22123%22%7D HTTP/1.1
```

**メリット:**
- CDN/ブラウザキャッシュが有効
- パフォーマンス向上（特に静的データ）

**デメリット:**
- URLに含まれるため、ログに記録される可能性
- URL長制限（2048文字程度）
- Mutationには使用不可

## NestJSでGETを有効化

Apollo Serverはデフォルトでは `csrfPrevention` が有効で、GETリクエストが制限されます。

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: true,
  // GETリクエストを許可
  csrfPrevention: false,
  // または特定のオリジンのみ許可
  csrfPrevention: {
    requestHeaders: ['x-apollo-operation-name', 'apollo-require-preflight'],
  },
}),
```

## 実装上の注意点

GETを許可する場合、Mutationは必ずPOSTに制限してください。

```typescript
// Mutation の例
@Mutation(() => UserResponseDto)
async createUser(
  @Args('input') input: CreateUserInputDto,
): Promise<UserResponseDto> {
  // POSTのみ受け付ける（Apollo Serverがデフォルトで検証）
  return this.commandBus.execute(new CreateUserCommand(input));
}

// Query の例（GET可）
@Query(() => UserResponseDto)
async user(@Args('id') id: string): Promise<UserResponseDto> {
  return this.queryBus.execute(new GetUserQuery({ id }));
}
```

# 3. リクエスト形式: 4つのフィールドの役割

GraphQLのHTTPリクエストは以下の4つのフィールドで構成されます。

```typescript
interface GraphQLRequest {
  query: string;            // 必須
  variables?: object;       // 推奨
  operationName?: string;   // 複数操作時に必須
  extensions?: object;      // カスタム拡張
}
```

## 3.1. query（必須）

GraphQLクエリ文字列。

```json
{
  "query": "query GetUser($id: ID!) { user(id: $id) { name email } }"
}
```

## 3.2. variables（推奨）

動的な値を注入するためのオブジェクト。

```json
{
  "query": "query GetUser($id: ID!) { user(id: $id) { name email } }",
  "variables": { "id": "123" }
}
```

**変数を使わない例（非推奨）:**

```json
{
  "query": "{ user(id: \"123\") { name email } }"
}
```

この方式は以下の問題があります。

- SQLインジェクション的な攻撃リスク
- クエリキャッシュが効かない（値が変わるたびに別クエリと認識）
- 型チェックが不十分

## 3.3. operationName（複数操作時に必須）

1つのクエリ文字列に複数の操作を含む場合、実行する操作を指定します。

```json
{
  "query": "query GetUser($id: ID!) { user(id: $id) { name } } query GetPost($id: ID!) { post(id: $id) { title } }",
  "operationName": "GetUser",
  "variables": { "id": "123" }
}
```

**NestJSでの実装:**

Apollo Serverは自動的に `operationName` を解釈しますが、独自のロギングやメトリクスで使用できます。

```typescript
// カスタムプラグインで operationName を記録
import { Plugin } from '@nestjs/apollo';
import { ApolloServerPlugin } from '@apollo/server';

@Plugin()
export class LoggingPlugin implements ApolloServerPlugin {
  async requestDidStart() {
    return {
      async didResolveOperation(requestContext) {
        console.log('Operation:', requestContext.operationName);
      },
    };
  }
}
```

## 3.4. extensions（カスタム拡張）

トレーシング情報やメタデータを送信するためのフィールド。

```json
{
  "query": "{ user(id: \"123\") { name } }",
  "extensions": {
    "tracing": true,
    "cacheControl": { "maxAge": 300 }
  }
}
```

Apollo Serverでは自動的にトレーシング情報を `extensions` に含めることができます。

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: true,
  plugins: [ApolloServerPluginInlineTrace()],
}),
```

# 4. レスポンス形式: data, errors, extensions の使い分け

GraphQLのHTTPレスポンスは以下の形式です。

```typescript
interface GraphQLResponse {
  data?: object | null;
  errors?: Array<{
    message: string;
    locations?: Array<{ line: number; column: number }>;
    path?: Array<string | number>;
    extensions?: object;
  }>;
  extensions?: object;
}
```

## 4.1. 成功時（data）

```json
{
  "data": {
    "user": {
      "name": "John Doe",
      "email": "john@example.com"
    }
  }
}
```

## 4.2. エラー時（errors）

GraphQLでは部分的なエラーをサポートします。

```json
{
  "data": {
    "user": {
      "name": "John Doe",
      "email": null
    }
  },
  "errors": [
    {
      "message": "メールアドレスの取得に失敗しました",
      "path": ["user", "email"],
      "extensions": {
        "code": "INTERNAL_SERVER_ERROR"
      }
    }
  ]
}
```

## NestJSでのエラーハンドリング

カスタム例外クラスを使用して、GraphQLエラーに変換します。

```typescript
// 例外クラス
export class UserNotFoundError extends Error {
  constructor(userId: string) {
    super(`ユーザー（ID: ${userId}）が見つかりません`);
  }
}

// GraphQLエラーへの変換
import { GraphQLError } from 'graphql';

@Catch(UserNotFoundError)
export class GraphQLErrorFilter implements ExceptionFilter {
  catch(exception: UserNotFoundError) {
    throw new GraphQLError(exception.message, {
      extensions: {
        code: 'USER_NOT_FOUND',
        http: { status: 404 },
      },
    });
  }
}
```

## 4.3. extensions（メタデータ）

パフォーマンス情報やトレーシングデータを含めます。

```json
{
  "data": { "user": { "name": "John Doe" } },
  "extensions": {
    "tracing": {
      "version": 1,
      "startTime": "2024-01-01T00:00:00.000Z",
      "endTime": "2024-01-01T00:00:00.123Z",
      "duration": 123000000
    }
  }
}
```

Apollo Serverでは `ApolloServerPluginInlineTrace` を使用して自動的に追加されます。

# 5. HTTPステータスコードと Content-Type の正しい使い方

## 5.1. HTTPステータスコード

GraphQLでは、ほとんどの場合 `200 OK` を返します。これはGraphQL仕様の特徴です。

```
成功:           200 OK
部分的エラー:   200 OK（errors フィールド含む）
完全なエラー:   200 OK（data: null, errors あり）
```

ただし、以下の場合は他のステータスコードを返すべきです。

| ステータス | 状況 |
|-----------|------|
| 400 Bad Request | クエリ構文エラー、バリデーションエラー |
| 401 Unauthorized | 認証エラー |
| 500 Internal Server Error | サーバー内部エラー（予期しない例外） |

## NestJSでの実装

Apollo Serverはデフォルトでこの挙動を実装しています。

```typescript
GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: true,
  formatError: (error) => {
    // カスタムエラーフォーマット
    return {
      message: error.message,
      extensions: {
        code: error.extensions?.code || 'INTERNAL_SERVER_ERROR',
        statusCode: error.extensions?.http?.status || 500,
      },
    };
  },
}),
```

## 5.2. Content-Type の新仕様

GraphQL仕様では新しいMIMEタイプが定義されました。

```
新仕様: application/graphql-response+json
旧仕様: application/json（後方互換性のため残存）
```

**新仕様の利点:**
- GraphQLレスポンスであることが明確
- プロキシやCDNがGraphQL専用の最適化を適用可能
- 将来の拡張に対応しやすい

## NestJSでの実装

Apollo Server v4以降では自動的に対応しています。

```typescript
// レスポンスヘッダー（Apollo Server v4+）
Content-Type: application/graphql-response+json; charset=utf-8
```

クライアント側でも `Accept` ヘッダーで指定できます。

```bash
curl -X POST http://localhost:3000/graphql \
  -H "Content-Type: application/json" \
  -H "Accept: application/graphql-response+json" \
  -d '{"query":"{ user(id: \"123\") { name } }"}'
```

# 6. 認証と認可: GraphQL処理前 vs リゾルバ内

## 設計原則

GraphQL公式ドキュメントでは以下のように推奨されています。

- **認証（Authentication）**: GraphQL処理の**前**にHTTPミドルウェアで実施
- **認可（Authorization）**: リゾルバ内でビジネスロジックとして実施

## 6.1. 認証（HTTP層）

FirebaseやJWTトークンの検証はHTTP層で行います。

```typescript
// Firebase認証ガード
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { GqlExecutionContext } from '@nestjs/graphql';
import * as admin from 'firebase-admin';

@Injectable()
export class GqlFirebaseAuthGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const ctx = GqlExecutionContext.create(context);
    const request = ctx.getContext().req;

    // Authorization ヘッダーからトークンを取得
    const token = request.headers.authorization?.replace('Bearer ', '');

    if (!token) {
      throw new Error('認証トークンがありません');
    }

    try {
      // Firebase Admin SDKでトークン検証
      const decodedToken = await admin.auth().verifyIdToken(token);
      request.user = decodedToken; // リクエストにユーザー情報を注入
      return true;
    } catch (error) {
      throw new Error('認証に失敗しました');
    }
  }
}
```

リゾルバで使用:

```typescript
@Query(() => UserResponseDto)
@UseGuards(GqlFirebaseAuthGuard) // 認証ガードを適用
async me(@CurrentUser() user: admin.auth.DecodedIdToken): Promise<UserResponseDto> {
  return this.queryBus.execute(new GetUserQuery({ uid: user.uid }));
}
```

## 6.2. 認可（リゾルバ層）

リソースへのアクセス権限チェックはビジネスロジック層で実施します。

```typescript
// コマンドハンドラ内での認可
@CommandHandler(DeleteUserCommand)
export class DeleteUserCommandHandler implements ICommandHandler<DeleteUserCommand> {
  async execute(command: DeleteUserCommand): Promise<void> {
    const { userId, requestUserId } = command;

    // リソース取得
    const user = await this.userRepository.findById(userId);

    if (!user) {
      throw new UserNotFoundError(userId);
    }

    // 認可チェック（自分自身または管理者のみ削除可能）
    if (user.id !== requestUserId && !this.isAdmin(requestUserId)) {
      throw new ForbiddenError('このユーザーを削除する権限がありません');
    }

    // 削除処理
    await this.userRepository.delete(user);
  }

  private isAdmin(userId: string): boolean {
    // 管理者判定ロジック
    return false;
  }
}
```

## なぜ分離するのか

| 層 | 責務 | 理由 |
|----|------|------|
| HTTP層（認証） | トークン検証 | すべてのリクエストで共通、早期リジェクト可能 |
| リゾルバ層（認可） | リソースアクセス権限 | ビジネスルールに依存、リソースごとに異なる |

この分離により、以下のメリットがあります。

- 認証エラーはGraphQL処理前に401で返せる
- 認可ロジックはテストしやすい（モック不要）
- リゾルバはビジネスロジックに集中できる

# メリットとデメリット

## GraphQL over HTTPのメリット

| メリット | 詳細 |
|---------|------|
| 単一エンドポイント | クライアント実装がシンプル |
| 柔軟なデータ取得 | 必要なフィールドのみ取得可能 |
| 型安全性 | スキーマファーストで型チェック |
| イントロスペクション | APIドキュメントが自動生成される |

## GraphQL over HTTPのデメリット

| デメリット | 詳細 | 対策 |
|-----------|------|------|
| キャッシュが効きにくい | POSTではHTTPキャッシュ不可 | GETを許可、またはApollo Clientのキャッシュ使用 |
| N+1問題 | ネストしたクエリでパフォーマンス劣化 | DataLoaderを使用 |
| ファイルアップロード | 標準仕様にない | `graphql-upload` ライブラリ使用 |
| 複雑性の制御 | クエリが深すぎるとサーバー負荷大 | 深さ制限、コスト計算を導入 |

# まとめ

GraphQL over HTTPの5つの設計ポイント:

1. **単一エンドポイント**: `/graphql` ですべての操作を処理（必要に応じて複数エンドポイント）
2. **POST vs GET**: 基本はPOST、Queryのみキャッシュ目的でGET許可
3. **リクエスト形式**: `query`, `variables`, `operationName`, `extensions` を正しく使い分ける
4. **レスポンス形式**: `data`, `errors`, `extensions` で部分的エラーをサポート
5. **HTTPステータス**: ほとんど `200 OK`、Content-Typeは `application/graphql-response+json` が標準

NestJS + Apollo Serverを使用すると、これらの仕様の多くがデフォルトで実装されています。ただし、認証・認可の分離やエラーハンドリングは設計が必要です。

公式ドキュメント（https://graphql.org/learn/serving-over-http/）を参照しながら、プロジェクトの要件に合わせた実装を選択してください。

## 参考資料

- [GraphQL公式ドキュメント: Serving over HTTP](https://graphql.org/learn/serving-over-http/)
- [Apollo Server Documentation](https://www.apollographql.com/docs/apollo-server/)
- [NestJS GraphQL Documentation](https://docs.nestjs.com/graphql/quick-start)
- [GraphQL over HTTP Specification](https://github.com/graphql/graphql-over-http)
