# item 43 람다보다는 메서드 참조를 사용하라

---

### 람다보다 메서드 참조

- 메서드 참조를 이용하면 람다보다 더 간단하게 표현이 가능하다.
    ```java
        // 람다
        map.merge(key, 1, (cnt, inc) -> cnt + inc);
        // 메서드 참조
        map.merge(key, 1, Integer::sum);
    ```

- 람다로 할 수 없는 일이라면 메서드 참조로도 할 수 없다. (예외는 아래에 있다.)

- 메서드 참조에는 기능을 잘 드러내는 이름을 지어줄 수 있고 친절할 설명을 문서로 남길 수도 있다.

### 메서드 참조의 유형 5가지

[참고글](https://ryumodrn.tistory.com/103)
- 1. 정적 메서드를 가리키는 메서드 참조
- 인스턴스 메서드를 참조하는 유형
    - 2. 수신 객체(참조대상 인스턴스)를 특정하는 한정적 인스턴스 메서드 참조
        - 정적 메서드 참조와 비슷
            - 함수 객체가 받는 인수와 참조되는 메서드가 받는 인수가 똑같다.
    - 3. 수신 객체(참조대상 인스턴스)를 특정하지 않는 비한정적 인스턴스 메서드 참조
        - 함수 객체를 적용하는 시점에 수신 객체를 알려준다.
            - 수신객체 전달용 매개변수가 매개변수 목록의 첫 번째로 추가된다.
            - 그 뒤로는 참조되는 메서드 선언에 정의된 매개변수들이 뒤 따른다.
                - 주로 매핑과 필터 함수에 쓰인다.
- 클래스 생성자를 가리키는 메서드 참조와 배열 생성자를 가리키는 메서드 참조
    - 4. 클래스 생성자 메서드 참조
    - 5. 배열 생성자 메서드 참조

- 그림 정리 참고 (p261)

* 람다로는 불가능하나 메서드 참조로 가능한 유일한 예는 제네릭 함수 타입 구현이다.
    ```java
    public interface G1 {
        <E extends Exception> Object m() throws E;
    }
  
    public interface G2 {
        <F extends Exception> Object m() throws E;
    }
  
    public interface G extends G1, G2 {}
  
    // 이 때 함수형 인터페이스 G를 함수 타입으로 표현하면 다음과 같다.
    <F extends Exception> () -> String throws F
  
    // 이처럼 함수형 인터페이스를 위한 제네릭 함수 타입은 
    // 메서드 참조 표현식으로 구현가능하지만,
    // 람다식으로는 불가능하다. 
    // 제네릭 람다식이라는 문법이 존재하지 않기 때문
    ```

우리 예제
```java
Optional.ofNullable(skuImages).orElse(Collections.emptyList())
    .stream()
    .collect(Collectors.toMap(SkuImage::getPriority, SkuImage::getImagePath, (a, b) -> a))
```