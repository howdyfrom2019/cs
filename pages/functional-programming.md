#함수형 프로그래밍
<br />

선언형 프로그래밍, **무엇을(what)** 의 관점에서 프로그래밍 하는 것.
```javascript
// 명령형 프로그래밍의 예
function makeMonthArrWithOrder() {
  const months = new Array(12);
  for (let i = 1; i <= 12; i++) {
    months.push(i);
  }
}

// 선언형 프로그래밍의 예
function makeMonthArrWithFC() {
  return Array.from({ length: 12 }, (_, i) => i + 1);
}
```
<br />

##함수형 프로그래밍의 특징
<br />

* **불변성(Immutability)**
* **First-class, higher-order functions**
* **Lazy evaluation**


### 불변성

함수형 프로그래밍에서는 오직 입력값에만 영향을 받기 때문에 프로그램을 예측하기가 쉽다.  
이를 바탕으로 쉽게 최적화가 가능하다.  

* 이전 함수의 값을 상태로써 사용할 수 있다.
* 멀티 스레드 환경에서 자원공유 문제에서 자유로워질 수 있다!

### First-class, Higher-order-functions

함수가 객체와 같은 1등 시민으로써 취급되기 때문에 핵심코드를 간결히 표현 가능하다.  
고차함수(map, filter, reduce 등)이나 커링(currying)을 가능하게 하는 중요한 요소이다.

```javascript
// 인수로 함수를 가질 수 있다. 꼭 객체처럼
const nextArr = arr.map((v, i, arr) => {
  //....
})
```

### Lazy evaluation

지연 연산은 실제로 값이 쓰이기 전까지 계산을 미뤄두는 것이다.  
미리 계산해서 저장하지 않기 때문에 공간을 절약할 수 있기 때문에 프로그램 성능에 좋은 영향을 준다.  
주로 메모이제이션과 함께 사용되어 필요한 계산을 수행하고 캐싱한 값을 재사용한다.
<br />
