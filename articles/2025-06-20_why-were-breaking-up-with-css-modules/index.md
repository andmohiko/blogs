---
title: Why We're Breaking Up with CSS Modules
slug: why-were-breaking-up-with-css-modules
description: 今まではCSS Modulesを使ってきたのですが、最近考えが変わり、今後はTailwind CSSを推したいと思います。今回は、この考えの変化について詳しく話したいと思います。
date: '2025-06-20'
---

## はじめに

今まではCSS Modulesを使ってきたのですが、最近考えが変わり、今後はTailwind CSSを推したいと思います。今回は、この考えの変化について詳しく話したいと思います。

## tl;dr

バイブコーディングとの相性を考えた結果、

- 責務が分離されているCSS ModulesはむしろAIが文脈を把握しづらくなり、本来メリットだったことが逆に足枷になっている
- AIにコーディングを任せるなら可読性が低くてもいいのでTailwind CSSのデメリットが小さくなっている

という考えになり、バイブコーディング寄りのプロジェクトではTailwind CSSを選択していきます。

## Reactにおけるスタイリング手法の比較

まず前提として、Reactにおけるスタイリングの選択肢を整理してみましょう。

- グローバルCSS
- CSS-in-JS
- コンポーネントライブラリ
- CSS Modules
- Tailwind CSS

### 1. グローバルCSS

普通にCSSファイルを作成してスタイルを書く方法です。デメリットとして、グローバルにクラス名が当たるため、同じクラス名を使用していた場合に予期せぬ影響を他に与えてしまう可能性があることが大きいと思います。

### 2. CSS-in-JS

CSS-in-JSは、JavaScriptファイル内でCSSを直接記述する手法です。コンポーネントのロジックとスタイルを同じファイルに書けるのが特徴です。代表的なライブラリとして、emotionやstyled-componentsなどがあります。

```tsx
// @emotion/react の例
function Button({ children }) {
  return (
    <button
      css={{
        backgroundColor: 'blue',
        color: 'white',
        padding: '8px 16px',
        border: 'none',
        borderRadius: '4px',
      }}
    >
      {children}
    </button>
  );
}

// styled-components の例
const StyledButton = styled.button`
  background-color: blue;
  color: white;
  padding: 8px 16px;
  border: none;
  border-radius: 4px;
`;
```

メリットとしては、

- ローカルスコープスタイル：クラス名の衝突を自動的に防げる
- コロケーション：スタイルとコンポーネントを同じファイルに書ける
- JavaScriptの変数やpropsを直接スタイルで使用可能

ということが挙げられます。
デメリットとしては、

- パフォーマンス問題
   - ランタイムでスタイルのシリアライゼーションが発生
   - レンダリング時にCSSルールが挿入され、ブラウザがDOM全体のスタイル再計算を実行
- バンドルサイズの増加
- React Server Componentsとの相性の悪さ

などがあります。
これらの解決策としてZero Runtime CSS-in-JSというアプローチもありますが、デファクトスタンダードなライブラリがまだ存在しないため、プロダクションでは採用しづらいと感じています。

メリットとしてスタイルとコンポーネントを同じファイルに書けることを挙げていますが、筆者はむしろこれはデメリットだと感じており、ドキュメントの構造とスタイリングという責務が異なるものは別のファイルに書きたいと思ってしまいます。

### 3. コンポーネントライブラリ

コンポーネントライブラリは、あらかじめスタイルが適用されたコンポーネントを提供するライブラリです。Material UI、Chakra UI、Mantineなどが代表的です。

```tsx
// Mantine の例
import { Button } from '@mantine/core';

function Example() {
  return (
    <div>
      <Button variant="filled" color="blue">
        プライマリボタン
      </Button>
    </div>
  );
}
```

スタイルだけでなく、コンポーネントの機能や振る舞い（ロジック）も実装されているため、実装コストが低いことがメリットとしてあります。
一方で、デメリットとしては

- デザインの自由度が低い
- propsでの指定や、ライブラリ特有の指定方法があるため、デザインのカスタムが複雑

などが挙げられます。
デザインや世界観へのこだわりが大切なプロダクト開発ではあまり使わないかもしれません。

このデメリットへの解決策として、ヘッドレスUIライブラリがあります。ヘッドレスUIライブラリは、コンポーネントの機能や振る舞いのみを提供し、スタイルは開発者が自由に実装できるライブラリです。Radix UI、Headless UI、React Ariaなどが代表的です。

```tsx
// Radix UI の例
import { Button } from '@radix-ui/react-button';

function CustomButton() {
  return (
    <Button className="custom-button">
      {/* 機能はライブラリが提供、スタイルは自分で実装 */}
      クリックしてください
    </Button>
  );
}
```

これにより、コンポーネントの複雑なロジック（アクセシビリティ対応など）はライブラリに任せつつ、デザインは完全にカスタマイズできるため、デザインの自由度とコンポーネントの品質を両立できます。
Mantineはスタイルを読み込まないで使用することでヘッドレスUIライブラリとして使用することもできるため、弊社ではMantineを使用する場面が多かったです。

### 4. CSS Modules

CSS Modulesは、CSSファイルをJavaScriptモジュールとしてインポートする仕組みです。ビルド時にクラス名に一意のハッシュ値が自動的に付与され、ローカルスコープを実現します。

```
/* Button.module.css */
.button {
  padding: 8px 16px;
  border: none;
  border-radius: 4px;
  font-size: 14px;
  cursor: pointer;
  transition: background-color 0.2s;
}

// Button.tsx
import styles from './Button.module.css';

function Button() {
  return (
    <button className={styles.button}>
      {children}
    </button>
  );
}
```

メリットとしては、ネイティブのCSSを書きつつ、グローバルCSSのデメリットであるスタイルの衝突を解決しています。
デメリットとしては、JavaScriptの変数を直接使用できないことや、ファイルが分割されてしまうことが挙げられます。特に、1行のCSSのためにファイルが1つ増えることをストレスに感じることがあるかもしれません。また、クラス名を自分で考えなければいけないということを負荷に感じるという声も聞きます。

### 5. Tailwind CSS

Tailwind CSSは、ユーティリティファーストのCSSフレームワークです。あらかじめ定義された小さなユーティリティクラスを組み合わせてスタイリングを行います。

```tsx
// Tailwind CSS の例
function Button() {
  return (
    <button className="bg-blue-500 hover:bg-blue-600 text-white font-medium py-2 px-4 rounded transition-colors">
      クリックしてください
    </button>
  );
}
```

メリットとしては、

- CSS-in-JSのデメリットであるパフォーマンスの問題も解決している
- インラインで直接クラスを指定するため、CSS Modulesのデメリットであるクラス名を毎回考える必要もない

ことが挙げられます。
デメリットとしては、

- HTMLの部分にスタイル情報がたくさん入るため、ドキュメントの構造とスタイルという異なる責務が1つのファイルに書かれる
- クラス名がどんどん長くなり、可読性・保守性が低い。他人が書いたTailwindはメンテナンスしたくない

また、独自のクラス名を覚える必要があるため、学習コストが高いこともデメリットとして挙げられますが、筆者はそこまで学習コストを感じませんでした。

## これまでの結論：CSS Modulesが最適解

ここまでのメリット・デメリットを考えると、デザインへのこだわりや世界観へのこだわりを持ってプロダクト開発をしている弊社では、CSS Modulesが一番良い選択なのではないかと思って今まで使ってきました。
CSS Modulesの本質はCSSをReactに取り込む仕組みであり、CSS Modulesが廃れてもCSSという資産は残ります。筆者はこれが一番のメリットだと考えていました。

## 考えの変化：なぜTailwind CSSに移行するのか

ただ、ここに来て私の考えは変わり、今後はTailwind CSSを使っていきたいと思います。
最近はバイブコーディングする機会も増え、AIが生成するコードの品質を高めることが重要だと考えています。いくつかの技術スタックでバイブコーディングしてみた結果、AIとの相性が良いのはTailwindだと感じました。
スタイルをコンポーネントのtsxファイルに書くことでAIが開発状況を理解しやすく、シングルファイルコンポーネントにするメリットの方が高いと最近は思うようになりました。
前述の通り、Tailwindのデメリットとして可読性・保守性の低さがありますが、すべてAIに書かせるなら可読性も保守性も優先度が下がります。

もちろん、引き続きエンジニアがCSSを書くプロジェクトもあるため、Tailwindの採用条件としては次の条件が判断軸になってくると思います。

- バイブコーディングの比率が高いプロジェクト
- 適切にコンポーネントを分割することで、Tailwindの可読性がなるべく下がらないように書ける場合

エンジニアが自分でCSSを書くことの方が多いなら、引き続きCSS Modulesを使いたいと思います。

## さいごに

CSS Modulesは今でも良い技術だと思います。特にCSSという資産が残り続けるのは魅力的です。

ただ、バイブコーディングすることが増えてくると、AIとの相性という観点も判断軸に入ってきます。実際にいろいろな技術スタックでバイブコーディングしてみた結果、自分で書くならCSS Modules、AIと一緒に書くならTailwind CSSという使い分けをしていくことになりそうです。
