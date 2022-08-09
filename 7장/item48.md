# item 47 스트림 병렬화는 주의해서 적용하라

---

### 병렬 스트림을 올바르고 빠르게 작성하는 일은 여전히 어려운 작업이다.

- 동시성 프로그래밍을 할 때에 안전성(safety)과 응답 가능(liveness)상태를 유지하기 위해 애써야 하는데
  - 병렬 스트림 파이프라인 프로그래밍에서도 다를 바 없다.
  
[메르센 소수](https://ko.wikipedia.org/wiki/%EB%A9%94%EB%A5%B4%EC%84%BC_%EC%86%8C%EC%88%98)
```java
// 메르센 소수를 생성
public static void main(String[] args) {
    primes().map(p -> TWO.pow(p.intValueExact())).subtract(ONE))
        .filter(mersenne -> mersenne.isProbablePrime(50))
        .limit(20)
        .forEach(System.out::println)
}

static Stream<BigInteger> primes() {
    Stream.iterate(TWO, BigInteger::nextProbablePrime);
}
```

- 이 프로그램을 실행하고 속도를 높이고 싶어 parallel()을 호출하겠다고 생각하면 어떻게 될까?
  - 안타깝게도 프로그램은 아무것도 출력하지 못하면서 CPU는 90%나 잡아먹는 상태가 무한히 계속된다(응답 불가)
  - 느려진 원인은 스트림 라이브러리가 파이프라인을 병렬화하는 방법을 찾아내지 못했기 때문
  - 환경이 아무리 좋아도 데이터 소스가 Stream.iterate거나 중간연산으로 limit를 쓰면 파이프라인 병렬화로는 성능 개선을 기대할 수 없다.
  - 그러나 위의 코드는 2가지를 모두 가지고 있다.
  - [예제](https://woodcock.tistory.com/28) **이전값이 있어야 다음 값을 계산할 수 있습니다.**
  - 파이프라인 병렬화는 limit를 다룰 때 CPU 코어가 남는다면 원소를 몇 개 더 처리한 후 제한된 개수 이후의 결과를 버려도 아무런 해가 없다고 가정한다.
    - 하지만 위 코드의 경우 새롭게 메르센 소수를 찾을 때 마다 그 전 소수를 찾을 때보다 2배정도 더 오래 걸린다.
    - 원소 하나를 계산하는 비용이 대략 그 이전까지의 원소를 전부 계산한 비용을 합친 것 만큼 든다는 뜻이다.
    - 그래서 마비된다.
  
- 대체로 스트림의 소스가 ArrayList, HashMap, HashSet, ConcurrentHashMap의 인스턴스거나 배열, int 범위, long 범위일 때 병렬화 효과가 가장 좋다.
  - 해당 자료구조들은 모두 데이터를 원하는 크기로 정확하고 손쉽게 나눌 수 있어서 일을 다수의 스레드에 분배하기 좋다는 특징이 있기 때문
  - 나누는 작업은 Spliterator가 담당하며 Spliterator 객체는 Stream이나 Iteralbe의 spliterator 메서드로 얻어올 수 있다.
  - 또 원소들을 순차적으로 실행할 때 참조 지역성이 뛰어나다는 것이다.
    - 이웃한 원소의 참조들이 메모리에 연속해서 저장되어 있다는 뜻
    - 참조들이 가리키는 실제 객체들이 떨어져 있을 수 있는데 이러면 참조 지역성이 나빠진다.
    - 참조 지역성이 낮으면 스레드는 데이터가 주 메모리에서 캐시 메모리로 전송되어 오기를 기다리며 시간을 멍하니 보내게 된다.
    - 따라서 참조 지역성은 다량의 데이터를 처리하는 벌크 연산을 병렬화할 때 중요한 요소로 작용한다.
    - 가장 뛰어난 참조지역성 자료구조는 기본 타입 배열이다.
  
- 스트림 파이프라인의 종단 연산 동작 역시 병렬 수행 효율에 영향을 준다.
  - 종단 연산 중 병렬화에 가장 적합한 것은 축소(reduction)다.
  - 축소는 파이프라인에서 만들어진 모든 원소를 합치는 작업으로 min, max, count, sum 과 같이 완성된 형태로 제공되는 메서드 중 하나를 선택해 수행한다.
  - allMatch, anyMatch, noneMatch 처럼 조건에 맞으면 바로 반환되는 메서드도 병렬화에 적합하다.
  - 가변 축소 [(mutable reduction)](https://www.logicbig.com/tutorials/core-java-tutorial/java-util-stream/collect.html#:~:text=Mutable%20reductions%20collect%20the%20desired,implemented%20as%20collect()%20methods.) 을 수행하는 Stream의 collect 메서드는 병렬화에 적합하지 않다.
    - 컬렉션 합치는 부담이 크기 때문
  
- 직접 구현한 Stream, Iterable, Collection이 병렬화의 이점을 제대로 누리게하고 싶다면 spliterator 메서드를 반드시 재정의하고 결과 스트림의 병렬화 성능을 강도높게 테스트해야한다.
  - 고효율 spliterator는 쉽지않다.
  
- 스트림을 잘못 병렬화하면 성능이 나빠질 뿐만 아니라 결과 자체가 잘못되거나 예상 못한 동작이 발생할 수 있다.
  - 결과가 잘못되거나 오동작하는 것은 안전 실패 (safety failure)
    - 안전 실패는 병렬화한 파이프라인이 사용하는 mappers, filters, 프로그래머가 제공한 다른 함수 객체가 명세대로 동작하지 않을 경우 발생할 수 있다.
    - Stream 명세는 이 때 사용되는 함수 객체에 관한 엄중한 규약을 정의해놨다.
      - reduce 연산에 accumulator(누적기) 와 combiner(결합기) 함수는 반드시 결합법칙을 만족(associative)하고 간섭받지 않(non-interfering)고 상태를 가지지 않(stateless)아야 한다.
      - 위 규칙을 지키지 않더라도 순차적으로 수행한다면 올바른 결과를 얻을 수도 있다.
        - 하지만 병렬로 수행하면 참혹한 실패로 이어진다.
   
- 결과가 순차적으로 정돈되지 않을 경우, 출력 순서를 순차 버전처럼 정렬하고 싶다면 forEach -> forEachOrdered로 바꿔주면 된다.
  
- 파이프라인이 수행하는 진짜 작업이 병렬화에 드는 추가 비용을 상쇄하지 못한다면 성능 향상이 미미할 수 있다.
  - 추정하는 방법
    - 스트림 안의 원소 수와 원소당 수행되는 코드 줄 수를 곱했을 때 최소 수십만을 되어야 성능 향상을 맛볼 수 있다.
  
- 스트림 병렬화는 오직 최적화 수단임
  - 변경 전후로 반드시 성능을 테스트 해 병렬화를 사용할 가치가 있는지 확인해야 한다.
    - 보통은 병렬 스트림 파이프라인도 공통의 포크-조인풀에서 수행(즉, 같은 스레드 풀을 사용)하므로 잘못된 파이프라인 하나가 시스템의 다른 부분 성능까지 악영향을 줄 수 있다.
  
- 조건이 잘 갖춰지면 parallel 메서드 호출하나로 거의 코어 프로세스 수에 비례하는 성능향상을 만끽할 수 있다.
```java
// 스트림 병렬화에 적합한 함수, n보다 작거나 같은 소수의 개수를 계산하는 함수
static long pi(long n) {
    return LongStream.rangeClosed(2, n)
        .parallel() // 추가로 시간 단축 기대 가능
        .mapToObj(BigInteger::valueOf)
        .filter(i -> i.isProbablePrime(50))
        .count();
}
```


우리 예제
[병렬화 foreach](https://www.4te.co.kr/876)
```java
// inventory 에 다수 포진
SettlementDailyStockSnapshotJobConfig
parallel
```