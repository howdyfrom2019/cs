# Hooks

<br />

React의 hook을 통한 상태의 업데이트는 큰 틀에서 아래와 같은 흐름으로 진행된다.

1. 훅을 통한 컴포넌트 상태 업데이트.
2. 업데이트를 반영할 Work를 scheduler에게 전달. Scheduler는 스케줄링된 task를 적절한 시기에 실행.
3. Work을 통해 VDOM 재조정 진행.
4. Work를 진행하면 발생한 변경점을 적용.
5. 사용자의 상호작용으로 이벤트 발생 후 등록된 핸들러 실행으로 1번으로 돌아감.

## Hook을 통한 상태 업데이트

<br />

훅은 어떤 경로로 컴포넌트에 접근할 수 있고, 어떻게 상태관리를 하는지 알아보려고 한다.

> useEffect는 reconciler에서 다룬다. 상태 업데이트는 오직 useState, useReducer에서 다룰 수 있다.

React의 코어는 컴포넌트 모델인 **React Element** 만 알고 있다.  
이 Element는 인스턴스로 만들기 전의 클래스와 유사한 형태인데 hooks는 그 안에서 상태를 관리할 수 있다.  
React Element는 **reconciler**로 인한 확장을 통해 hooks의 상태 등으로 확장이 가능하다.  
그러나 실제 패키지를 확인해보면 reconciler로 확장되는 코드를 확인해볼 수 없다.

```jsx
// react > React.js
import { useState, useEffect, ... } from './ReactHooks'
import ReactSharedInternals from './ReactSharedInternals' // 의존성을 주입받는 징검다리

const React = {
  useState,
  useEffect,
  __SECRET_INTERNALS_DO_NOT_USE_OR_YOU_WILL_BE_FIRED: ReactSharedInternals,
  /*...*/
}

export default React
```

```jsx
  
import ReactCurrentDispatcher from './ReactCurrentDispatcher'

function resolveDispatcher() {
  const dispatcher = ReactCurrentDispatcher.current
  return dispatcher
}

export function useState(initialState) {
  const dispatcher = resolveDispatcher()
  return dispatcher.useState(initialState)
}

export function useEffect(create, inputs) {
  const dispatcher = resolveDispatcher()
  return dispatcher.useEffect(create, inputs)
}
/*...*/
```

```jsx
  
const ReactCurrentDispatcher = {
  current: null,
}

export default ReactCurrentDispatcher
```

다른 패키지로부터 의존성을 주입받아 **ReactCurrentDispatcher.current**로 갱신된다는 걸 알 수 있다.  
이 의존성은 **shared**라는 패키지에서부터 주입받게 된다.

결론적으로 훅이 개발자에게 도달되는 흐름은 아래와 같다.

> **reconciler > shared/ReactSharedInternal.js > react/ReactSharedInternal > react/ReactCurrentDispatcher > react/ReactHooks > react > 개발자**

## reconciler로부터의 hooks 주입.

<br />

hook을 주입하는 가장 최상단은 reconciler라는 걸 알 수 있는데, 실제 삽입은 **reconciler/renderWithhHooks.js**에서 이루어진다.

```javascript
// reconciler > ReactFiberHooks.js
export function renderWithHooks(
  current: Fiber,
  workInProgress: Fiber,
  Component: any,
  props: any,
  refOrContext: any,
  nextRenderExpirationTime: ExpirationTime
) {
  /*...*/
  currentlyRenderingFiber = workInProgress // 현재 작업 중인 fiber를 전역으로 잡아둠
  nextCurrentHook = current !== null ? current.memoizedState : null

  ReactCurrentDispatcher.current =
    nextCurrentHook === null ? HooksDispatcherOnMount : HooksDispatcherOnUpdate;

  let children = Component(props, refOrContext)

  /*컴포넌트 재호출 로직..*/

  const renderedWork = currentlyRenderingFiber
  renderedWork.memoizedState = firstWorkInProgressHook

  ReactCurrentDispatcher.current = ContextOnlyDispatcher

  currentlyRenderingFiber = null;
  //...
}
```

> reconciler 패키지의 모든 전역변수(firstWorkInProgressHook, nextCurrentHook..)는 작업 중인 컴포넌트의 context를 따릅니다.   
> 컴포넌트의 작업이 끝나면 모두 초기화시켜서 다음 컴포넌트에서 사용이 가능하도록 합니다.

* **firstWorkInProgressHook** | memoizedState로 훅과 컴포넌트를 매핑.
* **memoizedState** | null이라면, 최초 마운트를 의미. 아니라면 업데이트를 의미.
* **ContextOnlyDispatcher** | 훅 구현체를 삽입. 그런데 이 hook은 올바르지 않은 사용에 대한 error throw를 목적으로 함.

```javascript
export const ContextOnlyDispatcher: Dispatcher = {
  useState: throwInvalidHookError,
  useEffect: throwInvalidHookError,
  /*...*/
};
```

## 훅의 본체는 어떻게 만들어지는가?

<br />

hook은 mountState에서 생성되며, 연결리스트로 관리된다.

```javascript
  
function mountState(initialState) {
  const hook = mountWorkInProgressHook();
  //..
}

function mountWorkInProgressHook(): Hook {
  // hook 객체에 대해서는 업데이트 구현체에서 자세히 다룹니다.
  const hook: Hook = {
    memoizedState: null, // 컴포넌트에 적용된 마지막 상태 값
    queue: null, // 훅이 호출될 때마다 update를 연결 리스트로 queue에 집어넣습니다.
    next: null, // 다음 훅을 가리키는 포인터

    // 업데이트 구현체에서 설명
    baseState: null,
    baseUpdate: null,
  }

  if (workInProgressHook === null) {
    // 맨 처음 실행되는 훅인 경우 연결 리스트의 head로 잡아둠
    firstWorkInProgressHook = workInProgressHook = hook
  } else {
    // 두번 째부터는 연결 리스트에 추가
    workInProgressHook = workInProgressHook.next = hook
  }
  return workInProgressHook
}
```

firstWorkInProgressHook은 훅 연결리스트이 head로 컴포넌트 실행이 끝났을 때 fiber(인스턴스)에 저장되어 컴포넌트와 훅 리스트를 연결해준다.
workInProgressHook은 현재 처리중인 훅을 나타내는 동시에 리스트의 tail 포인터로 사용된다.

한편 만약 우리가 <code>const [] = useState(getBlankString());</code> 처럼 함수를 실행해서 initialState를 넣어준다면 어떻게 될까요?

```javascript
if (typeof initialState === 'function') {
    // 생성자 함수일 경우
    initialState = initialState()
  }
  hook.memoizedState = hook.baseState = initialState
//...
```

타입 체크를 통해 해당 반환값을 얻어옵니다.  
최초 실행 이후부터는 mount 구현체가 아닌 update 구현체를 사용하므로 initialState가 실행되는 일은 없습니다.

## Update를 담는 queue

<br />

```javascript
// reconciler > ReactFiberHooks.js
function mountState(initialState) {
  /*...*/
  // hook.memoizedState = hook.baseState = initialState;

  const queue = (hook.queue = {
    last: null, // 마지막 update
    dispatch: null, // push 함수

    lastRenderedReducer: basicStateReducer,
    lastRenderedState: initialState,
  })

  const dispatch = (queue.dispatch = dispatchAction.bind(
    null,
    currentlyRenderingFiber,
    queue
  ))
  return [hook.memoizedState, dispatch]
}

function basicStateReducer(state, action) {
  return typeof action === 'function' ? action(state) : action;
}
```

setState()로 사용하는 dispatch 함수는 queue의 push 함수입니다.  
외부에 노출되는 함수인 만큼, bind를 통해 context를 넣어 줄 필요가 있습니다.


## Hook은 어떻게 리렌더링을 시키는가?

<br />

한편, fiber의 queue에 값을 업데이트할 뿐만 아니라, 컴포넌트 리렌더링을 위해 scheduler에게 Work를 예약해야합니다.  
방금 bind 된 dispatchAction() 안에서 실행되는데요. 이 bind() 때문에 생각해볼 문제가 생깁니다.  

```jsx
function FC() {
  const [a, setA] = useState(0)
  if (a === 1) setA(2)
  return <button onClick={() => setA(1)}></button>
}
```

VDOM의 reconciler는 변경 전 노드와 변경 후 노드 트리를 비고 비교하게 됩니다.  
문제는 실행된 dispatch함수가 bind로 인해 고정된 context를 가진다는 점입니다. 그렇기 때문에 **지금 이 호출이 어떤 노드를 변경하는지**에 대해서 분기처리를 해줄 필요가 있습니다.

```javascript
// reconciler > ReactFiberHooks.js
function dispatchAction(fiber, queue, action) {
  const alternate = fiber.alternate

  if (
    fiber === currentlyRenderingFiber ||
    (alternate !== null && alternate === currentlyRenderingFiber)
  ) {
    // Render phase update
  } else {
    // idle update
  }
}
```


## 유휴 상태에서의 dispatchAction

<br />

지금까지 내용으로 dispatchAction() 함수가 어떤 일을 하는지 정리해보겠습니다.

1. update 객체를 만든다.
2. 컴포넌트로부터 업데이트된 값을 queue에 저장한다.
3. 불필요한 렌더링을 방지한다.
4. UI 렌더링을 위해 Work를 스케쥴링한다. 

Work를 스케쥴링 할때, 이전의 상태와 같냐 같지않냐를 비교합니다.  
상태를 업데이트하는 이 시점에 expirationTime을 기록합니다.  

* `expirationTime`
  * dispatchAction()이 여러 번 호출되는 경우 매번 Work에 스케쥴링되지 않도록 막습니다.

## Render phase 에서의 updateAction 처리

<br />

상태가 변경됨에 따라 Work의 스케쥴링에 맞춰 render phase가 진행됩니다. 이때 dispatchAction()이 실행된다면 어떤 순서일까요?
이때는 렌더링 방지라던가, Work에 스케쥴링을 해줄 필요가 전혀 없습니다. 더 이상 변경이 없을 때 까지 상태 값만 변경해주면 됩니다.  
**다만 action의 소비를 위해 update 객체를 담아둘 저장소가 필요합니다.**

```javascript
reconciler > ReactFiberHooks.js

  
function dispatchAction(fiber, queue, action) {
  // const alternate = fiber.alternate
  if (...) {
    didScheduleRenderPhaseUpdate = true; // renderWithHooks()에게 컴포넌트 재실행을 알려줄 플래그

    const update = {
      expirationTime: renderExpirationTime,
      action,
      suspenseConfig: null,
      eagerReducer: null,
      eagerState: null,
      next: null,
    };

    if (renderPhaseUpdates === null) {
      renderPhaseUpdates = new Map(); // update 임시 저장소
    }

    const firstRenderPhaseUpdate = renderPhaseUpdates.get(queue);
    if (firstRenderPhaseUpdate === undefined) {
      renderPhaseUpdates.set(queue, update);
    } else {
      // Append the update to the end of the list.
      let lastRenderPhaseUpdate = firstRenderPhaseUpdate;
      while (lastRenderPhaseUpdate.next !== null) {
        lastRenderPhaseUpdate = lastRenderPhaseUpdate.next;
      }
      lastRenderPhaseUpdate.next = update;
    }
  } else {
    /*idle update..*/
  }
}
```

`didScheduleRenderPhaseUpdate`는 Render phase에서 update action이 일어났는지 확인하는 플래그로 사용됩니다.  
이 플래그는 `renderWithHooks`가 실행될 때 재호출 분기로 가게 돕는데요.

이때 최초 마운트될 때 `ReactCurrentDispatcher.current`에 심어뒀던 훅 구현체가 업데이트를 위한 구현체로 교체됩니다.  

```javascript
reconciler > ReactFiberHooks.js

  
export function renderWithHooks(...) {
  /*...*/
  if (didScheduleRenderPhaseUpdate) {
    do {
      didScheduleRenderPhaseUpdate = false
      // 무한 루프 방지와 업데이트 구현체에게 Render phase update를 알려주는 플래그
      numberOfReRenders += 1 

      //이하 훅 업데이트 구현체에서 Render phase update를 소비하는데 필요한 변수들을 설정
      nextCurrentHook = current !== null ? current.memoizedState : null
      nextWorkInProgressHook = firstWorkInProgressHook
      currentHook = null
      workInProgressHook = null

      ReactCurrentDispatcher.current = HooksDispatcherOnUpdate // 업데이트 구현체 주입

      children = Component(props, refOrContext) // 컴포넌트 재호출
    } while (didScheduleRenderPhaseUpdate)

    renderPhaseUpdates = null // Render phase update 저장소 초기화
    numberOfReRenders = 0
  }
  /*...*/
}
```

> 참고로 numberOfReRenders는 hook을 사용할 때 실수로 인해 확인할 수 있는 Too many re-renders. React limits the number of renders to prevent an infinite > loop.” 메세지에서 사용됩니다. (25회로 제한 되어있습니다.)

# Q&A

<br />

다음 질문에 대답할 수 있어야합니다.

**Q. setState()는 호출될 때마다 컴포넌트가 리렌더링 되나요?**
> 꼭 렌더링되는 건 아닙니다.  
> 만약 호출될 때 넣어준 값이 현재 상태와 동일하다면 내부 처리 분기에서 걸러집니다.  
> 만약 다르다면, 재호출 로직에 의해 한까번에 묶여서 한 번만 렌더링됩니다.  
> Work를 한 번만 스케쥴링 하고 그 사이에 일어난 모든 업데이트는 render phase에서 한 번에 소비됩니다.

**클릭을 통해 업데이트 된 상태를 기준으로 한 번 더 setState()시켜서 호출합니다. 이때에도 렌더링이 한 번 일어나나요?**
```jsx
//ex)
const FC = () => {
  const [state, setState] = useState(0);
  
  if (state === 1) setState(2);
  return <button onClick={setState((prev) => prev + 1)}>+</button>
}
```
> 네 if 문으로 실행된 업데이트는 클릭을 했을 때 발생한 업데이트 로직에서 호출됩니다.
> 재호출 시점에서 발생하는 여러 업데이트를 묶어 한번만 실행하게 됩니다.
> 여기서 추가 업데이트가 발생하지 않을 때까지 함수를 재실행하는데 최소 2번 최대 25번까지 재 실행될 수 있습니다.

**Q.재호출 로직이 브라우저 렌더링 리소스에 낭비를 일으킬 수 있나요?**
> 이 호출은 Render Phase에서 동작합니다.
> 아직 DOM이 반영된 것이 아니기 때문에 Virtual DOM에서 변경점을 적용하는 과정입니다.
> 실제 DOM의 조작은 Commut phase에서 진행되기 때문에 컴포넌트가 재호출된다고 하여도 브라우저 렌더링 측면에서는 리소스가 낭비되지 않습니다.

출처 | https://goidle.github.io/react/in-depth-react-hooks_1/