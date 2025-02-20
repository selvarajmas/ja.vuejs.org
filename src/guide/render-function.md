# Render 関数

Vue では、大多数のケースにおいてテンプレートを使ってアプリケーションを構築することを推奨していますが、完全な JavaScript プログラミングの力が必要になる状況もあります。そこでは私たちは **render 関数** を使うことができます。

さあ、 `render()` 関数が実用的になる例に取りかかりましょう。例えば、アンカーつきの見出しを生成したいとします:

```html
<h1>
  <a name="hello-world" href="#hello-world">
    Hello world!
  </a>
</h1>
```

アンカーつきの見出しはとても頻繁に使われますので、コンポーネントにするべきです:

```vue-html
<anchored-heading :level="1">Hello world!</anchored-heading>
```

コンポーネントは、`level` の値に応じた見出しを生成する必要があります。手っ取り早くこれで実現しましょう:

```js
const { createApp } = Vue

const app = createApp({})

app.component('anchored-heading', {
  template: `
    <h1 v-if="level === 1">
      <slot></slot>
    </h1>
    <h2 v-else-if="level === 2">
      <slot></slot>
    </h2>
    <h3 v-else-if="level === 3">
      <slot></slot>
    </h3>
    <h4 v-else-if="level === 4">
      <slot></slot>
    </h4>
    <h5 v-else-if="level === 5">
      <slot></slot>
    </h5>
    <h6 v-else-if="level === 6">
      <slot></slot>
    </h6>
  `,
  props: {
    level: {
      type: Number,
      required: true
    }
  }
})
```

このテンプレートは良いものには思えません。冗長なだけでなく、 `<slot></slot>` がすべての見出しのレベルにコピーされています。そして、アンカー要素を追加する時にはすべての `v-if/v-else-if` の分岐にまたコピーしなければなりません。

ほとんどのコンポーネントでテンプレートがうまく働くとはいえ、明らかにこれはそうではないものの 1 つです。そこで、 `render()` 関数を使ってこれを書き直してみましょう。

```js
const { createApp, h } = Vue

const app = createApp({})

app.component('anchored-heading', {
  render() {
    return h(
      'h' + this.level, // タグ名
      {}, // props/属性
      this.$slots.default() // 子供の配列
    )
  },
  props: {
    level: {
      type: Number,
      required: true
    }
  }
})
```

`render()` 関数の実装はとても単純ですが、コンポーネントインスタンスのプロパティについてよく理解している必要があります。この場合では、 `anchored-heading` の内側の `Hello world!` のように `v-slot` ディレクティブなしで子供を渡した時には、その子供は `$slots.default()` のコンポーネントインスタンスに保持されるということを知っている必要があります。もしまだ知らないのであれば、 **render 関数に取り掛かる前に [インスタンスプロパティ API](../api/instance-properties.html) を通読することをおすすめします。**

## DOM ツリー

render 関数に取り掛かる前に、ブラウザがどのように動くのかについて少し知っておくことが重要です。この HTML を例にしましょう:

```html
<div>
  <h1>My title</h1>
  Some text content
  <!-- TODO: Add tagline -->
</div>
```

ブラウザはこのコードを読み込むと、血縁関係を追跡するために家系図を構築するのと同じように、全てを追跡する [「DOM ノード」のツリー](https://javascript.info/dom-nodes)を構築します。

上の HTML の DOM ノードツリーはこんな感じになります。

![DOM ツリーの可視化](/images/dom-tree.png)

すべての要素はノードです。テキストのすべてのピースはノードです。コメントですらノードです！それぞれのノードは子供を持つことができます。 (つまり、それぞれのノードは他のノードを含むことができます)

これらすべてのノードを効率的に更新することは難しくなり得ますが、ありがたいことに、それを手動で行う必要はありません。代わりに、Vue にどのような HTML を表示させたいのかをテンプレートで伝えます:

```html
<h1>{{ blogTitle }}</h1>
```

または render 関数:

```js
render() {
  return h('h1', {}, this.blogTitle)
}
```

そしてどちらの場合でも、 `blogTitle` が変更されたとしても Vue が自動的にページを最新の状態に保ちます。

## 仮想 DOM ツリー

Vue は、実際の DOM に反映する必要のある変更を追跡するために **仮想 DOM** を構築して、ページを最新の状態に保ちます。この行をよく見てみましょう:

```js
return h('h1', {}, this.blogTitle)
```

`h()` 関数が返すものはなんでしょうか？これは、 _正確には_ 実際の DOM 要素ではありません。それが返すのは、ページ上にどんな種類のノードをレンダリングするのかを Vue に伝えるための情報をもったプレーンなオブジェクトです。この情報には子供のノードの記述も含まれます。私たちは、このノードの記述を *仮想ノード* と呼び、通常 **VNode** と省略します。「仮想 DOM」というのは、Vue コンポーネントのツリーから構成される VNode のツリー全体のことなのです。

## `h()` の引数

`h()` 関数は VNode を作るためのユーティリティです。もっと正確に `createVNode()` と名づけられることもあるかもしれませんが、頻繁に使用されるので、簡潔さのために `h()` と呼ばれます。

```js
// @returns {VNode}
h(
  // {String | Object | Function} tag
  // HTMLタグ名、コンポーネント、非同期コンポーネント、
  // または関数型コンポーネント。
  //
  // 必須
  'div',

  // {Object} props
  // テンプレート内で使うであろう
  // 属性、プロパティ、イベントに対応するオブジェクト
  //
  // 省略可能
  {},

  // {String | Array | Object} children
  // `h()` で作られた子供のVNode、
  // または文字列(テキストVNodeになる)、
  // またはスロットをもつオブジェクト
  //
  // 省略可能
  [
    'Some text comes first.',
    h('h1', 'A headline'),
    h(MyComponent, {
      someProp: 'foobar'
    })
  ]
)
```

props がない場合は、通常 children を第 2 引数として渡すことができます。それがあいまいな場合は、 `null` を第 2 引数として渡して、 children を第 3 引数にしておけます。

## 完全な例

この知識によって、今度は書き始めたコンポーネントを完成させることができます:

```js
const { createApp, h } = Vue

const app = createApp({})

/** 子供のノードから再帰的にテキストを取得する */
function getChildrenTextContent(children) {
  return children
    .map(node => {
      return typeof node.children === 'string'
        ? node.children
        : Array.isArray(node.children)
        ? getChildrenTextContent(node.children)
        : ''
    })
    .join('')
}

app.component('anchored-heading', {
  render() {
    // 子供のテキストからケバブケース(kebab-case)のIDを作成する
    const headingId = getChildrenTextContent(this.$slots.default())
      .toLowerCase()
      .replace(/\W+/g, '-') // 英数字とアンダースコア以外の文字を-に置換する
      .replace(/(^-|-$)/g, '') // 頭と末尾の-を取り除く

    return h('h' + this.level, [
      h(
        'a',
        {
          name: headingId,
          href: '#' + headingId
        },
        this.$slots.default()
      )
    ])
  },
  props: {
    level: {
      type: Number,
      required: true
    }
  }
})
```

## 制約

### VNode は一意でなければならない

コンポーネント内のすべての VNode は一意でなければなりません。つまり、下のような render 関数は無効だということです:

```js
render() {
  const myParagraphVNode = h('p', 'hi')
  return h('div', [
    // おっと - VNode が重複しています!
    myParagraphVNode, myParagraphVNode
  ])
}
```

もしあなたが本当に同じ要素、コンポーネントを何回もコピーしたいなら、ファクトリー関数を使えばできます。例えば、次の render 関数は 20 個の同じ段落をレンダリングする完全に正しい方法です。

```js
render() {
  return h('div',
    Array.from({ length: 20 }).map(() => {
      return h('p', 'hi')
    })
  )
}
```

## コンポーネントの VNodes を作る

コンポーネントの VNode を作るためには、 `h` の第 1 引数にコンポーネントそのものを渡します:

```js
render() {
  return h(ButtonCounter)
}
```

コンポーネントを名前解決する必要がある場合は、 `resolveComponent` で呼び出せます:

```js
const { h, resolveComponent } = Vue

// ...

render() {
  const ButtonCounter = resolveComponent('ButtonCounter')
  return h(ButtonCounter)
}
```

`resolveComponent` は、テンプレートがコンポーネントを名前解決するために内部的に使っているものと同じ関数です。

`render` 関数は通常、 [グローバルに登録された](/guide/component-registration.html#グローバル登録) コンポーネントにだけ `resolveComponent` を使う必要があります。 [ローカルのコンポーネント登録](/guide/component-registration.html#ローカル登録) は通常、完全に省略できます。次のような例を考えてみましょう:

```js
// これを単純化してみると
components: {
  ButtonCounter
},
render() {
  return resolveComponent('ButtonCounter')
}
```

コンポーネントの名前を登録して、それを調べるというよりも、直接使うことができます:

```js
render() {
  return h(ButtonCounter)
}
```

## テンプレートの機能をプレーンな JavaScript で置き換える

### `v-if` と `v-for`

何であれ、プレーンな JavaScript で簡単に実現できることについては、Vue の render 関数は固有の代替手段を提供していません。例えば、テンプレートでの `v-if` や `v-for` の使用:

```html
<ul v-if="items.length">
  <li v-for="item in items">{{ item.name }}</li>
</ul>
<p v-else>No items found.</p>
```

これは、render 関数では JavaScript の `if`/`else` と `map()` で書き換えることができます。

```js
props: ['items'],
render() {
  if (this.items.length) {
    return h('ul', this.items.map((item) => {
      return h('li', item.name)
    }))
  } else {
    return h('p', 'No items found.')
  }
}
```

テンプレートでは、 `<template>` タグを使って `v-if` や `v-for` ディレクティブを当てておくと便利です。 `render` 関数に移行するときには、 `<template>` タグは不要となり、破棄することができます。

### `v-model`

`v-model` ディレクティブは、テンプレートのコンパイル中に `modelValue` と `onUpdate:modelValue` プロパティに展開されます - 私たちはそれらのプロパティを自分自身で提供する必要があります:

```js
props: ['modelValue'],
emits: ['update:modelValue'],
render() {
  return h(SomeComponent, {
    modelValue: this.modelValue,
    'onUpdate:modelValue': value => this.$emit('update:modelValue', value)
  })
}
```

### `v-on`

私たちは適切なプロパティ名をイベントハンドラに与える必要があります。例えば、 `click` イベントをハンドルする場合は、プロパティ名は `onClick` になります。

```js
render() {
  return h('div', {
    onClick: $event => console.log('clicked', $event.target)
  })
}
```

#### イベント修飾子

`.passive`、`.capture`、 `.once` イベント修飾子は、キャメルケースを使ってイベント名の後につなげます。

例えば:

```js
render() {
  return h('input', {
    onClickCapture: this.doThisInCapturingMode,
    onKeyupOnce: this.doThisOnce,
    onMouseoverOnceCapture: this.doThisOnceInCapturingMode
  })
}
```

その他すべてのイベントおよびキー修飾子については、特別な API は必要ありません。
なぜなら、ハンドラーの中でイベントのメソッドを使用することができるからです:

| 修飾子                                                 | ハンドラーでの同等の記述                                                                                              |
| ---------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| `.stop`                                              | `event.stopPropagation()`                                                                                            |
| `.prevent`                                           | `event.preventDefault()`                                                                                             |
| `.self`                                              | `if (event.target !== event.currentTarget) return`                                                                   |
| キー:<br>例 `.enter`                               | `if (event.key !== 'Enter') return`<br><br>`'Enter'` キーを適切な [キー](http://keycode.info/) に変更 |
| 修飾キー:<br>`.ctrl`, `.alt`, `.shift`, `.meta` | `if (!event.ctrlKey) return`<br><br>`altKey`、`shiftKey`、`metaKey` も同様                       |

これらすべての修飾子を一緒に使った例がこちらです:

```js
render() {
  return h('input', {
    onKeyUp: event => {
      // イベントを発行した要素がイベントが紐づけられた要素ではない場合は
      // 中断する
      if (event.target !== event.currentTarget) return
      // 押されたキーが Enter ではなく、
      // また Shift キーが同時に押されていなかった場合は中断する
      if (!event.shiftKey || event.key !== 'Enter') return
      // イベントの伝播(propagation)を止める
      event.stopPropagation()
      // この要素のデフォルトの keyup ハンドラが実行されないようにする
      event.preventDefault()
      // ...
    }
  })
}
```

### スロット

[`this.$slots`](../api/instance-properties.html#slots) から取得した VNode の配列でスロットの中身にアクセスすることができます:

```js
render() {
  // `<div><slot></slot></div>`
  return h('div', this.$slots.default())
}
```

```js
props: ['message'],
render() {
  // `<div><slot :text="message"></slot></div>`
  return h('div', this.$slots.default({
    text: this.message
  }))
}
```

コンポーネントの VNodes の場合、引数 children を配列ではなくオブジェクトとして `h` に渡す必要があります。各プロパティは、同名のスロットに移植するために使われます:

```js
render() {
  // `<div><child v-slot="props"><span>{{ props.text }}</span></child></div>`
  return h('div', [
    h(
      resolveComponent('child'),
      null,
      // { name: props => VNode | Array<VNode> } の形で
      // 子供のオブジェクトを `slots` として渡す
      {
        default: (props) => h('span', props.text)
      }
    )
  ])
}
```

スロットは関数として渡され、子コンポーネントが各スロットのコンテンツの作成を制御できるようになっています。リアクティブなデータは、親コンポーネントではなく子コンポーネントの依存関係として登録されるように、スロット関数内でアクセスする必要があります。逆に `resolveComponent` の呼び出しは、スロット関数の外で行うべきで、そうしないと間違ったコンポーネントへの相対的な解決になってしまいます:

```js
// `<MyButton><MyIcon :name="icon" />{{ text }}</MyButton>`
render() {
  // resolveComponent の呼び出しはスロット関数の外側でなければなりません
  const Button = resolveComponent('MyButton')
  const Icon = resolveComponent('MyIcon')

  return h(
    Button,
    null,
    {
      // アロー関数を使って `this` の値を保持します
      default: (props) => {
        // リアクティブなプロパティは、子のレンダリングの依存関係になるように
        // スロット関数の内側で読み込む必要があります
        return [
          h(Icon, { name: this.icon }),
          this.text
        ]
      }
    }
  )
}
```

コンポーネントが親からスロットを受け取った場合、そのスロットを子のコンポーネントに直接渡せます:

```js
render() {
  return h(Panel, null, this.$slots)
}
```

また、必要に応じて個別に渡したり、ラップすることもできます:

```js
render() {
  return h(
    Panel,
    null,
    {
      // スロット関数を渡したい場合は次のようになります
      header: this.$slots.header,

      // スロットを何らかの方法で操作する必要がある場合は、
      // 新しい関数でそれをラップする必要があります
      default: (props) => {
        const children = this.$slots.default ? this.$slots.default(props) : []

        return children.concat(h('div', 'Extra child'))
      }
    }
  )
}
```

### `<component>` と `is`

裏では、テンプレートは `resolveDynamicComponent` をつかって `is` 属性を実装しています。 `render` 関数で `is` 属性がもつ、すべての柔軟性が必要な場合は、同じ関数を使うことができます:

```js
const { h, resolveDynamicComponent } = Vue

// ...

// `<component :is="name"></component>`
render() {
  const Component = resolveDynamicComponent(this.name)
  return h(Component)
}
```

`is` と同じように `resolveDynamicComponent` は、コンポーネント名、HTML 要素名、コンポーネントのオプションオブジェクトをサポートします。

しかし、通常このレベルの柔軟性はいりません。 `resolveDynamicComponent` をより直接的な代替手段で置き換えることができる場合が多いです。

例えば、コンポーネント名をサポートするだけならば、代わりに `resolveComponent` が使えます。

VNode が常に HTML 要素ならば、 `h` にその要素名を直接渡せます:

```js
// `<component :is="bold ? 'strong' : 'em'"></component>`
render() {
  return h(this.bold ? 'strong' : 'em')
}
```

同じように、 `is` に渡された値がコンポーネントのオプションオブジェクトならば、なにも解決する必要はなく、 `h` の第 1 引数として直接渡せます。

`<template>` タグと同様に、 `<component>` タグはテンプレートの中で構文上のプレースホルダとしてのみ必要で、 `render` 関数に移行するときには破棄してください。

### カスタムディレクティブ

カスタムディレクティブは、 [`withDirectives`](/api/global-api.html#withdirectives) を使って VNode に適用できます:

```js
const { h, resolveDirective, withDirectives } = Vue

// ...

// <div v-pin:top.animate="200"></div>
render () {
  const pin = resolveDirective('pin')

  return withDirectives(h('div'), [
    [pin, 200, 'top', { animate: true }]
  ])
}
```

[`resolveDirective`](/api/global-api.html#resolvedirective) は、テンプレートが内部でディレクティブの名前解決をするのと同じ関数です。これは、まだディレクティブの定義オブジェクトに直接アクセスしていない場合にのみ必要です。

### 組み込みコンポーネント

`<keep-alive>`、`<transition>`、`<transition-group>`、 `<teleport>` などの [組み込みコンポーネント](/api/built-in-components.html) はデフォルトではグローバルに登録されません。これによりバンドラーが Tree Shaking（ツリーシェイキング）を行い、コンポーネントが使われている場合にだけビルドに含まれるようになります。しかし、これは `resolveComponent` や `resolveDynamicComponent` を使ってアクセスできないということです。

テンプレートはこれらのコンポーネントを特別扱いしていて、使われるときには自動的にインポートします。自分で `render` 関数を書く場合は、自身でそれらをインポートする必要があります:

```js
const { h, KeepAlive, Teleport, Transition, TransitionGroup } = Vue

// ...

render () {
  return h(Transition, { mode: 'out-in' }, /* ... */)
}
```

## Render 関数の返り値

これまで見てきた例は、 `render` 関数が 1 つのルート VNode を返してきました。しかし、これには代替手段があります。

文字列を返すと、ラップする要素のないテキストの VNode が作成されます:

```js
render() {
  return 'Hello world!'
}
```

また、ルートノードでラップしない子の配列を返せます。これは Fragment を作成します:

```js
// テンプレートの `Hello<br>world!` に相当します
render() {
  return [
    'Hello',
    h('br'),
    'world!'
  ]
}
```

データの読み込み中など、コンポーネントがなにもレンダリングしない必要がある場合は、単に `null` を返すことができます。これは DOM のコメントノードとしてレンダリングされます。

## JSX

たくさんの `render` 関数を書いていると、こういう感じのものを書くのがつらく感じるかもしれません:

```js
h(
  resolveComponent('anchored-heading'),
  {
    level: 1
  },
  {
    default: () => [h('span', 'Hello'), ' world!']
  }
)
```

特に、テンプレート版がそれにくらべて簡潔な場合は:

```vue-html
<anchored-heading :level="1"> <span>Hello</span> world! </anchored-heading>
```

これが、Vue で JSX を使い、テンプレートに近い構文に戻す [Babel プラグイン](https://github.com/vuejs/jsx-next) が存在する理由です。

```jsx
import AnchoredHeading from './AnchoredHeading.vue'

const app = createApp({
  render() {
    return (
      <AnchoredHeading level={1}>
        <span>Hello</span> world!
      </AnchoredHeading>
    )
  }
})

app.mount('#demo')
```

JSX がどのように JavaScript に変換されるのか、より詳細な情報は、 [使用方法](https://github.com/vuejs/jsx-next#installation) を見てください。

## 関数型コンポーネント

関数型コンポーネントとは、それ自体にはなんの状態も持たないコンポーネントの別方式です。これはコンポーネントのインスタンスを作成しないでレンダリングされるため、通常のコンポーネントのライフサイクルを無視します。

関数型コンポーネントを作成するには、オプションオブジェクトではなく、単純な関数を使います。この関数は、事実上、コンポーネントの `render` 関数です。関数型コンポーネントには `this` の参照がないため、 Vue は `props` を最初の引数として渡します:

```js
const FunctionalComponent = (props, context) => {
  // ...
}
```

第 2 引数の `context` には 3 つのプロパティが含まれます: `attrs`、`emit`、`slots` です。これらはインスタンスプロパティの [`$attrs`](/api/instance-properties.html#attrs)、[`$emit`](/api/instance-methods.html#emit)、[`$slots`](/api/instance-properties.html#slots) に相当します。

コンポーネントで使う通常の設定オプションのほとんどは、関数型コンポーネントでは使えません。しかし、 [`props`](/api/options-data.html#props) や [`emits`](/api/options-data.html#emits) はプロパティとして追加定義することができます。:

```js
FunctionalComponent.props = ['value']
FunctionalComponent.emits = ['click']
```

`props` オプションが指定されていない場合、この関数に渡される `props` オブジェクトは `attrs` と同じく、すべての属性が含まれます。 `props` オプションが指定されていない場合、プロパティ名はキャメルケースに正規化されません。

関数型コンポーネントは、通常のコンポーネントと同様に登録したり、実行したりすることができます。関数を `h` の第 1 引数として渡すと、その関数は関数型コンポーネントとして扱われます。

## テンプレートのコンパイル

あなたは Vue のテンプレートが実際に Render 関数にコンパイルされることに興味があるかもしれません。これは通常知っておく必要のない実装の詳細ですが、もし特定のテンプレートの機能がどのようにコンパイルされるか知りたいのなら、これが面白いかもしれません。これは、 `Vue.compile` を使用してテンプレートの文字列をライブコンパイルする小さなデモです:

<iframe src="https://vue-next-template-explorer.netlify.app/" width="100%" height="420"></iframe>
