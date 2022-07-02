#item 11 equals를 재정의하려거든 hascode도 재정의하라

---

#### equals를 재정의한 클래스에서 모두 hashcode도 재정의해야 한다.

- 하지 않는다면?
    - hashcode 일반 규약을 어기게 되어 해당 클래스의 인스턴스를 hashMap이나 hashSet의 원소로 사용할 때 문제를 일으킬 것
    - 규약
        1. equals 비교에 사용되는 정보가 변하지 않았다면 hashcode 메서드는 몇 번을 호출해도 일관되게 항상 같은 값을 반환해야 한다.
            - 단, 어플리케이션을 다시 실행한다면 값이 달라져도 상관없다.
        2. equals(Object)가 두 객체를 같다고 판단했다면, 두 객체의 hashcode는 같은 값을 반환해야 한다.
        3. equals가 두 객체를 다르다고 판단했더라도, 두 객체의 hashcode가 서로 다른 값을 반환할 필요는 없다.  
            - 단, 다른 객체에 대해서는 다른 값을 반환해야 해시테이블 성능이 좋아짐
    - 문제가 되는 조항은 b다. 논리적으로 같은 객체는 같은 해시코드를 반환해야한다.
    ```
    Map<PhoneNumber, String> map = new HashMap<>();
    m.put(new PhoneNumber(707, 867, 5309), "제니");
    m.get(new PhoneNumber(707, 867, 5309)) // null -> 해시 코드가 같지 않기 때문
    ```

---

#### 올바른 hashcode 메서드의 모습

```
@override
public int hashCode() {
    return 42;
}
```

- 위 메서드는 모든 객체에서 똑같은 해시코드를 반환하니 적법하지만, 모든 객체에 같은 값만 내어주므로 모든 객체가 해시테이블의 버킷하나에 담기게 된다.
    - 마치 연결리스트 처럼 동작한다.
- 좋은 hash 함수라면 서로 다른 인스턴스에 다른 해시코드를 반환해야 한다. (규약 c)
- 이상적인 hash 함수는 주어진 서로 다른 인스턴스들을 32비트 정수 범위에 균일하게 분배해야 한다.
- 좋은 hashCode를 작성하는 방법
    1. int 변수 result를 선언한 수 값 c로 초기화한다. 
    2. 해시코드 c를 계산한다.
        1. 기본 타입 필드라면, Type.hashCode(f) 수행, Type은 기본 타입의 박싱 클래스다.
        2. 참조 타입 필드면서 equals 메서드가 필드의 equals를 재귀적으로 호출해 비교한다면 hashCode도 재귀적으로 호출한다.  
            - 계산이 복잡해 질 것 같으면 표준형을 만들어 표준형 hashCode를 호출한다.  
            - 필드 값이 null이면 0을 사용한다.
        3. 필드가 배열이라면 핵심 원소 각각을 별도 필드처럼 다룬다.
           - 배열에 핵심 원소가 하나도 없다면 0
           - 모든 원소가 핵심 원소라면 Arrays.hashCode를 사용한다.
        4. 해시코드를 계산한다. 
        5. 계산한 해시코드 c로 result를 갱신한다.
            ```
            result = 31 * result + c;
            ```
    3. result를 반환한다.
- 파생 필드는 해시코드 계산에서 제외해도 된다. 
- equals 비교에 사용되지 않은 필드는 반드시 제외해야 한다.
- ii.e에서 31 * result는 필드를 곱하는 순서에 따라 result 값이 달라지게 한다. 
- ii.e에서 31을 곱할 숫자로 정한 이유는 31이 홀수면서 소수(prime number)이기 때문이다.
    - 짝수라면 숫자를 곱해 오버플로가 발생하면 정보를 잃게 된다. [시프트로 하나가 밀려서](https://velog.io/@indongcha/hashCode%EC%99%80-31)
    - 홀수라면 왜 괜찮을까?
- 소수를 곱하는 이유는 해시 함수의 분배가 좋아지기 때문 [링크](https://stackoverflow.com/questions/3613102/why-use-a-prime-number-in-hashcode)

별개
- [hashMap에서 해시충돌 시 저장](https://leedo.me/36)은 16비트 오른쪽으로 이동 후 XOR연산으로 마스킹 처리
    - [D2 참조](https://d2.naver.com/helloworld/831311)
- [hashMap 해결](https://cantcoding.tistory.com/90)
    - [이유](https://mhwan.tistory.com/59)


---

#### 전형적 hashCode 메서드의 모습
```
// 전형적 hashCode 메서드 모습
@Override
public int hashCode() {
    int result = Short.hashCode(areaCode);
    result = 31 * result + Short.hashCode(prefix);
    result = 31 * result + Short.hashCode(lineNum);
    return result;
}
```

- 핵심 필드가 3개인 경우의 계산
    - 비결정적 요소가 전혀없으므로 동치인 인스턴스들은 같은 해시코드를 가질 것이다.
- 해시 충돌이 더욱 적은 방법을 꼭 써야한다면 구아바의 com.google.common.hash.Hashing을 참고하자. [링크](https://guava.dev/releases/21.0/api/docs/com/google/common/hash/Hashing.html)
- Objects를 이용하면 비슷한 수준의 hashCode 함수를 hash 메서드 한 줄로 작성이 가능하지만 느리다.
    - 입력 인수를 담기 위한 배열이 만들어지고, 입력 중 기본 타입이있다면 박싱 & 언박싱 과정을 거치기 때문
```
// Objects.hash 함수 이용
@Override
public int hashCode() {
    return Objects.hash(lineNum, prefix, areaCode);
}
```
- 클래스가 불변이고 해시코드를 계산하는 비용이 크다면, 매번 새로 계산하기보다는 캐싱하는 방식을 고려
    - 해당 객체가 해시의 키로 주로 쓰일 것 같은 경우 
        - 인스턴스가 만들어질 때 해시코드를 계산
    - 해당 객체가 해시의 키로 주로 쓰이지 않을 것 같은 경우
        - lazy initialization 전략을 고민해볼 것 (지연 초기화 전략)
            - 지연 초기화를 하려면 tread-safe 하도록 신경써야 함
```
// 쓰레드 안정성 고려
private int hashCode; // 자동으로 0으로 초기화된다.

@Override
public int hashCode() {
    int result = hashCode;
    if (result == 0) {
        result = Short.hashCode(areaCode);
        result = 31 * result + Short.hashCode(prefix);
        result = 31 * result + Short.hashCode(lineNum);
        hashCode = result;
    }
    return result;
}
```

#### hashCode 유의점

- 성능을 높이기 위해 해시코드 계산 시 핵심 필드를 생략해서는 안 된다.
    - 속도를 챙기고 해시 품질을 잃게 되어 해시테이블의 성능을 심각하게 떨어뜨릴 수 있다.
        - 특정 영역에 몰린 인스턴스의 해시코드를 넓은 범위로 고르게 퍼트려주는 효과가 있는 필드를 생략하면 해시테이블의 성능을 떨어뜨리게 된다.
- hashCode가 반환하는 값의 생성 규칙을 API 사용자에게 자세히 공표하지 말자.
    - 클라이언트가 몰라야 이 값에 의지하지 않게 되고 추후에 계산 방식을 유연하게 바꿀 수 있다.
    