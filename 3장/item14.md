#item 10 Comparable을 구현할 지 고려하라

---

#### 서론

- comparedTo는 equals, hashcode, clone들과 다르게 Object의 메서드가 아님
    - Comparable 인터페이스의 유일무이 메서드
- 성격은 거의 equals와 비슷하지만 compareTo와 다른 점이 2가지 있음
    - 단순 동치성 비교에 더해 순서까지 비교 가능
    - 제네릭
- Comparable을 구현했다는 것은 인스턴스들간에 자연적 순서가 있음을 뜻함
    - String 클래스는 Comparable을 구현하고 있음
- 구현 후 많은 [제네릭 알고리즘](http://ycpcs.github.io/cs201-fall2015/lectures/lecture15.html) 과 컬렉션의 힘을 누릴 수 있음
- 사실상 자바 플랫폼 라이브러리의 모든 값 클래스와 열거타입이 Comparable을 구현
    - 알파벳, 숫자, 연대 같이 순서가 명확한 값 클래스를 작성한다면 반드시 Comparable 인터페이스를 구현하자.

---

#### 모양

```
public interface Comparable<T> {
    public int compareTo(T o);
}
```

---

#### compareTo 메서드의 일반 규약

- 이 객체와 주어진 객체의 순서를 비교
    - 이 객체가 주어진 객체보다 작으면 음의 정수를, 같으면 0을, 크면 양의 정수를 반환
    - 비교할 수 없다면 ClassCastException을 던짐
- sgn(표현식) 표기는 수학에서 말하는 [부호 함수(ginum fucntion)](https://namu.wiki/w/%EB%B6%80%ED%98%B8%20%ED%95%A8%EC%88%98) 을 뜻함, 표현식의 값이 음수,0,양수일 때 각각 -1, 0, 1 을 반환

1. Comparable을 구현한 클래스는 모든 x,y에 대해 sgn(x.compareTo(y)) == -sgn(y.compareTo(x))여야 한다.
    - sgn(x.compareTo(y)) == -sgn(y.compareTo(x))
2. Comparable을 구현한 클래스는 추이성을 보장해야 한다. (x.compareTo(y) > 0 && y.compareTo(z) > 0) 이면 x.compareTo(z) > 0
    - x.compareTo(y) > 0 && y.compareTo(z) > 0 => x.compareTo(z) > 0
3. Comparable을 구현한 클래스는 모든 z에 대해 x.compareTo(y) == 0 이면 sgn(x.compareTo(z)) == sgn(y.compareTo(z))
    - x.compareTo(y) == 0 -> sgn(x.compareTo(z)) == sgn(y.compareTo(z))
4. 필수는 아니지만, (x.compareTo(y) == 0) == (x.equals(y))
    - (x.compareTo(y) == 0) == (x.equals(y))
    - 해당 권고를 지키지 않을 경우 명시해야 한다.
        - "주의 이 클래스의 순서는 equals 메서드와 일관되지 않는다."
    
#### compareTo 메서드의 일반 규약 상세 내용
1. 두 객체의 순서를 바꿔도 같은 예상한 결과가 나와야 함 
   - 첫 번째가 두 번째보다 작으면 두 번째가 첫 번째보다 커야 한다는 뜻
2. 첫 번째가 두 번째보다 크고 두 번째가 세번째보다 크다면 첫 번째가 세 번째보다 큼
3. 크기가 같은 객체들끼리는 어떤 다른 객체와 비교하더라도 항상 같아야 함
- 주의사항
    - 기존 클래스를 확장한 구체 클래스에서 새로운 값 컴포넌트를 추가했다면 compareTo 규약을 지킬 방법이 없다.
    - 우회법 : 확장하는 대신 독립된 클래스를 만들고 이 클래스에 원래 클래스를 가리키는 필드를 두고 뷰 메서드를 반환
- 필수는 아니지만 꼭 지키킬 원하는 것 
    - compareTo 메서드로 수행한 동치성 테스트의 결과가 equals와 같아야 한다는 것.
        - 이를 잘 지켜야 compareTo의 순서와 equals의 결과가 일관될 수 있다.
        - 일관되지 않은 클래스도 동작을 하지만 이 클래스 객체를 컬렉션에 넣으면 컬렉션 인터페이스에 정의된 동작과 엇박자를 낼 것
            - 인터페이스들이 equals 메서드 규약을 따른다고 되어 있지만 정렬된 컬렉션들이 동치성을 비교할 때는 compareTo를 쓰기 때문
            - ex BigDecimal 의 경우
              ```
                  BigDecimal b1 = new BigDecimal("1.0");
                  BigDecimal b2 = new BigDecimal("1.00");
                  b1.equals(b2); // false
                  b1.compareTo(b2); // 0
                  HashSet<BigDecimal> hs = new HashSet<>();
                  hs.add(b1);
                  hs.add(b2);
                  TreeSet<BigDecimal> ts = new TreeSet<>();
                  ts.add(b1);
                  ts.add(b2);
                  assert hs.size() == 2; // true
                  assert ts.size() == 1; // true
              ```
---

#### compareTo에 관해

- compareTo는 타입이 다른 객체를 신경쓰지 않아도 됨
    - 타입이 다른 객체가 들어오면 ClassCastException 예외를 던져도 되고 대부분 그렇게 한다.
- 다른 타입 사이의 비교도 허용하지만 보통 비교할 객체들이 구현한 공통 인터페이스를 매개로 이루어진다.
- compareTo 규약을 지키지 못하면 비교를 활용하는 클래스와 어울리지 못함
- 활용하는 예 TreeSet, TreeMap, Collections, Array

---

#### compareTo 작성 요령

- Comparable은 타입을 인수로 받는 제네릭 인터페이스이므로 compareTo 메서드의 인수 타입은 컴파일타임에 정해짐
    - 입력 인수의 타입을 확인하거나 형변환 할 필요가 없음
- 인수가 null 이라면 NullPointerException이 던져질 것
- compareTo 메서드는 각 필드의 동치를 비교하지 않고 순서를 비교함
    - 객체 참조 필드를 비교하려면 compareTo 메서드를 재귀적으로 호출한다.
- Comparable을 구현하지 않은 필드나 표준이 아닌 순서로 비교해야 한다면 Comparator를 이용
    - 비교자는 직접만들거나 자바가 제공하는 것을 사용하면 된다.

---

#### compareTo 일반 패턴

- 싱글 필드 비교
```
public final class CaseInsensitiveString implements Comparable<CaseInsensitiveString> {
    private String s;
    
    public int compareTo(CaseInsensitiveString cis) {
        return String.CASE_INSENSITIVE_ORDER.compare(s, cis.s);
    }
}
```
- 멀티 필드 비교
```
public int compareTo(PhoneNumber pn) {
    int result = Short.compare(areaCode, pn.areaCdoe);
    if (result == 0) {
        result = Short.compare(prefix, pn.prefix);
        if (result == 0) {
            result = Short.compare(lineNum, pn.lineNum);
        }
    }
    
    return result;
}
```  

- 박싱 기본 타입 클래스들에 새로 추가된 정적 메서드 compare를 이용 (a < b, a > b 를 사용하는 방식은 비추천)


---

#### 자바 8 의 Comparator 인터페이스의 비교

- Comparator 인터페이스가 일련의 비교자 생성 메서드(comparator construction method)와 팀을 꾸려 메서드 연쇄 방식으로 비교자를 생성
    - ex) thenComparing
- 좋지만 약간의 성능저하가 뒤따름
- 참고로 정적 임포트 기능을 이용하면 정적 비교자 생성 메서드들을 이름만으로 사용할 수 있어 코드가 훨씬 깔끔해짐
```
private static final Comparator<PhoneNumber> COMPARATOR = 
    comparingInt((PhoneNumber pn) -> pn.areaCode)
        .thenComparingInt(pn -> pn.prefix)
        .thenComparingInt(pn -> pn.lineNUm);

public int compareTo(PhoneNumber pn) {
    return COMPARATOR.compare(this, pn);
}
```
- 위 코드에서 클래스를 초기화 할 때 비교자 생성 메서드 2개를 이용해 비교자를 생성
    1. comparingInt가 객체 참조를 int 타입 키에 매핑하는 [키 추출 함수](https://docs.oracle.com/middleware/1213/coherence/java-reference/com/tangosol/util/extractor/KeyExtractor.html) 를 인수로 받아 순서를 정하는 비교자를 반환하는 정적 메서드
        - (PhoneNumber pn) 을 명시한 것은 자바의 타입 추론 능력이 타입을 알아낼 만큼 갈력하지 않기 때문에 컴파일 되도록 도와준 것
    2. thenComparingInt를 수행한다. thenComparingInt는 Comparator의 인스턴스 메서드로 int 키 추출자 함수를 입력받아 다시 비교자를 반환
        - 원하는 만큼 연달아 호출 가능
- Comparator는 수많은 보조 생성 메서드들로 중무장 중
    - ex. thenComparaing, comparing, comparingInt, comparingLong ......
- 객체 참조용 비교자 생성 메서드는 comparing 정적 메서드 2개가 다중 정의됨
    1. 키 추출자를 받아서 키의 자연적 순서를 사용 (keyExtractor)
       ```
           public static <T, U extends Comparable<? super U>> Comparator<T> comparing(
            Function<? super T, ? extends U> keyExtractor)
              {
              Objects.requireNonNull(keyExtractor);
              return (Comparator<T> & Serializable)
              (c1, c2) -> keyExtractor.apply(c1).compareTo(keyExtractor.apply(c2));
              }
       ```
    2. 키 추출자 하나와 추출된 키를 비교할 비교자까지 2개의 인수를 받음 (keyExtractor, keyComparator)
       ```
            public static <T, U> Comparator<T> comparing(
                Function<? super T, ? extends U> keyExtractor,
                Comparator<? super U> keyComparator)
                {
                    Objects.requireNonNull(keyExtractor);
                    Objects.requireNonNull(keyComparator);
                    return (Comparator<T> & Serializable)
                        (c1, c2) -> keyComparator.compare(keyExtractor.apply(c1),
                                                          keyExtractor.apply(c2));
                }
       ```
    
- thenComparaing 인스턴스 메서드가 3개 다중 정의 되어 있음
    1. 비교자 하나만 인수로 받아 비교자를 통해 부차 순서를 정함
        ```
          default Comparator<T> thenComparing(Comparator<? super T> other) {
                Objects.requireNonNull(other);
                return (Comparator<T> & Serializable) (c1, c2) -> {
                    int res = compare(c1, c2);
                    return (res != 0) ? res : other.compare(c1, c2);
                };
            }
        ```
    2. 비교자, 키 추출자를 인수로 받아 해당 키의 자연적 순서로 보조 순서를 정함
        ```
           default <U extends Comparable<? super U>> Comparator<T> thenComparing(
           Function<? super T, ? extends U> keyExtractor)
           {
           return thenComparing(comparing(keyExtractor));
           }
        ```
    3. 키 추출자와 하나의 추출 된 키 비교할 비교자까지 2개의 인수를 받아서 순서를 정함
        ```
           default <U> Comparator<T> thenComparing(
            Function<? super T, ? extends U> keyExtractor,
            Comparator<? super U> keyComparator)
              {
              return thenComparing(comparing(keyExtractor, keyComparator));
              }

        ```

---

#### 주의사항

- 값의 차를 기준으로 첫 번째 값이 두 번재 값보다 작으면 음수, 같으면 0, 크면 양수를 반환하는 compareTo는 사용하면 안 됨
    - [정수 오버플로 문제](https://stackoverflow.com/questions/70298676/how-can-compareto-method-avoid-overflowing)
      - ex) a : int값 21억 b : int값 -21억 일 경우 
    - [부동 소수점 문제](https://www.geeksforgeeks.org/problem-in-comparing-floating-point-numbers-and-how-to-compare-them-correctly/)
      - ex) double a = (0.3 * 3) + 0.1;
        double b = 1;
        compareFloatNum(a, b);
        a is : 0.99999999999999988898
        b is : 1
        - 같지 않음

- 추천 방법    
    - 정적 compare 메서드를 활용한 비교자
      ```
      static Comparator<Object> hashCodeOrder = new Comparator<>() {
        public int compare(Object o1, Object o2) {
            return Integer.compare(o1.hashCode(), o2.hashCode());
        }
      }
      ```
    - 비교자 생성 메서드를 활용한 비교자
      ```
      static Comparator<Object> hashCodeOrder = 
        Comparator.comparingInt(o -> o.hashCode());
      ```
    
---

#### 요약

- 순서를 고려해야 하는 값 클래스를 작성한다면 꼭 Comparable 인터페이스를 구현하자.
    - 인스턴스들을 쉽게 정렬하고 검색하고 비교 기능을 제공하는 컬렉션과 어우러지도록 해야한다.
- compareTo 메서드에서 필드의 값을 비교할 떄 a < b, a < b 와 은 연산자를 쓰지 말자.
- 박싱된 기본 타입 클래스가 제공하는 정적 compare 메서드나 비교자 생성 메서드를 활용하자.


