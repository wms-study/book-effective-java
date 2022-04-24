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
  - 따라서 한 클래스의 인스턴스가 다른 곳으로 빈번히 전달될 것임
    - 해당 객체를 사용하는 다른 객체들이 어떻게 반응할 지 알 수 없음
  
#### 일반 규약 상세 내용
- 대칭성
  ```
  public final class CaseInsensitiveString {
    private final String s;
    public CaseInsensitiveString(String s) {
      this.s = Objects.requireNonNull(s);
    }
    
    @Override public boolean equals(Object o) {
      if (o instanceof CaseInsensitiveString)
        return s.equalsIgnoreCase((CaseInsensitiveString) o).s;
      if (o instanceof String)
        return s.qualsIgnoreCase((String) o);
      return false
    }
  }
  
  CaseInsensitiveString cis = new CaseInsensitiveString("Ex1");
  String s = "ex1";
  
  cis.equals(s); // true
  s.equals(cis); // false 
  // 대칭성 위배
  
  List<CaseInsensitiveString> list = new ArrayList<>();
  list.add(cis);
  list.contains(s); // true를 반환할 가능성이 있음 -> 반응을 알 수 없음
  ```
  
  변해야 할 결과 (String & CaseInsensitiveString 은 동치관계가 불가능하다고 생각하고 버려야 함)

  ```
  public final class CaseInsensitiveString {
    private final String s;
    public CaseInsensitiveString(String s) {
      this.s = Objects.requireNonNull(s);
    }
    
    @Override public boolean equals(Object o) {
        return o instanceof CaseInsensitiveString &&
          ((CaseInsensitiveString) o).s.equalsIgnoreCase(s);
    }
  }
  ```
- 추이성
  - 구체 클래스를 확장해 새로운 값을 추가하면서 equals 규약을 만족시킬 방법은 존재하지 않음
  - 리스코프 치환 원칙에 따르면, 어떤 타입의 모든 메서드가 하위 타입에서도 똑같이 잘 작동해야 한다.
    - point 클래스를 상속받은 어떠한 클래스든 point로써 활용될 수 있어야 한다.
      - but getClass 검사로 하는 경우는 상위 클래스의 point와 상속받은 클래스의 클래스가 다르기 때문에 equals를 항상 false를 반환한다.
        - instanceof 로 하면 true 반환 가능하다.

  - 상속을 이용한 값 추가는 아니지만 컴포지션을 이용한다면 만족시킬 방법이 존재하게 된다.
  ```
  public class ColorPoint {
    private final Point p;
    private final Color c;
    public ColorPoint(int x, int y, Color color) {
      point = new Point(x, y);
      this.color = Objects.requeireNonNull(color);
    }
  
    /*
      point 뷰를 반환
    */
    public POint asPoint() {
      return point;
    }
    
    @Override public boolean equals(Object o) {
      if(!(o instanceof ColorPoint))
          return false;
      ColorPoint cp = (ColorPoint) o;
      return cp.point.equals(point) && cp.color.equals(color); // point를 바로 비교
    }
  }
  ```
  
  - java.sql.Timestamp는 대표적으로 java.util.Date를 확장한 후 nanoseconds를 추가했기 때문에 대칭성을 위배하고 있다.
    - 따라서 equals를 만족하지 않는 경우가 있다. 절대 따라해서는 안된다.
  
- 일관성
  - 두 객체가 같다면 영원히 같아야 한다는 뜻
    - 가변 객체의 경우
      - 비교 시점에 따라 같을수도 다를수도 있음
    - 불변 객체의 경우
      - 한번 다르면 끝까지 달라야 한다.
  - 불변 클래스의 경우 불변 클래스로 만들 지 심사숙고해야 함
    - equals가 한번 같다고 한 객체는 영원히 같다고 답하고, 다르다고 한 객체는 영원히 다르다고 답해야 하기 때문
  - 신뢰할 수 없는 자원이 끼어들게 해서는 안 됨
    - 유동ip와 같은 요소들이 equals에 들어가게 되면 같은 URL로 요청하는 것이라도 일관성을 유지할 수 없게 됨
  
- null 아님
  - 묵시적 null 검사가 더 나음
  ```
    // 명시적 null 검사
    @Override public boolean equals(Object o) {
      if(o == null)
        return false;
    }
  
    // 묵시적 null 검사 -> 이 쪽이 더 나음 instanceof 는 null을 자동으로 false로 반환 해 줌
    @Override public boolean equals(Object o) {
      if(!(o instanceof ColorPoint))
          return false;
    }
  ```

  