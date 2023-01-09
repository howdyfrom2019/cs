# 상속 vs 합성

<br />

코드의 재사용성을 바탕으로 확장을 용이하게 만드는 두 가지 방법.

<table>
    <thead>
        <th></th>
        <th>상속(Inheritance)</th>
        <th>합성(Composition)</th>
    </thead>
    <tbody>
        <tr>
            <td>의존성이 해결되는 시점</td>
            <td>부모 - 자식 간 의존성은 컴파일 타임에 해결</td>
            <td>두 객체간 의존성은 런타임에 해결</td>
        </tr>
        <tr>
            <td>관계성</td>
            <td>is-a 관계</td>
            <td>has-a 관계</td>
        </tr>
        <tr>
            <td>클래스 간 결합 정도</td>
            <td>부모의 구현에 의존 결합도 높음</td>
            <td>구현에 의존하지 않고 인터페이스에 의존</td>
        </tr>
        <tr>
            <td>객체간 관계성</td>
            <td>클래스 사이의 정적인 관계</td>
            <td>객체 간 동적인 관계</td>
        </tr>
        <tr>
            <td>재사용성</td>
            <td>부모 클래스에서 물려받음</td>
            <td>포함되는 객체의 인터페이스를 재사용.</td>
        </tr>
    </tbody>
</table>

## 상속

<br />

부모 클래스를 포괄적으로 정의하고, 자식 클래스에서 보다 자세하게 확장함.  
> class Person => class Student / class Teacher 등

* 상속을 제대로 활용하기 위해서는 부모 클래스의 구현을 상세히 알아야해서 `결합도가 높아짐`.
* 상속 관계는 `컴파일 타임`에 결정되며, 당연히 정적으로 결정됨.
* 여러 조합을 이용해야하는 경우에는 조합 별로 클래스를 넣어줘야함. (클래스 폭발 문제)
  * ex) 햄버거 => 치즈버거/빅맥/.......

> 상속은 클래스 간 관계를 파악하기 쉽고 재사용할 수 있는 쉬운 방법인건 맞지만 가장 효율적인 방법은 아니다.


## 합성

<br />

상속으로의 확장 대신, 클래스의 인스턴스를 참조하게 하는 설계이다.  
> class Car <=> class Engine / class Student <=> class Subject

```javascript
class Student {
  constructor(subject) {
    this.subject = subject;
  }
  
  study() {
    console.log(`${this.subject.name} 공부 중..`);
  }
  
  drop() {
    console.log(`성적: ${this.subject.score}점`);
  }
}

class Subject {
  constructor(name, score) {
    this.name = name;
    this.score = score;
  }
}

const student = new Student(new Subject("수학", 90));
```

이런 방식을 다른 말로 `포워딩(forwarding)`이라고도 하며,  
인스턴스를 참조하는 메소드를 `포워딩 메소드(forwarding method)`라고도 부른다.