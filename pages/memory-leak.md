#메모리 누수
<br />

####Q. 웹 페이지가 멈췄다. 원인은 무엇일까?

* 클로저의 잘못된 사용
* 의도치않게 생성된 전역 변수
* 분리된 DOM 노드
* 콘솔 출력
* 해제하지 않은 타이머

## V8 엔진에서의 메모리
<br />

자바스크립트의 엔진은 **Memory Heap**과 **Call stack**으로 구성되어 있다.  
자바스크립트는 싱글 스레드를 가진다. (콜 스택이 하나임)

1. **Stack Memory** | 단순 변수(원시타입)을 저장.
2. **Heap Memory** | 복잡한 객체를 저장. 참조 데이터 타입이라고도 함.

<img src="../images/stack&heap%20memory.png" alt="memory diff">  

Heap Memory의 경우는 call stack에 참조 하는 주소를 넣고, 실제 데이터는 heap 영역에 삽입한다.

<br />

### 클로저의 잘못된 사용
```javascript
function fn1() {
    let a = new Array(10000);
    return a;
}

let res = [];

function myClick() {
    res.push(fn1());
}
```

버튼이 눌릴 때마다 res 배열에 새로운 10000크기만큼의 배열이 추가된다.  
콜 스택은 지워지지만, 힙 메모리가 가비지 콜렉팅되지 않기 때문에 점점 메모리가 늘어난다.  

### 의도치 않은 전역변수 사용.  

가비지 콜렉터는 전역변수는 제외되고 실행되기 때문에 메모리 공간에서 해제되지 않는다.
언제나 전역변수를 사용할 땐 유의해서 사용해야한다.

### 분리된 DOM 노드
```javascript
let btn = document.querySelector('button')
let child001 = document.querySelector('.child001')
let root = document.querySelector('#root')

btn.addEventListener('click', function() {
    root.removeChild(child001)
})
```

위 코드에서 버튼을 클릭할 때 dom 노드를 제거하고 가비지 컬렉팅 되기를 기대하지만,  
전역변수로 선언되어있는 child001 변수로 인해 메모리 해제가 안되기 때문에 실제로는 가비지 컬렉팅이 되지않는다.  
이 코드를 개선하기 위해서는 전역변수가 아닌 지역변수로 해당 변수를 옮겨줘야한다.

### 콘솔 출력

코딩 테스트 등 상황에서 알 수 있지만, 콘솔 출력은 꽤 비용이 큰 작업이다.
개발 환경일때 실행할 수 있도록 다음과 같이 작성해보자.

```javascript
if (process.env.iS_DEV) {
  console.log(obj);
}
```


### 해제되지 않는 반복문

**while 문** 등을 사용하거나, **setInterval** 등을 사용하는 경우에 escape 조건을 명확히 명시해야한다.  


<br />

자바스크립트 가비지 컬렉션은 자동으로 이루어지지만, 특정 변수들의 메모리를 수동으로 해제하는 일들이 필요하다.  
또, 반복문 등 상황에서 메인 스레드를 너무 오래 점유하고 있는 경우도 문제가 될 수 있다.