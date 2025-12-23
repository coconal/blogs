---
title: react 渲染key 为什么不用index?
date: 2025-12-23 16:00:00
categories:
  - react
tags:
  -
# slug:
# description:
# cover: /images/cover/my-first-post.jpg
---


## 底层原因

在 `react` 渲染的时候，虚拟dom，也就是 `fiber` 节点并不会删除，而是先存储到一个 `map` 结构里面，在渲染同级节点的时候看有没有复用，然后删除定义的 `map` 中对应 `key` 的值。最终还存储在 `map` 里的节点直接在 `浏览器dom` 中删除。下面是部分源码。

```js
function mapRemainingChildren(
    currentFirstChild: Fiber,
  ): Map<string | number | ReactOptimisticKey, Fiber> {
    // Add the remaining children to a temporary map so that we can find them by
    // keys quickly. Implicit (null) keys get added to this set with their index
    // instead.
    const existingChildren: Map<
      | string
      | number
      // This type is only here for the case when enableOptimisticKey is disabled.
      // Remove it after it ships.
      | ReactOptimisticKey,
      Fiber,
    > = new Map();

    let existingChild: null | Fiber = currentFirstChild;
    while (existingChild !== null) {
      if (existingChild.key === null) {
        existingChildren.set(existingChild.index, existingChild);
      } else if (
        enableOptimisticKey &&
        existingChild.key === REACT_OPTIMISTIC_KEY
      ) {
        // For optimistic keys, we store the negative index (minus one) to differentiate
        // them from the regular indices. We'll look this up regardless of what the new
        // key is, if there's no other match.
        existingChildren.set(-existingChild.index - 1, existingChild);
      } else {
        existingChildren.set(existingChild.key, existingChild);
      }
      existingChild = existingChild.sibling;
    }
    return existingChildren;
  }

  function reconcileChildrenArray(
    returnFiber: Fiber,
    currentFirstChild: Fiber | null,
    newChildren: Array<any>,
    lanes: Lanes,
  ): Fiber | null {
    // This algorithm can't optimize by searching from both ends since we
    // don't have backpointers on fibers. I'm trying to see how far we can get
    // with that model. If it ends up not being worth the tradeoffs, we can
    // add it later.

    // Even with a two ended optimization, we'd want to optimize for the case
    // where there are few changes and brute force the comparison instead of
    // going for the Map. It'd like to explore hitting that path first in
    // forward-only mode and only go for the Map once we notice that we need
    // lots of look ahead. This doesn't handle reversal as well as two ended
    // search but that's unusual. Besides, for the two ended optimization to
    // work on Iterables, we'd need to copy the whole set.

    // In this first iteration, we'll just live with hitting the bad case
    // (adding everything to a Map) in for every insert/move.

    // If you change this code, also update reconcileChildrenIterator() which
    // uses the same algorithm.

    let knownKeys: Set<string> | null = null;

    let resultingFirstChild: Fiber | null = null;
    let previousNewFiber: Fiber | null = null;

    let oldFiber = currentFirstChild;
    let lastPlacedIndex = 0;
    let newIdx = 0;
    let nextOldFiber = null;
    for (; oldFiber !== null && newIdx < newChildren.length; newIdx++) {
      if (oldFiber.index > newIdx) {
        nextOldFiber = oldFiber;
        oldFiber = null;
      } else {
        nextOldFiber = oldFiber.sibling;
      }
      const newFiber = updateSlot(
        returnFiber,
        oldFiber,
        newChildren[newIdx],
        lanes,
      );
      if (newFiber === null) {
        // TODO: This breaks on empty slots like null children. That's
        // unfortunate because it triggers the slow path all the time. We need
        // a better way to communicate whether this was a miss or null,
        // boolean, undefined, etc.
        if (oldFiber === null) {
          oldFiber = nextOldFiber;
        }
        break;
      }

      if (__DEV__) {
        knownKeys = warnOnInvalidKey(
          returnFiber,
          newFiber,
          newChildren[newIdx],
          knownKeys,
        );
      }

      if (shouldTrackSideEffects) {
        if (oldFiber && newFiber.alternate === null) {
          // 命中了这个位置的“槽位”，但没复用旧 fiber
          // 所以我们需要删除旧的child
          deleteChild(returnFiber, oldFiber);
        }
      }
      lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
      if (previousNewFiber === null) {
        // TODO: Move out of the loop. This only happens for the first run.
        resultingFirstChild = newFiber;
      } else {
        // TODO: Defer siblings if we're not at the right index for this slot.
        // I.e. if we had null values before, then we want to defer this
        // for each null value. However, we also don't want to call updateSlot
        // with the previous one.
        previousNewFiber.sibling = newFiber;
      }
      previousNewFiber = newFiber;
      oldFiber = nextOldFiber;
    }

    if (newIdx === newChildren.length) {
      // We've reached the end of the new children. We can delete the rest.
      deleteRemainingChildren(returnFiber, oldFiber);
      if (getIsHydrating()) {
        const numberOfForks = newIdx;
        pushTreeFork(returnFiber, numberOfForks);
      }
      return resultingFirstChild;
    }

    if (oldFiber === null) {
      // If we don't have any more existing children we can choose a fast path
      // since the rest will all be insertions.
      for (; newIdx < newChildren.length; newIdx++) {
        const newFiber = createChild(returnFiber, newChildren[newIdx], lanes);
        if (newFiber === null) {
          continue;
        }
        if (__DEV__) {
          knownKeys = warnOnInvalidKey(
            returnFiber,
            newFiber,
            newChildren[newIdx],
            knownKeys,
          );
        }
        lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
        if (previousNewFiber === null) {
          // TODO: Move out of the loop. This only happens for the first run.
          resultingFirstChild = newFiber;
        } else {
          previousNewFiber.sibling = newFiber;
        }
        previousNewFiber = newFiber;
      }
      if (getIsHydrating()) {
        const numberOfForks = newIdx;
        pushTreeFork(returnFiber, numberOfForks);
      }
      return resultingFirstChild;
    }

    // Add all children to a key map for quick lookups.
    const existingChildren = mapRemainingChildren(oldFiber);

    // Keep scanning and use the map to restore deleted items as moves.
    for (; newIdx < newChildren.length; newIdx++) {
      const newFiber = updateFromMap(
        existingChildren,
        returnFiber,
        newIdx,
        newChildren[newIdx],
        lanes,
      );
      if (newFiber !== null) {
        if (__DEV__) {
          knownKeys = warnOnInvalidKey(
            returnFiber,
            newFiber,
            newChildren[newIdx],
            knownKeys,
          );
        }
        if (shouldTrackSideEffects) {
          const currentFiber = newFiber.alternate;
          if (currentFiber !== null) {
            // The new fiber is a work in progress, but if there exists a
            // current, that means that we reused the fiber. We need to delete
            // it from the child list so that we don't add it to the deletion
            // list.
            if (
              enableOptimisticKey &&
              currentFiber.key === REACT_OPTIMISTIC_KEY
            ) {
              existingChildren.delete(-newIdx - 1);
            } else {
              existingChildren.delete(
                currentFiber.key === null ? newIdx : currentFiber.key,
              );
            }
          }
        }
        lastPlacedIndex = placeChild(newFiber, lastPlacedIndex, newIdx);
        if (previousNewFiber === null) {
          resultingFirstChild = newFiber;
        } else {
          previousNewFiber.sibling = newFiber;
        }
        previousNewFiber = newFiber;
      }
    }

    if (shouldTrackSideEffects) {
      // Any existing children that weren't consumed above were deleted. We need
      // to add them to the deletion list.
      existingChildren.forEach(child => deleteChild(returnFiber, child));
    }

    if (getIsHydrating()) {
      const numberOfForks = newIdx;
      pushTreeFork(returnFiber, numberOfForks);
    }
    return resultingFirstChild;
  }
```

整体思路基本上就是我上面所说，我再次将上面的代码总结为一句话
***用 mapRemainingChildren 把剩余旧节点放进 existingChildren (key → oldFiber)，遍历新 children 时通过 key/index 找到可复用的旧 fiber 并从 Map 中删除，最后对 Map 中仍然存在的旧 fiber 调用 deleteChild，实现“根据 key 对比后删除多余旧节点”的逻辑。***

## 索引作为key会发生什么呢

### 案例1

```js
function App() {

  const [arr, setArr] = useState([
    'one', 'two', 'three'
  ]);
  return (
    <div className="App" >
      <div>

        {
          arr.map((item, index) => {
            return <div key={index}>{item}</div>
          })
        }
      </div>
      <button onClick={() => { setArr(['two', 'three', 'one']) }}>更改</button>
    </div>
  );
}
```

渲染如图
![img1](/img/diff/diff-ex1.png)

更改后会如何呢
![img2](/img/diff/diff-ex2.png)

没有什么问题，那么什么情况下会出现变化呢
我们更改一下代码

### 案例2

```js
function App() {

  const [items, setItems] = useState(initialItems);

  const shuffle = () => {
    // 简单用反转来模拟“顺序变化”
    setItems((prev) => [...prev].reverse());
  };

  return (
    <div className="App" style={{ padding: 20 }}>
      <h1>key 的设计示例</h1>

      <h2>错误示例：使用 index 作为 key</h2>
      <p>
        在 A 这一行的输入框中输入内容，然后点击「打乱顺序」，
        你会看到输入的内容跑到了别的字母上。
      </p>
      {items.map((item, index) => (
        <div key={index} style={{ marginBottom: 8 }}>
          <span style={{ display: 'inline-block', width: 20 }}>{item.label}：</span>
          {/* 不受 React 控制的 input，用来暴露 DOM 复用的问题 */}
          <input placeholder={`给 ${item.label} 输入点内容`} />
        </div>
      ))}

      <hr style={{ margin: '24px 0' }} />

      <h2>正确示例：使用 id 作为 key</h2>
      <p>
        在 A 那一行输入内容，再点「打乱顺序」，你会看到这一行整体移动，
        输入的内容会跟着 A 这一项一起移动。
      </p>
      {items.map((item) => (
        <div key={item.id} style={{ marginBottom: 8 }}>
          <span style={{ display: 'inline-block', width: 20 }}>{item.label}：</span>
          <input placeholder={`给 ${item.label} 输入点内容`} />
        </div>
      ))}

      <button
        onClick={shuffle}
        style={{
          marginTop: 24,
          padding: '6px 12px',
          cursor: 'pointer',
        }}
      >
        打乱顺序
      </button>
    </div>
  );
}
```

页面如下

![img3](/img/diff/diff-ex3.png)

反转后页面如下

![img4](/img/diff/diff-ex4.png)

现在可以明显看出区别，使用 `index` 会导致文本text会更改，但是dom还是会复用。这也解释了为什么上一个例子使用 `index` 但是区别不大。看似没有复用，但实际上还是复用了节点，`React` 看到 `key` 还是 `0`、`1`、`2`，就会认为「第 0 个旧节点」 = 「第 0 个新节点」，只是内容从 one 变成了 two。
