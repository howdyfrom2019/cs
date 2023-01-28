# 비동기 호출

<br />

* Callback
* Promise
* async & await

## Callback

<br />

* 다른 함수가 실행을 끝낸 뒤 실행되는 함수를 말한다.
* 코드를 통해 명시적으로 호출하는 함수가 아니라 정의한 함수가 어떤 이벤트나 특정시점에서 실행되도록 시스템에서 호출하는 함수를 의미한다.
* 파라미터로 함수를 전달 받아 함수의 내부에서 실행된다.

### Q. 콜백함수를 쓰는 이유

> JS의 비동기적 프로그래밍을 활용.  
> 싱글스레드를 논블록킹으로 동작하게 하기 위해서  
> (이벤트 루프가 싱글스레드임.)

### 콜백 함수 사용할 때 주의점

* 콜백을 중첩하기 시작하면서 들여쓰기 수준이 감당하기 힘들어질 정도로 깊어짐.
* 이런 문제를 **콜백 지옥** 이라고 하고, `Promise`, `async/await` 등을 사용해 방지할 수 있다.

## async/await

<br />

* `비동기식 코드`를 `동기식`으로 표현하여 간단하게 나타냄.
* `async & await`는 `Promise` 객체를 반환하기 때문에 then을 사용할 수 있다.

## Promise vs Callback

<br />

* Promise 객체는 비동기 작업이 어떤 결과를 반환하냐에 따른 값을 콜백 형태로 구성합니다.
* `return value`를 사용할 수 있다.
* `error handling(try, catch)` 방식이 동기식과 유사함.
* `대기(pending)`, `이행(fulfilled)`, `거부(rejected)` 단계로 나뉨.

```javascript
function getData() {
  return new Promise((resolve, reject) => {
    const data = 100;
    return data;
  });

  getData()
    .then((resolve) => {
      console.log(resolve);
    })
    .catch((reject) => {
      console.log(reject);
    })
}

Promise.all([getData, getData]).then((val) => //...)
```