# Vue.js

HTML テンプレートと JavaScript のリアクティブなデータバインディングを組み合わせたプログレッシブ・フロントエンドフレームワークです。**Composition API による直感的な状態管理**・軽量なバンドルサイズ・緩やかな学習曲線を特徴とし、React と並ぶフロントエンドの主要選択肢です。日本・アジア企業での採用が多く、HTML に近い書き方が好まれる現場で選ばれます。

---

## はじめて読む人へ

Vue.js を使うと「このデータが変われば画面のこの部分が自動的に更新される」という宣言的 UI を書けます。`v-for` や `v-if` のようなディレクティブで HTML に直接ロジックを記述でき、JavaScript を最小限に抑えながら動的な UI を構築できます。

### 読む前に押さえること

- [HTML / CSS](HTML-CSS.md) — テンプレートの基礎
- [JavaScript 基礎](JavaScript.md) — アロー関数・Promise・配列操作
- [TypeScript](TypeScript.md) — 型アノテーション（省略可だが推奨）

### 読み終えたら説明できること

- `ref` と `reactive` の違いと使い分けを説明できる
- `v-for`・`v-if`・`v-model` を使ったコンポーネントを書ける
- Composition API で Composable を設計できる

---

## プロジェクトの作成

```bash
npm create vue@latest my-app
# TypeScript・Vue Router・Pinia・ESLint を選択
cd my-app
npm install
npm run dev
```

---

## Composition API の基本

Vue 3 では `<script setup>` + Composition API が推奨スタイルです。

### ref と reactive

```vue
<script setup lang="ts">
import { ref, reactive, computed } from 'vue'

// ref: プリミティブな値をリアクティブに
const count = ref(0)
const message = ref('こんにちは')

// reactive: オブジェクト全体をリアクティブに
const user = reactive({
  name: '田中',
  age: 21,
})

// computed: 依存する値が変わると自動再計算
const doubled = computed(() => count.value * 2)

function increment() {
  count.value++  // ref は .value でアクセス
}
</script>

<template>
  <p>{{ message }}、{{ user.name }} さん</p>
  <p>カウント: {{ count }} / 2倍: {{ doubled }}</p>
  <button @click="increment">+1</button>
</template>
```

`ref` は `.value` が必要ですが、テンプレート内では自動アンラップされるため不要です。`reactive` はオブジェクト全体がそのままリアクティブになるため、ネストしたプロパティも `.value` なしでアクセスできます。

### watch

```vue
<script setup lang="ts">
import { ref, watch, watchEffect } from 'vue'

const keyword = ref('')
const results = ref<string[]>([])

// watch: 特定の値を監視して副作用を実行
watch(keyword, async (newVal, oldVal) => {
  if (newVal.length < 2) return
  results.value = await fetchSearch(newVal)
})

// watchEffect: 内部で参照したすべてのリアクティブ値を自動追跡
watchEffect(() => {
  document.title = `検索: ${keyword.value}`
})
</script>
```

---

## テンプレート構文

### 主なディレクティブ

```vue
<script setup lang="ts">
import { ref } from 'vue'
const items = ref(['Apple', 'Banana', 'Cherry'])
const isVisible = ref(true)
const inputText = ref('')
const color = ref('red')
</script>

<template>
  <!-- v-for: リスト描画（:key は必須）-->
  <ul>
    <li v-for="(item, index) in items" :key="index">
      {{ index + 1 }}. {{ item }}
    </li>
  </ul>

  <!-- v-if / v-else-if / v-else: 条件分岐 -->
  <p v-if="items.length > 5">多い</p>
  <p v-else-if="items.length > 0">少ない</p>
  <p v-else>空</p>

  <!-- v-show: display の切替（DOM は残る）-->
  <p v-show="isVisible">表示されています</p>

  <!-- v-model: 双方向バインディング -->
  <input v-model="inputText" placeholder="入力してください" />
  <p>入力値: {{ inputText }}</p>

  <!-- v-bind (:): 属性バインディング -->
  <p :style="{ color: color }">赤いテキスト</p>
  <button :disabled="items.length === 0">送信</button>

  <!-- v-on (@): イベントハンドリング -->
  <button @click="isVisible = !isVisible">切替</button>
</template>
```

### イベント修飾子

```vue
<template>
  <!-- .prevent: event.preventDefault() -->
  <form @submit.prevent="onSubmit">...</form>

  <!-- .stop: event.stopPropagation() -->
  <button @click.stop="onClick">クリック</button>

  <!-- .once: 一度だけ発火 -->
  <button @click.once="onFirstClick">最初の1回だけ</button>

  <!-- .enter: Enter キーのみ -->
  <input @keyup.enter="onEnter" />
</template>
```

---

## コンポーネント

### props と emit

```vue
<!-- components/ScoreCard.vue -->
<script setup lang="ts">
const props = defineProps<{
  title: string
  score: number
  isHighScore?: boolean
}>()

const emit = defineEmits<{
  reset: []
  update: [value: number]
}>()
</script>

<template>
  <div :class="{ highlight: props.isHighScore }">
    <h2>{{ props.title }}</h2>
    <p>スコア: {{ props.score }}</p>
    <button @click="emit('reset')">リセット</button>
    <button @click="emit('update', props.score + 10)">+10</button>
  </div>
</template>
```

```vue
<!-- App.vue（親）-->
<script setup lang="ts">
import { ref } from 'vue'
import ScoreCard from './components/ScoreCard.vue'

const score = ref(0)
</script>

<template>
  <ScoreCard
    title="得点板"
    :score="score"
    :is-high-score="score >= 100"
    @reset="score = 0"
    @update="(v) => (score = v)"
  />
</template>
```

### スロット

```vue
<!-- components/Card.vue -->
<template>
  <div class="card">
    <header><slot name="header">デフォルトタイトル</slot></header>
    <main><slot /></main>          <!-- デフォルトスロット -->
    <footer><slot name="footer" /></footer>
  </div>
</template>
```

```vue
<!-- 使う側 -->
<Card>
  <template #header>カスタムタイトル</template>
  <p>本文コンテンツ</p>
  <template #footer>フッター</template>
</Card>
```

---

## Composable（カスタム hooks）

ロジックを再利用可能な関数として切り出す Vue のパターンです（React の Custom Hook と同等）。

```typescript
// composables/useFetch.ts
import { ref } from 'vue'

export function useFetch<T>(url: string) {
  const data = ref<T | null>(null)
  const loading = ref(false)
  const error = ref<string | null>(null)

  async function execute() {
    loading.value = true
    error.value = null
    try {
      const res = await fetch(url)
      data.value = await res.json()
    } catch (e) {
      error.value = String(e)
    } finally {
      loading.value = false
    }
  }

  return { data, loading, error, execute }
}
```

```vue
<script setup lang="ts">
import { onMounted } from 'vue'
import { useFetch } from '@/composables/useFetch'

type User = { id: number; name: string }

const { data, loading, error, execute } = useFetch<User[]>('/api/users')
onMounted(() => execute())
</script>

<template>
  <p v-if="loading">読み込み中...</p>
  <p v-else-if="error">エラー: {{ error }}</p>
  <ul v-else>
    <li v-for="user in data" :key="user.id">{{ user.name }}</li>
  </ul>
</template>
```

---

## ライフサイクルフック

!!! info ""
    ```
    作成 → マウント → 更新 → アンマウント
    
    setup()            ← データの初期化
      ↓
    onBeforeMount()    ← DOM 生成前
      ↓
    onMounted()        ← DOM 生成後（API 呼び出し・イベントリスナー登録）
      ↓
    onUpdated()        ← 再描画後
      ↓
    onUnmounted()      ← 破棄直前（クリーンアップ）
    ```

```vue
<script setup lang="ts">
import { onMounted, onUnmounted } from 'vue'

function handleResize() { /* ... */ }

onMounted(() => {
  window.addEventListener('resize', handleResize)
})

onUnmounted(() => {
  window.removeEventListener('resize', handleResize)
})
</script>
```

---

## Vue Router

```typescript
// router/index.ts
import { createRouter, createWebHistory } from 'vue-router'
import HomeView from '@/views/HomeView.vue'
import UserView from '@/views/UserView.vue'

const router = createRouter({
  history: createWebHistory(),
  routes: [
    { path: '/', component: HomeView },
    { path: '/users/:id', component: UserView, props: true },
    { path: '/:pathMatch(.*)*', redirect: '/' },
  ],
})

export default router
```

```vue
<script setup lang="ts">
import { useRouter, useRoute } from 'vue-router'

const router = useRouter()
const route = useRoute()

const userId = route.params.id  // URL パラメータ

function goHome() {
  router.push('/')
}
</script>

<template>
  <RouterLink to="/">ホームへ</RouterLink>
  <RouterLink :to="`/users/${userId}`">ユーザー詳細</RouterLink>
  <RouterView />
</template>
```

---

## Pinia（状態管理）

コンポーネントをまたぐ共有状態を管理します。Vuex の後継として Vue 公式が推奨しています。

```typescript
// stores/useAuthStore.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useAuthStore = defineStore('auth', () => {
  const user = ref<{ name: string; email: string } | null>(null)

  const isLoggedIn = computed(() => user.value !== null)

  function login(name: string, email: string) {
    user.value = { name, email }
  }

  function logout() {
    user.value = null
  }

  return { user, isLoggedIn, login, logout }
})
```

```vue
<script setup lang="ts">
import { useAuthStore } from '@/stores/useAuthStore'

const auth = useAuthStore()
</script>

<template>
  <p v-if="auth.isLoggedIn">{{ auth.user?.name }} さんログイン中</p>
  <button v-if="auth.isLoggedIn" @click="auth.logout()">ログアウト</button>
  <button v-else @click="auth.login('田中', 'tanaka@example.com')">ログイン</button>
</template>
```

---

## Vue vs React の比較

| 観点 | Vue.js | React |
|------|--------|-------|
| **テンプレート** | HTML ベース（ディレクティブ）| JSX（JS 中心）|
| **双方向バインディング** | `v-model` でシンプル | 手動（`onChange` + `setState`）|
| **状態管理** | `ref` / `reactive`（組み込み）| `useState`（組み込み）|
| **グローバル状態** | Pinia（公式）| Zustand・Jotai など（選択が必要）|
| **ルーティング** | Vue Router（公式）| React Router（サードパーティ）|
| **学習コスト** | 低め（HTML に近い）| 高め（JSX・関数的思考）|
| **採用規模** | 日本・アジア企業で多い | グローバルで最多 |
| **向いているケース** | HTML + JS からの移行・中小規模 | 大規模・Next.js との組み合わせ |

---

## 確認問題

1. `ref` と `reactive` の違いを説明し、それぞれ使うべき場面を示してください。
2. `v-show` と `v-if` の違いを説明し、どちらを使うべきかの判断基準を示してください。
3. Composable（`useFetch` など）を作る利点を React の Custom Hook と比較して説明してください。

---

## 関連ページ

- [JavaScript 基礎](JavaScript.md) — Vue テンプレートの前提知識
- [TypeScript](TypeScript.md) — `defineProps` の型定義
- [React](React.md) — Vue と比較されるフロントエンドフレームワーク
- [Next.js](Next.js.md) — React ベースの SSR フレームワーク（Vue 版は Nuxt.js）
- [Astro](Astro.md) — Vue コンポーネントを Islands として利用可能

---

[← ホームへ](Home)
