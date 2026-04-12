---
title: "JavaScript and TypeScript Standards"
description: "JavaScript・TypeScriptコーディング標準 - 命名規則・型定義・async/await・ESLint/Prettier / JavaScript and TypeScript standards - naming, types, async/await, ESLint/Prettier"
version: "1.0.0"
status: "Stable"
last_updated: "2026-02-20T00:00+09:00"
lang: "ja"
---

# JavaScript and TypeScript Standards

**JavaScript and TypeScript規定** - JavaScript および TypeScript のコーディング標準を定義




**前提条件**:
- TypeScript `strict`モード有効
- ESLint + Prettier によるフォーマット
- ES2020以降の構文を使用
- Node.js 20.11以降（`import.meta.dirname`利用時）

---

## 目次

1. 基本フォーマット
2. 命名規則
3. 変数宣言
4. TypeScript型定義規約
5. 関数
6. オブジェクト・配列
7. 文字列
8. 比較・条件分岐
9. 非同期処理・エラーハンドリング
10. モジュール
11. ESLint/Prettier推奨設定
12. JSDoc/TSDocコメント規約

---

## 1. 基本フォーマット

| 項目 | 規定 |
|------|------|
| セミコロン | **使用する** |
| インデント | **半角スペース2個** |
| クォート | **シングルクォート基本**（ダブル使用時は明示的に指定） |
| 行末スペース | 禁止 |
| ファイル末尾 | 改行1つ |
| 最大行長 | 100文字（目安） |

**注意**: JSON（`.json`、`.prettierrc`、`tsconfig.json`等）は仕様上ダブルクォートのみ有効。本規約のシングルクォート基本は`.js`/`.ts`ファイルに適用。

```typescript
// ✅ Good
const message = 'Hello, World';
const html = '<div class="container">Content</div>';

// ❌ Bad
const badMsg = "Hello, World"   // ダブルクォート、セミコロンなし
```

---

## 2. 命名規則

### 2.1 識別子のケーススタイル

| 対象 | スタイル | 例 |
|------|----------|-----|
| 変数・関数・メソッド | camelCase | `userName`, `calculateTotal` |
| 定数（グローバル/static readonly） | SCREAMING_SNAKE_CASE | `MAX_RETRY_COUNT`, `API_BASE_URL` |
| クラス・型・interface・enum | PascalCase | `UserService`, `ApiResponse` |
| enumメンバー | PascalCase | `Status.Pending` |
| ファイル名 | kebab-case | `user-service.ts`, `api-client.ts` |
| 型パラメータ | 単一大文字 or PascalCase | `T`, `TKey`, `TValue` |

### 2.2 命名の原則

- **意味のある名前**: 略語・1文字変数は避ける（ループカウンタ`i`, `j`等は例外）
- **略語の扱い**: 単語として扱う（`loadHttpUrl`、~~`loadHTTPURL`~~）
- **Boolean変数**: `is`, `has`, `can`, `should`, `did`, `will` プレフィックス
- **プライベート**: アンダースコアプレフィックス禁止（TypeScriptの`private`を使用）
- **interface**: `I`プレフィックス禁止（~~`IUserService`~~）

```typescript
// ✅ Good
const isActive = true;
const hasPermission = false;
const canEdit = user.role === 'admin';
const shouldRefresh = Date.now() > lastUpdate + CACHE_TTL;

// ❌ Bad
const active = true;      // Boolean判別不明
const flag = false;       // 意味不明
const _privateVar = 123;  // アンダースコア禁止
```

### 2.3 単数形・複数形

| 対象 | 規則 | 例 |
|------|------|-----|
| 配列・コレクション | 複数形 | `users`, `items`, `orderIds` |
| 単一オブジェクト | 単数形 | `user`, `item`, `currentOrder` |
| Map/Set | 複数形 or 用途を示す名前 | `userMap`, `uniqueIds` |

---

## 3. 変数宣言

### 3.1 const / let / var

| キーワード | 使用 |
|-----------|------|
| `const` | **デフォルト**（再代入しない場合） |
| `let` | 再代入が必要な場合のみ |
| `var` | **禁止** |

```typescript
// ✅ Good
const maxRetries = 3;
const users: User[] = [];  // 空配列は型注釈必須
users.push(newUser);  // OK: 中身の変更はconst可

let retryCount = 0;
retryCount++;  // 再代入が必要

// ❌ Bad
var config = {};  // var禁止（型注釈の有無は論点外）
let name = 'John';  // 再代入しないならconst
```

### 3.2 宣言のルール

- **1宣言1変数**: `let a = 1, b = 2;` は禁止
- **使用箇所で宣言**: スコープ先頭でまとめて宣言しない
- **未使用変数**: 禁止（ESLintで検出）

---

## 4. TypeScript型定義規約

### 4.1 禁止事項

| 項目 | ルール | 代替 |
|------|--------|------|
| `any` | **禁止** | `unknown`を使用 |
| `@ts-ignore` | **禁止** | 型エラーを根本解決 |
| `@ts-expect-error` | **原則禁止** | テスト時のみ例外的に許可 |
| Wrapper型 | **禁止** | `string`, `boolean`, `number`を使用 |
| `const enum` | **禁止** | 通常の`enum`を使用 |

```typescript
// ✅ Good
let data: unknown;
const userName: string = 'hello';  // Wrapper型（String）との対比のため型注釈を明示

// ❌ Bad
let input: any;
// @ts-ignore
const value: string = someFunction();
const name: String = 'hello';  // Wrapper型は禁止
```

### 4.1.1 enum vs ユニオン型

| 項目 | 推奨 |
|------|------|
| 文字列の列挙 | **ユニオン型** |
| 数値の列挙（連番） | `enum`可 |
| ビットフラグ | `enum`可 |

```typescript
// ✅ 推奨: 文字列ユニオン型
type Status = 'pending' | 'approved' | 'rejected';

// ✅ 許可: 数値enum（連番が必要な場合）
enum HttpStatus {
  Ok = 200,
  NotFound = 404,
  InternalError = 500,
}

// ❌ 避ける: 文字列enum
enum StatusEnum {
  Pending = 'pending',
  Approved = 'approved',
}
```

### 4.2 型推論の活用

| 場面 | 型注釈 |
|------|--------|
| リテラル初期化 | 省略可 |
| `new`式 | 省略可 |
| 空配列・空オブジェクト | **必須** |
| 複雑な戻り値 | 明示推奨 |

```typescript
// ✅ 型推論に任せる
const count = 15;
const name = 'hello';
const user = new User();

// ❌ 不要な型注釈（リテラル・new式から推論可能）
const total: number = 15;
const isActive: boolean = true;

// ✅ 型注釈必須（空の初期化）
const items: string[] = [];
const map: Map<string, number> = new Map();
const config: Record<string, unknown> = {};
```

### 4.3 配列型の記法

| 型の複雑さ | 記法 | 例 |
|------------|------|-----|
| 単純型 | `T[]` | `string[]`, `number[]` |
| 多次元（単純） | `T[][]` | `string[][]` |
| 複合型 | `Array<T>` | `Array<string \| number>` |
| 読み取り専用（単純） | `readonly T[]` | `readonly string[]` |
| 読み取り専用（複合） | `ReadonlyArray<T>` | `ReadonlyArray<string \| number>` |

```typescript
// ✅ Good
let names: string[];
let matrix: number[][];
let ids: Array<string | number>;
let items: Array<{ id: number; name: string }>;

// ❌ Bad
let badNames: Array<string>;     // 単純型はT[]
let badIds: (string | number)[]; // 複合型はArray<T>
```

### 4.4 type vs interface

#### 主要スタイルガイド比較

| 項目 | Google | TypeScript Deep Dive | Zenn (kiman) | 本規約 |
|------|--------|---------------------|--------------|--------|
| デフォルト | オブジェクト型はinterface | 使い分け | type基本 | **type基本** |
| ユニオン型・交差型 | type | type | type | **type** |
| extends/implements | interface | interface | interface | **interface** |
| 宣言マージ | interface | interface | interface | **interface** |

**補足**: 各ガイドの立場は完全一致しない。プロジェクト判断として「type基本、extends/implements・宣言マージのみinterface」を採用する。

参照元:
- [Google TypeScript Style Guide](https://google.github.io/styleguide/tsguide.html)
- [TypeScript Handbook: Everyday Types](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html)
- [TypeScript Handbook: Object Types](https://www.typescriptlang.org/docs/handbook/2/objects.html)
- [TypeScript Handbook: Declaration Merging](https://www.typescriptlang.org/docs/handbook/declaration-merging.html)

#### 使い分けルール

| ユースケース | 推奨 | 例 |
|--------------|------|-----|
| ユニオン型 | `type` | `type Status = 'active' \| 'inactive'` |
| 交差型 | `type` | `type Admin = User & { role: string }` |
| プリミティブエイリアス | `type` | `type ID = string \| number` |
| タプル | `type` | `type Point = [number, number]` |
| オブジェクト形状 | `type` | `type User = { name: string }` |
| extends/implements | `interface` | クラス実装時 |
| 宣言マージ | `interface` | ライブラリ型拡張 |

#### なぜtypeを基本とするか

1. **ユニオン型・交差型に対応**: interfaceでは表現できない
2. **拡張性**: 後からユニオン型への変更が容易
3. **一貫性**: 「迷ったらtype」のシンプルなルール
4. **交差型トラブル回避**: 交差型（`A & B`）で矛盾プロパティが`never`化する挙動を明示的に扱いやすい

```typescript
// ✅ type: 基本
type Status = 'pending' | 'approved' | 'rejected';
type ID = string | number;
type User = {
  id: ID;
  name: string;
  status: Status;
};

// ✅ interface: 継承・実装
interface Repository<T> {
  find(id: string): T | undefined;
  save(entity: T): void;
}

class UserRepository implements Repository<User> {
  find(id: string): User | undefined { /* ... */ }
  save(entity: User): void { /* ... */ }
}

// ✅ interface: ライブラリ型の拡張（宣言マージ）
declare module '@mui/material/styles' {
  interface Theme {
    customProperty: string;
  }
}
```

### 4.5 型アサーション

| 項目 | ルール |
|------|--------|
| 構文 | `as`構文を使用（`<T>`は禁止） |
| 使用 | **原則避ける**、使用時はコメント必須 |
| ダブルアサーション | `unknown`経由（`any`経由は禁止） |
| オブジェクトリテラル | 型注釈を使用 |
| `as const` | **推奨** |

```typescript
// ✅ Good
const x = someValue as Foo;  // 理由をコメントで補足
const user: User = { name: 'John' };
const converted = value as unknown as TargetType;

// ✅ as const推奨
const DIRECTIONS = ['north', 'south', 'east', 'west'] as const;

// ❌ Bad
const val = <Foo>someValue;
const badUser = { name: 'John' } as User;
const bad = value as any as TargetType;
```

### 4.6 null vs undefined

| 場面 | 推奨 | 理由 |
|------|------|------|
| 値の欠如（デフォルト） | `undefined` | JavaScript標準動作 |
| オプショナルプロパティ | `?` | 暗黙的undefined |
| DOM API / 外部API | `null` | API仕様に従う |
| 明示的な「値なし」 | `null` | 意図的な空値 |
| 両方チェック | `== null` | null/undefined同時判定 |

```typescript
// ✅ オプショナルプロパティ
type User = {
  name: string;
  nickname?: string;  // string | undefined
};

// ✅ null/undefinedチェック
if (value == null) {
  // value is null or undefined
}

// ✅ DOM APIはnull
const element: HTMLElement | null = document.getElementById('app');
```

### 4.7 import type

| 場面 | ルール |
|------|--------|
| 型のみのインポート | `import type`を使用 |
| 値と型の混在 | インライン`type`を使用 |
| 型の再エクスポート | `export type`を使用 |

```typescript
// ✅ 型のみ
import type { User, Config } from './types';

// ✅ 値と型の混在
import { createUser, type User } from './user';

// ✅ 型の再エクスポート
export type { User } from './types';
```

### 4.8 Utility Types活用

| Utility Type | 用途 | 例 |
|--------------|------|-----|
| `Partial<T>` | 全プロパティをオプショナル | 更新用DTO |
| `Required<T>` | 全プロパティを必須 | バリデーション後 |
| `Readonly<T>` | 読み取り専用 | イミュータブル |
| `Pick<T, K>` | 特定プロパティ抽出 | 部分型 |
| `Omit<T, K>` | 特定プロパティ除外 | 部分型 |
| `Record<K, V>` | キー・値指定オブジェクト | マップ型 |

```typescript
type User = {
  id: string;
  name: string;
  email: string;
  createdAt: Date;
};

type UserUpdate = Partial<Pick<User, 'name' | 'email'>>;
type UserSummary = Pick<User, 'id' | 'name'>;
type UserWithoutTimestamp = Omit<User, 'createdAt'>;
```

---

## 5. 関数

### 5.1 主要ガイド・公式資料の傾向

| 項目 | Google | React Docs | MUI（実装例） | 本規約 |
|------|--------|------------|---------------|--------|
| 名前付き関数 | function宣言 | 記述例あり | function宣言中心 | **function宣言** |
| 無名関数/コールバック | アロー関数 | 記述例あり | アロー関数多用 | **アロー関数** |
| Reactコンポーネント | - | 関数コンポーネント | function宣言中心 | **function宣言** |
| クラスメソッド | 通常記法 | - | 通常記法 | **通常記法** |
| プロパティとしてのアロー関数 | 避ける | - | - | **避ける** |

参照元:
- [Google TypeScript Style Guide](https://google.github.io/styleguide/tsguide.html)
- [React Docs: Your First Component](https://react.dev/learn/your-first-component)
- [MUI TypeScript Guide](https://mui.com/material-ui/guides/typescript/)

### 5.2 使い分けルール

| ユースケース | 推奨 | 理由 |
|--------------|------|------|
| 名前付き関数・ユーティリティ | `function` 宣言 | 巻き上げ、明示的、スタックトレース |
| Reactコンポーネント | `function` 宣言 | React公式・MUI・Google準拠 |
| クラスメソッド | 通常記法 | thisバインディングは呼び出し側で制御 |
| コールバック・無名関数 | アロー関数 | 簡潔、thisのレキシカルスコープ |
| イベントハンドラ（インライン） | アロー関数 | thisの明確なバインド |
| 即時実行関数（IIFE） | アロー関数 | 簡潔 |

### 5.3 なぜfunction宣言を基本とするか

1. **使用できない場面が少ない**: アロー関数はコンストラクタ、ジェネレータ、super呼び出しで使用不可
2. **thisバインディングは呼び出し側の責務**: メソッド参照渡し時は`() => obj.method()`で包む
3. **スタックトレースの可読性**: function宣言は関数名が表示される
4. **業界標準との整合性**: Google、React公式、MUIが採用

### 5.4 コード例

```typescript
// ✅ 名前付き関数: function宣言
function calculateTotal(items: Item[]): number {
  return items.reduce((sum, item) => sum + item.price, 0);
}

// ✅ Reactコンポーネント: function宣言
function UserProfile({ user }: UserProfileProps) {
  const handleClick = () => {
    console.log('clicked');
  };

  return (
    <div onClick={handleClick}>
      {user.name}
    </div>
  );
}

// ✅ コールバック: アロー関数
const doubled = numbers.map((n) => n * 2);

// ✅ イベントハンドラ登録: アロー関数で包む
element.addEventListener('click', () => handler.onClick());

// ❌ Bad: 名前付き関数にアロー関数を使用
const computeSum = (values: number[]): number => {
  return values.reduce((sum, v) => sum + v, 0);
};

// ❌ Bad: メソッド参照を直接渡す（thisが壊れる）
setTimeout(obj.method, 1000);
// ✅ Good: アロー関数で包む
setTimeout(() => obj.method(), 1000);
```

### 5.5 その他のルール

- **デフォルト引数**: 末尾に配置、副作用禁止
- **レストパラメータ**: `arguments`の代わりに使用
- **パラメータの再代入**: 禁止
- **単一引数のアロー関数**: 括弧推奨（`(x) => x * 2`）

---

## 6. オブジェクト・配列

### 6.1 リテラル構文

```typescript
// ✅ Good: リテラル構文
const user: User = { name: 'John', age: 30 };
const items: string[] = ['apple', 'banana'];

// ✅ Good: 空の初期化（型注釈必須）
const config: Record<string, unknown> = {};
const queue: number[] = [];

// ❌ Bad: コンストラクタ
const obj = new Object();
const arr = new Array();
```

### 6.2 ショートハンド

```typescript
const name = 'John';
const age = 30;

// ✅ Good: プロパティショートハンド
const user = { name, age };

// ✅ Good: メソッドショートハンド
const obj = {
  greet() {
    return 'Hello';
  },
};
```

### 6.3 分割代入

```typescript
// ✅ Good: オブジェクト分割代入
const { name, age } = user;
const { name: userName } = user;  // リネーム

// ✅ Good: 配列分割代入
const [first, second] = items;
const [, , third] = items;  // スキップ

// ✅ Good: デフォルト値
type Options = { name?: string };
const options: Options = {};
const { name: guestName = 'Guest' } = options;
function greet({ name = 'Guest' }: Options) {}
```

### 6.4 スプレッド構文

```typescript
// ✅ Good: 配列のコピー・結合
const arrayCopy = [...original];
const merged = [...arr1, ...arr2];

// ✅ Good: オブジェクトのコピー・マージ
const objectCopy = { ...original };
const combined = { ...defaults, ...overrides };
```

---

## 7. 文字列

### 7.1 テンプレートリテラル

```typescript
// ✅ Good: 変数埋め込み
const message = `Hello, ${name}!`;

// ✅ Good: 複数行
const html = `
  <div>
    <p>${content}</p>
  </div>
`;

// ❌ Bad: 文字列連結
const badMessage = 'Hello, ' + name + '!';
```

### 7.2 クォートの使い分け

```typescript
// ✅ 基本: シングルクォート
const name = 'John';

// ✅ HTMLを含む場合: シングルクォート
const html = '<div class="container">Content</div>';

// ✅ シングルクォートを含む場合: テンプレートリテラル
const message = `It's a beautiful day`;
```

---

## 8. 比較・条件分岐

### 8.1 等価演算子

| 演算子 | 使用 |
|--------|------|
| `===`, `!==` | **基本** |
| `== null` | **許可**（null/undefined同時チェック） |
| `!= null` | **許可**（null/undefined同時チェック） |
| `==`, `!=`（その他） | **禁止** |

```typescript
// ✅ Good
if (value === 'active') {}
if (count !== 0) {}
if (value == null) {}  // null or undefined
if (value != null) {}  // value is not null and not undefined

// ✅ Good: 早期リターンで== nullを使用
if (value == null) {
  return;
}
// valueはnull/undefinedでないことが保証される

// ❌ Bad
if (value == 'active') {}
if (count != 0) {}
```

### 8.2 早期リターン

```typescript
// ✅ Good: 早期リターン
function process(user: User | null): string {
  if (user == null) {
    return 'No user';
  }
  if (!user.isActive) {
    return 'Inactive';
  }
  return user.name;
}

// ❌ Bad: ネストが深い
function processNested(user: User | null): string {
  if (user == null) {
    return 'No user';
  } else {
    if (user.isActive) {
      return user.name;
    } else {
      return 'Inactive';
    }
  }
}
```

### 8.3 三項演算子

```typescript
// ✅ Good: 単純な条件
const status = isActive ? 'Active' : 'Inactive';

// ❌ Bad: ネストした三項演算子
const result = isActive ? (isPremium ? 'Premium' : 'Standard') : 'Inactive';

// ✅ Good: 複雑な場合はif文
let label: string;
if (!isActive) {
  label = 'Inactive';
} else if (isPremium) {
  label = 'Premium';
} else {
  label = 'Standard';
}
```

### 8.4 短絡評価・Nullish Coalescing・Optional Chaining

| 演算子 | 使用 |
|--------|------|
| `??`（Nullish Coalescing） | **基本**（デフォルト値設定） |
| `?.`（Optional Chaining） | **基本**（ネストしたプロパティアクセス） |
| `\|\|`（論理OR）でのデフォルト値設定 | **禁止** |
| `&&`チェーンでのプロパティアクセス | **禁止**（`?.`で代替） |
| `&&`による条件付き実行 | 1式のみ許可（2項以上のチェーン禁止） |

`||`は`0`、`''`（空文字）、`false`、`NaN`をfalsyとして扱うため、意図しないフォールバックが発生する。`??`は`null`と`undefined`のみをフォールバック対象とし、有効な値を上書きしない。

```typescript
// ✅ Good: Nullish Coalescing
const port = config.port ?? 3000;          // config.port が 0 でも 0 が使われる
const name = user.displayName ?? 'Guest';  // '' でも '' が使われる

// ❌ Bad: 論理OR（0 や '' が無視される）
const port = config.port || 3000;          // config.port が 0 だと 3000 になる
const name = user.displayName || 'Guest';  // '' だと 'Guest' になる
```

```typescript
// ✅ Good: Optional Chaining
const city = user?.address?.city;
const result = callback?.();
const value = arr?.[0];

// ❌ Bad: && チェーン
const city = user && user.address && user.address.city;
```

```typescript
// ✅ Good: Optional Chaining + Nullish Coalescing
const city = user?.address?.city ?? 'Unknown';
const label = item?.meta?.label ?? 'Untitled';
```

```typescript
// ✅ Good: 単純な条件付き実行（1式で完結する場合）
isEnabled && initialize();

// ❌ Bad: 2項以上のチェーン
response && response.data && processData(response.data);
// ✅ Good: Optional Chainingで書き直す
response?.data && processData(response.data);
// ✅ Good: if文で書く
if (response?.data) {
  processData(response.data);
}
```

参照元:
- [MDN: Nullish coalescing operator (??)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing)
- [MDN: Optional chaining (?.)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining)

### 8.5 オブジェクトマップパターン

| 場面 | 推奨 |
|------|------|
| 3分岐以上の値マッピング | **オブジェクトマップ**（`Record<Key, Value>`） |
| 副作用を伴う制御フロー分岐 | `switch`文 |
| フォールスルーが意図的に必要 | `switch`文 |

```typescript
// ✅ Good: オブジェクトマップ
const statusLabel: Record<Status, string> = {
  active: 'アクティブ',
  inactive: '非アクティブ',
  pending: '保留中',
  suspended: '停止',
};
const label = statusLabel[user.status];

// ❌ Bad: switch文による値マッピング
function getLabel(status: Status): string {
  switch (status) {
    case 'active':
      return 'アクティブ';
    case 'inactive':
      return '非アクティブ';
    case 'pending':
      return '保留中';
    case 'suspended':
      return '停止';
  }
}
```

`Record<Key, Value>`で定義することで、キーの網羅性がコンパイル時に保証される。新しいキーが追加された場合、エントリがなければ型エラーになる。

```typescript
type UserRole = 'viewer' | 'editor' | 'admin' | 'owner';

// 型エラー: 'owner' が不足 → コンパイル時に検出
const rolePermissions: Record<UserRole, string[]> = {
  viewer: ['read'],
  editor: ['read', 'write'],
  admin: ['read', 'write', 'delete'],
  // owner が未定義 → TypeScriptがエラーを出す
};
```

```typescript
// ✅ switch文が適切: 分岐ごとに異なる副作用
switch (action.type) {
  case 'fetch':
    await fetchData();
    break;
  case 'save':
    await saveData(action.payload);
    notify('保存しました');
    break;
  case 'delete':
    await deleteData(action.id);
    break;
}
```

参照元:
- [Google TypeScript Style Guide](https://google.github.io/styleguide/tsguide.html)

---

## 9. 非同期処理・エラーハンドリング

### 9.1 async/await基本

```typescript
import { z } from 'zod';

const userSchema = z.object({
  id: z.string(),
  name: z.string(),
});
type User = z.infer<typeof userSchema>;

// ✅ Good: async/await
async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  if (!response.ok) {
    throw new Error(`Failed to fetch user: ${response.status}`);
  }
  const data: unknown = await response.json();
  return userSchema.parse(data);
}

// ❌ Bad: .then()チェーン（複雑な場合）
function fetchUserLegacy(id: string): Promise<User> {
  return fetch(`/api/users/${id}`)
    .then((response) => {
      if (!response.ok) {
        throw new Error(`Failed to fetch user: ${response.status}`);
      }
      return response.json();
    });
}
```

### 9.2 エラーハンドリング

```typescript
// ✅ Good: try-catch
async function processData(): Promise<void> {
  try {
    const data = await fetchData();
    await saveData(data);
  } catch (error: unknown) {
    if (error instanceof NetworkError) {
      console.error('Network error:', error.message);
    } else if (error instanceof ValidationError) {
      console.error('Validation error:', error.message);
    } else {
      throw error;  // 未知のエラーは再スロー
    }
  }
}

// ✅ Good: Errorのみをthrow
throw new Error('Something went wrong');

// ❌ Bad: 文字列をthrow
throw 'Something went wrong';
```

### 9.3 並列実行

```typescript
// ✅ Good: Promise.all（すべて成功が必要）
const [users, posts] = await Promise.all([
  fetchUsers(),
  fetchPosts(),
]);

// ✅ Good: Promise.allSettled（部分的失敗を許容）
const results = await Promise.allSettled([
  fetchUsers(),
  fetchPosts(),
]);

results.forEach((result) => {
  if (result.status === 'fulfilled') {
    console.log(result.value);
  } else {
    console.error(result.reason);
  }
});
```

### 9.4 タイムアウト

```typescript
// ✅ Good: AbortController
async function fetchWithTimeout(
  url: string,
  timeoutMs: number
): Promise<Response> {
  const controller = new AbortController();
  const timeoutId = setTimeout(() => controller.abort(), timeoutMs);

  try {
    const response = await fetch(url, { signal: controller.signal });
    return response;
  } finally {
    clearTimeout(timeoutId);
  }
}
```

---

## 10. モジュール

### 10.1 インポート順序

1. 標準ライブラリ / Node.js組み込み
2. 外部パッケージ（node_modules）
3. 内部モジュール（エイリアス: `@/`等）
4. 相対パス（親ディレクトリ `../`）
5. 相対パス（同ディレクトリ `./`）

各グループ間は空行で区切る。

**注意**: 絶対パス（ルートパス `/src/...`）は環境依存のため原則使用しない。パスエイリアス（`@/`等）で代替する。

```typescript
// ✅ Good
import fs from 'node:fs';
import path from 'node:path';

import React from 'react';
import { useQuery } from '@tanstack/react-query';

import { ApiClient } from '@/lib/api-client';
import type { User } from '@/types';

import { formatDate } from '../utils/date';

import { Button } from './button';
import styles from './styles.module.css';
```

### 10.2 エクスポート

| 項目 | ルール |
|------|--------|
| デフォルトエクスポート | **禁止** |
| 名前付きエクスポート | **推奨** |
| ワイルドカードインポート | 避ける（名前空間インポートは可） |

**注意**: 外部ライブラリのデフォルトインポート（`import React from 'react'`等）は許可。
禁止対象は自プロジェクト内のモジュールのデフォルトエクスポート。

```typescript
// ✅ Good: 名前付きエクスポート
export function createUser() {}
export const DEFAULT_CONFIG = {};
export type { User };

// ❌ Bad: デフォルトエクスポート（自プロジェクト内）
export default function createUserDefault() {}

// ✅ Good: 名前空間インポート
import * as utils from './utils';
utils.formatDate();

// ❌ Bad: ワイルドカード再エクスポート
export * from './utils';
```

---

## 11. ESLint/Prettier推奨設定

### 11.1 ESLint（Flat Config形式、ESLint 9+）

`import.meta.dirname` は Node.js 20.11以降を前提とする。未対応環境では `new URL('.', import.meta.url)` などで代替する。

```javascript
// eslint.config.js
import eslint from '@eslint/js';
import tseslint from 'typescript-eslint';
import eslintConfigPrettier from 'eslint-config-prettier';

export default tseslint.config(
  eslint.configs.recommended,
  ...tseslint.configs.strictTypeChecked,
  ...tseslint.configs.stylisticTypeChecked,
  eslintConfigPrettier,
  {
    languageOptions: {
      parserOptions: {
        projectService: true,
        tsconfigRootDir: import.meta.dirname,
      },
    },
    rules: {
      // any禁止
      '@typescript-eslint/no-explicit-any': 'error',
      // 未使用変数
      '@typescript-eslint/no-unused-vars': [
        'error',
        { argsIgnorePattern: '^_' },
      ],
      // function宣言推奨（関数式に対してのみ適用）
      'func-style': ['error', 'declaration', { allowArrowFunctions: true }],
      // デフォルトエクスポート禁止
      'no-restricted-exports': [
        'error',
        {
          restrictDefaultExports: {
            direct: true,
            named: true,
            defaultFrom: true,
            namedFrom: true,
            namespaceFrom: true,
          },
        },
      ],
    },
  }
);
```

### 11.2 Prettier

```jsonc
// .prettierrc
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 100,
  "bracketSpacing": true,
  "arrowParens": "always"
}
```

### 11.3 TypeScript設定

```jsonc
// tsconfig.json（抜粋）
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}
```

---

## 12. JSDoc/TSDocコメント規約

### 12.1 基本ルール

- 公開API（export）には**必須**
- 内部実装は**省略可**（自明な場合）
- 型情報は**記載不要**（TypeScriptで表現）
- サンプルコードのコメントに句読点を入れない
- 1行で済むコメントを複数行で書かない
- 1行で済む説明は `//` を使用し、`/** ... */` を使わない
- 正当な括弧での注釈は許容する

### 12.2 記法

```typescript
/**
 * ユーザー情報を取得する
 *
 * @param id - ユーザーID
 * @returns ユーザー情報 見つからない場合はundefined
 * @throws {NetworkError} ネットワークエラー時
 *
 * @example
 * ```typescript
 * const user = await fetchUser('123');
 * console.log(user?.name);
 * ```
 */
async function fetchUser(id: string): Promise<User | undefined> {
  // ...
}

// ユーザーを表す型
type User = {
  // 一意識別子
  id: string;
  // 表示名
  name: string;
  // メールアドレス（オプション）
  email?: string;
};
```

### 12.3 禁止事項

```typescript
// ❌ Bad: 型情報の重複記載
/**
 * @param {string} id - ユーザーID
 * @returns {Promise<User>}
 */
async function getUser(id: string): Promise<User> {}

// ❌ Bad: 自明なコメント
// ユーザーIDを返す
function getUserId(): string {
  return this.id;
}
```

---

## 出典・参考情報
### 主要な一次情報（公式）

- [Google TypeScript Style Guide](https://google.github.io/styleguide/tsguide.html)
- [TypeScript Handbook: Everyday Types](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html)
- [TypeScript Handbook: Object Types](https://www.typescriptlang.org/docs/handbook/2/objects.html)
- [TypeScript Handbook: Declaration Merging](https://www.typescriptlang.org/docs/handbook/declaration-merging.html)
- [ESLint: func-style](https://eslint.org/docs/latest/rules/func-style)
- [ESLint: no-restricted-exports](https://eslint.org/docs/latest/rules/no-restricted-exports)
- [MDN: Nullish coalescing operator (??)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Nullish_coalescing)
- [MDN: Optional chaining (?.)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining)
- [Node.js ESM (`import.meta.dirname`)](https://nodejs.org/api/esm.html)
- [React Docs: Your First Component](https://react.dev/learn/your-first-component)
- [MUI TypeScript Guide](https://mui.com/material-ui/guides/typescript/)

### 補足資料（コミュニティ）

- [TypeScript Deep Dive 日本語版](https://typescript-jp.gitbook.io/deep-dive/styleguide)
- [Zenn - TypeScript コーディング規約](https://zenn.dev/kiman/articles/db95ceeb925fe4)
- [Qiita - functionとアロー関数](https://qiita.com/nicco_mirai/items/0bd0d0bcee72497d4b42)
