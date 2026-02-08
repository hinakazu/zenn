---
title: "GraphQL公式に学ぶQuery/Mutation命名規約"
type: "tech"
topics: ["graphql", "api", "nestjs", "backend"]
published: true
---

# はじめに

GraphQLのQuery/Mutationの命名規約について、チーム内で議論になったことはありませんか？

「Queryフィールドは`getUser`？それとも`user`だけ？」「Mutationは`updateUser`で良い？もっと具体的にすべき？」といった疑問に対し、GraphQL公式ドキュメントと仕様書から導き出される**ベストプラクティス**をまとめます。

# 公式が明示する3つの原則

GraphQL公式ドキュメント（graphql.org）では、以下の3つの原則が示されています。

## 1. オペレーション名は明示的に付ける

```graphql
# Good: オペレーション名がある
query GetUserProfile {
  user(id: "123") {
    name
    email
  }
}

# Bad: オペレーション名がない（開発環境では許容されるが非推奨）
{
  user(id: "123") {
    name
  }
}
```

**理由:**
- サーバーサイドのログに記録される（どのクエリが重いか特定可能）
- デバッグツールで識別しやすい
- 複数オペレーションを1リクエストで送る場合は必須

## 2. Mutationは目的特化型にする

公式ドキュメントでは、汎用的なCRUD操作ではなく**purpose-built**（目的特化型）のMutationを推奨しています。

```graphql
# Bad: 汎用的すぎる
mutation UpdateUser($id: ID!, $name: String, $email: String, $age: Int) {
  updateUser(id: $id, name: $name, email: $email, age: $age) {
    id
  }
}

# Good: 目的が明確
mutation UpdateUserEmail($id: ID!, $email: String!) {
  updateUserEmail(id: $id, email: $email) {
    id
    email
  }
}

mutation UpdateUserProfile($id: ID!, $name: String!, $age: Int!) {
  updateUserProfile(id: $id, name: $name, age: $age) {
    id
    name
    age
  }
}
```

**メリット:**
- 必須項目を`Non-Null`型で強制できる（`updateUserEmail`では`email`が必須）
- バリデーションロジックが単純化される
- 変更履歴の追跡が容易（監査ログに`updateUserEmail`と記録される）
- 権限制御が細かく設定できる（メール変更は本人のみ、プロフィール変更は管理者も可、など）

## 3. Queryフィールドは名詞、Mutationフィールドは動詞

公式ドキュメントと仕様書の例を見ると、**Queryフィールドには一貫して名詞のみ**が使われています。

```graphql
# 公式ドキュメント・仕様書の例（すべて名詞）
type Query {
  hero(episode: Episode): Character   # not getHero
  human(id: ID!): Human               # not findHuman
  droid(id: ID!): Droid               # not getDroid
  user(id: 4): User                   # not getUser（仕様書）
  me: User                            # not getMe（仕様書）
}
```

`query { user(id: "123") { ... } }` と書いた時点で「取得する」ことは自明なので、`get` / `find` は冗長です。

一方、**Mutationフィールドには動詞プレフィックスが必須**です。操作の種類（作成・更新・削除）を区別する必要があるためです。

```graphql
type Mutation {
  createReview(episode: Episode!, review: ReviewInput!): Review
  updateHumanName(id: ID!, name: String!): Human
  deleteStarship(id: ID!): ID!
}
```

**まとめると:**

| 種別 | 命名パターン | 例 |
|------|------------|---|
| Query（単一） | 名詞 | `user(id:)`, `hero(episode:)`, `me` |
| Query（一覧） | 複数形名詞 | `users`, `posts`, `reviews` |
| Mutation | 動詞 + 名詞 | `createReview`, `updateHumanName`, `deleteStarship` |

# 仕様書から読み取れる暗黙の慣習

GraphQL仕様（spec.graphql.org）と公式ドキュメントの例示から、以下の暗黙の慣習が読み取れます。

## 大文字小文字の使い分け

```graphql
# オペレーション名: PascalCase（クライアント側の識別用）
query UserProfile { ... }
mutation CreateReview { ... }

# 型名: PascalCase
type User { ... }
type Review { ... }

# フィールド名: camelCase
type User {
  firstName: String
  createdAt: DateTime
}

# Enum値: UPPER_SNAKE_CASE
enum Episode {
  NEWHOPE
  EMPIRE
  JEDI
}
```

## 予約語の回避

`__`（ダブルアンダースコア）プレフィックスはGraphQLのイントロスペクション用に予約されています。

```graphql
# NG: 予約語と衝突
type __User { ... }

# OK
type User { ... }
```

# 実装例（NestJS + GraphQL）

NestJSでの実装例を示します。

```typescript
// ユーザーのメールアドレス更新（目的特化型）
@Mutation(() => UserResponseDto)
async updateUserEmail(
  @Args('input') input: UpdateUserEmailInputDto,
): Promise<UserResponseDto> {
  return this.commandBus.execute(
    new UpdateUserEmailCommand({
      userId: input.userId,
      email: input.email,
    })
  );
}

// ユーザープロフィール更新（目的特化型）
@Mutation(() => UserResponseDto)
async updateUserProfile(
  @Args('input') input: UpdateUserProfileInputDto,
): Promise<UserResponseDto> {
  return this.commandBus.execute(
    new UpdateUserProfileCommand({
      userId: input.userId,
      name: input.name,
      age: input.age,
    })
  );
}

// ユーザー一覧取得（フィールド名は名詞: users）
@Query(() => [UserResponseDto], { name: 'users' })
async users(
  @Args() args: PaginationArgs,
): Promise<UserResponseDto[]> {
  return this.queryBus.execute(
    new GetUsersQuery({ limit: args.limit, offset: args.offset })
  );
}
```

# まとめ

GraphQL公式ドキュメントから導き出される命名規約のポイントは以下の3つです。

1. **オペレーション名は明示的に付ける**（デバッグ・ログ記録のため）
2. **Mutationは目的特化型にする**（`updateUser`ではなく`updateUserEmail`）
3. **Queryは名詞、Mutationは動詞**（`user`/`me`/`users` vs `createReview`/`updateHumanName`）

特に「Queryフィールドに`get`/`find`を付けない」点は、REST脳からの脱却として重要です。GraphQLでは`query { user(id: "123") }` の時点で取得であることが自明なため、動詞プレフィックスは冗長です。

GraphQLスキーマ設計の参考になれば幸いです。

## 参考資料

- [GraphQL公式ドキュメント - Queries and Mutations](https://graphql.org/learn/queries/)
- [GraphQL公式ドキュメント - Schemas and Types](https://graphql.org/learn/schema/)
- [GraphQL仕様書](https://spec.graphql.org/)
