# 데이터 바인딩
<br />

화면상의 데이터(View)와 브라우저 메모리에 있는 데이터(Model)을 묶어서 데이터를 동기화하는 것을 의미한다.  
예를 들어, HTML에서 서버나 자바스크립트에서 받아온 데이터로 `동적으로 구성 요소를 변경`해주는 기능을 의미한다.

```tsx
import React, { useState, useCallback } from "pages/react/react";

interface DataType {
  id: number;
  title: string;
  url: string;
}

const DataBinding = () => {
  const [data, setData] = useState<DataType>({ id: 0, title: "", url: "" });

  const onHandleNewDataFetching = useCallback(() => {
    setData({
      id: 1,
      title: "구글 코리아",
      url: "www.google.co.kr"
    })
  }, []);

  return (
    <div>
      <span>{data.id}</span>
      <span>{data.title}</span>
      <img src={data.url} alt={"img"}/>
      <button onClick={onHandleNewDataFetching}>데이터 변경하기</button>
    </div>
  )
}

export default DataBinding;
```

위 코드는 `DataType`을 만족하는 값으로 `data` 상태를 변경해주는 기능을 수행한다.  
데이터 바인딩은 크게 두 가지 측면으로 나뉘고 그 안에서도 `컴포넌트 간`, `컴포넌트 내`에서 동작하는 기능이 나뉜다.  

* 단방향 데이터 바인딩(One-way Data Binding)
* 양방향 데이터 바인딩(Two-way Data Binding)

## 단방향 데이터 바인딩
<br />

### 1) 컴포넌트 내에서 `단방향 바인딩`
* Model(JavaScript)에서 View(HTML)로 데이터를 동기화하는 것을 의미한다.
* 단방향에서는 View(HTML)에서 Model(JavaScript)로 데이터를 직접적으로 갱신하는게 **불가능**하다.
  * 이벤트 함수(onClick, onChange 등)을 사용하여 함수를 호출하여 Model에서 View로 다시 변경해야한다.

### 2) 컴포넌트 간 `단방향 바인딩`
* 부모 컴포넌트에서 자식 컴포넌트로 데이터가 전달되는 구조를 의미한다.
* SPA 프레임워크의 React가 대표적인 예시이다.


## 양방향 데이터 바인딩
<br />

### 1) 컴포넌트 내에서 `양방향 바인딩`
* `Model(JavaScript)` - `ViewModel` - `View(HTML)`의 구조를 띈다.
* 중간의 ViewModel로 인해 하나로 묶여서 Model이나 View 중에 하나만 바뀌어도 함께 변경되는 것을 의미한다.

### 2) 컴포넌트 간 `양방향 바인딩`
* 부모 컴포넌트에서 자식 컴포넌트로는 Props를 통해 데이터를 전달하고, 자식 컴포넌트에서 부모 컴포넌트로 Emit Event를 통해서 데이터를 전달하는 구조이다.
* SPA 프레임워크의 Vue.js, Angular가 대표적인 예시이다.