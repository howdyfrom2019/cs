#React 톺아보기

<br />

## 패키지 구조

<br />

리액트는 react 코어, renderer, Virtual Dom과 관련된 **reconciler**, 비동기 실행기인 **scheduler**와 **event**로 나눌 수 있습니다.  

### 1. React
* 컴포넌트 정의와 관련된 패키지.
* React Element를 만드는 createElement()와 개발자들에게 다른 패키지의 모듈을 제공하는 중간 다리 역할을 합니다.
* 다른 패키지의 의존성이 없어서 여러 플랫폼에서 사용할 수 있습니다.

### 2. renderer
* react-dom, react-native-renderer 등 호스트 렌더링에 의존적인 패키지.
* 호스트와 react를 연결하는 역할을 함.
* reconciler와 legacy-event 패키지에 의존성을 가짐.

### 3. scheduler
* SyntheticEvent라는 명칭으로 내부적으로 개발된 이벤트 시스템입니다.
* 개발자가 event를 사용하기 전 리액트에서 추가적인 제어를 하기위해 호스트 event를 wrapping합니다.
* 이벤트 풀링, 이벤트 위임 등을 사용하여 구현되어 있습니다.

### 4. reconciler
* 리액트의 핵심 패키지.
* v15 이전에는 스택 기반 구현에서 v16부터는 fiber architecture를 도입함.
