#item 10 equals는 일반 규약을 지켜 재정의하라

---

#### equals의 재정의는 쉬워보이지만 잘못할 경우 끔찍한 결과를 초래한다.
#### 가장 쉬운 길은 재정의하지 않는 것이다.
#### 다음 4가지 상황에서는 재정의하지 않는 것이 최선이다.  
- 각 인스턴스가 본질적으로 고유한 경우
  - 값을 표현하는 것이 아닌 동작하는 개체를 표현하는 클래스   
    (ex. Thread)
- 인스턴스의 '논리적 동치성(local equality)'를 검사할 일이 없는 경우
  - 논리적 동치성이 필요치 않은 경우 Object의 기본 equals 만으로 해결이 된다.
- 상위 클래스에서 재정의한 equals가 하위 클래스에도 딱 들어맞는 경우
  - 상속 받은 구현체들을 확인했을 때 같은 방식으로도 된다면 필요없음
    (ex. Set 구현체 <- AbstractSet의 equals를 상속 받아서 사용, Map 구현체 <- AbstractMap의 equals를 상속 받아서 사용)
- 클래스가 private이거나 package-private이고 equals 메서드를 호출할 일이 없는 경우
  - 이런 경우에 override 해서 exception을 던져주면 위험을 철저히 회피할 수 있음
  ```
  @Override
  public boolean equals(Object o) {
    throw new AssertionError(); // 호출 금지!
  }
  ```

#### 재정의 해야할 때는 언제 일까?
- 객체 식별성이 아니라 논리적 동치성을 확인해야 할 때(상위 클래스의 equals가 논리적 동치성을 비교하도록 재정의도 되지 않았고)  
  - ex. 값 객체의 경우 참조 주소보다는 값이 같은 지 알고 싶어 할 것
    - 이 경우 재정의 후에 Map key와 Set의 원소로 사용할 수 있음
- 값 클래스라고 해도 값이 같은 인스턴스가 둘 이상 만들어지지 않음을 보장하는 인스턴스 통제 클래스라면 equals를 재정의 안 해도 됨   
  - ex. Enum class // TODO : ( 인스턴스가 둘 이상 어떻게 안 만들어 지는 지 링크 추가해야 함.)
    - 이 경우 어짜피 논리적으로 같은 인스턴스가 2개 이상 만들어지지 않으니 사실상 객체 식별성 = 논리적 동치성 의 의미를 가지게 된다.

---


#### 동치관계와 equals
- 동치관계란?
  - 집합을 **서로 같은 원소들로 이뤄진 부분집합(동치류, 동치 클래스 equivalence class)** 으로 나누는 연산
- equals 메서드가 쓸모 있으려면 모든 원소가 같은 동치류에 속한 어떤 원소와도 서로 교환이 가능해야 한다.
  - a class 내부 [1,2,3,4,5] b 내부 [1,2,3,4,5] -> 여기서 1,2,3,4,5 모든 원소가 교환이 가능해야 equals가 쓸모 있게 됨

#### equals 재정의 시 일반 규약 (동치관계를 만족시키기 위한 5가지 요건)
- 반사성
  - null 아닌 모든 참조 값 x에 대해 x.equals(x) 는 true 다.
    - x = x => true
- 대칭성
  - null 아닌 모든 참조 값 x, y에 대해 x.equals(y)가 true 라면 y.equals(x)도 true 다.
    - x = y if true, y = x => true
- 추이성
  - null 아닌 모든 참조 값 x, y, z에 대해 x.equals(y)가 true, y.equals(z)도 true 라면 x.equals(z)도 true 다.
    - x = y && y = z if true, x = z => true
- 일관성
  - null 아닌 모든 참조 값 x, y에 대해 x.equals(y)를 반복호출해도 항상 true or false
    - x = y if true always true
- null 아님
  - null 아닌 모든 참조 값 x에 대해 x.equals(null) 는 false 다.
    - x != null

#### equals 일반 규약을 어길 경우
- 원인 코드 파악이 어려워짐
- 세상에 홀로 존재하는 클래스는 없다 by John Donne
  - 따라서 한 클래스의 인스턴스가 다른 곳으로 빈번히 전달될 것임(equals 규약을 어길 경우 다른 곳에 영향이 갈 수 있음을 표현하려고 한 것 같음)

