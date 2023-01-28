# 상태가 변경되어 리렌더링될 때 어디에서 값을 가져올까?

<br />

컴포넌트가 새로 마운트 되고 나서 `useState()`훅은 마운트 될때와는 다른 곳에서 값을 가져옵니다.  
업데이트 구현체인 `updateState()`를 사용해서 가져오게 됩니다.

```javascript
reconciler > ReactFiberHooks.js

  
function updateState(initialState) {
  return updateReducer(basicStateReducer, initialState)
}
```

훅 객체는 최초 마운트 될 때, 아래와 같은 값으로 훅 구현체를 생성합니다.  
지금 불러온 업데이트 구현체는 이전에 만들어놨던 훅 구현체를 불러오고, 훅의 Queue에 담겨있는 update를 모부 소모해서 최종 값을 구합니다.  
그 진행과정을 `baseUpdate와 baseState`에 기입하게 됩니다.

```javascript
 const hook: Hook = {
    memoizedState: null, // 컴포넌트에 적용된 마지막 상태 값
    queue: null, // 훅이 호출될 때마다 update를 연결 리스트로 queue에 집어넣습니다.
    next: null, // 다음 훅을 가리키는 포인터

    // 업데이트 구현체에서 설명
    baseState: null,
    baseUpdate: null,
  }
```

호출된 `updateRecuer`는 기존 훅을 작업용 훅으로 만들고, 재사용합니다.
그 처리는 `updateWorkInProgresHook` 함수로부터 받아오게됩니다.

```javascript
function updateReducer(reducer, initialArg, init) {
  const hook = updateWorkInProgressHook()
  /*...*/
}

function updateWorkInProgressHook() {
  if (nextWorkInProgressHook !== null) {
    // Render phase update로 인해 재호출될 경우 아래에서 만들어논 객체를 재사용..
    workInProgressHook = nextWorkInProgressHook
    nextWorkInProgressHook = workInProgressHook.next
    // current hook
    currentHook = nextCurrentHook
    nextCurrentHook = currentHook !== null ? currentHook.next : null
  } else {
    currentHook = nextCurrentHook
    // 작업용 훅 객체를 만든다.
    const newHook: Hook = {
      memoizedState: currentHook.memoizedState,

      baseState: currentHook.baseState,
      queue: currentHook.queue,
      baseUpdate: currentHook.baseUpdate,

      next: null,
    }

    if (workInProgressHook === null) {
      // This is the first hook in the list.
      workInProgressHook = firstWorkInProgressHook = newHook
    } else {
      // Append to the end of the list.
      workInProgressHook = workInProgressHook.next = newHook
    }
    nextCurrentHook = currentHook.next
  }
}
```

## Circular Linked List로 관리되는 baseUpdate, baseState, update

<br />

일반적인 선형 연결리스트가 아닌 원형 연결리스트로 업데이트 내역이 관리가 됩니다.

```javascript
const last = queue.last
if (last === null) {
  update.next = update
} else {
  const first = last.next
  if (first !== null) {
    update.next = first
  }
  last.next = update
}
queue.last = update
```

먼저 `setState()의 호출`과 `update의 소비`에 대해서 다시 한번 정리해보겠습니다.

1. 컴포넌트 상태 변경을 위해 `setState()` 호출.
2. 훅 내부의 `queue`에 `update`가 연결리스트 형태로 추가.
3. 리렌더링 후 컴포넌트 재호출.
4. `queue`에서 `update`를 꺼내서 소비.

문제는 4번 과정을 매번 root에서부터 타고 내려간다면 이미 소비된 update를 또 처리하게 됩니다.  
이미 소비된 update는 GC로부터 제거되게끔 연결을 끊어줄 필요가 있는 것이죠.  
이미 처리한 update와 처리할 update 정보를 표시해주는 프로퍼티가 바로 `baseUpdate, baseState` 입니다.  

**원형 연결리스트의 head는 어디에서 찾나요?**

정답은 `head를 알 필요가 없다`입니다.  
컴포넌트가 재호출 되지 않으면 update 정보를 갱신할 필요가 없습니다. last.next의 존재 여부에 따라 head를 훅의 baseUpdate로 가지고 있는지를 파악해서 끊어내는 작업을 처리합니다.

## update 소비하기

<br />

update 정보를 담고 있는 자료구조는 hook 내부의 queue 이외에도 있습니다.  
Render phase에서 `dispatchAction`이 호출되었을 때 저장해둔 `renderPhaseUpdates 해시맵`입니다. 일단 로직을 먼저 보고, 두 분기를 모두 처리하는 코드를 확인하겠습니다.  

```javascript
function updateReducer(reducer, initialArg, init) {
  const last = queue.last
  const baseUpdate = hook.baseUpdate
  const baseState = hook.baseState

  // 1. 적용시킬 update의 head를 가지고 온다.
  let first
  if (baseUpdate !== null) {
    if (last !== null) {
      last.next = null // 1-2 연결을 끊는다.
    }
    first = baseUpdate.next // 1-1 baseUpdate의 head 참조
  } else {
    first = last !== null ? last.next : null // 1-1 Circular Linked List의 head 참조
  }
  //..
}
```

최초 컴포넌트 재호출 로직 실행시에는 baseUpdate가 비어있기 때문에 아래 분기로 확인합니다.  
두 번째 호출부터는 baseUpdate를 갱신해주기 때문에 윗 분기에서 첫 노드를 넣어줍니다.  

```javascript
  // 2. head부터 tail까지 차례로 리듀서에 action을 던져 결괏값을 취한다.
if (first !== null) {
  let newState = baseState
  let prevUpdate = baseUpdate
  let update = first
  do {
    const action = update.action
    newState = reducer(newState, action)
    prevUpdate = update
    update = update.next
  } while (update !== null && update !== first)

  // 3. update를 모두 소비했다면 최종 상태값을 저장한다.
  hook.memoizedState = newState
  hook.baseUpdate = prevUpdate // 적용된 update의 tail pointer
  hook.baseState = newState // baseUpdate의 결괏값
}

const dispatch = queue.dispatch
return [hook.memoizedState, dispatch] // 최종 상태 값 반환
```

first부터 모든 큐를 비우면서 값을 갱신하고, memoizedState, baseUpdate, baseState를 갱신합니다.

## Render phase Update 적용

<br />

업데이트 로직은 동일합니다. 다만, 해시맵 자료구조이기 때문에 tail을 update하는 로직이 없습니다.

```javascript
function updateReducer(reducer, initialArg, init) {
  /*...*/
  if (numberOfReRenders > 0) {
    const dispatch = queue.dispatch

    if (renderPhaseUpdates !== null) {
      const firstRenderPhaseUpdate = renderPhaseUpdates.get(queue)

      if (firstRenderPhaseUpdate !== undefined) {
        renderPhaseUpdates.delete(queue)
        let newState = hook.memoizedState
        let update = firstRenderPhaseUpdate

        do {
          const action = update.action
          newState = reducer(newState, action)
          update = update.next
        } while (update !== null)
  
        hook.memoizedState = newState
        queue.lastRenderedState = newState
        return [newState, dispatch]
      }
    }
    return [hook.memoizedState, dispatch]
  }
  /*...*/
}
```

**출처 | https://goidle.github.io/react/in-depth-react-hooks_2/**