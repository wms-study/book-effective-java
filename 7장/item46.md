# item 46 스트림에서는 부작용 없는 함수를 사용하라

---

### 스트림 패러다임

- 스트림 패러다임의 핵심은 계산을 일련의 변환으로 재구성하는 부분이다.
  - 각 변환 단계는 가능한 한 이전 단계의 결과를 받아 처리하는 순수 함수여야 한다.
    - 순수 함수란 오직 입력만이 결과에 영향을 주는 함수
    - 다른 가변 상태를 참조하지 않고 함수 스스로도 다른 상태를 변경하지 않는다.
    - 이렇게 하려면 스트림 연산에 건네는 함수 객체는 모두 부작용(side effect)가 없어야 한다.
  - 잘못 사용한 예
  ```java
  // 단어별 빈도수를 세어 Map으로 만듦
  Map<String, Long> freq = new HashMap<>();
  try (Stream<String> words = new Scanner(file).tokens()) {
    words.forEach(word -> {
        freq.merge(word.toLowerCase(), 1L, Long::sum);
    });
  }
  ```
  - 위 코드는 스트림 코드라 할 수 없고 스트림 코드를 가장한 반복적 코드다.
  - 종단 연산인 forEach 내부에서 외부 상태 (Map)을 수정하는 람다를 실행하면서 문제가 생긴다.
  - 올바른 예
  ```java
  // 단어별 빈도수를 세어 Map으로 만듦
  Map<String, Long> freq;
  try (Stream<String> words = new Scanner(file).tokens()) {
      freq = words
        .collect(groupingBy(String::toLowerCase, counting()));
  }
  ```
  - 짧고 명확함
  - forEach연산은 종단 연산 중 기능이 가장 적고 가장 덜 스트림 답다.
    - 대놓고 반복적이어서 병렬화할 수도 없다.
    - 스트림 계산 결과를 보고할 때만 사용하고 계산하는 데는 쓰지말자.
    
- 축소(reduction) 전략을 캡슐화한 블랙박스 객체라고 생각하기 바란다.
  - 축소는 스트림의 원소들을 객체 하나에 취합한다는 뜻이다.
  
- toList, toSet, toMap 과 같은 Collector 들이 존재한다.

### 맵 수집기 toMap 

- 문자열을 열거 타입 상수에 매핑
```java
private static final Map<String, Operation> stringToEnum = 
        Stream.of(values()).collect(
                toMap(Object::toString, e -> e));
```
  - 스트림 원소 다수가 같은 키를 사용한다면 파이프라인이 IllegalStateException을 던지며 종료될 것이다.

- 각 키와 해당 키의 특정 원소를 연관 짓는 맵을 생성하는 수집기
```java
// 음악가의 베스트 앨범을 연관 짓는 맵을 생성
Map<Artist, Album> topHits = albums.collect(
        toMap(Album::artist, a->a, maxBy(comparing(Album::sales))));
```

- 마지막에 쓴 값을 취합하는 수집기
```java
// newVal로 취합하는 수집기
toMap(keyMapper, valueMapper, (oldVal, newVal) -> newVal);
```

- toConcurrentMap 도 존재한다.

- groupingBy는 원소들을 카테고리 별로 모아놓은 맵 수집기를 반환한다.
```java
Map<String, List<String>> wordsMap = words.collect(gorupingBy(word -> alphabetize(word)))
```

- groupingBy의 리스트 외의 값을 갖는 맵을 생성하게 하려면 분류 함수와 함께 다운스트림(downstream) 수집기도 명시해야 한다.
```java
// 다운스트림으로 counting을 쓴다면 Long이 된다.
Map<String, Long> wordsMap = words.collect(gorupingBy(word -> alphabetize(word), counting()))
```

- groupingBy의 다운스트림 수집기에 더해 맵 팩터리도 지정할 수 있게 해준다.
  - 참고로 이 메서드는 점층적 인수 목록 패턴(telescoping argument list pattern)에 어긋난다.
  - mapFactory 매개변수가 downStream 매개변수보다 앞에 놓인다.
  - 해당 버전을 사용하면 맵과 그 안에 담긴 컬렉션의 타입을 모두 지정할 수 있다. 
  - 값이 TreeSet인 TreeMap을 반환하는 수집기를 만들 수 있다.[예제 3.7](https://umanking.github.io/2021/07/31/java-stream-grouping-by-example/)
  
