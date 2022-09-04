# item 79 과도한 동기화는 피하라

---

### 과도한 동기화는 성능을 떨어뜨리고 교착상태에 빠뜨리고 심지어는 예측할 수 없는 동작을 낳음

- 응답 불가와 안전 실패를 피하려면 동기화 메서드나 동기화 블록 안에서는 제어를 절대로 클라이언트에 양도하면 안 된다.
  - 동기화된 영역 안에서 재정의할 수 있는 메서드는 호출되면 안 됨
  - 클라이언트가 넘겨준 함수 객체를 호출해서도 안 됨
  
- 동기화된 영역을 포함한 클래스 관점에서는 이런 메서드는 모두 바깥 세상에서 온 외계인
  - 메서드가 무슨 일을 할 지 알지 못하며 통제할 수도 없다는 뜻
  - 외계인 메서드가 하는 일에 따라 동기화된 영역은 예외를 일으키거나 교착상태에 빠지거나 데이터를 훼손할 수 있다.
  
- 관찰자 패턴

```java
// 잘못된 코드, 동기화 블록 안에서 외계인 메서드를 호출
public class ObservableSet<E> extends ForwardSet<E> {
    public ObservableSet(Set<E> set) {
        super(set);
    }
    
    private final List<SetObserver<E>> observers = new ArrayList<>();
    
    public void addObserver(SetObserver<E> observer) {
        synchronized (observers) {
            observers.add(observer);
        }
    }
    
    public boolean removeObserver(SetObserver<E> observer) {
      synchronized (observers) {
        return observers.remove(observer);
      }
    }
    
    private void notifyElementAdded(E element) {
        synchronized (observers) {
            for (SetObserver<E> observer : observers) 
                observer.added(this, element);
        }
    }
    
    @Override public boolean add(E element) {
        boolean added = super.add(element);
        if (added)
            notifyElementAdded(element);
        return added;
    }
    
    @Override public boolean addAll(Collection<? extends E> c) {
        boolean result = true;
        for (E element : c) 
            result |= add(element); // notifyElementAdded 를 호출한다.
        return result;
    }
}
```

관찰자들은 addObserver와 removeObserver 메서드를 호출 해 구독을 신청하거나 해지 두 경우 모두 callback 메서드를 건낸다.
```java
@FunctionalInterface public interface SetObserver<E> {
    // ObservableSet에 원소가 더해지면 호출된다.
    void added(ObservableSet<E> set, E element);
}
```

출력 부분
```java
// 0부터 99까지 set에 add
public class Main {
  public static void main(String[] args) {
    ObservableSet<Integer> set = new ObservableSet<>(new HashSet<>());
    
    set.addObserver((s, e) -> System.out.println(e));
    
    for (int i = 0; i < 100; i++)
        set.add(i);
  }
}
```

### 이제 23까지 정숫값 출력하다가 값이 23이면 자기 자신을 제거(구독해지)하는 관찰자를 추가
```java
set.addObserver(new SetObserver<>() {
    public void added(ObservableSet<Integer> s, Integer e) {
        System.out.println(e);
        if (e == 23)
            s.removeObserver(this);
    }
});
```

  - 예상 : 23까지 출력한 후 관찰자 자신을 구독해지 한 후 종료
  - 실제 : 그렇게 진행되지 않는다!
  
  - 이 프로그램은 23까지 출력한 다음 ConcurrentModificationException을 던진다.
    - 관찰자의 added 메서드 호출이 일어난 시점이 notifyElementAdded가 관찰자들의 리스트를 순회하는 도중이기 때문이다.
    
  ![img.png](https://user-images.githubusercontent.com/17772475/188304635-29db80c6-6123-4618-93e0-e57c55d5e141.png)
  
  - added 메서드는 ObservableSet의 removeObserver 메서드를 호출하고, 이 메서드는 다시 observers.remove 메서드를 호출한다.
  
  - 리스트에서 원소를 제거하려는데 마침 지금은 이 리스트를 순회하는 도중이다. 허용되지 않는 동작이다.
  
  - notifyElementAdded 메서드에서 수행하는 순회는 동기화 블록 안에 있으므로 동시 수정이 일어나지 않도록 보장
    - but 정작 자신이 콜백을 거쳐 되돌아와 수정하는 것까지 막지는 못한다.

### 이번에는 구독해지 관찰자를 작성하는데 removeObserver 를 직접 호출하지 않고 실행자 서비스(ExecutorService)를 사용해 다른 스레드에 부탁
```java
set.addObserver(new SetObserver<>() {
        public void added(ObservableSet<Integer> s, Integer e) {
          System.out.println(e);
          if (e == 23) {
              ExecutorService exec = Executors.newSingleThreadExecutor();
              try {
                  exec.submit(() -> s.removeObserver(this)).get();
              } catch (ExecutionException | InterruptedException ex) {
                  throw new AssertionError(ex);
              } finally {
                  exec.shutdown();
              }
          }
        }
});
```

- 프로그램을 실행하면 예외는 나지 않지만 교착 상태에 빠진다.
  - 백그라운드 스레드가 s.removeObserver를 호출하면 관찰자를 잠그려 시도하지만 락을 얻을 수 없다.
  - 메인 스레드가 이미 락을 쥐고 있기 때문
  - 그와 동시에 메인 스레드는 백그라운드 스레드가 관찰자를 제거하기만을 기다리는 중이다.
    - 교착 상태!
  
- 실제 시스템에서도 동기화된 영역 안에서 외계인 메서드를 호출하여 교착상태에 빠지는 사례는 자주 있다.

- 예외와 교착상태에서는 운이 좋았다.
  - 동기화 영역이 보호하는 자원(관찰자)은 외계인 메서드(added)가 호출될 때 일관된 상태였음
- 똑같은 상황이지만 불변식이 임시로 깨진 경우
  - 자바 언어의 락은 재진입(reentrant)을 허용 -> 교착상태에 빠지지는 않음
  
- 예외를 발생시킨 첫 번째 예시라면 외계인 메서드를 호출하는 스레드느느 이미 락을 쥐고 있으므로 다음번 락 획득도 성공
  - 락이 보호하는 데이터에 대해 개념적으로 관련이 없는 다른 작업이 진행 중인데도 획득이 성공 됨
    - 참혹한 결과로 이어질 수 있음
  - 문제의 주 원인은 락이 제 구실을 하지 못했기 때문
    - 재진입 가능 락은 객체 지향 멀티스레드 프로그램을 쉽게 구현할 수 있도록 해주지만 응답 불가(교착 상태)가 될 상황을 안전 실패(데이터 훼손)로 변모 시킬 수 있음
  
- 대부분은 외계인 메서드 호출을 동기화 블록 바깥으로 옮기면 된다.
  - notifyElementAdded 메서드에서라면 관찰자 리스트를 복사해 쓰면 락 없이도 안전하게 순회할 수 있음
  -   이 방식을 적용하면 앞서 두 예제에서의 예외 발생과 교착 상태 증상이 사라진다.
  
```java
private void notifyElementAdded(E element) {
    List<SetObserver<E>> snapshot = null;
    synchronized(observers) {
        snapshot = new ArrayList<>(observers);
    }
    for (SetObserver<E> observer : snapshot)
        observer.added(this, element);
}
```


### 동기화 블록 바깥으로 옮기는 더 나은 방법
  - 동시성 컬렉션 라이브러리의 CopyOnWriteArrayList가 정확히 이 목적으로 특별히 설계된 것
  - 이름이 말해주듯 ArrayList를 구현한 클래스로 내부를 변경하는 작업은 항상 깨끗한 복사본을 만들어 수행하도록 구현
  - 내부의 배열은 절대 수정되지 않으니 순회할 때 락이 필요 없어 매우 빠름
  - 다른 용도로 쓰인다면 CopyOnWriteArrayList는 끔찍이 느리겠지만 수정할 일은 드물고 순회만 빈번히 일어나는 관찰자 리스트 용도로는 최적

### CopyOnWriteArrayList를 이용한 ObservableSet 구현
  - 명시적으로 동기화한 곳이 사라졌다는 것에 주목하자.
```java
private final List<SetObserver<E>> observers = new CopyOnWriteArrayList<>();

public void addObserver(SetObserver<E> observer) {
    observers.add(observer);
}

public boolean removeObserver(SetObserver<E> observer) {
    return observers.remove(observer);
}

private void notifyElementAdded(E element) {
    for (SetObserver<E> observer : observers)
        observer.added(this, element);
}
```

### 동기화 영역 바깥에서 호출되는 외계인 메서드를 열린 호출 (open call)이라 함

- 외계인 메서드는 얼마나 오래 실행될 지 알 수 없는데 동기화 영역 안에서 호출된다면 그동안 다른 스레드는 보호된 자원을 사용하지 못하고 대기해야 함
- 열린 호출은 실패 방지 효과 외에도 동시성 효율을 크게 개선해준다.

### 기본 규칙은 동기화 영역에서는 가능한 한 일을 적게 하는 것이다.

- 락을 얻고 공유 데이터를 검사하고 필요하면 수정하고 락을 놓는다(release)
  - 오래 걸리는 작업이라면 '공유 중인 가변 데이터는 동기화에 사용하라'는 지침을 어기지 않으면서 동기화 영역 바깥으로 옮기는 방법을 찾아보자.
  
### 동기화 성능 측면

- 자바 동기화 비용은 빠르게 낮아져 왔지만 과도한 동기화를 피하는 일은 중요
  - 멀티 코어가 일반화된 오늘날, 동기화가 초래하는 진짜 비용은 락을 얻는데 드는 CPU시간이 아니다. 
  - 경쟁하느라 낭비하는 시간, 병렬로 실행할 기회를 잃고 모든 코어가 메모리를 일관되게 보기 위한 지연시간이 진짜 비용
  - 가상머신의 코드 최적화를 제한한다는 점도 과도한 동기화의 또 다른 숨은 비용이다.
  
### 가변 클래스를 작성하려거든 해야할 2가지

1. 동기화를 전혀 하지말고, 그 클래스를 동시에 사용해야 하는 클래스가 외부에서 알아서 동기화하게 하자.
2. 동기화를 내부에서 수행해 스레드 안전한 클래스로 만들자. (item 82)

- 단, 클라이언트가 외부에서 객체 전체에 락을 거는 것보다 동시성을 월등히 개선할 수 있을 때만 2번을 택하자.

java.util -> 1번
java.util.concurrent -> 2번

- StringBuffer 인스턴스는 거의 항상 단일 스레드에서 쓰였음에도 내부적으로 동기화를 수행
  - 뒤늦게 StringBuilder 등장
- java.util.Random -> java.util.concurrent.ThreadLocalRandom 으로 대체

- 선택하기 어렵다면 동기화하지 말고 대신 문서에 "스레드 안전하지 않다"고 명기하자.

### 클래스를 내부에서 동기화하기로 했다면 

- 락 분할 (lock splitting)[링크](https://www.demo2s.com/java/java-interview-question-what-is-lock-splitting-technique.html)
- 락 스트라이핑 (lock striping)[링크](https://www.baeldung.com/java-lock-stripping) [링크2](https://aroundck.tistory.com/4529)
- 비차단 동시성 제어 (nonblocking concurrency control) [링크](https://jenkov.com/tutorials/java-concurrency/non-blocking-algorithms.html) [링크2](https://stackoverflow.com/questions/50717086/why-non-blocking-concurrency-is-better-than-blocking-concurrency)

### 여러 스레드가 호출할 가능성이 있는 메서드가 정적 필드를 수정한다면 그 필드를 사용하기 전에 반드시 동기화

- but 클라이언트가 여러 스레드로 복제돼 구동되는 상황이라면 다른 클라이언트에서 이 메서드를 호출하는 걸 막을 수 없으니 외부에서 동기화할 방법이 없다.
- 이 정적 필드가 심지어 private라도 서로 관련 없는 스레드들이 동시에 일고 수정할 수 있게 된다.
  - 사실상 전역 변수와 같아진다는 뜻이다.
- generateSerialNubmer 메서드에 쓰인 nextSerialNumber 필드가 이러한 사례다.

## 정리

- 교착상태와 데이터 훼손을 피하려면 동기화 영역 안에서 외계인 메서드를 절대 호출하지 말자.
- 동기화 영역 안에서의 작업은 최소한으로 줄이자.
- 가변 클래스를 설계할 때는 스스로 동기화해야 할 지 고민하자.
- 멀티코어 세상인 지금은 과도한 동기화를 피하는게 과거 어느 때보다 중요
- 합당한 이유가 있을 때만 내부에서 동기화하고, 동기화됐는지 여부를 문서에 명확히 밝히자.