# Scheduler

<br />

`reconciler`는 비동기 호출이 필요한 **Work**의 실행 제어권을 **scheduler**에게 위임합니다.  
`reconciler`는 VDOM 재조정 작업 전에 설정해줘야 하는 부분을 처리하고, **scheduler**는 스케줄링된 태스크 중에서 우선순위 반영하고, 적절한 시기에 작업의 실행과 중단을 조정합니다.  

**scheduler**는 host config라는 모듈이 존재합니다.  
비동기 api, performance, isInputPending 등 메소드가 존재합니다.

## isInputPending
> isInputPending API는 브라우저가 메인 스레드 점유가 필요할 때 작업을 중단하고 위임할 수 있습니다.  
`concorruent mode`에서 Render phase가 진행 중일 때 UIEvent가 발생하거나 브라우저의 렌더링 작업의 방해를 막지 않게 하기 위해 적절히 콜 스택을 비워줬었습니다.

## 1. dispatchAction

<br />

hooks의 상태 변경을 돕는 dispatchAction에서 scheduling도 같이 처리합니다.
```javascript
reconciler > ReactFiberHooks.js

  
function dispatchAction(...) {
  if (...) { 
    /* Render phase update... */
  } else { 
    /* idle update... */
    scheduleWork(fiber, expirationTime);
  }
}
```

## scheduleWork

<br />

포워딩 된 해당 함수에서는 **Work**를 스케줄링 하기 전에 아래와 같은 일들을 부가적으로 처리해야합니다.

1. 상태가 변경된 해당 컴포넌트에서 이벤트가 발생했음을 `expirationTime`을 새깁니다.
2. 이벤트가 발생한 컴포넌트의 VDOM `root`를 가지고 옵니다.
3. root에 스케줄 정보를 기록합니다.

### root는 무엇을 의미할까요?

각 컴포넌트에서 영역별로 `root`가 생성됩니다. 그리고 root 마다 VDOM이 같이 생성됩니다.  
이때 요청이 들어온 **Work**와 이미 스케줄링된 **Work** 사이에 교통정리가 필요합니다. 이 역할을 reconciler가 해주며, 그 기준이 바로 root에 새겨진 스케줄 정보입니다.

## expirationTime 새기기

<br />

```javascript
   
export function scheduleUpdateOnFiber(fiber, expirationTime) {
  const root = markUpdateTimeFromFiberToRoot(fiber, expirationTime)
}

export const scheduleWork = scheduleUpdateOnFiber;
```

업데이트가 발생한 fiber에 expirationTime을 새깁니다. 그리고 root를 찾기 위해 위쪽 노드로 찾아 올라갑니다.  
이때 거쳐가는 fiber(노드)에도 시간을 새깁니다. expirationTime이 아닌 이벤트가 발생함을 알려주는 childExpirationTime을 새깁니다.  
*(이는 재조정 작업에서 매우 중요하게 사용됩니다)*

```javascript
reconciler > ReactFiberWorkLoop.js

  
function markUpdateTimeFromFiberToRoot(fiber, expirationTime) {
  // expirationTime을 새긴다.
  if (fiber.expirationTime < expirationTime) {
    fiber.expirationTime = expirationTime
  }
  let alternate = fiber.alternate
  if (alternate !== null && alternate.expirationTime < expirationTime) {
    alternate.expirationTime = expirationTime
  }

  // root를 찾는다.
  let node = fiber.return
  let root = null
  if (node === null && fiber.tag === HostRoot) {
    root = fiber.stateNode // Host root의 stateNode가 root이다.
  } else {
    while (node !== null) {
      alternate = node.alternate
      // childExpirationTime를 새긴다.
      if (node.childExpirationTime < expirationTime) {
        node.childExpirationTime = expirationTime
        if (
          alternate !== null &&
          alternate.childExpirationTime < expirationTime
        ) {
          alternate.childExpirationTime = expirationTime
        }
      } else if (
        alternate !== null &&
        alternate.childExpirationTime < expirationTime
      ) {
        alternate.childExpirationTime = expirationTime
      }

      if (node.return === null && node.tag === HostRoot) {
        root = node.stateNode
        break
      }
      node = node.return
    }
  }

  return root
}
```

이때 alternate와 node에 동시에 expirationTime과 childExpirationTime을 새깁니다.  
`fiber`를 bind로 묶어서 고정해놨기 때문에 작업중인 노드가 current인지 workInProgress인지 모르기 때문입니다.

## 스케줄링 요청 전 Work를 동기적으로 처리 확인하기

<br />

```javascript
reconciler > ReactFiberWorkLoop.js

  
export function scheduleUpdateOnFiber(fiber, expirationTime) { // scheduleWork()
  // const root = markUpdateTimeFromFiberToRoot(fiber, expirationTime);

  if (expirationTime === Sync) {
    // VDOM 첫 생성 판단
    if (
      (executionContext & LegacyUnbatchedContext) !== NoContext &&
      (executionContext & (RenderContext | CommitContext)) === NoContext
    ) {
      performSyncWorkOnRoot(root);
    } else {
      ensureRootIsScheduled(root);
      if (executionContext === NoContext) {
        flushSyncCallbackQueue();
      }
    }
  } else {
    ensureRootIsScheduled(root);
  }

  /*...*/
}
```

1. [Sync] 먼저 **Work**를 동기적으로 호출해야하는지 판단합니다. 대기 expirationTime은 Sync입니다.
   * [최초 생성 시] VDOM이 처음 생길 때 충족 됨. 이때 Work를 바로 진행하여 VDOM을 빠르게 완성함. `performSyncWorkOnRoot()`
   * [재조정 작업 시] `ensureRootIsScheduled()` 를 통해 재조정 작업이 필요한 root에 Work를 스케줄링 요청하고 정보를 새김. reconciler가 비었을 때는 `flushSyncCallbackQueue()`를 사용해 sync queue에 담겨있는 Work를 바로 실행함.
2. [Async] Work 스케줄링을 요청합니다. 비동기 전용 Work가 스케줄링 됩니다.

스케줄링과 관련된 코드는 `ensureRootIsScheduled(root)`에서 실행되고, 위 함수에서는 스케줄 요청 전 후로 reconciler 입장에서의 추가 작업이 위치할 수 있는 함수입니다.

## ensureRootIsScheduled

<br />

### scheduler에게 Work를 전달하기 전 사전 준비

<br />

Work의 본격적인 스케줄링 전에 사전 설정해줄 정보들이 있습니다. 새로 요청하는 작업의 우선순위 결정과 expirationTime 등에 필요한 스케줄 정보입니다.

```javascript
reconciler > ReactFiberWorkLoop.js

  
function ensureRootIsScheduled(root) {
  /*...*/
  const existingCallbackNode = root.callbackNode // 스케줄링되어 있는 Task 객체
  const expirationTime = getNextRootExpirationTimeToWorkOn(root)
  const currentTime = requestCurrentTimeForUpdate()
  const priorityLevel = inferPriorityFromExpirationTime(
    currentTime,
    expirationTime
  )
  /*...*/
}
```

```javascript
  
const HIGH_PRIORITY_EXPIRATION = __DEV__ ? 500 : 150
const HIGH_PRIORITY_BATCH_SIZE = 100
const LOW_PRIORITY_EXPIRATION = 5000
const LOW_PRIORITY_BATCH_SIZE = 250

function inferPriorityFromExpirationTime(
  currentTime: ExpirationTime,
  expirationTime: ExpirationTime
): ReactPriorityLevel {
  if (expirationTime === Sync) {
    return ImmediatePriority
  }
  if (expirationTime === Never || expirationTime === Idle) {
    return IdlePriority
  }

  const msUntil =
    expirationTimeToMs(expirationTime) - expirationTimeToMs(currentTime)

  if (msUntil <= 0) {
    return ImmediatePriority
  }

  if (msUntil <= HIGH_PRIORITY_EXPIRATION + HIGH_PRIORITY_BATCH_SIZE) {
    return UserBlockingPriority
  }
  if (msUntil <= LOW_PRIORITY_EXPIRATION + LOW_PRIORITY_BATCH_SIZE) {
    return NormalPriority
  }

  return IdlePriority
}
```

`msUntil`은 우리가 일반적으로 생각하는 시간입니다. 낮을 수록 바로 시작해야하는 작업이고, 클 수록 우선순위가 밀리는 작업입니다.  
우선순위가 밀리는 작업은 얼마나 여유있는지를 확인하고 해당하는 우선순위를 반환합니다.

### 교통정리 하기

<br />

스케줄링에 필요한 정보를 모두 구했으니 root에 Work가 예약되어 있는지 확인합니다. 예약되어 있다면 우선순위를 따져 기존 Work를 취소할 지 결정합니다.

```javascript
function ensureRootIsScheduled(root) {
  /*...*/
  /* const priorityLevel = ...; */

  if (existingCallbackNode !== null) {
    const existingCallbackPriority = root.callbackPriority
    const existingCallbackExpirationTime = root.callbackExpirationTime
    if (
      existingCallbackExpirationTime === expirationTime &&
      existingCallbackPriority >= priorityLevel
    ) {
      // Existing callback is sufficient.
      return
    }
    cancelCallback(existingCallbackNode) // scheduler에게 취소 요청
  }
  /*...*/
}
```

### root에 스케줄링 정보 새기기

<br />

Work의 스케줄을 요청한 뒤 root에 정보를 새깁니다. 역시 Sync 옵션을 따지는데 `scheduleUpdateOnFiber()`에서는 **Work를 동기 호출** 하기 위한 분기 확인이었고 `ensureRootIsScheduled()`는 **Work의 진행을 동기적으로 처리**해야하는지에 대한 분기를 의미합니다.

```javascript
reconciler > ReactFiberWorkLoop.js

  
function ensureRootIsScheduled(root) {
  /*...*/
  /* if (existingCallbackNode !== null) { ... } */

  root.callbackExpirationTime = expirationTime
  root.callbackPriority = priorityLevel

  let callbackNode
  if (expirationTime === Sync) {
    // Sync React callbacks are scheduled on a special internal queue
    callbackNode = scheduleSyncCallback(performSyncWorkOnRoot.bind(null, root))
  } else {
    callbackNode = scheduleCallback(
      priorityLevel,
      performConcurrentWorkOnRoot.bind(null, root),
      // Compute a task timeout based on the expiration time. This also affects
      // ordering because tasks are processed in timeout order.
      { timeout: expirationTimeToMs(expirationTime) - now() }
    )
  }

  root.callbackNode = callbackNode // Task를 root에 저장한다.
}
```

여기에서 주목할 점은 계속 어떤 객체에 함수나 객체를 포워딩해주는 처리만 되고 실제로 스케줄링의 처리와 관련된 코드들이 등장하지 않았다는 점입니다.  
리액트 팀은 reconciler와 scheduler 사이의 의존도를 낮추기 위해 따로 모듈을 두었습니다. `scheduleSyncCallback()` 과 `scheduleCallback()`으로 분리되어 있습니다.

## 비동기 스케줄링 & 동기 스케줄링 처리

<br />

### scheduleCallback

<br />

```javascript
  
import * as Scheduler from 'scheduler';
const { unstable_scheduleCallback: Scheduler_scheduleCallback } = Scheduler;

function scheduleCallback(
  reactPriorityLevel: ReactPriorityLevel,
  callback: SchedulerCallback,
  options: SchedulerCallbackOptions | void | null
) {
  const priorityLevel = reactPriorityToSchedulerPriority(reactPriorityLevel)
  return Scheduler_scheduleCallback(priorityLevel, callback, options)
}
```

독립적으로 작동하는 reconciler와 scheduler 간의 우선순위가 다르기 때문에 콜백 전에 따로 우선순위를 반환합니다.

### scheduleSyncCallback()

<br />

```javascript
  
function scheduleSyncCallback(callback: SchedulerCallback) {
  // Push this callback into an internal queue. We'll flush these either in
  // the next tick, or earlier if something calls `flushSyncCallbackQueue`.
  if (syncQueue === null) {
    syncQueue = [callback]
    // Flush the queue in the next tick, at the earliest.
    immediateQueueCallbackNode = Scheduler_scheduleCallback(
      Scheduler_ImmediatePriority,
      flushSyncCallbackQueueImpl
    )
  } else {
    // Push onto existing queue. Don't need to schedule a callback because
    // we already scheduled one when we created the queue.
    syncQueue.push(callback)
  }
  return fakeCallbackNode // Task는 없으니 가짜를 반환
}
```

동기 호출의 경우는 내부 큐에 쌓아 둡니다. 이 큐를 처리하는 함수인 `flushSyncCallbackQueueImpl()`을 스케줄링 하여 처리합니다. 만약 해당 작업이 우선순위에서 밀려 큐를 다 비우지 못했다면 다음 tick에서 scheduler에 의해 처리됩니다.

## flushSyncCallbackQueueImpl()

<br />

```javascript
  
function flushSyncCallbackQueueImpl() {
  if (!isFlushingSyncQueue && syncQueue !== null) {
    // 중복 실행 방지 플래그
    isFlushingSyncQueue = true
    let i = 0
    try {
      const isSync = true
      const queue = syncQueue
      runWithPriority(ImmediatePriority, () => {
        for (; i < queue.length; i++) {
          let callback = queue[i]
          do {
            callback = callback(isSync) // performSyncWorkOnRoot()
          } while (callback !== null)
        }
      })
      syncQueue = null
    } catch (error) {
      // 에러가 발생한 callback만 버린다.
      if (syncQueue !== null) {
        syncQueue = syncQueue.slice(i + 1)
      }
      // Resume flushing in the next tick
      Scheduler_scheduleCallback(
        Scheduler_ImmediatePriority,
        flushSyncCallbackQueue
      )
      throw error
    } finally {
      isFlushingSyncQueue = false
    }
  }
}
```

큐를 할당하고 처리하는 과정에서 `runWithPriority`를 사용하여 콜백을 넣어주는 걸 확인할 수 있습니다.
이 함수는 scheduler에게 해당 함수가 어느정도의 우선순위가 있는지를 알려주기 위해 사용합니다.

```javascript
  
function unstable_runWithPriority(priorityLevel, eventHandler) {
  switch (priorityLevel) {
    case ImmediatePriority:
    case UserBlockingPriority:
    case NormalPriority:
    case LowPriority:
    case IdlePriority:
      break;
    default:
      priorityLevel = NormalPriority;
  }

  var previousPriorityLevel = currentPriorityLevel;
  currentPriorityLevel = priorityLevel;

  try {
    return eventHandler();
  } finally {
    currentPriorityLevel = previousPriorityLevel;
  }
}
```

이렇게 Work의 모든 작업을 scheduler가 관리하고 reconciler와 독립적으로 우선순위를 매겨 작업을 관리하는 과정을 간략히 살펴볼 수 있었습니다.  

reconciler는 현재 진행되는 Work 관련 추가 작업이 필요할 때 scheduler의 컨텍스트 우선순위만 참고하면 됩니다. 일반 큐처럼 순서대로 넣어지는게 아니라 우선순위에 따라 언제든 중지되고 재실행될 수 있습니다. 

**출처 | https://goidle.github.io/react/in-depth-react-scheduler_1/**