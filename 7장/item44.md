# item 44 표준 함수형 인터페이스를 사용하라

---

### 함수 객체를 매개변수로 받는 생성자와 메서드를 많이 만들어야 한다. 

- 이 때 함수형 매개변수 타입을 올바르게 선택해야 한다.

- LinkedHashMap에서 removeEldestEntry를 재정의하면 캐시로 사용할 수 있다.
    ```java
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > 100;
    }
    ```
    - 위와 같이 재정의하면 100개가 될 때까지 커지다가 그 이상이 되면 키가 더해질 때마다 가장 오래된 원소를 하나씩 제거한다.
    - 즉, 가장 최근 원소 100개를 유지한다.
    
- 람다를 이용하면 더 잘해낼 수 있다.
    - 오늘날 다시구현한다면 함수 객체를 받는 정적 팩터리나 생성자를 제공했을 것이다.
    ```java
    @FunctionalInterface
    interface EldestEntryRemovalFunction<K, V> {
      boolean remove(Map<K, V> map, Map.Entry<K, V> eldest);
    }
    ```
  
  - 필요한 용도에 맞는게 있다면 직접 구현하지 말고 표준 함수형 인터페이스를 활용하라. 
    (위의 EldestEntryRemovalFunction java.util.function 패키지에 이미 존재)
    ```java
    BiPredicate<Map<K, V>, Map.Entry<K, V>>
    ```
  
  - 표준 함수형 인터페이스들은 디폴트 메서드들을 많이 제공하므로 다른 코드와의 상호운용성도 크게 좋아질 것이다.
  

### 기억하면 좋을 것 같은 표준 함수형 인터페이스

- Operator 인터페이스 
  - 반환값과 인수의 타입이 같은 함수를 뜻한다.
    - 인수가 1개인 UnaryOperator
    - 인수가 2개인 BinaryOperator
- Predicate 인터페이스는 인수 하나를 받아 boolean을 반환하는 함수
- Function 인터페이스는 인수와 반환 타입이 다른 함수
- Supplier 인터페이스는 인수를 받지 않고 값을 반환하는 함수
- Consumer 인터페이스는 인수를 하나 받고 반환값은 없는 (인수를 소비하는) 함수를 뜻한다.

- 그림 참고 (p264)

- 기본 인터페이스는 기본 타입인 int, long, double용으로 3개씩 변형이 생겨남
  - ex. IntPredicate, LongBinaryOperator
- Function의 변형만 매개변수화 됨
    LongFunction<int[]> long 인수를받아 int[]를 반환
  
- 입력과 결과 타입이 모두 기본타입이면 접두어로 SrcToResult를 하용한다. 
    long을 받아 int를 반환하면 **LongToIntFunction이** 되는 식
- 입력을 매개변수화하고 접두어로 ToResult를 사용
    ToLongFunction<int[]> 은 int[] 인수를 받아 long을 반환
  
남은 부분은 추가로 참고 p265 BiPredicate, BiFunction, BiConsumer 등등

- BooleanSupplier 인터페이스는 boolean 을 반환하도록 한 supplier의 변형
  - 표준 함수형 인터페이스 중 boolean을 이름에 명시한 유일한 인터페이스
  
- 표준 함수형 인터페이스의 대부분은 기본 타입만 지원한다.
  - 그렇다고 기본 함수형 인터페이스에 박싱된 기본 타입을 넣어 사용하지는 말자. (note!)
  동작은 하지만 박싱된 기본 타입 대신 기본 타입을 사용하라 라는 조언을 위배한다.
      - 박싱된 기본 타입을 넣을 시 계산량이 많을 때는 성능이 처참히 느려질 수 있다.
  
- 표준 함수형 인터페이스없이 직접 사용해야할 때는 인터페이스 중 필요한 용도에 맞는 것이 없는 경우다.
- 인터페이스 중 필요한 용도에 존재하더라도 따로 사용하고 있는 경우 : Comparator
  - API에서 굉장히 자주 사용되는데 지금의 이름이 용도를 아주 훌륭히 설명한다.
  - 구현하는 쪽에서 반드시 지켜야 할 규약을 담고 있다.
  - 비교자들을 변환하고 조합해주는 디폴트 메서드들을 듬뿍 담고 있다.
  
### 전용 함수형 인터페이스 구현 고려 

- 세가지 중 하나 이상을 만족한다면 고민
  - 자주 쓰이고, 이름 자체가 용도를 명확히 설명한다.
  - 반드시 따라야 할 규약이 있다.
  - 유용한 디폴트 메서드를 제공할 수 있다.
  
- 주의해서 설계해야 함
  - 왜냐하면 interface를 설계하는 것이기 때문
  
- 직접 만든 함수형 인터페이스에는 항상 @FunctionalInterface 애너테이션을 사용하라.
  - 유용한 점
    - 코드나 설명 문서를 읽을 이에게 람다용으로 설계된 것임을 알려준다.
    - 인터페이스가 추상 메서드 하나만 가지고 있어야 컴파일되게 확인해준다.
    - 유지보수 과정에서 누군가 실수로 메서드를 추가하지 못하게 막아준다.
  
### 주의사항

- 서로 다른 함수형 인터페이스를 같은 위치의 인수로 받는 메서드들을 다중 정의해서는 안된다.
  - 클라이언트에게 모호함만을 안겨줄 뿐이고 이로 인해 문제가 될 수 있다. (예제 찾기)
  
- 서로 다른 함수형 인터페이스를 같은 위치의 인수로 사용하는 다중정의를 피하는 것이다.

우리 예제
```java
// inventory
UnaryOperator -> StandardDateType 
```
```java
@FunctionalInterface
public interface Action<V extends Vo> {
    // 메서드가 1개만 존재!
    void action(Event event, State state, V vo);
    
    default Action<V> andThen(Action<V> after) {
        return (event, resultState, vo) -> {
          action(event, resultState, vo);
          after.action(event, resultState, vo);
        };
    }
}
```