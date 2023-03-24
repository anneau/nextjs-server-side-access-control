---
# try also 'default' to start simple
theme: seriph
# random image from a curated Unsplash collection by Anthony
# like them? see https://unsplash.com/collections/94734566/slidev
background: https://source.unsplash.com/collection/94734566/1920x1080
# apply any windi css classes to the current slide
class: 'text-center'
# https://sli.dev/custom/highlighters.html
highlighter: shiki
# show line numbers in code blocks
lineNumbers: false
# some information about the slides, markdown enabled
info: |
  ## Slidev Starter Template
  Presentation slides for developers.

  Learn more at [Sli.dev](https://sli.dev)
# persist drawings in exports and build
drawings:
  persist: false
# page transition
transition: slide-left
# use UnoCSS
css: unocss
---

# Next.jsにおけるサーバーサイドアクセスコントロール

エイチーム×レバレジーズ フロントエンド勉強会

<div class="abs-br m-6 flex gap-2">
  <a href="https://github.com/anneau" target="_blank" alt="GitHub"
    class="text-xl slidev-icon-btn opacity-50 !border-none !hover:text-white">
    <carbon-logo-github />
  </a>
</div>

<!--
The last comment block of each slide will be treated as slide notes. It will be visible and editable in Presenter Mode along with the slide. [Read more in the docs](https://sli.dev/guide/syntax.html#notes)
-->

---
layout: image-right
image: /images/profile.jpg
---

# 小堀 輝（こぼり ひかる）

- エイチームライフデザイン リードエンジニア
- React, Next.js, TypeScript
- 🍷🐈🎮

<div class="flex flex-row gap-2">
<img src="/images/wasabi.jpg" class="h-150px" />
</div>

<style>
h1 {
  background-color: #2B90B6;
  background-image: linear-gradient(45deg, #4EC5D4 10%, #146b8c 20%);
  background-size: 100%;
  -webkit-background-clip: text;
  -moz-background-clip: text;
  -webkit-text-fill-color: transparent;
  -moz-text-fill-color: transparent;
}
</style>

<!--
Here is another comment.
-->

---
layout: center
class: text-center
---

# 今日お話しするのは

---
layout: center
class: text-center
---

# Next.jsにおけるアクセスコントロールの話

---
layout: image-right
image: /images/access_control.png
---

# アクセスコントロールとは？

### 以下のようなパターン

- マイページへのアクセスはトークンの検証ができないとアクセスできないようにする
<br />

- 社内管理画面はIPアドレス制限を行う
<br />

- 提携業社様用の管理画面は特別なトークンを持っていないとアクセスできないようにする
<br />

- ABテストでページを出し分ける

---
---

# アクセスコントロールをどこでするか？

1. クライアントサイド
2. サーバーサイド
3. エッジサーバーサイド

1にはいくつか課題があり、2や3を採用していけるといいよ。
<br />
という話と、僕たちのチームでNext.jsを用い、どのように実現しているのか？という話をします。

---
---

# クライアントサイドアクセスコントロール

アクセスコントロールが必要なページで、以下のようなhooksを呼び出し、制御を行う

```tsx {all|6|10|12-14}
import { useEffect } from 'react';
import { useAuth } from './firebase';
import { useRouter } from 'next/router';

export const useAssertIsAuthenticated = () => {
  const { user, loading } = useAuth();
  const { replace } = useRouter();

  useEffect(() => {
    if (loading) return;

    if (!user) {
      replace('/sign-in');
    }
  }, [user, loading, replace]);

  return { loading };
};
```

一見、良さそうに見えるがいくつか問題がある

---
---

# クライアントサイドアクセスコントロールの問題

1. ローディング状態を管理しないといけないこと
2. （認可が必要な場合）認証情報をクライアントサイドに持たないといけない
3. アクセスコントロールのコードが各ページにバラけてしまい、漏れが発生する可能性が上がる

---
---

# クライアントサイドアクセスコントロールの問題

1. ローディング状態を管理しないといけないこと  

```tsx
const { loading } = useAssertIsAuthenticated();
const data = useQuery(pageQuery, { skip: loading });

if (loading || !data) return <Loading />;
```

→ コードの保守性が崩れるし、作り方によっては、画面のちらつきが発生する可能性もある

---
---

# クライアントサイドアクセスコントロールの問題

2. （認可が必要な場合）認証情報をクライアントサイドに持たないといけない  

```ts {2}
export const useAssertIsAuthenticated = () => {
  const { user, loading } = useAuth(); // ここの内部でindexedDBやlocalStorageにアクセスしている
 ...
}
```

→ セキュリティ的なリスクが上がる（XSSされてしまった時にトークンの流出する可能性がある）

---
---

# クライアントサイドアクセスコントロールの問題

3. アクセスコントロールのコードが各ページにバラけてしまい、漏れが発生する可能性が上がる  
→ レイアウトの共通化などにより減らせはするが、根本的に解決するのは難しい

---
layout: center
class: text-center
---

# ではどうすればいいのか？

---
---

# 僕たちのサービスでやっている2パターンの解決策

1. サーバーサイドアクセスコントロール  
→ getServerSideProps
2. エッジサーバーサイドアクセスコントロール  
→ Middleware

---
layout: center
class: text-center
---

# 1. サーバーサイドアクセスコントロール  

---
---

# サーバーサイドアクセスコントロール  

Next.jsのサーバーサイドでアクセスコントロールをしてしまうという手法

```ts
export async function getServerSideProps(context: GetServerSidePropsContext) {
  const cookies = nookies.get(context);
  const { session } = cookies;
  if (!isAuthenticated(session)) {
    return {
      redirect: {
        parament: false,
        destination: '/sign-in',
      },
    };
  }
  return {
    props: {},
  };
}
```

---
---

# サーバーサイドアクセスコントロール

クライアントサイドアクセスコントロールで抱えていた課題の解消

1. ローディング状態を管理しないといけないこと  
→ 🤗 クライアントサイドがレンダリングされる時にはアクセスコントロール済みなのでローディングは不要
2. （認可が必要な場合）認証情報をクライアントサイドに持たないといけないためセキュリティリスクが上がる  
→ 🤗 http onlyなCookieにのみ認証情報を持たせることができる
3. アクセスコントロールのコードが各ページにバラけてしまい、漏れが発生する可能性が上がる  
→ 😢 getServerSidePropsは各ページに書かないといけないので依然としてある

---
layout: center
class: text-center
---

# 2. エッジサーバーサイドアクセスコントロール  

---
layout: image-right
image: /images/middleware_access_control.png
---

# エッジサーバーサイドアクセスコントロール

Next.jsのMiddlewareを用いた手法

- Next.js v12.2.0でstableになった機能
- エッジサーバーで動くことが期待されている
  - Vercelにデプロイするとエッジサーバー上で実行される
  - セルフホストの場合は、オリジンサーバー上で実行される
- SSRでもSG（SSG）であっても実行可能
  - SGの場合はキャッシュされたコンテンツが変えるよりも前段で実行される

---
---

# エッジサーバーサイドアクセスコントロール  

Next.jsのMiddlewareでアクセスコントロールしてしまう手法

```ts
export async function middleware(request: NextRequest, event: NextFetchEvent) {
  const session = request.cookies.get('session')?.value ?? '';

  if (!(await isAuthenticated(event, session))) {
    return NextResponse.redirect(new URL('/sign-in', request.url));
  }

  return NextResponse.next();
}
```

---
---

# エッジサーバーサイドアクセスコントロール

クライアントサイドアクセスコントロールで抱えていた課題の解消

1. ローディング状態を管理しないといけないこと  
→ 🤗 クライアントサイドがレンダリングされる時にはアクセスコントロール済みなのでローディングは不要
2. （認可が必要な場合）認証情報をクライアントサイドに持たないといけないためセキュリティリスクが上がる  
→ 🤗 http onlyなCookieにのみ認証情報を持たせることができる
3. アクセスコントロールのコードが各ページにバラけてしまい、漏れが発生する可能性が上がる  
→ 🤗 Middlewareは全ページで共通に実行されるため、まとめてアクセスコントロールが可能

# まとめ

Next.jsでのアクセスコントロール手法

1. クライアントサイド
2. サーバーサイド
3. エッジサーバーサイド

1にはいくつか問題があるので、僕たちのチームでは1→2→3といったように移行をしていったよ。
