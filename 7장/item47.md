# item 47 반환 타입으로는 스트림보다 컬렉션이 낫다

---

### 스트림은 반복(iteration)을 지원하지 않는다.

- API를 스트림만 반환하도록 짜놓으면 반환된 스트림을 forEach로 반복하길 원하는 사용자는 당연히 불만을 토로할 것
  - but 사실 Stream 인터페이스는 Iterable 인터페이스가 정의한 추상 메서드를 전부 포함하고
  - Iterable 인터페이스가 정의한 방식대로 동작한다.
  - forEach로 스트림을 반복할 수 없는 까닭은 바로 Stream이 Iterable을 확장하지 않아서다.
  - 우회로도 없다.
  
```java
// 스트림을 반복하기 위한 '끔찍한' 우회 방법
for (ProcessHandle ph: (Iterable<ProcessHandle>)
                        ProcessHandle.allProcesses()::iterator) {
    // process 들을 처리한다.
}
```
- 위 코드는 동작은 하지만 너무 난잡하고 직관성이 떨어진다.

- 어댑터 사용
```java
// 어댑터
public static <E> Iterable<E> iterableOf(Stream<E> stream) {
    return stream::iterator;
}
```
```java
// 어댑터 사용
for (ProcessHandle ph: iterableOf(ProcessHandle.allProcesses())) {
    // process 들을 처리한다.
}
```

- 반대로 API를 Iterable만 반환하면 스트림 파이프라인에서 처리하려는 프로그래머는 성을 낼 것이다.
- 어댑터

```java
// 어댑터
public static <E> Stream<E> streamOf(Iterable<E> iterable) {
    return StreamSupport.stream(iterable.spliterator(), false);
}
```

- 공개 API를 작성할 때는 스트림 파이프라인을 사용하는 사람과 반복문에서 쓰려는 사람 모두를 배려해야 한다. 

- Collection 인터페이스는 Iterable의 하위 타입이고 stream 메서드도 제공하니 반복과 스트림을 동시에 지원한다. 
  - 따라서 공개 API의 반환타입에는 Collection이나 그 하위 타입을 쓰는게 일반적으로 최선이다.
  

- 단지 컬렉션을 반환한다는 이유로 덩치 큰 시퀀스를 메모리에 올려서는 안된다.

- 반환할 시퀀스가 크지만 표현을 간결하게 할 수 있다면 전용 컬렉션을 구하는 방안을 검토해보자.
  - AbstractList를 이용하면 전용 컬렉션을 손쉽게 구현할 수 있다.
  ```java
  public class PowerSet {
      public static final <E> Collection<Set<E>> of(Set<E> s) {
          List<E> src = new ArrayList<>(s);
  
          if (src.size() > 30)
              throw new IllegalArgumentException("Set too big " + s);
  
          return new AbstractList<Set<E>>() {
  
              @Override
              public int size() {
                  return 1 << src.size(); // 2 to the power srcSize
              }
  
               @Override 
              public boolean contains(Object o) {
                  return o instanceof Set && src.containsAll((Set)o);
              }
  
              @Override
              public Set<E> get(int index) {
                  Set<E> result = new HashSet<>();
  
                  for (int i = 0; index != 0; i++, index >>= 1)
                      if ((index & 1) == 1)
                          result.add(src.get(i));
  
                  return result;
              }
          };
      }
  
       public static void main(String[] args) {
          Set s = new HashSet(Arrays.asList(args));
          System.out.println(PowerSet.of(s));
      }
  }
  ```
  
- contains와 size만 더 구현하면 AbstractCollection -> Collection 구현체를 작성할 수 있다.
  - contains와 size를 구현하는게 불가능할 때는 켈렉션보다는 스트림이나 Iterable을 반환하는 편이 낫다.
  
