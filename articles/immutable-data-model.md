---
title: "イミュータブルデータモデル実践ガイド: DDD + CQRS + Prismaで実現する進化するDB設計"
emoji: "📚"
type: "tech"
topics: ["database", "ddd", "prisma", "nestjs", "datamodeling"]
published: true
---

# はじめに

データベース設計において「更新日時」「削除フラグ」「マスタ/トランザクション分類」を当たり前のように使っていませんか。これらは実は思考停止のサインかもしれません。

本記事では、kawasima氏や増田亨氏が提唱する「イミュータブルデータモデル」の実践手法を、Prisma + NestJS + DDD/CQRSの環境で実装する方法を解説します。UPDATE文を最小化し、事実を消さず、NULLを排除することで、監査性が高く進化しやすいデータモデルを構築できます。

# イミュータブルデータモデルとは

イミュータブルデータモデルは、データベース上のレコードを極力変更せず、新しい事実をINSERTのみで記録していく設計思想です。

## 核となる3原則

1. **事実は消さない、上書きしない（INSERT ONLY）**
2. **NULLを避ける（イベント分割でNULLableカラムを不要にする）**
3. **複数の日時属性はエンティティ分割のサイン**

これにより以下のメリットが得られます。

- 完全な監査証跡
- データの時系列分析が容易
- 業務の変化に強い（カラム追加でなくテーブル追加で対応）
- バグによるデータ破壊のリスク低減

# テーブル分類: リソース(R)とイベント(E)

イミュータブルデータモデルでは、テーブルを「リソース」と「イベント」の2種類に分類します。

## 判定方法: 「〜する」テスト

エンティティ名に「〜する」をつけて自然かどうかで判定します。

| 分類 | 判定方法 | 日時属性 | 例 |
|------|---------|---------|-----|
| **リソース(R)** | 「〜する」が不自然 | なし（または作成日時のみ） | 会員、商品、ドクター |
| **イベント(E)** | 「〜する」が自然 | ただ1つの日時 | 注文する、フォローする |

```typescript
// リソース: 「会員する」は不自然
model User {
  id        String   @id @default(uuid()) @db.Uuid
  email     String   @unique
  createdAt DateTime @default(now()) @map("created_at")
}

// イベント: 「フォローする」は自然
model UserFollow {
  id           String   @id @default(uuid()) @db.Uuid
  followerId   String   @map("follower_id") @db.Uuid
  followedId   String   @map("followed_id") @db.Uuid
  followedAt   DateTime @default(now()) @map("followed_at") // 日時属性は1つだけ

  @@unique([followerId, followedId])
}
```

**重要な原則: イベントテーブルは日時属性を1つだけ持つ**

複数の日時属性（`createdAt`, `updatedAt`, `deletedAt`など）がある場合、それは隠れたイベントが未抽出のサインです。

## 命名規則

### DBテーブル名: 名詞形

データベース上のテーブル名は名詞形で統一します。

```prisma
// OK: 名詞形
model Order { }           // 注文
model Enrollment { }      // 入会
model UserFollow { }      // フォロー

// NG: 動詞形
model FollowUser { }
model CreateOrder { }
```

### DDDドメインイベントクラス: 過去形

一方、DDDのドメインイベントクラスは過去形で命名します。

```typescript
// DDD Domain Event: 過去形
export class OrderStartedDomainEvent extends DomainEventBase {
  constructor(
    public readonly aggregateId: string,
    public readonly orderedAt: Date,
  ) {
    super({ aggregateId });
  }
}

export class UserFollowedDomainEvent extends DomainEventBase {
  constructor(
    public readonly aggregateId: string,
    public readonly followerId: string,
    public readonly followedAt: Date,
  ) {
    super({ aggregateId });
  }
}
```

命名パターン: `[名詞][過去形動詞]DomainEvent`

これはMicrosoftのDDDドキュメントやEvent Sourcingのベストプラクティスに準拠しています。

### 避けるべき修飾語

以下の修飾語は無意味なので避けます。

- 情報、データ、処理、〜物、マスタ、記録、管理、履歴

```typescript
// NG
model UserInformation { }  // 「情報」は不要
model OrderData { }        // 「データ」は不要
model ProductMaster { }    // 「マスタ」は不要

// OK
model User { }
model Order { }
model Product { }
```

# 交差エンティティ化: リソース同士を直接FK参照しない

リソース同士を直接外部キーで参照すると、NULLableカラムやUPDATE操作が必要になります。交差エンティティ（イベント）を仲介することで解決します。

## アンチパターン: 直接FK参照

```prisma
// NG: 社員に部門IDを直接持たせる
model Employee {
  id           String    @id @default(uuid()) @db.Uuid
  name         String
  departmentId String?   @map("department_id") @db.Uuid  // NULLable
  updatedAt    DateTime  @updatedAt @map("updated_at")   // UPDATE発生

  department   Department? @relation(fields: [departmentId], references: [id])
}

model Department {
  id        String   @id @default(uuid()) @db.Uuid
  name      String
  employees Employee[]
}
```

**問題点:**
- 配属前の新入社員や部門に属さない役員を表現できない（departmentIdがNULL）
- 異動時にUPDATEが発生
- 過去の配属履歴が失われる

## 推奨: 交差エンティティで仲介

```prisma
// OK: リソースは日時なし
model Employee {
  id        String   @id @default(uuid()) @db.Uuid
  name      String
  createdAt DateTime @default(now()) @map("created_at")

  assignments EmployeeDepartmentAssignment[]
}

model Department {
  id        String   @id @default(uuid()) @db.Uuid
  name      String
  createdAt DateTime @default(now()) @map("created_at")

  assignments EmployeeDepartmentAssignment[]
}

// 交差エンティティ（イベント）: 日時属性は1つだけ
model EmployeeDepartmentAssignment {
  id           String   @id @default(uuid()) @db.Uuid
  employeeId   String   @map("employee_id") @db.Uuid
  departmentId String   @map("department_id") @db.Uuid
  assignedAt   DateTime @default(now()) @map("assigned_at")  // 配属された日時

  employee   Employee   @relation(fields: [employeeId], references: [id])
  department Department @relation(fields: [departmentId], references: [id])

  @@unique([employeeId, departmentId, assignedAt])
}
```

**Before/After比較:**

```
Before（直接参照）:
Employee ─FK(departmentId)→ Department
  └ NULLable外部キー、UPDATE発生

After（交差エンティティ）:
Employee ← EmployeeDepartmentAssignment → Department
  └ 全てNOT NULL、INSERT ONLY
```

**メリット:**
- NULLが不要（配属されていない社員は単に紐付きレコードがない）
- UPDATEが不要（異動時は新しい配属レコードをINSERT）
- 完全な履歴が残る（いつどの部門に所属していたか）

## NestJS CQRS実装例

```typescript
// commands/assign-employee-to-department/assign-employee-to-department.command.ts
export class AssignEmployeeToDepartmentCommand extends CommandBase {
  readonly employeeId: string;
  readonly departmentId: string;
  readonly assignedAt: Date;

  constructor(props: AssignEmployeeToDepartmentCommand) {
    super(props);
    this.employeeId = props.employeeId;
    this.departmentId = props.departmentId;
    this.assignedAt = props.assignedAt;
  }
}

// commands/assign-employee-to-department/assign-employee-to-department.command-handler.ts
@CommandHandler(AssignEmployeeToDepartmentCommand)
export class AssignEmployeeToDepartmentCommandHandler
  implements ICommandHandler<AssignEmployeeToDepartmentCommand> {

  constructor(
    @Inject(EMPLOYEE_ASSIGNMENT_REPOSITORY)
    private readonly repository: EmployeeAssignmentRepositoryPort,
  ) {}

  async execute(command: AssignEmployeeToDepartmentCommand): Promise<void> {
    // 交差エンティティを生成（INSERT ONLY）
    const assignment = EmployeeDepartmentAssignmentEntity.create(
      uuid(),
      {
        employeeId: command.employeeId,
        departmentId: command.departmentId,
        assignedAt: command.assignedAt,
      }
    );

    // INSERTのみ。UPDATEは発生しない
    await this.repository.insert(assignment);
  }
}
```

# リソースの状態変化: バージョニングパターン

リソース自体が状態変化する場合（プロフィール更新、設定変更など）、どう表現するか。

## アンチパターン: updatedAtでUPDATE

```prisma
// NG: UPDATEで状態を上書き
model UserProfile {
  id        String   @id @default(uuid()) @db.Uuid
  userId    String   @unique @map("user_id") @db.Uuid
  nickname  String
  bio       String
  updatedAt DateTime @updatedAt @map("updated_at")  // UPDATE発生
}
```

**問題点:**
- 過去のプロフィールが失われる
- いつ何を変更したかわからない

## 推奨: userId + version パターン

状態変更を新バージョンレコードのINSERTで表現します。

```prisma
model UserProfile {
  id        String   @id @default(uuid()) @db.Uuid
  userId    String   @map("user_id") @db.Uuid
  version   Int      // バージョン番号
  nickname  String
  bio       String
  createdAt DateTime @default(now()) @map("created_at")  // このバージョンが作成された日時

  @@unique([userId, version])
}
```

**運用:**

```typescript
// 初回プロフィール作成
INSERT INTO UserProfile (id, userId, version, nickname, bio, createdAt)
VALUES ('uuid1', 'user123', 1, 'Alice', 'Hello', NOW());

// プロフィール更新（UPDATEではなくINSERT）
INSERT INTO UserProfile (id, userId, version, nickname, bio, createdAt)
VALUES ('uuid2', 'user123', 2, 'Alice Chen', 'Hello World', NOW());
```

**クエリ例:**

```typescript
// queries/get-user-profile/get-user-profile.query-handler.ts
@QueryHandler(GetUserProfileQuery)
export class GetUserProfileQueryHandler implements IQueryHandler<GetUserProfileQuery> {
  constructor(
    @Inject(USER_PROFILE_REPOSITORY)
    private readonly repository: UserProfileRepositoryPort,
  ) {}

  async execute(query: GetUserProfileQuery): Promise<UserProfileResponseDto> {
    // 最新バージョンを取得
    const latestProfile = await this.prisma.userProfile.findFirst({
      where: { userId: query.userId },
      orderBy: { version: 'desc' },
      take: 1,
    });

    if (!latestProfile) {
      throw new UserProfileNotFoundError();
    }

    return this.mapper.toResponse(latestProfile);
  }
}

// 履歴を取得
async getProfileHistory(userId: string): Promise<UserProfile[]> {
  return this.prisma.userProfile.findMany({
    where: { userId },
    orderBy: { version: 'asc' },
  });
}
```

**適用例（本プロジェクト）:**

```prisma
// ユーザープロフィール
model UserProfile {
  id        String   @id @default(uuid()) @db.Uuid
  userId    String   @map("user_id") @db.Uuid
  version   Int
  // ... プロフィール属性
  createdAt DateTime @default(now()) @map("created_at")
  @@unique([userId, version])
}

// ユーザー肌質
model UserSkinType {
  id        String   @id @default(uuid()) @db.Uuid
  userId    String   @map("user_id") @db.Uuid
  version   Int
  skinType  String   @map("skin_type")
  createdAt DateTime @default(now()) @map("created_at")
  @@unique([userId, version])
}

// ユーザーレシピ
model UserRecipe {
  id        String   @id @default(uuid()) @db.Uuid
  userId    String   @map("user_id") @db.Uuid
  version   Int
  // ... レシピ属性
  createdAt DateTime @default(now()) @map("created_at")
  @@unique([userId, version])
}
```

**NestJS CQRS実装例:**

```typescript
// commands/update-user-profile/update-user-profile.command-handler.ts
@CommandHandler(UpdateUserProfileCommand)
export class UpdateUserProfileCommandHandler
  implements ICommandHandler<UpdateUserProfileCommand> {

  constructor(
    @Inject(USER_PROFILE_REPOSITORY)
    private readonly repository: UserProfileRepositoryPort,
  ) {}

  async execute(command: UpdateUserProfileCommand): Promise<IdResponse> {
    // 最新バージョンを取得
    const currentProfile = await this.repository.findLatestByUserId(command.userId);

    if (!currentProfile) {
      throw new UserProfileNotFoundError();
    }

    // 新バージョンのエンティティを生成（versionをインクリメント）
    const newProfile = UserProfileEntity.create(
      uuid(),
      {
        userId: command.userId,
        version: currentProfile.getProps().version + 1,
        nickname: command.nickname,
        bio: command.bio,
      }
    );

    // 新バージョンをINSERT（既存レコードは変更しない）
    await this.repository.insert(newProfile);

    return new IdResponse(newProfile.id);
  }
}
```

# 対になるイベントの設計パターン

フォロー/アンフォロー、注文/キャンセルのような対になるイベントをどう設計するか。3つのパターンがあります。

## パターンA: イベント分割（純粋イミュータブル）

対になるイベントをそれぞれ別テーブルとして記録します。

```prisma
// フォローイベント
model UserFollow {
  id         String   @id @default(uuid()) @db.Uuid
  followerId String   @map("follower_id") @db.Uuid
  followedId String   @map("followed_id") @db.Uuid
  followedAt DateTime @default(now()) @map("followed_at")

  @@unique([followerId, followedId])
}

// アンフォローイベント
model UserUnfollow {
  id           String   @id @default(uuid()) @db.Uuid
  followerId   String   @map("follower_id") @db.Uuid
  followedId   String   @map("followed_id") @db.Uuid
  unfollowedAt DateTime @default(now()) @map("unfollowed_at")

  @@unique([followerId, followedId, unfollowedAt])
}
```

**現在の状態を取得:**

```typescript
// queries/is-following/is-following.query-handler.ts
async isFollowing(followerId: string, followedId: string): Promise<boolean> {
  // 最後のフォローイベント
  const lastFollow = await this.prisma.userFollow.findFirst({
    where: { followerId, followedId },
    orderBy: { followedAt: 'desc' },
  });

  // 最後のアンフォローイベント
  const lastUnfollow = await this.prisma.userUnfollow.findFirst({
    where: { followerId, followedId },
    orderBy: { unfollowedAt: 'desc' },
  });

  if (!lastFollow) return false;
  if (!lastUnfollow) return true;

  // 最後のイベントがフォローならtrue、アンフォローならfalse
  return lastFollow.followedAt > lastUnfollow.unfollowedAt;
}
```

**メリット:**
- 全ての事実が残る（何回フォロー/アンフォローしたか）
- 監査・分析に強い
- 時系列分析が容易

**デメリット:**
- 現在の状態取得にクエリが複雑
- テーブル数が増える

## パターンB: ロングタームイベント

始まりと終わりのあるイベントを、スーパータイプ + 詳細イベントで表現します。

```prisma
// スーパータイプ: 現在のステータスを保持
model UserFollowState {
  id          String   @id @default(uuid()) @db.Uuid
  followerId  String   @map("follower_id") @db.Uuid
  followedId  String   @map("followed_id") @db.Uuid
  status      String   // 'FOLLOWING' | 'NOT_FOLLOWING'
  updatedAt   DateTime @updatedAt @map("updated_at")  // 唯一のUPDATE許容箇所

  @@unique([followerId, followedId])
}

// 詳細イベント: フォロー
model UserFollowEvent {
  id         String   @id @default(uuid()) @db.Uuid
  followerId String   @map("follower_id") @db.Uuid
  followedId String   @map("followed_id") @db.Uuid
  followedAt DateTime @default(now()) @map("followed_at")

  @@unique([followerId, followedId, followedAt])
}

// 詳細イベント: アンフォロー
model UserUnfollowEvent {
  id           String   @id @default(uuid()) @db.Uuid
  followerId   String   @map("follower_id") @db.Uuid
  followedId   String   @map("followed_id") @db.Uuid
  unfollowedAt DateTime @default(now()) @map("unfollowed_at")

  @@unique([followerId, followedId, unfollowedAt])
}
```

**現在の状態を取得:**

```typescript
async isFollowing(followerId: string, followedId: string): Promise<boolean> {
  const state = await this.prisma.userFollowState.findUnique({
    where: {
      followerId_followedId: { followerId, followedId },
    },
  });

  return state?.status === 'FOLLOWING';
}
```

**メリット:**
- 現在の状態取得が高速
- 一定の履歴が残る（詳細イベントテーブル）

**デメリット:**
- スーパータイプにUPDATEが発生（唯一の許容箇所）

## パターンC: DELETE許容（実践派）

現在の状態だけ必要で履歴不要なら、DELETE-then-INSERTも選択肢です。

```prisma
model UserFollow {
  id         String   @id @default(uuid()) @db.Uuid
  followerId String   @map("follower_id") @db.Uuid
  followedId String   @map("followed_id") @db.Uuid
  followedAt DateTime @default(now()) @map("followed_at")

  @@unique([followerId, followedId])
}
```

**運用:**

```typescript
// フォロー
await this.prisma.userFollow.create({
  data: {
    id: uuid(),
    followerId,
    followedId,
    followedAt: new Date(),
  },
});

// アンフォロー（DELETE）
await this.prisma.userFollow.delete({
  where: {
    followerId_followedId: { followerId, followedId },
  },
});

// 現在の状態取得
const following = await this.prisma.userFollow.findUnique({
  where: {
    followerId_followedId: { followerId, followedId },
  },
});

return !!following;
```

**メリット:**
- シンプル
- 現在の状態取得が高速

**デメリット:**
- 履歴が失われる

## 選定基準

どのパターンを選ぶべきか、判断フローチャート:

```
[「いつフォローしたか」「いつアンフォローしたか」が重要?]
  YES → パターンA（イベント分割）
  NO  → [履歴は完全に不要?]
          YES → パターンC（DELETE許容）
          NO  → パターンB（ロングタームイベント）
```

| ケース | 推奨パターン |
|--------|-------------|
| 「いつ発生したか」「いつ解除したか」が重要（監査、分析） | A: イベント分割 |
| 現在の状態だけわかればよい（履歴不要） | C: DELETE許容 |
| 状態取得の高速性 + 一定の履歴が必要 | B: ロングタームイベント |

# Event Sourcingとの関係性

イミュータブルデータモデルとEvent Sourcingは近い概念ですが、同一ではありません。

| 項目 | イミュータブルデータモデル | Event Sourcing |
|------|--------------------------|----------------|
| **思想** | 事実をINSERTで記録、UPDATE最小化 | 状態変更を全てイベントとして記録 |
| **リソース** | リソーステーブルは存在する（日時なし） | 現在状態はイベントから再構築 |
| **クエリ** | リソース or イベントを直接クエリ | イベントストリームから再生 |
| **複雑度** | 中程度 | 高（CQRS必須、スナップショット必要） |
| **適用範囲** | 部分的適用可能 | システム全体で採用が基本 |

**イミュータブルデータモデルはEvent Sourcingのライト版**と考えることができます。Event Sourcingの厳密さを必要としない場合に有効です。

# DDDとの組み合わせ

イミュータブルデータモデルはDDD(Domain-Driven Design)と相性が良いです。

## ドメインイベントとの統合

```typescript
// domain/user-follow.entity.ts
export class UserFollowEntity extends AggregateRoot<UserFollowProps> {
  static create(id: string, props: CreateUserFollowProps): UserFollowEntity {
    const entity = new UserFollowEntity({ id, props });

    // ドメインイベントを発火
    entity.addEvent(new UserFollowedDomainEvent({
      aggregateId: id,
      followerId: props.followerId,
      followedId: props.followedId,
      followedAt: props.followedAt,
    }));

    return entity;
  }

  validate(): void {
    if (this.props.followerId === this.props.followedId) {
      throw new Error('自分自身をフォローできません');
    }
  }
}

// commands/follow-user/follow-user.command-handler.ts
@CommandHandler(FollowUserCommand)
export class FollowUserCommandHandler implements ICommandHandler<FollowUserCommand> {
  constructor(
    @Inject(USER_FOLLOW_REPOSITORY)
    private readonly repository: UserFollowRepositoryPort,
  ) {}

  async execute(command: FollowUserCommand): Promise<IdResponse> {
    // エンティティ生成（バリデーション + ドメインイベント追加）
    const follow = UserFollowEntity.create(uuid(), {
      followerId: command.followerId,
      followedId: command.followedId,
      followedAt: new Date(),
    });

    // 永続化（INSERT ONLY）
    await this.repository.insert(follow);
    // ドメインイベントをpublish（永続化後）
    await follow.publishEvents(this.logger, this.eventEmitter);

    return new IdResponse(follow.id);
  }
}
```

**ポイント:**
- エンティティの生成/変更時にドメインイベントを追加
- 永続化後にドメインイベントをpublish
- イベントハンドラで副作用を処理（メール送信、通知など）

## リポジトリパターン

```typescript
// database/user-follow.repository.port.ts
export abstract class UserFollowRepositoryPort {
  abstract findOneById(id: string): Promise<UserFollow | null>;
  abstract findFollows(followerId: string): Promise<UserFollow[]>;
  abstract insert(entity: UserFollowEntity): Promise<void>;
  abstract delete(entity: UserFollowEntity): Promise<void>;
}

// database/user-follow.repository.ts
@Injectable()
export class UserFollowRepository implements UserFollowRepositoryPort {
  constructor(
    private readonly prisma: PrismaClient,
    private readonly mapper: UserFollowMapper,
    private readonly eventEmitter: EventEmitter2,
    @Inject(LOGGER) private readonly logger: LoggerPort,
  ) {}

  async insert(entity: UserFollowEntity): Promise<void> {
    const record = this.mapper.toPersistence(entity);

    await this.prisma.userFollow.create({
      data: {
        id: record.id,
        followerId: record.followerId,
        followedId: record.followedId,
        followedAt: record.followedAt,
      },
    });

    // ドメインイベントをpublish
    await entity.publishEvents(this.logger, this.eventEmitter);
  }
}
```

# アンチパターン集

## 1. 全エンティティへの一律「登録日時・更新日時」付与

```prisma
// NG: 思考停止で全テーブルにcreatedAt/updatedAtを追加
model Product {
  id        String   @id @default(uuid()) @db.Uuid
  name      String
  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")  // 本当に必要?
}
```

**問題点:**
- `updatedAt`は「いつ何が変わったか」を記録しない
- ビジネス的意味のない日時が増える
- 「更新日時があるからUPDATEしていい」という思考に陥る

**改善策:**
- リソーステーブルは`createdAt`のみ
- 状態変化が重要ならバージョニングパターンを使う

## 2. 「マスタ」「トランザクション」分類

```
# NG: 曖昧な分類
master/
  - product_master.sql
  - user_master.sql

transaction/
  - order_transaction.sql
  - payment_transaction.sql
```

**問題点:**
- 「ユーザーはマスタだけど、プロフィール変更はトランザクション?」のような不毛な議論が発生
- 分類が曖昧でチーム内で解釈が分かれる

**改善策:**
- 「リソース」「イベント」の2分類に統一
- 「〜する」テストで機械的に判定

## 3. 削除フラグ・削除日時

```prisma
// NG: 削除フラグは物理設計の早すぎる最適化
model User {
  id        String    @id @default(uuid()) @db.Uuid
  email     String    @unique
  isDeleted Boolean   @default(false) @map("is_deleted")
  deletedAt DateTime? @map("deleted_at")
}
```

**問題点:**
- 論理削除は一見安全だが、ユニーク制約の問題が発生する
- 「削除された」という状態も業務的意味がある場合、イベントとして抽出すべき

**改善策:**

```prisma
// OK: 退会をイベントとして記録
model User {
  id        String   @id @default(uuid()) @db.Uuid
  email     String   @unique
  createdAt DateTime @default(now()) @map("created_at")
}

model UserWithdrawal {
  id          String   @id @default(uuid()) @db.Uuid
  userId      String   @unique @map("user_id") @db.Uuid
  reason      String
  withdrawnAt DateTime @default(now()) @map("withdrawn_at")

  user User @relation(fields: [userId], references: [id])
}
```

**現在の状態を取得:**

```typescript
async isActiveUser(userId: string): Promise<boolean> {
  const withdrawal = await this.prisma.userWithdrawal.findUnique({
    where: { userId },
  });

  return !withdrawal;
}
```

## 4. リソースのNULLable外部キー更新パターン

```prisma
// NG: 交差エンティティ化で解決できる
model Employee {
  id           String    @id @default(uuid()) @db.Uuid
  departmentId String?   @map("department_id") @db.Uuid  // NULLable
  updatedAt    DateTime  @updatedAt @map("updated_at")   // UPDATE発生
}
```

このパターンは「交差エンティティ化」セクションで解説した通り、交差エンティティで解決します。

## 5. 極端な適用: 「何が何でもUPDATE禁止」

```typescript
// NG: セッショントークンの有効期限更新までINSERTにする必要はない
model UserSession {
  id        String   @id @default(uuid()) @db.Uuid
  version   Int      // 極端な適用
  token     String
  expiresAt DateTime
}
```

**改善策:**
- 一時的なデータ（セッション、キャッシュ）はUPDATE/DELETE許容
- ビジネス的意味のある事実のみイミュータブルに

# 実践上の注意点

## 1. 完全なUPDATE排除は非現実的

イミュータブルデータモデルは「UPDATEを最小化する」設計指針であり、絶対ルールではありません。

**UPDATE許容箇所:**
- パフォーマンスカウンター（閲覧数、いいね数）
- 一時的なデータ（セッション、キャッシュ）
- ステータス管理（ロングタームイベントのスーパータイプ）

## 2. パフォーマンスとのトレードオフ

イベント分割でテーブル数が増えると、現在の状態取得クエリが複雑になる場合があります。

**対策:**
- 頻繁にアクセスする「現在の状態」はマテリアライズドビュー化
- パターンB（ロングタームイベント）の採用

```sql
-- マテリアライズドビューで現在のフォロー状態を高速化
CREATE MATERIALIZED VIEW current_user_follows AS
SELECT DISTINCT ON (follower_id, followed_id)
  follower_id,
  followed_id,
  followed_at
FROM (
  SELECT follower_id, followed_id, followed_at, 'FOLLOW' as event_type
  FROM user_follow
  UNION ALL
  SELECT follower_id, followed_id, unfollowed_at as followed_at, 'UNFOLLOW' as event_type
  FROM user_unfollow
) events
ORDER BY follower_id, followed_id, followed_at DESC;

-- 定期的にREFRESH
REFRESH MATERIALIZED VIEW current_user_follows;
```

## 3. エンティティ数の増加による複雑さ

イベント分割でテーブル数が増えます。

**対策:**
- 業務上の結びつきでグルーピング（モジュール化）
- ネーミング規則の徹底（`User` + `UserFollow` + `UserUnfollow`）

```
src/modules/user-follow/
├── domain/
│   ├── user-follow.entity.ts
│   └── user-unfollow.entity.ts
├── database/
│   ├── user-follow.repository.ts
│   └── user-unfollow.repository.ts
└── ...
```

## 4. チーム内での合意形成

イミュータブルデータモデルは従来の設計思想と異なるため、チーム内での理解と合意が必要です。

**推奨アプローチ:**
- 小さく始める（一部のテーブルから適用）
- 具体的なメリットを示す（監査証跡、バグ調査の容易さ）
- ペアプログラミングで知識共有

# まとめ

イミュータブルデータモデルは、以下の原則で進化しやすいデータ設計を実現します。

## 核となる原則

1. **テーブル分類はリソース(R)とイベント(E)の2種類**
   - 「〜する」テストで判定
2. **事実は消さない、上書きしない（INSERT ONLY）**
3. **NULLを避ける（イベント分割で解決）**
4. **複数の日時属性はエンティティ分割のサイン**

## 実践パターン

| パターン | 適用ケース |
|---------|----------|
| **交差エンティティ化** | リソース同士の関係（所属、フォロー） |
| **バージョニング** | リソースの状態変化（プロフィール更新） |
| **イベント分割（A）** | 対イベントで履歴が重要 |
| **ロングタームイベント（B）** | 対イベントで現在状態取得が頻繁 |
| **DELETE許容（C）** | 対イベントで履歴不要 |

## メリット

- 完全な監査証跡
- 時系列分析が容易
- 業務の変化に強い
- バグによるデータ破壊のリスク低減

## 注意点

- 完全なUPDATE排除は非現実的（最小化が目標）
- パフォーマンスとのトレードオフ（マテリアライズドビューで対策）
- チーム内での合意形成が重要

イミュータブルデータモデルは銀の弾丸ではありませんが、適切に適用することでメンテナブルなデータ設計を実現できます。

# 参考資料

本記事は以下の資料を参考に作成しました。

## 理論・設計思想

- [イミュータブルデータモデル - kawasima](https://scrapbox.io/kawasima/%E3%82%A4%E3%83%9F%E3%83%A5%E3%83%BC%E3%82%BF%E3%83%96%E3%83%AB%E3%83%87%E3%83%BC%E3%82%BF%E3%83%A2%E3%83%87%E3%83%AB)
  - イミュータブルデータモデルの提唱者による解説
- [増田亨さん「設計の考え方とやり方」- asken](https://tech.asken.inc/entry/2022/08/24/084632)
  - DDDとイミュータブルデータモデルの組み合わせ

## 実践事例

- [実践イミュータブルデータモデル - 令和トラベル](https://engineering.reiwatravel.co.jp/blog/newt-point-immutable-data-model)
  - ポイントシステムでの適用事例
- [データモデリング初学者がイミュータブルデータモデリングを実践 - エニグモ](https://tech.enigmo.co.jp/entry/2022/12/12/100000)
  - 初学者視点での実践レポート
- [イミュータブルデータモデリングのチャレンジ - ikkitang](https://www.ikkitang1211.site/entry/stafes2021-16)
  - 実践上の課題と対策

## 技術詳細

- [イミュータブルデータモデルから業務データの世代管理を考える - Zenn](https://zenn.dev/mashi/articles/cf8f871afe21e9)
  - バージョニングパターンの詳細
- [イミュータブルでゆこう - Zenn](https://zenn.dev/cacbahbj/articles/9a17170967fb50)
  - 実装上のTips
- [イミュータルデータモデルって美味しいの？ - Qiita](https://qiita.com/javano/items/a1021fdf7d1f090a13ca)
  - メリット・デメリットの整理

## DDD・Event Sourcing

- [Microsoft - Domain Events Design and Implementation](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/domain-events-design-implementation)
  - ドメインイベントの公式ドキュメント
- [Event Sourcing Explained - BaytechConsulting](https://www.baytechconsulting.com/blog/event-sourcing-explained-2025)
  - Event Sourcingの解説
- [How I Name Events - Arrange Act Assert](https://arrangeactassert.com/posts/how-i-name-events/)
  - イベント命名のベストプラクティス

## サンプルコード

- [kawasima/immutable-datamodel-example - GitHub](https://github.com/kawasima/immutable-datamodel-example)
  - kawasima氏による実装例
