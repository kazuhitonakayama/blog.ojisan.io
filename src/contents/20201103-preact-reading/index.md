---
path: /preact-reading
created: "2020-11-03"
title: preact コードリーディング
visual: "./visual.png"
tags: [preact]
userId: sadnessOjisan
isFavorite: false
isProtect: false
---

preact 完全に理解した記念ブログです。jsx/compat は含んでいません。

去年のクリスマスイブの飲み会に友人に preact のコードリーディングを勧められた（クリスマスに何しとるねん）のがきっかけで読んでいました。どうも、コードベースが小さく、型の情報が合ったり、コメントが充実していたりして、仮想 DOM 系のライブラリがどう実装されているかを知るにはちょうど良いとのことでした。実際 React 本体は読もうとしてもなかなか進まなかったりしたという経験があったりもして、まずは preact から挑戦してみることにしました。

## preact とは

React の軽量版です。
読むと分かったのですが、本家といろいろレンダリングプロセスが違ったりしました。

で、(p)react は、

- 状態を持て、書き換えも可能である
- 状態を書き換えるとそれに対応して HTML が書き換わる

という特徴があると思います。
それがどのようにして実現されているか見ていきましょう。

## 前提となる知識

### 仮想 DOM とは何か

### jsx と h 関数

## preact の全体感

### ビルド

microbundle というツールで行われています。
これは rollup のラッパーで作者が preact のビルド設定をデフォルトに設定したものです。
zero config でできます。

### 言語

JavaScript で実装されています。
TypeScript ではありません。
ただし JSDoc に型情報があり、型を出力しています。
ちなみに TypeScript で実装しようとするとエラーがたくさん出るので現実的では無さそうです。

### フォルダ構成

src 配下を解説します。

### データ構造

要素は VNode という形式で回されます。
これは preact 内部での要素表現です。
これを引数にとったり出力したりなどします。

VNode の定義はこうです。

```ts
// preact というname space で定義されている VNode
interface VNode<P = {}> {
  type: ComponentType<P> | string
  props: P & { children: ComponentChildren }
  key: Key
  ref?: Ref<any> | null
  startTime?: number
  endTime?: number
}

export interface VNode<P = {}> extends preact.VNode<P> {
  type: string | ComponentFactory<P>
  props: P & { children: preact.ComponentChildren }
  _children: Array<VNode<any>> | null
  _parent: VNode | null
  _depth: number | null
  _dom: PreactElement | null
  _nextDom: PreactElement | null
  _component: Component | null
  _hydrating: boolean | null
  constructor: undefined
  _original?: VNode | null
}
```

### 呼出し関係

## コードリーディング

render から読み進めていきましょう。
目標は、

```jsx
import { h, render, Component } from "preact"

class App extends Component {
  state = {
    age: 19,
  }

  componentDidMount() {
    this.setState({ age: 12 })
  }

  render() {
    return h("h1", null, `${this.state.age}才`)
  }
}

render(h(App, null, null), document.body)
```

がどうして動作するかを理解することです。

### render

render の実装はこうなっています。

```js
import { EMPTY_OBJ, EMPTY_ARR } from "./constants"
import { commitRoot, diff } from "./diff/index"
import { createElement, Fragment } from "./create-element"
import options from "./options"

const IS_HYDRATE = EMPTY_OBJ

export function render(vnode, parentDom, replaceNode) {
  if (options._root) options._root(vnode, parentDom)
  let isHydrating = replaceNode === IS_HYDRATE
  let oldVNode = isHydrating
    ? null
    : (replaceNode && replaceNode._children) || parentDom._children
  vnode = createElement(Fragment, null, [vnode])
  let commitQueue = []
  diff(
    parentDom,
    ((isHydrating ? parentDom : replaceNode || parentDom)._children = vnode),
    oldVNode || EMPTY_OBJ,
    EMPTY_OBJ,
    parentDom.ownerSVGElement !== undefined,
    replaceNode && !isHydrating
      ? [replaceNode]
      : oldVNode
      ? null
      : parentDom.childNodes.length
      ? EMPTY_ARR.slice.call(parentDom.childNodes)
      : null,
    commitQueue,
    replaceNode || EMPTY_OBJ,
    isHydrating
  )

  commitRoot(commitQueue, vnode)
}

export function hydrate(vnode, parentDom) {
  render(vnode, parentDom, IS_HYDRATE)
}
```

簡略下のため hydrate 周りは省いて説明します。

まず、render は

```js
render(<App />, document.getElement("body"))
```

などのようにして呼ばれます。

render ではこの `<App />` が `createElement` を通して VNode という形式に変換されます。

そして `diff` で、parentDOM に対して VNode から DOM のツリーを作ります。
この diff 自体は ツリーを作るための関数ではないのですが、差分更新を行った結果ツリーができあがるので初回の render 呼び出しで呼ばれます。

そして最後に `commitRoot` にて、HTML ができたあとに各コンポーネントが持っていた componentDidMount などの関数を実行します。
それらの処理は diff を取る時に commitQueue に詰め込まれているので、それを commitRoot に渡します。

### diff

diff 関数は次のようになっています。
この関数が根幹の起点になるためかなり長いです。

全体像は[こちら]()ですが、長すぎて追いにくいので大事なところ以外削って、分岐の条件などをみやすくします。
これから読んでいくコードはこのような関数です。

```js
export function diff(
  parentDom,
  newVNode,
  oldVNode,
  globalContext,
  isSvg,
  excessDomChildren,
  commitQueue,
  oldDom,
  isHydrating
) {
  newType = newVNode.type

  try {
    // labelとうい機能.
    outer: if (typeof newType == "function") {
      // 渡されたVNodeのtypeがコンポーネントの場合
      let c, isNew, oldProps, oldState, snapshot, clearProcessingException
      let newProps = newVNode.props

      if (oldVNode._component) {
        c = newVNode._component = oldVNode._component
        clearProcessingException = c._processingException = c._pendingError
      } else {
        // 渡されたVNodeのtypeがfunctionであればComponentFactoryなので分岐
        // ClassComponent じゃなくて FC の可能性もあるのでその分岐
        if ("prototype" in newType && newType.prototype.render) {
          newVNode._component = c = new newType(newProps, componentContext)
        } else {
          newVNode._component = c = new Component(newProps, componentContext)
          c.constructor = newType
          c.render = doRender
        }

        // 作ったコンポーネントに値を詰め込む
        c.props = newProps
        if (!c.state) c.state = {}
        isNew = c._dirty = true
        c._renderCallbacks = []
      } // この 処理により必ず _nextState はなんらかの値を持つ。c.stateの初期値は {}

      if (c._nextState == null) {
        c._nextState = c.state
      }

      oldProps = c.props
      oldState = c.state

      if (isNew) {
        // 新しく渡ってきたコンポーネントの場合(VNodeがfunctionでないとき)
        if (
          newType.getDerivedStateFromProps == null &&
          c.componentWillMount != null
        ) {
          c.componentWillMount()
        }

        if (c.componentDidMount != null) {
          c._renderCallbacks.push(c.componentDidMount)
        }
      } else {
        // コンポーネントを新しく作らなかった場合(VNodeがfunctionのとき)
        if (
          newType.getDerivedStateFromProps == null &&
          newProps !== oldProps &&
          c.componentWillReceiveProps != null
        ) {
          c.componentWillReceiveProps(newProps, componentContext)
        }

        // 再レンダリング抑制
        if (
          (!c._force &&
            c.shouldComponentUpdate != null &&
            c.shouldComponentUpdate(
              newProps,
              c._nextState,
              componentContext
            ) === false) ||
          newVNode._original === oldVNode._original
        ) {
          if (c._renderCallbacks.length) {
            commitQueue.push(c)
          }

          reorderChildren(newVNode, oldDom, parentDom)
          break outer
        }

        if (c.componentWillUpdate != null) {
          c.componentWillUpdate(newProps, c._nextState, componentContext)
        }

        if (c.componentDidUpdate != null) {
          c._renderCallbacks.push(() => {
            c.componentDidUpdate(oldProps, oldState, snapshot)
          })
        }
      }

      let renderResult = isTopLevelFragment ? tmp.props.children : tmp

      // 子コンポーネントの差分を取る
      diffChildren(
        parentDom,
        Array.isArray(renderResult) ? renderResult : [renderResult],
        newVNode,
        oldVNode,
        globalContext,
        isSvg,
        excessDomChildren,
        commitQueue,
        oldDom,
        isHydrating
      )
    } else if (
      excessDomChildren == null &&
      newVNode._original === oldVNode._original
    ) {
      // typeがfunctionでない && 過剰なchildren(excessDomChildren) がない場合
      newVNode._children = oldVNode._children
      newVNode._dom = oldVNode._dom
    } else {
      // typeがfunctionでない && 過剰なchildren(excessDomChildren) がある場合
      newVNode._dom = diffElementNodes(
        oldVNode._dom,
        newVNode,
        oldVNode,
        globalContext,
        isSvg,
        excessDomChildren,
        commitQueue,
        isHydrating
      )
    }
  } catch (e) {
    // 元に戻す
    newVNode._original = null
    if (isHydrating || excessDomChildren != null) {
      newVNode._dom = oldDom
      newVNode._hydrating = !!isHydrating
      excessDomChildren[excessDomChildren.indexOf(oldDom)] = null
    }
    options._catchError(e, newVNode, oldVNode)
  }

  return newVNode._dom
}
```

では、これらを 1 つ 1 つ見ていきましょう。

#### newType

まず最初に `newType = newVNode.type;` が定義されます。
これは この `newType` は `string | ComponentFactory<P>;` を取りうります。
ComponentFactory は JavaScript の世界では function ですが、これは ClassComponent, FunctionComponent であることを示します。

実は diff は render 以外からも呼ばれ、Component は入れ子になるので、評価タイミングによってはコンポーネントが渡ってきます。
そのときに分岐をするためにこの識別子が必要となります。

#### label

分岐の説明に入る前に

```js
outer: if (typeof newType == 'function') {
```

についてみましょう。
JS でこんな JSON みたいなことを生でかけましたっけ？
これは label という機能で、break したときにここに戻すみたいなことができます。
goto みたいものなのであまり使われてはいません。

#### diff を取る対象がコンポーネントの場合

比較対象にコンポーネントを作ります。
もしすでにあるのならばそれを使いまわし、なければ新しく作ります。

```js
if (oldVNode._component) {
  c = newVNode._component = oldVNode._component
  clearProcessingException = c._processingException = c._pendingError
} else {
  // 渡されたVNodeのtypeがfunctionであればComponentFactoryなので分岐
  // ClassComponent じゃなくて FC の可能性もあるのでその分岐
  if ("prototype" in newType && newType.prototype.render) {
    newVNode._component = c = new newType(newProps, componentContext)
  } else {
    newVNode._component = c = new Component(newProps, componentContext)
    c.constructor = newType
    c.render = doRender
  }

  // 作ったコンポーネントに値を詰め込む
  c.props = newProps
  if (!c.state) c.state = {}
  c.context = componentContext
  c._globalContext = globalContext
  isNew = c._dirty = true
  c._renderCallbacks = []
} // この 処理により必ず _nextState はなんらかの値を持つ。c.stateの初期値は {}

if (c._nextState == null) {
  c._nextState = c.state
}
```

新しく作る場合、その新しい VNode の type がコンポーネントかどうかに着目します。
もしそれがコンポーネントならばその constructor を呼び出して使いまわし、そうでなければ Component のインスタンスを作ります。

```js
if ("prototype" in newType && newType.prototype.render) {
  newVNode._component = c = new newType(newProps, componentContext)
} else {
  newVNode._component = c = new Component(newProps, componentContext)
  c.constructor = newType
  c.render = doRender
}
```

もし 新しく Component インスタンスを作った場合は必要な値を初期化します。

```js
// 作ったコンポーネントに値を詰め込む
c.props = newProps
if (!c.state) c.state = {}
isNew = c._dirty = true
c._renderCallbacks = []
```

そして、コンポーネントを使いまわした場合、新しく作った場合の共通の初期化を行います。

```js
// この 処理により必ず _nextState はなんらかの値を持つ。c.stateの初期値は {}
if (c._nextState == null) {
  c._nextState = c.state
}

oldProps = c.props
oldState = c.state
```

次にライフサイクルの実行を行います。

```js
if (isNew) {
  // 新しく渡ってきたコンポーネントの場合(VNodeがfunctionでないとき)
  if (
    newType.getDerivedStateFromProps == null &&
    c.componentWillMount != null
  ) {
    c.componentWillMount()
  }

  if (c.componentDidMount != null) {
    c._renderCallbacks.push(c.componentDidMount)
  }
} else {
  // コンポーネントを新しく作らなかった場合(VNodeがfunctionのとき)
  if (
    newType.getDerivedStateFromProps == null &&
    newProps !== oldProps &&
    c.componentWillReceiveProps != null
  ) {
    c.componentWillReceiveProps(newProps, componentContext)
  }

  // 再レンダリング抑制
  if (
    (!c._force &&
      c.shouldComponentUpdate != null &&
      c.shouldComponentUpdate(newProps, c._nextState, componentContext) ===
        false) ||
    newVNode._original === oldVNode._original
  ) {
    if (c._renderCallbacks.length) {
      commitQueue.push(c)
    }

    reorderChildren(newVNode, oldDom, parentDom)
    break outer
  }

  if (c.componentWillUpdate != null) {
    c.componentWillUpdate(newProps, c._nextState, componentContext)
  }

  if (c.componentDidUpdate != null) {
    c._renderCallbacks.push(() => {
      c.componentDidUpdate(oldProps, oldState, snapshot)
    })
  }
}
```

isNew つまりコンポーネントが新規作成ならば、`componentWillMount` と `componentDidMount` を実行します。

ここで面白いのは componentDidMount です。即時実行せずに renderQueue に詰め込んでいます。
これはマウントされた後に実行したいからです。
くわしくは commitRoot の説明でみていきましょう。

このコードブロックで面白いのは、`shouldComponentUpdate` です。
パフォチューの文脈で

- 再レンダリングするとその子もされる
- 再レンダリング抑制すればその子のレンダリングを止められる

という話を聞いたことはないでしょうか。

その挙動をまさしく再現しているのが次のコードです。

```js
// 再レンダリング抑制
if (
  (!c._force &&
    c.shouldComponentUpdate != null &&
    c.shouldComponentUpdate(newProps, c._nextState, componentContext) ===
      false) ||
  newVNode._original === oldVNode._original
) {
  if (c._renderCallbacks.length) {
    commitQueue.push(c)
  }

  reorderChildren(newVNode, oldDom, parentDom)
  break outer
}
```

`shouldComponentUpdate` があればこの時点で break しています。
このブロックの先には diffChildren があるのですが、それを実行しなくて済んでいるわけです。
つまり子の再レンダリングが抑制できています。

そして本命の

```js
diffChildren(
  parentDom,
  Array.isArray(renderResult) ? renderResult : [renderResult],
  newVNode,
  oldVNode,
  globalContext,
  isSvg,
  excessDomChildren,
  commitQueue,
  oldDom,
  isHydrating
)
```

です。

これは コンポーネントの children に対して diff を取る処理です。
diff を取る対象がコンポーネント(type=='function'の分岐の場合)の場合、実は diff という関数で diff をとっているのは `diffChildren` を呼び出すことです。

diffChildren は内部で diff を呼び（つまり関数を跨いだ再帰をしている）次第に diff を取る対象が primitivie な場合である分岐に入っていきます。

詳しくは diffChildren の説明で解説します。

この diffChildren が実行されると、あとは if の分岐から出て、`return newVNode._dom;` が実行されます。つまり差分をとった後の DOM が返されるわけです。
この newVNode.\_dom が差分をとった DOM になるのは、diff を取る中で引数を破壊的変更していくからなのですが、それについても後から見ていきます。

#### diff を取る対象が primitive の場合

diff を取る対象が primitive の場合のコードブロックは次の通りです。

```js
else if (
			excessDomChildren == null &&
			newVNode._original === oldVNode._original
		) {
      // typeがfunctionでない && 過剰なchildren(excessDomChildren) がない場合
			newVNode._children = oldVNode._children;
			newVNode._dom = oldVNode._dom;
		} else {
      // typeがfunctionでない && 過剰なchildren(excessDomChildren) がある場合
			newVNode._dom = diffElementNodes(
				oldVNode._dom,
				newVNode,
				oldVNode,
				globalContext,
				isSvg,
				excessDomChildren,
				commitQueue,
				isHydrating
			);
		}
```

まず前半の

```js
else if (
			excessDomChildren == null &&
			newVNode._original === oldVNode._original
		)
```

は、過剰な children がない、\_original に差分がないということですが、このとき何も変更が生じていないとみなせるので　 diff を取る必要がないことが事前に分かります。

ちなみにこの \_original は render から diff を読んだときはセットされていないのですが、いつセットされるものなのかは コンポーネントの説明で解説します。

もしこの分岐に入らなかった場合、つまり **type が function でない and（\_original に差分がある or 余剰な children がある）**場合、VNode に対して diffElementNodes が実行されます。

```js
newVNode._dom = diffElementNodes(
  oldVNode._dom,
  newVNode,
  oldVNode,
  globalContext,
  isSvg,
  excessDomChildren,
  commitQueue,
  isHydrating
)
```

これは vnode の差分を比較し、その差分を反映した dom を返す関数です。
これもあとで詳しく見ていきましょう。

#### diff が呼び出す関数を読む

お疲れ様です。
ここまでで diff は読めました。
しかし diff 関数自体は diff をとっているわけではなく、その本命が別にいることがわかりました。

これからそれらを読んでいきましょう。

diffChildren と diffElementNodes です。

先にいうと、 diffChildren は内部で diff を呼び出し続け、その結果いつかは diffElementNodes が呼ばれる分岐に入ります。
そこで先に diffElementNodes から見ていきましょう。

### diffElementNodes

diffElementNodes は 要素の props を比較して、更新があればそれを DOM に反映する処理の起点となるものです。
diffElementNodes の定義はこうなっています。

```js
function diffElementNodes(
  dom,
  newVNode,
  oldVNode,
  globalContext,
  isSvg,
  excessDomChildren,
  commitQueue,
  isHydrating
) {
  let i

  // 比較対象の抽出
  let oldProps = oldVNode.props
  let newProps = newVNode.props

  // svg かどうかで変わる処理があるのでフラグとして持つ
  isSvg = newVNode.type === "svg" || isSvg

  if (excessDomChildren != null) {
    for (i = 0; i < excessDomChildren.length; i++) {
      const child = excessDomChildren[i]
      if (
        child != null &&
        ((newVNode.type === null
          ? child.nodeType === 3
          : child.localName === newVNode.type) ||
          dom == child)
      ) {
        dom = child
        excessDomChildren[i] = null
        break
      }
    }
  }

  // dom がないときは作る
  if (dom == null) {
    if (newVNode.type === null) {
      return document.createTextNode(newProps)
    }

    dom = isSvg
      ? document.createElementNS("http://www.w3.org/2000/svg", newVNode.type)
      : document.createElement(
          newVNode.type,
          newProps.is && { is: newProps.is }
        )
    excessDomChildren = null
    isHydrating = false
  }

  if (newVNode.type === null) {
    if (oldProps !== newProps && (!isHydrating || dom.data !== newProps)) {
      dom.data = newProps
    }
  } else {
    // 更新するVNode typeがなんらかの値を持っている場合

    if (excessDomChildren != null) {
      excessDomChildren = EMPTY_ARR.slice.call(dom.childNodes)
    }

    oldProps = oldVNode.props || EMPTY_OBJ

    // props の diff を取って DOM に反映する関数. この関数は 実DOM を直接操作する
    diffProps(dom, newProps, oldProps, isSvg, isHydrating)

    i = newVNode.props.children

    // 新propsにchildrenがあるのならばchildrenに対しても差分を取る
    diffChildren(
      dom,
      Array.isArray(i) ? i : [i],
      newVNode,
      oldVNode,
      globalContext,
      newVNode.type === "foreignObject" ? false : isSvg,
      excessDomChildren,
      commitQueue,
      EMPTY_OBJ,
      isHydrating
    )

    // form周りの扱い. input 要素が value や checked を持っている場合の扱い
    if (
      "value" in newProps &&
      (i = newProps.value) !== undefined &&
      (i !== dom.value || (newVNode.type === "progress" && !i))
    ) {
      setProperty(dom, "value", i, oldProps.value, false)
    }
    if (
      "checked" in newProps &&
      (i = newProps.checked) !== undefined &&
      i !== dom.checked
    ) {
      setProperty(dom, "checked", i, oldProps.checked, false)
    }
  }

  return dom
}
```

それでは一つずつ見ていきましょう。

### diffChildren

```js
export function diffChildren(
  parentDom,
  renderResult,
  newParentVNode,
  oldParentVNode,
  globalContext,
  isSvg,
  excessDomChildren,
  commitQueue,
  oldDom,
  isHydrating
) {
  let i, j, oldVNode, childVNode, newDom, firstChildDom, refs
  let oldChildren = (oldParentVNode && oldParentVNode._children) || EMPTY_ARR

  let oldChildrenLength = oldChildren.length
  if (oldDom == EMPTY_OBJ) {
    if (excessDomChildren != null) {
      oldDom = excessDomChildren[0]
    } else if (oldChildrenLength) {
      oldDom = getDomSibling(oldParentVNode, 0)
    } else {
      oldDom = null
    }
  }

  newParentVNode._children = []
  for (i = 0; i < renderResult.length; i++) {
    childVNode = renderResult[i]

    if (childVNode == null || typeof childVNode == "boolean") {
      // JSXの中に{null}とか{true}を入れてる場合の挙動
      childVNode = newParentVNode._children[i] = null
    } else if (typeof childVNode == "string" || typeof childVNode == "number") {
      // JSXの中に{1}とか{"1"}を入れてる場合の挙動
      childVNode = newParentVNode._children[i] = createVNode(
        null,
        childVNode,
        null,
        null,
        childVNode
      )
    } else if (Array.isArray(childVNode)) {
      // JSXの中に{[1, <div>hoge</div>]}などを入れてる場合の挙動
      childVNode = newParentVNode._children[i] = createVNode(
        Fragment,
        { children: childVNode },
        null,
        null,
        null
      )
    } else if (childVNode._dom != null || childVNode._component != null) {
      // JSXの中に<div>hoge</div>などコンポーネントを入れ子にしている場合の挙動
      childVNode = newParentVNode._children[i] = createVNode(
        childVNode.type,
        childVNode.props,
        childVNode.key,
        null,
        childVNode._original
      )
    } else {
      childVNode = newParentVNode._children[i] = childVNode
    }

    if (childVNode == null) {
      // loopから抜けて次のloopに移る
      continue
    }

    childVNode._parent = newParentVNode
    childVNode._depth = newParentVNode._depth + 1

    oldVNode = oldChildren[i]

    if (
      oldVNode === null ||
      (oldVNode &&
        childVNode.key == oldVNode.key &&
        childVNode.type === oldVNode.type)
    ) {
      oldChildren[i] = undefined
    } else {
      for (j = 0; j < oldChildrenLength; j++) {
        oldVNode = oldChildren[j]
        // children のうち key と type が一致したものは children の比較をしない (break する)
        if (
          oldVNode &&
          childVNode.key == oldVNode.key &&
          childVNode.type === oldVNode.type
        ) {
          oldChildren[j] = undefined
          break
        }
        oldVNode = null
      }
    }

    // 上の比較で key や type が異なっていた場合は oldVNode は null なので、oldVNode は EMPTY_OBJ として diffを取る
    // key やtype が一致していれば oldVNode は oldChildren[j] で、この値を使って diff を取る。
    oldVNode = oldVNode || EMPTY_OBJ
    newDom = diff(
      parentDom,
      childVNode,
      oldVNode,
      globalContext,
      isSvg,
      excessDomChildren,
      commitQueue,
      oldDom,
      isHydrating
    )

    if (newDom != null) {
      if (firstChildDom == null) {
        firstChildDom = newDom
      }

      oldDom = placeChild(
        parentDom,
        childVNode,
        oldVNode,
        oldChildren,
        excessDomChildren,
        newDom,
        oldDom
      )
      if (!isHydrating && newParentVNode.type == "option") {
        parentDom.value = ""
      } else if (typeof newParentVNode.type == "function") {
        newParentVNode._nextDom = oldDom
      }
    } else if (
      oldDom &&
      oldVNode._dom == oldDom &&
      oldDom.parentNode != parentDom
    ) {
      oldDom = getDomSibling(oldVNode)
    }
  }

  newParentVNode._dom = firstChildDom

  if (excessDomChildren != null && typeof newParentVNode.type != "function") {
    for (i = excessDomChildren.length; i--; ) {
      if (excessDomChildren[i] != null) removeNode(excessDomChildren[i])
    }
  }

  // for ループの中で使用済みのものには undefined が詰め込まれているはず。それでも余っているものをここでunmountする
  for (i = oldChildrenLength; i--; ) {
    if (oldChildren[i] != null) unmount(oldChildren[i], oldChildren[i])
  }
}
```

## 読む上で出てくるであろう疑問とその答え

### VNode.type が function だとどうなるのか

### 新コンポーネントを作った後に oldProps = c.props したら意味がないのでは

まず oldProps, newProps はこのコードが呼ばれた段階であまり使わなくなります。
唯一使うのは、

```js
if (
  newType.getDerivedStateFromProps == null &&
  newProps !== oldProps &&
  c.componentWillReceiveProps != null
) {
  c.componentWillReceiveProps(newProps, componentContext)
}
```

のタイミングです。このときの比較の条件で使います。
このとき `newProps !== oldProps` が絶対に false になってこの処理が呼ばれないように見えますが、実際には大丈夫です。

そもそも componentWillReceiveProps は新規コンポーネントに対しては呼ばれないものです。
そして oldProps = c.props は新規コンポーネント作成でしか呼ばれないためです。

### なんで再帰構造になっているのか

木を辿るためです。

### key の比較のところがよく分からなかった

これは EMPTY_OBJECT が入った状態で diff が呼ばれる時のループを追うと良い。
newDom が作られることがわかる。
そのため DOM が作り直されることとなり再レンダリングが必然的には知りパフォーマンスが落ちる。