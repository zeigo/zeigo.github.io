---
title: React Fiber的实现
date: 2018-01-28 15:58:09
tags: [React, JavaScript]
category: JavaScript
description: React Fiber的实现
---
在React之前的版本中，进行更新时是连续的，将会阻塞整个线程。如果此时有一些重要的任务，比如用户交互、动画等被阻塞时，体验显然是很差的。而有些更新实际并不用着急完成，比如根据网络请求结果更新的内容，或者发生在画面以外。React16为了解决这一问题，而提出了Fiber的核心架构。

<!-- more -->

## 概念

- virtual stack frame

之前的React基于栈（stack），当函数被调用，会有新的栈帧（stack frame）进入栈，栈帧包含了这个函数要执行的工作的信息。执行完，栈帧出栈，当栈为空时才停止工作。

为了解决之前UI渲染的问题，有必要自定义调用栈的行为，从而可以根据意愿中断调用栈并手动操作栈帧。

fiber就是栈的特殊实现，将函数的调用栈帧转为内存中可控的fiber，从而可以决定何时执行这些工作（perform unit of work），或许可以视为将递归转为了循环。

- unit of work

Fiber引入了新的概念incremental rendering（渐进式渲染，将渲染工作分隔，分布到不同的帧）。

> incremental rendering: the ability to split rendering work into chunks and spread it out over multiple frames.

整个update的work可以分成不同的工作单元（unit of work），fiber就代表着工作单元。

## 基本属性

fiber包含着work的对象(what to do)和方式(how to do)，其属性就是为这两者服务。

### what

fiber的工作对象可以由`fiber.stateNode`得到。我们知道JSX解析后得到React Element，根据其type可以得到不同的实例，这些实例就是work的对象，而从实例也可以反回来得到fiber。type是string对应HostComponent实例（DOM元素，其`_reactInternalInstance`为fiber），type为构造函数对应ClassComponent实例（其`_reactInternalFiber`为fiber。需要注意React还会为`ReactDOM.render`的容器元素创建单独的HostRoot对象（其`current`为fiber）。

instance的相关信息

- tag 实例类型，重要：HostRoot 3, ClassComponent 2, HostComponent 5, HostText 6

- key 对应实例element的key

- type 对应实例element的type

- internalContextTag 表示是否默认异步渲染 AsyncUpdates 1

fiber之间的联系，可以反映出其实例的联系

- return  父元素

- child  第一个子元素

- sibling  相邻后一个兄弟元素

### how

- alternate

对于一个实例，最多会有两个对应的fiber（current和workInProgress），两者通过alternate互相指向，即current.alternate === workInProgress，workInProgress.alternate === current。current表示实例当前的状态，当需要更新时，会根据current复制出workInProgress，两者的信息相同，改变都发生在workInProgress上。当更新完成后，workInProgress会变成current。

- updateQueue 包含了更新的信息，形式有多种。

对于ClassComponent，是一个setState形成的Update对象组成的链表。

对于HostComponent，是DOM元素新旧属性对比得到的一个数组，描述了该如何更新其属性。比如`<div id="i1">`要变成`<div id="i2">`，会得到一个数组`["id", "i2"]`，表示将id变为i2。

- memorizedState 实例最终的state

- pendingProps 实例即将接受的props，作用在workInProgress上

- memorizedProps 实例最终的props

- expirationTime 完成工作的期限

```js
// 1单位代表10ms
NoWork = 0
Sync = 1
Never = 2147483647
// async任务：当前时间加上约1000ms，超时后为同步任务
```

- effectTag 更新造成side effect的类型

React将更新的差异视为side effect，其类型以二进制表示，好处是可以使用位运算，&取交，|取并。

```js
// 重要的几种
NoEffect // 0, 不操作
Placement // 2, 插入
Update // 4, 更新
Deletion // 8, 删除

fiber.effectTag & Deletion // 是否含Deletion
fiber.effectTag |= Placement // 加入Placement
fiber.effectTag &= ~Update // 删去Update
```

effect相关属性

- nextEffect 下一个要处理的effect

- firstEffect 子树中第一个要处理的effect

- lastEffect 子树中最后一个要处理的effect

## Scheduler

既然不使用栈，那各个工作单元的执行过程也不以栈的形式决定。React有自己的调度（scheduler），来决定how和when执行工作单元。以下根据React应用更新的流程来具体分析fiber及scheduler的运用，注意下面的代码为便于理解，相比源码做了一定的简化。

要更新一个React应用的状态，最常用的就是ReactDOM.render和setState，两者的流程大致相同。

### 找到更新对应的fiber

render对应的是HostRoot实例的fiber，是最顶层的fiber，初次渲染时需要新建HostRoot实例及其fiber；而setState对应的则是ClassComponent实例的fiber

### 计算更新的expirationTime

```js
function computeExpirationForFiber(fiber) {
  let expirationTime;
  if (expirationContext !== NoWork) {
    // An explicit expiration context was set;
    expirationTime = expirationContext;
  } else if (isWorking) {
    if (isCommitting) {
      // Updates that occur during the commit phase should have sync priority
      // by default.
      expirationTime = Sync;
    } else {
      // Updates during the render phase should expire at the same time as
      // the work that is being rendered.
      // render阶段，可能是componentWillMount等调用了setState，
      // 所以这个时候的setState会合并
      expirationTime = nextRenderExpirationTime;
    }
  } else {
    // No explicit expiration context was set, and we're not currently
    // performing work. Calculate a new expiration time.
    if (fiber.internalContextTag & AsyncUpdates) {
      // This is an async update
      expirationTime = computeAsyncExpiration();
    } else {
      // This is a sync update
      expirationTime = Sync;
    }
  }
  return expirationTime;
}
```

### insertUpdateIntoFiber

更新以Update的形式插入fiber的updateQueue，updateQueue的expirationTime保持为其中update里最小的expirationTime，也可以反映出fiber的expirationTime。

### scheduleWork

从当前的fiber开始，冒泡到root fiber，更新expirationTime。所以无论更新发生在哪里，最终都是从root开始工作的。

```js
while (node !== null) {
  if (node.expirationTime === NoWork || node.expirationTime > expirationTime) {
    node.expirationTime = expirationTime;
  } // 之前是NoWork或者新的expirationTime更小（时间更加紧迫），则更新expirationTime

  if (node['return'] === null) {
    if (node.tag === HostRoot) {
      var root = node.stateNode;
      requestWork(root, expirationTime);
    }
  }
  node = node['return'];
}
```

### requestWork

每当有更新发生，都会请求（request）开始工作，这时根据具体情况决定是否执行工作（performWork）。

```js
function requestWork(root, expirationTime) {

  // 把root加入scheduler中，由于页面可能有多个root，这里有firstScheduledRoot和
  // lastScheduledRoot两个变量来记录首尾，root的nextScheduledRoot指向
  // 下一个加入scheduler的root
  if (root.nextScheduledRoot === null) {
    // This root is not already scheduled. Add it.
    root.remainingExpirationTime = expirationTime;
    if (lastScheduledRoot === null) {
      firstScheduledRoot = lastScheduledRoot = root;
      root.nextScheduledRoot = root;
    } else {
      lastScheduledRoot.nextScheduledRoot = root;
      lastScheduledRoot = root;
      lastScheduledRoot.nextScheduledRoot = firstScheduledRoot;
    }
  } else {
    // This root is already scheduled, but its priority may have increased.
    var remainingExpirationTime = root.remainingExpirationTime;
    if (remainingExpirationTime === NoWork || expirationTime < remainingExpirationTime) {
      // Update the priority.
      root.remainingExpirationTime = expirationTime;
    }
  }

  if (isRendering) {
    // 如果此时正处于render阶段，说明这次update发生在componentWillMount等生命周期函数里，
    // 这次update会合并到之前的工作里
    return;
  }

  if (expirationTime === Sync) {
    performWork(Sync, null);
  } else {
    // 异步执行，requestIdleCallback，会根据expirationTime计算参数的timeout
    // 回调为performWork(NoWork)
    scheduleCallbackWithExpiration(expirationTime);
  }
}
```

### performWork

```js
function performWork(minExpirationTime, dl) {
  // minExpirationTime            dl
  //       Sync                  null
  //      NoWork       {didTimeout, timeRemaining()}
  deadline = dl;

  // 找到scheduler中expirationTime最小的root，设为nextFlushedRoot，
  // 并对应nextFlushedExpirationTime
  findHighestPriorityRoot();

  // 当找到了下个work并还没到deadline，performWorkOnRoot
  while (nextFlushedRoot !== null && nextFlushedExpirationTime !== NoWork && (minExpirationTime === NoWork || nextFlushedExpirationTime <= minExpirationTime) && !deadlineDidExpire) {
    performWorkOnRoot(nextFlushedRoot, nextFlushedExpirationTime);
    // Find the next highest priority work.
    findHighestPriorityRoot();
  }

  // 如果还有工作没做完，说明时间不够，继续异步
  if (nextFlushedExpirationTime !== NoWork) {
    scheduleCallbackWithExpiration(nextFlushedExpirationTime);
  }

  // Clean-up.
  deadline = null;
  deadlineDidExpire = false;
}
```

### performWorkOnRoot

work分为两个阶段，render和commit

```js
function performWorkOnRoot(root, expirationTime) {
  isRendering = true // 进入render阶段

  let finishedWork = root.finishedWork
  if (finishedWork !== null) {
    // 之前已完成render，但是时间不够commit而中断，现在继续commit
    root.finishedWork = null
    root.remainingExpirationTime = commitRoot(finishedWork)
  } else {
    finishedWork = renderRoot(root, expirationTime)
    // 如果renderRoot过程中时间不够而中断，finishedWork为null，不会commitRoot
    if (finishedWork !== null) {
      if (expirationTime < recalculateCurrentTime()) {
        // sync/expired work，直接commit
        root.remainingExpirationTime = commitRoot(finishedWork)
      } else if (!shouldYield()) { // async，还有时间剩余，直接commit
        root.remainingExpirationTime = commitRoot(finishedWork)
      } else { // 没时间剩余，之后再commit
        root.finishedWork = finishedWork
      }
    }
  }

  isRendering = false
}

// 判断暂停有两个地方用到
// 1.renderRoot的过程中，performUnitOfWork后要判断时间，
// 如果没时间，之后从nextUnitOfWork继续
// 2.renderRoot后要判断时间，如果没时间，之后从root.finishedWork继续
function shouldYield() {
  if (deadline === null) return false
  if (deadline.timeRemaining() > 1) return false
  deadlineDidExpire = true
  return true
}
```

### renderRoot

```js
// 当workLoop中performUnitOfWork因时间不够停止时，isReadyForCommit仍为false，返回null
function renderRoot(root, expirationTime) {
  isWorking = true
  root.isReadyForCommit = false

  // 判断是否从新开始，如果nextUnitOfWork不为null，证明之前有unit of work因时间不够
  // 还未完成，需要从中断的地方即nextUnitOfWork继续，
  // 否则根据current创建workInProgress从新开始
  if (root !== nextRoot || expirationTime !== nextRenderExpirationTime || nextUnitOfWork === null) {
    nextRoot = root
    nextRenderExpirationTime = expirationTime
    nextUnitOfWork = createWorkInProgress(root.current, null, nextRenderExpirationTime)
  }
  workLoop(expirationTime)
  isWorking = false
  return root.isReadyForCommit ? root.current.alternate : null
}
```

### workLoop

```js
function workLoop(expirationTime) {
  if (nextRenderExpirationTime === NoWork || nextRenderExpirationTime > expirationTime)
    return
  if (nextRenderExpirationTime < mostRecentCurrentTime) {
    // 如果是sync/expired work，不用考虑时间，执行每一个unit of work
    while (nextUnitOfWork !== null) {
      nextUnitOfWork = performUnitOfWork(nextUnitOfWork)
    }
  } else {
    // async任务到deadline后停止，nextUnitOfWork为下次继续的位置
    while (nextUnitOfWork !== null && !shouldYield()) {
      nextUnitOfWork = performUnitOfWork(nextUnitOfWork)
    }
  }
}
```

### performUnitOfWork

perform分为begin/complete两步。深度优先遍历，先begin当前的workInProgress，然后往下找child来执行下一步work，如果没有的话complete当前work，并找同级的sibling，还没有的话往上回溯return并complete，继续找sibling。

```js
function performUnitOfWork(workInProgress) {
  let current = workInProgress.alternate
  let next = beginWork(current, workInProgress, nextRenderExpirationTime)
  if (next === null) {
    next = completeUnitOfWork(workInProgress)
  }
  return next
}
```

### beginWork

diff算法所在，根据实例的类型决定reconcile的方法。通过对比来修改fiber树，确定Placement, Deletion, ContentRest等Tag，新建/更新ClassComponent实例, 根据pendingProps和updateQueue来确定workInProgress的memorizedProps/State

```js
// 返回下一个需要对比的节点，对应workInProgress.child
// 当到达叶节点时，无法再往下，返回null
function beginWork(current, workInProgress, renderExpirationTime) {
  if (workInProgress.expirationTime === NoWork || workInProgress.expirationTime > renderExpirationTime)
    return null
  // 低优先级可以不处理
  switch (workInProgress.tag) {
    case HostRoot:
      return updateHostRoot(current, workInProgress, renderExpirationTime)
    case HostComponent:
      return updateHostComponent(current, workInProgress, renderExpirationTime)
    case HostText:
      return updateHostText(current, workInProgress)
    case ClassComponent:
      return updateClassComponent(current, workInProgress, renderExpirationTime)
  }
}
```

### completeUnitOfWork

```js
// 当还有sibling时，不返回null，对sibling performUnitOfWork
function completeUnitOfWork(workInProgress) {
  while (true) {
    var current = workInProgress.alternate;
    var next = completeWork(current, workInProgress, nextRenderExpirationTime);
    // 具体处理

    var returnFiber = workInProgress['return'];
    var siblingFiber = workInProgress.sibling;
    // 重置expirationTime，一般重置为NoWork，如果还有未完成的update，子树还有updateQueue
    // 不为空，则为其中最小的expirationTime
    resetExpirationTime(workInProgress, nextRenderExpirationTime);

    if (returnFiber !== null) {
      // workInProgress子树的effect list连接到returnFiber的effect list上
      // 通过层层回溯，最终会连接到root fiber上
      if (returnFiber.firstEffect === null) {
        returnFiber.firstEffect = workInProgress.firstEffect;
      }
      if (workInProgress.lastEffect !== null) {
        if (returnFiber.lastEffect !== null) {
          returnFiber.lastEffect.nextEffect = workInProgress.firstEffect;
        }
        returnFiber.lastEffect = workInProgress.lastEffect;
      }

      var effectTag = workInProgress.effectTag;
      // 如果workInProgress本身有side effect，将workInProgress插到returnFiber
      // 的effect list的最后
      if (effectTag > PerformedWork) {
        if (returnFiber.lastEffect !== null) {
          returnFiber.lastEffect.nextEffect = workInProgress;
        } else {
          returnFiber.firstEffect = workInProgress;
        }
        returnFiber.lastEffect = workInProgress;
      }
    }

    if (siblingFiber !== null) {
      // If there is more work to do in this returnFiber, do that next.
      return siblingFiber;
    } else if (returnFiber !== null) {
      // If there's no more work in this returnFiber. Complete the returnFiber.
      workInProgress = returnFiber;
      continue;
    } else {
      // We've reached the root.
      var root = workInProgress.stateNode;
      root.isReadyForCommit = true;
      return null;
    }
  }

  return null;
}
```

### completeWork

主要针对HostComponent实例，包括新建DOM元素，根据pendingProps、memorizedProps得到updateQueue，设置effectTag等。

```js
function completeWork(current, workInProgress) {
  let newProps = workInProgress.pendingProps
  if (newProps === null) {
    newProps = workInProgress.memorizedProps
  } else {
    workInProgress.pendingProps = null
  }
  let oldProps = current !== null ? current.memorizedProps : null
  switch (workInProgress.tag) {
    case ClassComponent: {
      return null
    }
    case HostComponent: {
      let type = workInProgress.type
      if (current !== null && workInProgress.stateNode !== null) {
        // update
        let instance = workInProgress.stateNode
        let updatePayload = prepareUpdate(instance, type, oldProps, newProps)
        updateHostComponent(workInProgress, updatePayload)
      } else {
        let instance = createInstance(type, newProps, workInProgress)
        appendAllChildren(instance, workInProgress)
        finalizeInitialChildren(type, instance, newProps)
        workInProgress.stateNode = instance
      }
      return null
    }
    case HostText: {
      let newText = newProps;
      if (current !== null && workInProgress.stateNode !== null) {
        var oldText = current.memorizedProps
        updateHostText(workInProgress, oldText, newText)
      } else {
        let textInstance = createTextInstance(newText, workInProgress)
        workInProgress.stateNode = textInstance
      }
      return null
    }
    case HostRoot: {
      return null
    }
  }
}
```

### commitRoot

完成render阶段后，对比已经完成。通过begin和complete工作，需要做的更新保存在root fiber的firstEffect和lastEffect维持的effect list上，现在需要遍历链表把更新反映到实际的DOM视图上。

```js
// 遍历两次，最后返回值作为hostRoot的remainingExpirationTime
// 如果为NoWork，本次工作就结束了；否则还有工作没完成，会继续performWorkOnRoot
function commitRoot(finishedWork) {
  isWorking = true
  isCommitting = true
  let root = finishedWork.stateNode
  root.isReadyForCommit = false

  let firstEffect
  if (finishedWork.effectTag > PerformedWork) {
    // root本身有side effect，也要连接到effect list最后
    if (finishedWork.lastEffect !== null) {
      finishedWork.lastEffect.nextEffect = finishedWork
      firstEffect = finishedWork.firstEffect
    } else {
      firstEffect = finishedWork
    }
  } else {
    firstEffect = finishedWork.firstEffect
  }

  nextEffect = firstEffect

  while (nextEffect !== null) {
    commitAllHostEffects()
    // 第一次遍历，根据effectTag进行相应的DOM操作
    // 比如Placement要insertBefore/appendChild
    // Deletion要removeChild
  }

  root.current = finishedWork
  // workInProgress取代current
  nextEffect = firstEffect
  while (nextEffect !== null) {
    commitAllLifeCycles()
    // 第二次遍历，完成componentDidMount/componentDidUpdate及setState的回调等
  }
  isWorking = false
  isCommitting = false
  root.isReadyForCommit = false
  return root.current.expirationTime
}
```

## 总结

各个阶段的具体实现（begin/complete/commit）都比较繁琐，这里省略了，无非是switch case具体情况具体分析，这里重点还是放在整个更新的流程。个人认为难点还是在异步的状态管理上，之前的setState已经是异步，如今又提供了异步渲染，如何在保证性能的同时，建立数据状态与视图的映射？所幸React已经在底层实现，我们只需要声明式地构建应用。

## 参考

1. [React Fiber Architecture](https://github.com/acdlite/react-fiber-architecture)

1. [Lin Clark - A Cartoon Intro to Fiber - React Conf 2017](https://www.youtube.com/watch?v=ZCuYPiUIONs)