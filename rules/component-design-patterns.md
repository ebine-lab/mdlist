---
title: "Component Design Patterns"
description: "コンポーネント設計パターン - Atoms/Molecules/Organisms分類・Props設計・状態管理・ディレクトリ構造 / Component design patterns - Atoms/Molecules/Organisms, props design, state management, directory structure"
version: "1.1.0"
status: "Stable"
last_updated: "2026-03-28T23:09+09:00"
lang: "ja"
---

# Component Design Patterns

**コンポーネント設計パターン** — React コンポーネントの設計判断、実装パターン、アンチパターンを定義する。


---

## 目的

本文書は、再利用性・保守性・アクセシビリティを兼ね備えたコンポーネントを設計するための判断基準と実装ガイドを提供する。パターンの羅列ではなく、「いつ何を選ぶか」「なぜそうするか」「何が落とし穴か」を中心に構成する。

## 背景

コンポーネントの Props API 設計・状態管理・合成パターンに関する判断は、コードベース全体の保守性に直結する。特に React 19 の安定版リリース（2024-12-05）により、`forwardRef` の不要化（将来削除予定）・Context API の簡略化・新しい非同期パターン（Actions）が導入されたため、既存の設計慣行を見直す必要がある。

## 対象

- React 19 以降を使用するフロントエンド開発
- TypeScript `strict` モード有効
- 本プロジェクトの コンポーネントおよびカスタムフック設計全般

## 非対象

- Server Components の詳細設計（別ドキュメントで扱う）
- CSS-in-JS / スタイリング手法
- テストの詳細実装（別ドキュメントで扱う）

## 用語

| 用語 | 定義 |
| --- | --- |
| Controlled | 状態が親コンポーネントの props によって管理されるコンポーネント |
| Uncontrolled | 状態をコンポーネント自身が `useState` で管理するコンポーネント |
| Hybrid | 両モードを切り替えて動作するコンポーネント |
| asChild | 子コンポーネントに props をマージする合成パターン（Radix UI 由来） |
| ロービングタブインデックス | グループ内のフォーカスを矢印キーで制御するアクセシビリティパターン |
| Action | React 19 における非同期トランジションを起動する関数 |

## 構成の根拠

| パターン | 根拠 |
| --- | --- |
| Props API 設計 | React 公式ドキュメント、maxschmitt.me（Controlled/Uncontrolled）、certificates.dev |
| asChild vs as prop | jacobparis.com、radix-ui.com、yisukim.com（2024年のコンセンサス） |
| Context 最適化 | react.dev/reference/react/memo、kentcdodds.com、developerway.com |
| メモ化判断 | react.dev/reference/react/useMemo、joshwcomeau.com、debugbear.com（React Compiler） |
| Actions | react.dev/blog/2024/12/05/react-19（React 19 公式） |
| ロービングタブインデックス | joshuawootonn.com、WAI-ARIA APG |

---

## 記述規則

本文書のコード例は **do / don't の順**（推奨例を先に示し、次に非推奨例を示す）で記述する。読者が最初に正しい実装を把握し、非推奨パターンを「なぜ避けるか」という文脈で理解できるようにするため。

---

## 1. 設計判断の前提

### コンポーネント分類

コンポーネントを設計する前に、その役割を明確にする。役割の混在は単一責任の原則に違反し、テストと再利用を困難にする。

| 分類 | 役割 | 例 |
| --- | --- | --- |
| 表示（Presentational） | UI のレンダリングのみ。状態を持たない | `Button`, `Avatar`, `Badge` |
| ロジック（Container） | データ取得・状態管理のみ。DOM を直接レンダリングしない。Hooks 以前の設計で、現在は Custom Hooks による責務分離が主流 | `UserProfileContainer` |
| 合成（Compound） | 子コンポーネントと状態を共有する複合ウィジェット | `Tabs`, `Accordion`, `Dialog` |
| ユーティリティ | 機能を提供するが、UI を持たない | `ErrorBoundary`, `Portal` |

### 設計原則

**アクセシビリティファースト**: コンポーネントの設計段階から WAI-ARIA ロールとキーボード操作を組み込む。後付けのアクセシビリティ対応はコスト高になる。

**Props API の安定性**: 公開 Props の命名・型・デフォルト値は一度決定すると変更が困難になる。設計段階で消費者の視点から Props API を定義する。

**デザイントークン経由の参照**: 色・間隔・フォントサイズは必ずセマンティクストークン経由で参照する。プリミティブ値のハードコードを禁止する。

### ハードコーディングの禁止（コンポーネント層）

コンポーネント内部に意味の不明な定数値を直接記述することを禁止する。これはデザイントークンに限らず、全てのリテラル値に適用する。汎用的なルールは [coding-standards.md](coding-standards.md) を参照。本セクションはコンポーネント設計に固有の禁止事項を規定する。

#### コンポーネント固有の禁止対象

| 禁止対象 | 禁止例 | 対処 |
|---|---|---|
| 固定寸法 | `width: '300px'`, `height: '48px'` | デザイントークン `var(--size-*)` または props で外部注入 |
| z-index リテラル | `zIndex: 1000`, `z-index: 9999` | トークン `var(--z-index-*)` を定義し参照 |
| アニメーション値 | `transition: 'opacity 0.3s ease'` | トークン `var(--duration-*)`, `var(--easing-*)` |
| ブレークポイント | `@media (min-width: 768px)` | [css-breakpoints-guidelines.md](css-breakpoints-guidelines.md) のカスタムプロパティを使用 |
| アイコンサイズ | `fontSize: '24px'` | サイズバリアント props (`size="md"`) で制御 |
| 角丸 | `borderRadius: '8px'` | トークン `var(--radius-*)` |
| 影 | `boxShadow: '0 2px 4px rgba(0,0,0,0.1)'` | トークン `var(--shadow-*)` |

#### コンポーネント Props でのデフォルト値

Props のデフォルト値にもリテラルを使用しない。定数またはトークンから参照する。

```typescript
// ✅ 必須: 名前付き定数で意味を明示
const AVATAR_SIZE = {
  sm: 32,
  md: 40,
  lg: 56,
} as const;

type AvatarSize = keyof typeof AVATAR_SIZE;

function Avatar({ size = 'md' }: { size?: AvatarSize }) {
  const px = AVATAR_SIZE[size];
  // ...
}

// ❌ 禁止: Props デフォルト値にマジックナンバー
function Avatar({ size = 40, borderRadius = 999 }: AvatarProps) {
  // ...
}
```

#### 文字列リテラルの型安全化

コンポーネントのバリアント・ステートを文字列リテラルで分岐させる場合、`as const` 定数オブジェクトまたはユニオン型で型安全を確保する。散在する文字列比較を禁止する。

```typescript
// ✅ 必須: 型定義とトークンマッピング
const BADGE_VARIANTS = {
  success: 'var(--color-status-success)',
  warning: 'var(--color-status-warning)',
  error: 'var(--color-status-error)',
  info: 'var(--color-status-info)',
} as const;

type BadgeVariant = keyof typeof BADGE_VARIANTS;

function Badge({ variant }: { variant: BadgeVariant }) {
  return <span style={{ color: BADGE_VARIANTS[variant] }}>...</span>;
}

// ❌ 禁止: 散在する文字列リテラル
function Badge({ variant }: { variant: string }) {
  if (variant === 'success') return <span style={{ color: 'green' }}>...</span>;
  if (variant === 'error') return <span style={{ color: 'red' }}>...</span>;
}
```

---

## 2. Props API 設計

### 命名規則

| 種別 | 規則 | 例 |
| --- | --- | --- |
| Boolean Props | `is` / `has` / `can` プレフィックス | `isDisabled`, `hasError`, `canExpand` |
| イベントハンドラ | `on` プレフィックス + PascalCase の動詞 | `onChange`, `onOpenChange`, `onSelectionChange` |
| 子コンポーネント | `children` を基本とし、スロット的な子は名前を付ける | `header`, `footer`, `actions` |
| render props | `render` プレフィックス | `renderItem`, `renderEmpty` |

### Controlled / Uncontrolled / Hybrid（制御の三形態）

**選択基準**:

- 親が状態を必要としない → Uncontrolled
- 複数コンポーネント間の同期、または状態のプログラム的変更が必要 → Controlled
- ライブラリコンポーネントとして両方のユースケースを想定 → Hybrid

**重要制約**: コンポーネントはライフタイム中に制御モードを切り替えてはならない。`value` が `undefined` から具体的な値に変わる、またはその逆は React の警告を引き起こしバグの原因となる。

```typescript
// Hybrid コンポーネントの実装パターン
// value !== undefined を制御モードの判定条件とする

type AccordionProps = {
  // Controlled モード
  isOpen?: boolean;
  onOpenChange?: (isOpen: boolean) => void;
  // Uncontrolled モード
  defaultOpen?: boolean;
};

function Accordion({ isOpen, onOpenChange, defaultOpen = false }: AccordionProps) {
  const [internalOpen, setInternalOpen] = useState(defaultOpen);
  const isControlled = isOpen !== undefined;
  const open = isControlled ? isOpen : internalOpen;

  function handleToggle() {
    if (!isControlled) {
      setInternalOpen((prev) => !prev);
    }
    onOpenChange?.(!open);
  }

  return (
    <div>
      <button
        onClick={handleToggle}
        aria-expanded={open}
        aria-controls="accordion-content"
      >
        Toggle
      </button>
      <div id="accordion-content" hidden={!open}>
        Content
      </div>
    </div>
  );
}
```

### TypeScript による型安全な ARIA プロップ設計

`aria-label` と `aria-labelledby` を両方同時に指定すると `aria-labelledby` が優先され、`aria-label` は無視される。開発者の意図が不明確になるため、ユニオン型と `never` を使ってコンパイル時にどちらか一方のみを要求する。

```typescript
// aria-label と aria-labelledby の排他制約
type AccessibleNameProps =
  | { 'aria-label': string; 'aria-labelledby'?: never }
  | { 'aria-labelledby': string; 'aria-label'?: never };

type ButtonProps = AccessibleNameProps & {
  onClick: () => void;
  isDisabled?: boolean;
  children: React.ReactNode;
};

// ❌ コンパイルエラー: 両方同時に指定できない
// <Button aria-label="閉じる" aria-labelledby="title-id">×</Button>

// ✅ どちらか一方のみ
// <Button aria-label="閉じる">×</Button>
// <Button aria-labelledby="dialog-title">確認</Button>
```

### Prop Getters パターン

複数の関連 props（イベントハンドラ + ARIA 属性 + `id`）を返すカスタムフック。消費者はスプレッド構文で一括適用できる。

```typescript
type UseToggleReturn = {
  isOpen: boolean;
  getToggleProps: () => {
    onClick: () => void;
    'aria-expanded': boolean;
    'aria-controls': string;
  };
  getContentProps: () => {
    id: string;
    hidden: boolean;
  };
};

function useToggle(id: string): UseToggleReturn {
  const [isOpen, setIsOpen] = useState(false);

  return {
    isOpen,
    getToggleProps: () => ({
      onClick: () => setIsOpen((prev) => !prev),
      'aria-expanded': isOpen,
      'aria-controls': `${id}-content`,
    }),
    getContentProps: () => ({
      id: `${id}-content`,
      hidden: !isOpen,
    }),
  };
}

// 消費者側
function Accordion({ id }: { id: string }) {
  const { getToggleProps, getContentProps } = useToggle(id);

  return (
    <div>
      <button {...getToggleProps()}>見出し</button>
      <section {...getContentProps()}>コンテンツ</section>
    </div>
  );
}
```

### イベントハンドラの合成（callAll パターン）

ライブラリコンポーネントのイベントハンドラを消費者のハンドラと安全に合成する。

```typescript
function callAll<T extends unknown[]>(
  ...fns: Array<((...args: T) => void) | undefined>
): (...args: T) => void {
  return (...args) => fns.forEach((fn) => fn?.(...args));
}

// 使用例: ライブラリ内部の onClick と消費者の onClick を両方実行する
type ButtonProps = {
  onClick?: React.MouseEventHandler<HTMLButtonElement>;
  children: React.ReactNode;
};

function Button({ onClick, children }: ButtonProps) {
  function handleLibraryClick(e: React.MouseEvent<HTMLButtonElement>) {
    // ライブラリ内部の処理（ログ記録・アナリティクス等）
    console.log('button clicked');
  }

  return (
    <button onClick={callAll(handleLibraryClick, onClick)}>
      {children}
    </button>
  );
}
```

---

## 3. コンポーネント合成パターン

### Compound Components（Context ベース）

親コンポーネントが状態を管理し、子コンポーネントが Context 経由で状態にアクセスするパターン。`children` を自由に構成できる柔軟な API を実現する。

```typescript
import { createContext, useContext, useId, useState } from 'react';

// --- Context 定義 ---
type TabsContextValue = {
  activeTab: string;
  setActiveTab: (id: string) => void;
  baseId: string;
};

const TabsContext = createContext<TabsContextValue | null>(null);

function useTabsContext(): TabsContextValue {
  const ctx = useContext(TabsContext);
  if (!ctx) throw new Error('useTabsContext must be used within <Tabs>');
  return ctx;
}

// --- 親コンポーネント ---
type TabsProps = {
  defaultTab?: string;
  children: React.ReactNode;
};

function TabsRoot({ defaultTab, children }: TabsProps) {
  const baseId = useId();
  const [activeTab, setActiveTab] = useState(defaultTab ?? '');

  return (
    <TabsContext value={{ activeTab, setActiveTab, baseId }}>
      <div>{children}</div>
    </TabsContext>
  );
}

// --- 子コンポーネント ---

// TabList: role="tablist" には accessible name を付ける（aria-label または aria-labelledby）
function TabList({
  children,
  'aria-label': ariaLabel,
  'aria-labelledby': ariaLabelledby,
}: {
  children: React.ReactNode;
  'aria-label'?: string;
  'aria-labelledby'?: string;
}) {
  return (
    <div role="tablist" aria-label={ariaLabel} aria-labelledby={ariaLabelledby}>
      {children}
    </div>
  );
}

type TabTriggerProps = {
  value: string;
  children: React.ReactNode;
};

function TabTrigger({ value, children }: TabTriggerProps) {
  const { activeTab, setActiveTab, baseId } = useTabsContext();
  const isSelected = activeTab === value;

  return (
    <button
      role="tab"
      id={`${baseId}-tab-${value}`}
      aria-selected={isSelected}
      aria-controls={`${baseId}-panel-${value}`}
      tabIndex={isSelected ? 0 : -1}
      onClick={() => setActiveTab(value)}
      onKeyDown={(e) => {
        const tabList = e.currentTarget.closest('[role="tablist"]');
        if (!tabList) return;
        const tabs = Array.from(
          tabList.querySelectorAll<HTMLButtonElement>('[role="tab"]')
        );
        const currentIndex = tabs.indexOf(e.currentTarget);
        if (currentIndex === -1) return;

        let nextIndex = currentIndex;
        if (e.key === 'ArrowRight') nextIndex = (currentIndex + 1) % tabs.length;
        if (e.key === 'ArrowLeft') nextIndex = (currentIndex - 1 + tabs.length) % tabs.length;
        if (e.key === 'Home') nextIndex = 0;
        if (e.key === 'End') nextIndex = tabs.length - 1;
        if (nextIndex === currentIndex) return;

        e.preventDefault();
        const nextTab = tabs[nextIndex];
        nextTab.focus();
        nextTab.click();
      }}
    >
      {children}
    </button>
  );
}

type TabPanelProps = {
  value: string;
  children: React.ReactNode;
};

function TabPanel({ value, children }: TabPanelProps) {
  const { activeTab, baseId } = useTabsContext();

  return (
    <div
      role="tabpanel"
      id={`${baseId}-panel-${value}`}
      aria-labelledby={`${baseId}-tab-${value}`}
      hidden={activeTab !== value}
    >
      {children}
    </div>
  );
}

// Object.assign でまとめる: 関数への動的プロパティ追加は TypeScript が型エラーとして扱うため
const Tabs = Object.assign(TabsRoot, {
  List: TabList,
  Trigger: TabTrigger,
  Panel: TabPanel,
});

// --- 消費者側 ---
// <Tabs defaultTab="overview">
//   <Tabs.List aria-label="セクション切り替え">
//     <Tabs.Trigger value="overview">概要</Tabs.Trigger>
//     <Tabs.Trigger value="details">詳細</Tabs.Trigger>
//   </Tabs.List>
//   <Tabs.Panel value="overview">概要コンテンツ</Tabs.Panel>
//   <Tabs.Panel value="details">詳細コンテンツ</Tabs.Panel>
// </Tabs>
```

**React 19 での変更点**: `<TabsContext.Provider value={...}>` は `<TabsContext value={...}>` と書けるようになった。Provider は後方互換性のために残るが、新規コードでは省略形を使用する。

### ロービングタブインデックスの実装

タブグループ・ツールバー・ラジオグループなど、複合ウィジェットのキーボードナビゲーション実装に使用する。WAI-ARIA APG が定義するパターンに準拠する。

**原則**:
- グループ内の `tabIndex={0}` は常に 1 つだけ（現在フォーカス対象）
- それ以外はすべて `tabIndex={-1}`
- 矢印キーでフォーカスを移動し、同時に `tabIndex` を切り替える
- タブバックしたとき、最後にフォーカスされた要素に戻る

DOM から順序を取得するパターンを使うと、`children` API の自由度が増す:

```typescript
import { createContext, useCallback, useContext, useEffect, useMemo, useRef, useState } from 'react';

type RovingContext = {
  currentId: string;
  register: (id: string, el: HTMLElement) => void;
  unregister: (id: string) => void;
  focus: (id: string) => void;
  moveFocus: (direction: 'next' | 'prev' | 'first' | 'last') => void;
};

const RovingContext = createContext<RovingContext | null>(null);

function RovingRoot({ children }: { children: React.ReactNode }) {
  const elements = useRef(new Map<string, HTMLElement>());
  const [currentId, setCurrentId] = useState('');

  const getOrderedIds = useCallback((): string[] => {
    return [...elements.current.entries()]
      .sort(([, a], [, b]) => {
        const position = a.compareDocumentPosition(b);
        return position & Node.DOCUMENT_POSITION_FOLLOWING ? -1 : 1;
      })
      .map(([id]) => id);
  }, []);

  // currentId が変わったとき、DOM の focus を同期する（state updater 外で副作用を実行）
  useEffect(() => {
    if (currentId) {
      elements.current.get(currentId)?.focus();
    }
  }, [currentId]);

  const moveFocus = useCallback(
    (direction: 'next' | 'prev' | 'first' | 'last') => {
      const ids = getOrderedIds();
      // functional update で最新の prev を参照し次の id を返すのみ
      // focus は上の useEffect が currentId の変化を検知して実行する
      setCurrentId((prev) => {
        const currentIndex = ids.indexOf(prev);
        let nextIndex: number;
        switch (direction) {
          case 'next':  nextIndex = (currentIndex + 1) % ids.length; break;
          case 'prev':  nextIndex = (currentIndex - 1 + ids.length) % ids.length; break;
          case 'first': nextIndex = 0; break;
          case 'last':  nextIndex = ids.length - 1; break;
        }
        return ids[nextIndex];
      });
    },
    [getOrderedIds]
  );

  const register = useCallback((id: string, el: HTMLElement) => {
    elements.current.set(id, el);
    setCurrentId((prev) => prev || id); // 最初の登録のみ初期値をセット
  }, []);

  const unregister = useCallback((id: string) => {
    elements.current.delete(id);
  }, []);

  const focus = useCallback((id: string) => {
    setCurrentId(id);
  }, []);

  // currentId が変わるときのみ Context 値を再生成する
  const value = useMemo(
    () => ({ currentId, register, unregister, focus, moveFocus }),
    [currentId, register, unregister, focus, moveFocus]
  );

  return (
    <RovingContext value={value}>
      <div role="toolbar">{children}</div>
    </RovingContext>
  );
}

function RovingItem({ id, children }: { id: string; children: React.ReactNode }) {
  const ctx = useContext(RovingContext);
  if (!ctx) throw new Error('RovingItem must be used within RovingRoot');
  const ref = useRef<HTMLButtonElement>(null);
  const isActive = ctx.currentId === id;

  // register / unregister は useCallback で参照安定 → deps に含められる
  useEffect(() => {
    if (ref.current) ctx.register(id, ref.current);
    return () => ctx.unregister(id);
  }, [id, ctx.register, ctx.unregister]);

  return (
    <button
      ref={ref}
      tabIndex={isActive ? 0 : -1}
      onFocus={() => ctx.focus(id)}
      onKeyDown={(e) => {
        if (e.key === 'ArrowRight') ctx.moveFocus('next');
        if (e.key === 'ArrowLeft') ctx.moveFocus('prev');
        if (e.key === 'Home') ctx.moveFocus('first');
        if (e.key === 'End') ctx.moveFocus('last');
      }}
    >
      {children}
    </button>
  );
}
```

### Custom Hooks による責務分離

ロジックをコンポーネントから分離する基準:

- 同一のロジックを複数のコンポーネントで使う場合
- テストしやすい独立した単位として切り出したい場合
- ロジックが UI の構造と独立している場合

```typescript
// ✅ 良い例: ロジックをフックに分離
function useDisclosure(defaultOpen = false) {
  const [isOpen, setIsOpen] = useState(defaultOpen);
  const open = useCallback(() => setIsOpen(true), []);
  const close = useCallback(() => setIsOpen(false), []);
  const toggle = useCallback(() => setIsOpen((prev) => !prev), []);

  return { isOpen, open, close, toggle };
}

// 複数のコンポーネントで再利用
function Modal() {
  const { isOpen, open, close } = useDisclosure();
  // ...
}

function Dropdown() {
  const { isOpen, toggle } = useDisclosure();
  // ...
}
```

---

## 4. 多態的コンポーネント（Polymorphic Components）

### asChild パターン（推奨）

ライブラリコンポーネント（Radix UI, shadcn/ui が採用）の現代的な標準。`@radix-ui/react-slot` の `Slot` コンポーネントを使用する。

```typescript
// ✅ 推奨: asChild パターン
// npm install @radix-ui/react-slot

import { Slot } from '@radix-ui/react-slot';

type ButtonProps = {
  asChild?: boolean;
  children: React.ReactNode;
  className?: string;
  isDisabled?: boolean;
};

function Button({ asChild = false, children, className, isDisabled }: ButtonProps) {
  if (asChild) {
    return (
      <Slot
        className={className}
        aria-disabled={isDisabled || undefined}
        data-disabled={isDisabled ? '' : undefined}
        tabIndex={isDisabled ? -1 : undefined}
        onClick={(e: React.MouseEvent<HTMLElement>) => {
          if (!isDisabled) return;
          e.preventDefault();
          e.stopPropagation();
        }}
      >
        {children}
      </Slot>
    );
  }

  return (
    <button className={className} disabled={isDisabled}>
      {children}
    </button>
  );
}

// 消費者: リンクとして使いたい場合
// <Button asChild>
//   <a href="/dashboard">ダッシュボード</a>
// </Button>
// → <a href="/dashboard" class="..." ...> が生成される
```

**Slot の動作原理**: `React.cloneElement` ベースで、親の props を子の props にマージする。マージルールは「子の props が優先」。イベントハンドラは合成されるが、親側の処理を中断したい場合は `event.defaultPrevented` を確認する実装にする。

**asChild の制約と落とし穴**:

| 項目 | 内容 |
| --- | --- |
| 子要素の数 | 単一の React 要素のみ受け付ける |
| 型安全性 | 子コンポーネントへの props の型検査はコンパイル時に行われない |
| 複数 children | `Slottable` コンポーネントで対応可能 |
| 子が必要な props | 親から渡された props が子に存在しない場合、ランタイムで無視される |

### as プロップ（歴史的経緯と TypeScript の限界）

`as` プロップは要素の種類を動的に変更するパターンだが、TypeScript の型定義が複雑になる問題がある。

```typescript
// ❌ 避けるべき: as プロップの TypeScript 実装
// ネストされた多態的コンポーネントで TypeScript 5 の型解決に問題が発生する
type ButtonProps<T extends React.ElementType = 'button'> = {
  as?: T;
} & React.ComponentPropsWithoutRef<T>;

function Button<T extends React.ElementType = 'button'>({
  as: Component = 'button',
  ...props
}: ButtonProps<T>) {
  return <Component {...props} />;
}
// 問題: TypeScript の型推論が深くなると intellisense が著しく低下する
// 問題: ネストされた polymorphic コンポーネントで型エラーが発生する (TypeScript 5)
```

### 選択基準

| 状況 | 推奨 |
| --- | --- |
| アプリケーションコード | どちらでも可。`asChild` の方がシンプル |
| ライブラリ・デザインシステム | `asChild` パターン |
| TypeScript の型安全性が最優先 | `as` プロップを注意深く実装、または `asChild` |
| Radix UI / shadcn/ui を使用 | `asChild` パターン（標準）|

---

## 5. Ref の設計（React 19 対応）

### ref プロップ化（forwardRef の非推奨化）

React 19 から、関数コンポーネントは `ref` を標準の props として受け取れる。`forwardRef` は Deprecated とされ、将来のメジャーバージョンで削除予定。

```typescript
// ✅ React 19 以降（推奨）
type InputProps = {
  ref?: React.Ref<HTMLInputElement>;
  placeholder?: string;
};

function Input({ ref, placeholder, ...props }: InputProps) {
  return <input ref={ref} placeholder={placeholder} {...props} />;
}

// ⚠️ React 19 では非推奨（React 18 以前では必要だった）
import { forwardRef } from 'react';

const Input = forwardRef<HTMLInputElement, InputProps>(
  ({ placeholder, ...props }, ref) => {
    return <input ref={ref} placeholder={placeholder} {...props} />;
  }
);
```

**移行**: `npx codemod@latest react/19/migration-recipe` で自動移行できる。既存の `forwardRef` コードは引き続き動作する（後方互換性あり）。

**TypeScript 型定義**:

```typescript
// React 19 での ref の型定義
type CardProps = {
  ref?: React.Ref<HTMLDivElement>;
  children: React.ReactNode;
};

// Ref コールバックとクリーンアップ（React 19 新機能）
// React 19 では cleanup パターンを使う場合、el は常に要素（null にならない）。
// cleanup 関数を返したとき React がアンマウント時に自動で呼び出す
function AttachedElement({ id }: { id: string }) {
  return (
    <div
      ref={(el) => {
        // マウント時の処理
        analytics.track('element_mounted', { id });
        // クリーンアップ関数を返す（アンマウント時に React が呼ぶ）
        return () => analytics.track('element_unmounted', { id });
      }}
    />
  );
}
```

### useImperativeHandle による公開 API の制御

親コンポーネントに公開するメソッドを最小限に絞ることで、内部実装を隠蔽し将来の変更に耐えられる API を設計する。

**使用が適切な場面**:
- カスタム入力コンポーネントの `focus` / `reset` / `validate`
- Modal / Dialog の `open` / `close`
- アニメーションの `play` / `pause` / `reset`
- Canvas / Chart の描画・エクスポート機能

```typescript
import { useId, useImperativeHandle, useRef } from 'react';

// Modal コンポーネント: 内部実装を隠蔽しつつ宣言的 API を提供

type ModalHandle = {
  open: () => void;
  close: () => void;
};

type ModalProps = {
  ref?: React.Ref<ModalHandle>;
  title: string;
  children: React.ReactNode;
};

function Modal({ ref, title, children }: ModalProps) {
  const dialogRef = useRef<HTMLDialogElement>(null);
  // 同一ページに複数 Modal が存在しても id が衝突しない
  const titleId = useId();

  useImperativeHandle(
    ref,
    () => ({
      open: () => dialogRef.current?.showModal(),
      close: () => dialogRef.current?.close(),
    }),
    [] // 依存なし: メソッドは参照安定
  );

  return (
    <dialog ref={dialogRef} aria-labelledby={titleId}>
      <h2 id={titleId}>{title}</h2>
      {children}
    </dialog>
  );
}

// 消費者側
function App() {
  const modalRef = useRef<ModalHandle>(null);

  return (
    <>
      <button onClick={() => modalRef.current?.open()}>開く</button>
      <Modal ref={modalRef} title="確認">
        <p>本当に削除しますか？</p>
        <button onClick={() => modalRef.current?.close()}>キャンセル</button>
      </Modal>
    </>
  );
}
```

**設計原則**: `useImperativeHandle` は宣言的パターンで解決できない場合の最終手段として使用する。状態の代替としての使用は避け、props と `onChange` によるデータフローを優先する。

---

## 6. Context と状態管理

### React 19 の Context 変更

```typescript
// ✅ 新構文（React 19）
<ThemeContext value={theme}>
  {children}
</ThemeContext>

// ⚠️ 従来構文（後方互換のため引き続き動作）
<ThemeContext.Provider value={theme}>
  {children}
</ThemeContext.Provider>
```

### use() フックによる Promise と Context の消費

`useContext` と異なり、`use()` は条件分岐やループ内で呼び出せる。Promise と Context の両方を受け付ける。

```typescript
import { use, Suspense } from 'react';

// --- Promise の消費: Suspense と組み合わせる ---
function UserProfile({ userPromise }: { userPromise: Promise<User> }) {
  // Promise が解決するまで Suspense がフォールバックを表示する
  const user = use(userPromise);
  return <div>{user.name}</div>;
}

// 使用側
// <Suspense fallback={<Spinner />}>
//   <UserProfile userPromise={fetchUser(id)} />
// </Suspense>

// --- 条件分岐内での Context 消費 ---
// useContext と異なり条件分岐内でも呼び出せる
type FeatureContextValue = { config: { label: string } };
const FeatureContext = createContext<FeatureContextValue>({ config: { label: '' } });

function FeaturePanel({ isEnabled }: { isEnabled: boolean }) {
  if (!isEnabled) return null;
  // isEnabled が false の場合はここに到達しないため use() を条件内で使える
  const featureCtx = use(FeatureContext);
  return <div>{featureCtx.config.label}</div>;
}
```

**Suspense + ErrorBoundary の組み合わせ**: `use(promise)` は Promise が reject されると例外を投げる。`Suspense` はローディング中のフォールバックを担当し、エラー捕捉には `ErrorBoundary` が必要。

```typescript
// ✅ use(promise) を使うときの完全な構成
// <ErrorBoundary fallback={<ErrorMessage />}>
//   <Suspense fallback={<Spinner />}>
//     <UserProfile userPromise={fetchUser(id)} />
//   </Suspense>
// </ErrorBoundary>
//
// ErrorBoundary（下記 §12 参照）がなければ、fetch 失敗時にアプリ全体がクラッシュする
```

### Context 設計のアンチパターンと対策

Context の値が変更されると、その Context を消費するすべてのコンポーネントが再レンダリングされる。これは React の設計上の動作であり、適切な Context 設計で制御する。

**対策 1 — Context 分割（最も効果的）**:

```typescript
// ✅ 更新頻度ごとに Context を分割
const UserContext = createContext<User | null>(null);
const ThemeContext = createContext<'light' | 'dark'>('light');
const NotificationContext = createContext<Notification[]>([]);
```

**対策 2 — 状態と更新関数の分離**:

`useReducer` の `dispatch` は参照安定なため、アクションのみを使うコンポーネントは状態変更で再レンダリングされない。

```typescript
// ✅ 状態と dispatch を別 Context に分離
const CountStateContext = createContext<number>(0);
const CountDispatchContext = createContext<React.Dispatch<Action>>(() => {});

function CountProvider({ children }: { children: React.ReactNode }) {
  const [count, dispatch] = useReducer(reducer, 0);

  return (
    // useMemo 不要: dispatch は参照安定
    <CountStateContext value={count}>
      <CountDispatchContext value={dispatch}>
        {children}
      </CountDispatchContext>
    </CountStateContext>
  );
}
```

**対策 3 — useMemo による値の安定化**:

```typescript
// ✅ Context 値のオブジェクトをメモ化
function AuthProvider({ user, children }: AuthProviderProps) {
  const [status, setStatus] = useState<'idle' | 'loading'>('idle');

  // user または status が変わったときのみ新しいオブジェクトが生成される
  const value = useMemo(
    () => ({ user, status, setStatus }),
    [user, status]
  );

  return <AuthContext value={value}>{children}</AuthContext>;
}
```

**アンチパターン**: 無関係な値を単一 Context に詰め込む

```typescript
// ❌ 悪い例: 更新頻度が異なる値が同居
const AppContext = createContext({
  user: null,        // ログイン時のみ変わる
  theme: 'light',   // 設定変更時のみ変わる
  notifications: [], // 頻繁に変わる
  settings: {},      // 稀にしか変わらない
});
// notifications が変わるたびに user, theme, settings を使うコンポーネントも再レンダリング
```

**Kent C. Dodds の見解**: React は非常に高速。パフォーマンス問題が知覚できない段階で最適化するな。Context を分割しすぎるとメンテナンスが困難になる。

### Context vs Props の選択基準

| 状況 | 推奨 |
| --- | --- |
| 1〜2 階層の props の受け渡し | Props |
| 多くのコンポーネントで共有する値（テーマ、認証情報） | Context |
| 高頻度の更新（マウスポジション、アニメーション値） | Context は不向き。`useRef` または外部ストア（Zustand 等）を検討 |
| 複雑なグローバル状態 | Zustand / Jotai 等の状態管理ライブラリ |

---

## 7. レンダリング最適化とメモ化

### React Compiler（優先手段）

2025年10月に v1.0 安定版がリリースされた。ビルド時にコンポーネントと関数へ自動的にメモ化を適用できるため、手動での `useMemo` / `useCallback` / `React.memo` は多くのケースで削減できる。

導入環境:
- **Next.js 16.1**（2025年12月時点）: `reactCompiler: true` を `next.config.js` に設定
- **Expo SDK 54**: 新規プロジェクトでは有効。既存プロジェクトは設定を確認して有効化
- **Vite / その他**: `babel-plugin-react-compiler` を追加

```javascript
// next.config.js
/** @type {import('next').NextConfig} */
const nextConfig = {
  reactCompiler: true,
};

export default nextConfig;
```

**前提条件**: Rules of React を遵守している必要がある:
- コンポーネントは冪等（同じ props → 同じ出力）
- Props・state を直接変更しない
- 副作用は `useEffect` または イベントハンドラ内でのみ行う
- フックはトップレベルで呼ぶ

React DevTools でコンポーネントに **Memo ✨** バッジが表示されると Compiler が適用されている。

### 手動メモ化（Compiler 未導入時）

手動メモ化は問題が知覚できてから適用する。推測で追加しない。

**React.memo の適用基準**:
- 同じ props で頻繁に再レンダリングされる
- レンダリングのコストが高い（複雑な計算・大きな DOM ツリー）

**useMemo の適用基準**:
- 計算に 1ms 以上かかる
- 結果をメモ化した子コンポーネントに渡す
- `useEffect` の依存配列に含まれ、参照安定性が必要

**useCallback の適用基準**:
- `React.memo` でラップした子コンポーネントに渡す関数

```typescript
// ✅ 意味のあるメモ化
const sortedList = useMemo(
  () => [...items].sort((a, b) => a.name.localeCompare(b.name)),
  [items]
);

const MemoizedChart = memo(Chart); // 重い描画コンポーネント
const stableOnChange = useCallback(
  (value: string) => dispatch({ type: 'UPDATE', value }),
  [dispatch]
);
// <MemoizedChart onChange={stableOnChange} data={sortedList} />

// ❌ 過剰なメモ化（意味がない）
const greet = useCallback(() => 'Hello', []); // 再生成コストは無視できる
const doubled = useMemo(() => count * 2, [count]); // 計算が軽すぎる
```

**デバッグ**: React DevTools の Profiler でレンダリング時間を計測してから最適化する。

### useDeferredValue / useTransition

CPU を消費するレンダリングとユーザー操作の応答性を両立させる Concurrent Features。

```typescript
// useTransition: 重い状態更新を「非緊急」としてマーク
function SearchPage() {
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<Result[]>([]);
  const [isPending, startTransition] = useTransition();

  function handleSearch(e: React.ChangeEvent<HTMLInputElement>) {
    // 入力フィールドの更新は緊急（即時反映）
    setQuery(e.target.value);

    // 検索結果の更新は非緊急（中断可能）
    startTransition(() => {
      setResults(search(e.target.value));
    });
  }

  return (
    <>
      <input value={query} onChange={handleSearch} />
      {isPending && <Spinner />}
      <ResultList results={results} />
    </>
  );
}

// useDeferredValue: 値の更新を遅延させ古い値で表示を継続
function FilteredList({ filter }: { filter: string }) {
  const deferredFilter = useDeferredValue(filter);
  // filter が変わっても deferredFilter はすぐには変わらない
  // 重いフィルタリングは deferredFilter でのみ動作する
  const filtered = useMemo(() => expensiveFilter(deferredFilter), [deferredFilter]);

  return <List items={filtered} />;
}
```

---

## 8. 非同期・Actions パターン（React 19）

React 19 では、非同期操作（データ送信・楽観的更新）のボイラープレートを削減する新しいパターンが導入された。

### useActionState

フォームアクションの送信状態（pending / success / error）を一元管理する。従来の `useState` を複数組み合わせたパターンを置き換える。

```typescript
// ✅ React 19: useActionState で一元管理
import { useActionState } from 'react';

async function updateNameAction(
  _previousState: string | null,
  formData: FormData
): Promise<string | null> {
  const name = formData.get('name') as string;
  if (!name) return '名前を入力してください';
  await updateName(name);
  return null; // エラーなし
}

function NameForm() {
  const [errorMessage, submitAction, isPending] = useActionState(
    updateNameAction,
    null // 初期状態
  );

  return (
    <form action={submitAction}>
      <input name="name" type="text" />
      {errorMessage && (
        <p role="alert">{errorMessage}</p>
      )}
      <button type="submit" disabled={isPending}>
        {isPending ? '保存中…' : '保存'}
      </button>
      {/* 送信中の重複送信防止は disabled で実装する。 */}
    </form>
  );
}

// ❌ React 18 以前: 状態を個別に管理
function OldForm() {
  const [name, setName] = useState('');
  const [isPending, setIsPending] = useState(false);
  const [message, setMessage] = useState<string | null>(null);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setIsPending(true);
    const result = await updateName(name);
    setMessage(result.message);
    setIsPending(false);
  }
  // ...
}
```

### useOptimistic

非同期操作が完了する前に UI を即座に更新し、操作が失敗した場合に自動的にロールバックする。

```typescript
import { useOptimistic, startTransition } from 'react';

type Message = { id: string; text: string; pending?: boolean };

function MessageList({ messages }: { messages: Message[] }) {
  const [optimisticMessages, addOptimistic] = useOptimistic(
    messages,
    // reducer: 楽観的な状態の更新ロジック
    (state, newMessage: Message) => [...state, newMessage]
  );

  async function sendMessage(text: string) {
    const tempMessage: Message = {
      id: crypto.randomUUID(),
      text,
      pending: true,
    };

    startTransition(async () => {
      // 即座に UI を更新（楽観的）
      addOptimistic(tempMessage);
      // 実際の API 呼び出し
      await postMessage(text);
      // 完了後 messages が更新されて optimisticMessages もリセット
    });
  }

  return (
    <ul>
      {optimisticMessages.map((msg) => (
        <li key={msg.id} style={{ opacity: msg.pending ? 0.5 : 1 }}>
          {msg.text}
        </li>
      ))}
    </ul>
  );
}
```

### useFormStatus

フォームの状態（pending / data / method）を、prop drilling なしに子コンポーネントから参照する。

```typescript
import { useFormStatus } from 'react-dom';

// formAction: サーバーアクションを想定したスタブ（実際は Server Action として定義する）
async function formAction(_formData: FormData): Promise<void> {
  // サーバーサイド処理（例: データベース更新）
}

// Submit ボタンは <form> の子であればどこに配置しても FormStatus を参照できる
function SubmitButton() {
  const { pending } = useFormStatus();

  return (
    <button type="submit" disabled={pending}>
      {pending ? '送信中…' : '送信'}
    </button>
  );
}

// form の深いネスト内に配置可能
function ComplexForm() {
  return (
    <form action={formAction}>
      <fieldset>
        <legend>ユーザー情報</legend>
        <input name="name" />
        {/* SubmitButton は form の何階層下でも動作する */}
        <SubmitButton />
      </fieldset>
    </form>
  );
}
```

---

## 9. アクセシビリティとの統合

アクセシビリティはコンポーネント設計の後付けではなく、設計段階から組み込む。各セクションの実装例に ARIA 属性・キーボード操作を含めているが、ここでは統合的な観点を補足する。

### フォーカス管理

Dialog / Modal が開いたとき:
1. `aria-modal="true"` を設定する
2. 最初のフォーカス可能な要素または Dialog 自体にフォーカスを移動する
3. Tab / Shift+Tab のフォーカストラップを実装する
4. 閉じたとき、Dialog を開いたトリガー要素にフォーカスを戻す

```typescript
import { useEffect, useRef } from 'react';

type DialogProps = {
  isOpen: boolean;
  onClose: () => void;
  triggerRef: React.RefObject<HTMLElement | null>;
  children: React.ReactNode;
};

type OldDialogProps = {
  isOpen: boolean;
  onClose: () => void;
  children: React.ReactNode;
};

// ✅ <dialog> 要素を使う: フォーカストラップ・Escape・aria-modal をブラウザが処理する
function Dialog({ isOpen, onClose, triggerRef, children }: DialogProps) {
  const dialogRef = useRef<HTMLDialogElement>(null);

  useEffect(() => {
    const dialog = dialogRef.current;
    if (!dialog) return;

    if (isOpen) {
      dialog.showModal(); // フォーカストラップ・aria-modal・スクロールロックをブラウザが処理
    } else {
      dialog.close();
      triggerRef.current?.focus(); // 閉じたときトリガーにフォーカスを戻す
    }
  }, [isOpen, triggerRef]);

  useEffect(() => {
    // Escape キーで閉じたとき（ブラウザが自動処理）を React の状態に同期する
    const dialog = dialogRef.current;
    function handleClose() { onClose(); }
    dialog?.addEventListener('close', handleClose);
    return () => dialog?.removeEventListener('close', handleClose);
  }, [onClose]);

  return (
    // <dialog> 要素: showModal() により role="dialog" aria-modal="true" 相当が自動付与される
    <dialog ref={dialogRef}>
      {children}
    </dialog>
  );
}

// ❌ 避けるべき: <div role="dialog"> による手動実装
// フォーカストラップ・Escape 処理・スクロールロックをすべて自前で実装する必要がある
// 実装漏れによるアクセシビリティ不備が発生しやすい
// ブラウザサポートが不十分だった時代の手法であり現在は <dialog> 要素を使うべき
function OldDialog({ isOpen, onClose, children }: OldDialogProps) {
  const dialogRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (!isOpen) return;
    const focusable = dialogRef.current?.querySelector<HTMLElement>(
      'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
    );
    focusable?.focus();
    return () => { /* トリガーへのフォーカス戻しを忘れやすい */ };
  }, [isOpen]);

  if (!isOpen) return null;

  return (
    <div ref={dialogRef} role="dialog" aria-modal="true">
      {children}
    </div>
  );
}
```

### ライブリージョン

非同期操作の結果や動的に変化するコンテンツをスクリーンリーダーに通知する。

```typescript
import { useActionState } from 'react';

// formAction: string | null を返すサーバーアクション（エラーメッセージまたは null）
async function formAction(_prev: string | null, _data: FormData): Promise<string | null> {
  return null;
}

// ✅ aria-live による非同期通知
function StatusMessage({ message, type }: { message: string; type: 'info' | 'error' }) {
  return (
    <div
      role={type === 'error' ? 'alert' : 'status'}
      aria-live={type === 'error' ? 'assertive' : 'polite'}
      aria-atomic="true"
    >
      {message}
    </div>
  );
}

// ✅ フォーム送信結果の通知
// useActionState の状態型は string | null（§8 updateNameAction の戻り値に合わせている）
function Form() {
  const [errorMessage, submitAction, isPending] = useActionState(formAction, null);

  return (
    <form action={submitAction}>
      {/* errorMessage が変わったときにスクリーンリーダーが読み上げる */}
      {errorMessage && (
        <div role="alert" aria-live="assertive">
          {errorMessage}
        </div>
      )}
      <button disabled={isPending}>送信</button>
    </form>
  );
}
```

---

## 10. デザイントークン統合

### セマンティクストークン経由の参照ルール

コンポーネント内では必ずセマンティクストークンを使用する。プリミティブトークンへの直接参照はコンポーネント層で禁止する。

```typescript
// ✅ 正しい: セマンティクストークン経由（CSS Custom Properties を inline style で参照）
type ButtonProps = {
  children: React.ReactNode;
  className?: string;
};

function Button({ children, className }: ButtonProps) {
  return (
    <button
      className={className}
      style={{
        background: 'var(--color-action-primary)',   // セマンティクストークン
        padding: 'var(--spacing-2) var(--spacing-4)', // セマンティクストークン
        color: 'var(--color-text-on-action)',          // セマンティクストークン
      }}
    >
      {children}
    </button>
  );
}

// ❌ 禁止: プリミティブ値のハードコード
function BadButton1({ children }: { children: React.ReactNode }) {
  return (
    <button style={{ background: '#1a73e8', padding: '8px 16px' }}>
      {/* #1a73e8、8px 16px は禁止。テーマ切替・ダークモード対応が不可能になる */}
      {children}
    </button>
  );
}

// ❌ 禁止: プリミティブトークンの直接参照
function BadButton2({ children }: { children: React.ReactNode }) {
  return (
    <button style={{ background: 'var(--color-blue-600)' }}>
      {/* プリミティブトークン（--color-blue-600）の直接参照は禁止。
          セマンティクスの意味が失われ、テーマ変更時に追跡が困難になる */}
      {children}
    </button>
  );
}
```

### TypeScript による型安全なトークン参照

```typescript
import tokens from '@/design-system/tokens';

// トークンのキーを型として使う
type ColorToken = keyof typeof tokens.color;
type SpacingToken = keyof typeof tokens.spacing;

type CardProps = {
  backgroundColor?: ColorToken;
  padding?: SpacingToken;
  children: React.ReactNode;
};

function Card({ backgroundColor = 'surface-default', padding = '4', children }: CardProps) {
  return (
    <div
      style={{
        backgroundColor: `var(--color-${backgroundColor})`,
        padding: `var(--spacing-${padding})`,
      }}
    >
      {children}
    </div>
  );
}
```

---

## 11. Error Boundary

### 概要

`ErrorBoundary` はレンダリング中に発生した JavaScript エラーをキャッチし、クラッシュしたコンポーネントツリーの代わりにフォールバック UI を表示するコンポーネント。React では **クラスコンポーネントのみ** が実装できる（`getDerivedStateFromError` / `componentDidCatch` はクラス専用 API）。

### 実装

```typescript
import { Component, Suspense, type ErrorInfo, type ReactNode } from 'react';

type Props = {
  fallback: ReactNode;
  children: ReactNode;
};

type State = {
  hasError: boolean;
  error: Error | null;
};

class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false, error: null };

  static getDerivedStateFromError(error: Error): State {
    // 次のレンダリングでフォールバック UI を表示するよう state を更新する
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, info: ErrorInfo) {
    // エラーロギングサービスへの送信はここで行う
    console.error('ErrorBoundary caught:', error, info.componentStack);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback;
    }
    return this.props.children;
  }
}

// --- 消費者側 ---
function App() {
  return (
    <ErrorBoundary fallback={<p>エラーが発生しました。再読み込みしてください。</p>}>
      <Suspense fallback={<Spinner />}>
        <UserProfile userPromise={fetchUser(id)} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

### react-error-boundary（推奨ライブラリ）

クラスコンポーネントの自前実装を避けるには [`react-error-boundary`](https://github.com/bvaughn/react-error-boundary) を使用する。リセット機能・`useErrorBoundary` フック・関数コンポーネント形式のフォールバックなど、実用的な機能が揃っている。

```typescript
import { Suspense } from 'react';
import { ErrorBoundary } from 'react-error-boundary';

function ErrorFallback({ error, resetErrorBoundary }: { error: Error; resetErrorBoundary: () => void }) {
  return (
    <div role="alert">
      <p>エラーが発生しました: {error.message}</p>
      <button onClick={resetErrorBoundary}>再試行</button>
    </div>
  );
}

function App() {
  return (
    <ErrorBoundary FallbackComponent={ErrorFallback}>
      <Suspense fallback={<Spinner />}>
        <UserProfile userPromise={fetchUser(id)} />
      </Suspense>
    </ErrorBoundary>
  );
}
```

**キャッチできないエラー**:
- イベントハンドラ内のエラー（`try/catch` を使う）
- 非同期コード（`setTimeout` 等）のエラー
- サーバーサイドレンダリング中のエラー
- `ErrorBoundary` コンポーネント自身のエラー

---

## 12. 静的解析・テスト方針

### ESLint 設定

```javascript
// eslint.config.js（抜粋）
import eslintPluginJsxA11y from 'eslint-plugin-jsx-a11y';
import eslintPluginReactHooks from 'eslint-plugin-react-hooks';

export default [
  {
    plugins: {
      'jsx-a11y': eslintPluginJsxA11y,
      'react-hooks': eslintPluginReactHooks,
    },
    rules: {
      // アクセシビリティ
      'jsx-a11y/alt-text': 'error',
      'jsx-a11y/aria-props': 'error',
      'jsx-a11y/aria-role': 'error',
      'jsx-a11y/no-autofocus': 'warn',
      'jsx-a11y/interactive-supports-focus': 'error',
      // Hooks
      'react-hooks/rules-of-hooks': 'error',
      'react-hooks/exhaustive-deps': 'warn',
    },
  },
];
```

### コンポーネントのテスト方針

```typescript
// @testing-library/react を使用
// ロールとアクセシブルな名前でクエリする（実装詳細ではなく、利用者の視点でテスト）

import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

describe('Tabs', () => {
  test('キーボードで Tab 間を移動できる', async () => {
    render(
      <Tabs defaultTab="tab1">
        <Tabs.List>
          <Tabs.Trigger value="tab1">タブ 1</Tabs.Trigger>
          <Tabs.Trigger value="tab2">タブ 2</Tabs.Trigger>
        </Tabs.List>
        <Tabs.Panel value="tab1">コンテンツ 1</Tabs.Panel>
        <Tabs.Panel value="tab2">コンテンツ 2</Tabs.Panel>
      </Tabs>
    );

    const tab1 = screen.getByRole('tab', { name: 'タブ 1' });
    const tab2 = screen.getByRole('tab', { name: 'タブ 2' });

    tab1.focus();
    expect(tab1).toHaveAttribute('aria-selected', 'true');

    await userEvent.keyboard('{ArrowRight}');
    expect(tab2).toHaveFocus();
  });
});
```

---

## 出典・参考情報
- [React 19 ブログ（公式）](https://react.dev/blog/2024/12/05/react-19) - React 19 安定版リリースノート
- [React 公式: memo](https://react.dev/reference/react/memo) - メモ化の使用基準
- [React 公式: useMemo](https://react.dev/reference/react/useMemo) - メモ化の使用基準
- [React 公式: useImperativeHandle](https://react.dev/reference/react/useImperativeHandle) - Ref の公開 API 設計
- [React 公式: Sharing State Between Components](https://react.dev/learn/sharing-state-between-components) - Controlled / Uncontrolled の解説
- [React 公式: Error Boundaries](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary) - Error Boundary の実装
- [react-error-boundary](https://github.com/bvaughn/react-error-boundary) - Error Boundary 実装ライブラリ
- [Radix UI: Slot](https://www.radix-ui.com/primitives/docs/utilities/slot) - asChild パターンの実装
- [joshuawootonn.com: React Roving Tabindex](https://www.joshuawootonn.com/react-roving-tabindex) - ロービングタブインデックスの実装
- [kentcdodds.com: How to Optimize Your Context Value](https://kentcdodds.com/blog/how-to-optimize-your-context-value) - Context 最適化
- [developerway.com: How to Write Performant React Apps with Context](https://www.developerway.com/posts/how-to-write-performant-react-apps-with-context) - Context パフォーマンス
- [joshwcomeau.com: Understanding useMemo and useCallback](https://www.joshwcomeau.com/react/usememo-and-usecallback/) - メモ化の実践的な理解
- [debugbear.com: React Compiler](https://www.debugbear.com/blog/react-compiler) - React Compiler の詳解
- [WAI-ARIA APG: Tabs Pattern](https://www.w3.org/WAI/ARIA/apg/patterns/tabs/) - Tabs の ARIA 構造（tablist / tab / tabpanel）
- [WAI-ARIA Authoring Practices Guide](https://www.w3.org/WAI/ARIA/apg/) - ARIA パターンの一次情報
- [certificates.dev: Controlled vs Uncontrolled](https://certificates.dev/blog/controlled-vs-uncontrolled-components-in-react) - 制御の三形態
- [react.wiki: useImperativeHandle](https://react.wiki/hooks/use-imperative-handle/) - React 19 対応の useImperativeHandle 解説
