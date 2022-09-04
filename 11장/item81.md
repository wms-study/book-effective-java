# item 81 wait와 notify보다는 동시성 유틸리티를 애용하라

---

### wait와 notify로 사용해야 할 이유가 많이 줄어듦

- 고수준 동시성 유틸리티가 wait와 notify로 하드코딩해야 했던 전형적인 일들을 대신 처리해주기 때문
- wait와 notify는 올바르게 사용하기가 아주 까다로우니 고수준 동시성 유틸리티를 사용하자.

### java.util.concurrent 고수준 유틸리티는 세 범주로 나눌 수 있다.

- 실행자 프레임워크
- 동시성 컬렉션(concurrent collection)
- 동기화 장치(synchronizer)

### 동시성 컬렉션

- 동시성 컬렉션은 List, Queue, Map 과 같은 표준 컬렉션 인터페이스에 동시성을 가미해 구현한 고성능 컬렉션
- 높은 동시성에 도달하기 위해 동기화를 각자의 내부에서 수행
- 따라서 동시성 컬렉션에서 동시성을 무력화하는 건 불가능, 외부에서 락을 추가로 사용하면 오히려 속도가 느려짐
- 동시성 컬렉션에서 동시성을 무력화하지 못하므로 여러 메서드를 원자적으로 묶어 호출하는 일 역시 불가능
  - 그래서 여러 기본 동작을 하나의 원자적 동작으로 묶는 '상태 의존적 수정' 메서드들이 추가
- 일반 컬렉션 인터페이스에도 디폴드 메서드 형태로 추가됨

- Map의 putIfAbsent(key, value) 메서드는 주어진 키에 매핑된 값이 아직 없을 때만 새값
- 기존 값이 있었다면 값을 반환, 아니면 null 반환
- 이 메서드 덕에 스레드 안전 정규화 맵 구현 가능 (canonicalizing map)

- ConcurrentMap으로 구현한 동시성 정규화 맵 
```java
private static final ConcurrentMap<String, String> map = new ConcurrentHashMap<>();

public static String intern(String s) {
    String previousValue = map.putIfAbsent(s, s);
    return previousValue == null ? s : previousValue;
}
```

- get은 검색에 최적화 되어 있기 때문에 get을 먼저 호출하여 필요할 때만 putIfAbsent를 호출하면 더 빠름
```java
public static String intern(String s) {
    String result = map.get(s);
    if (result == null) {
        result = map.putIfAbsent(s, s);
        if (result == null)
            result = s;
    }
    return result;
}
```

- 동시성 컬렉션은 동기화한 컬렉션을 낡은 유산으로 만듦

- Collections.synchronizedMap 보다는 ConcurrentHashMap을 사용
- 컬렉션 인터페이스 중 일부는 작업이 성공적으로 완료될 때까지 기다리도록 확장됨
  
- BlockingQueue 에 추가된 메서드 중 take 는 큐의 첫 원소를 꺼냄
- 만약 큐가 비었다면 새로운 원소가 추가될 때까지 기다림
- 이런 특성 덕에 BlockingQueue는 작업 큐로 쓰기에 적합함

- 작업 큐는 하나 이상의 생산자(producer) 스레드가 작업(work)을 큐에 추가하고 하나 이상의 소비자(consumer) 스레드가 큐에 있는 작업을 꺼내 처리하는 형태
- 대부분의 실행자 서비스 구현체에서 BlockingQueue를 이용

### 동기화 장치는 스레드가 다른 스레드를 기다릴 수 있게 하여 서로 작업을 조율할 수 있게 해줌

- 가장 자주 쓰이는 동기화 장치는 [CountDownLatch](https://codechacha.com/ko/java-countdownlatch/#:~:text=CountDownLatch%EB%8A%94%20%EC%96%B4%EB%96%A4%20%EC%93%B0%EB%A0%88%EB%93%9C%EA%B0%80,%EC%B2%98%EB%A6%AC%EB%90%98%EB%8F%84%EB%A1%9D%20%ED%95%A0%20%EC%88%98%20%EC%9E%88%EC%8A%B5%EB%8B%88%EB%8B%A4.) 와 [Semaphore](https://jwprogramming.tistory.com/13) 다.
  - [CountDownLatch](https://codechacha.com/ko/java-countdownlatch/#:~:text=CountDownLatch%EB%8A%94%20%EC%96%B4%EB%96%A4%20%EC%93%B0%EB%A0%88%EB%93%9C%EA%B0%80,%EC%B2%98%EB%A6%AC%EB%90%98%EB%8F%84%EB%A1%9D%20%ED%95%A0%20%EC%88%98%20%EC%9E%88%EC%8A%B5%EB%8B%88%EB%8B%A4.)
  - [Semaphore](https://docs.oracle.com/javase/7/docs/api/java/util/concurrent/Semaphore.html)
  
- CyclicBarrier와 Exchanger는 그보다 덜 쓰인다.
- 가장 강력한 동기화 장치는 Phaser다.
- [정리](https://javabom.tistory.com/35)

### CountDownLatch(걸쇠)는 장벽으로 하나 이상의 스레드가 또 다른 하나 이상의 스레드 작업이 끝날 때까지 기다리게 함

- CountDownLatch의 유일한 생성자는 int값을 받으며 이 값이 래치의 countDown 메서드를 몇 번 호출해야 대기 중인 스레드들을 깨우는지 결정
- 간단한 장치를 활용하면 유용한 기능들을 놀랍도록 쉽게 구현할 수 있음

- CountDownLatch를 이용한 타이머 기능 구현
```java
public static logn time(Executor executor, int concurrency, Runnable action) throws InterruptedExcpetion {
        CountDownLatch ready = new CountDownLatch(concurrency);
        CountDownLatch start = new CountDownLatch(1);
        CountDownLatch done = new CountDownLatch(concurrency);
        
        for (int i = 0; i < concurrency; i++) {
            executor.execute(() -> {
                // 타이머에게 준비를 마쳤음을 알림
                ready.countDown();
                try {
                    // 모든 작업자 스레드가 준비될 떄까지 기다림
                    start.await();
                    action.run();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    // 타이머에게 작업을 마쳤음을 알린다.
                    done.countDown();
                }
            });
        }
        
        ready.await(); // 모든 작업자가 준비될 때까지 기다린다.
        long startNanos = System.nanoTime();
        start.countDown(); // 작업자들을 깨운다.
        down.await(); // 모든 작업자가 일을 끝마치기를 기다린다.
        return System.nanoTime(); - startNanos;
}
``` 

- time 메서드에 넘겨진 실행자(executor)는 concurrency 매개변수로 지정한 동시성 수준만큼의 스레드를 생성할 수 있어야 함
  - 그렇지 못하면 이 메서드는 결코 끝나지 않을 것 (= 스레드 기아 교착상태 (thread starvation deadlock))
- InterruptedException을 캐치한 작업자 스레드는 Thread.currentThread(), interrupt() 관용구를 사용해 인터럽트(interrupt)를 되살리고 자신은 run 메서드에서 빠져나온다.
  - 이렇게 해야 실행자가 인터럽트를 적절하게 처리할 수 있음
- 시간을 잴 때는 System.currentTimeMillis 보다는 System.nanoTime 
  - System.nanoTime은 더 정밀하며 시스템의 실시간 시계의 시간 보정에 영향받지 않는다.

- 3개의 CountDownLatch는 CyclicBarrier(or Phaser) 인스턴스 하나로 대체할 수 있음
  - 이렇게 하면 코드가 더 명료해지겠지만 이해하기는 더 어려워 질 것
  
### wait or notify 써야할 레거시의 경우

- wait 메서드는 스레드가 어떤 조건이 충족되기를 기다리게 할 때 사용
- 락 객체의 wait 메서드는 반드시 그 객체를 잠근 동기화 영역 안에서 호출해야 함
  
- wait를 사용하는 표준 방식
```java
synchronized(obj) {
    while(<조건이 충족되지 않았다>)
        obj.wait(); // (락을 놓고, 깨어나면 다시 잡는다.)
        
    ... // 조건이 충족됐을 때의 동작을 수행함
}
```

- wait 메서드를 사용할 때는 반드시 대기 반복문(wait loop) 관용구를 사용하라
  - 반복문 밖에서는 절대로 호출하지 말자.

- 대기 전에 조건을 검사해 이미 충족되었다면 wait를 건너뛰게 한 것은 응답 불가 상태를 예방하는 조치
  - 만약 조건이 이미 충족되었는데 스레드가 notify(or notifyAll) 메서드를 먼저 호출한 후 대기 상태로 빠지면 다시 스레드를 깨울 수 있다고 보장할 수 없음

- 한편, 대기 후에 조건을 검사해 조건이 충족되지 않았다면 다시 대기하게 하는 것은 안전 실패를 막는 조치
  - 만약 조건이 충족되지 않았는데 스레드가 동작을 이어가면 락이 보호하는 불변식을 깨뜨릴 위험이 있음

- 조건이 만족되지 않아도 스레드가 깨어날 수 있는 상황
  - 스레드가 notify를 호출한 다음, 대기 중이던 스레드가 깨어나는 사이에 다른 스레드가 락을 얻어 그 락이 보호하는 상태를 변경한다.
  - 조건이 만족되지 않았음에도 다른 스레드가 실수로 혹은 악의적으로 notify를 호출한다. 
    - 공개된 객체를 락으로 사용해 대기하는 클래스는 이런 위험에 노출된다.
    - 외부에 노출된 객체의 동기화된 메서드 안에서 호출하는 wait는 모두 이 문제의 영향을 받는다.
  - 깨우는 스레드는 지나치게 관대해서, 대기 중인 스레드 중 일부만 조건이 충족되어도 notifyAll을 호출해 모든 스레드를 깨울 수도 있다.
  - 대기 중인 스레드가 드물게 notify 없이도 깨어나는 경우가 있다. 허위 각성이라는 현상이다.
    - [허위 각성](https://en.wikipedia.org/wiki/Spurious_wakeup)
  
- 이와 관련해 notify 와 notifyAll 중 무엇을 선택하느냐에 대한 문제도 있음
  - 일반적으로 언제나 notifyAll을 사용하라는 조언이 합리적
  - 깨어나야 하는 모든 스레드가 깨어남을 보장하니 항상 정확한 결과를 얻을 것
  - 다른 스레드까지 깨어날 수도 있긴 하지만 프로그램의 정확성에는 영향을 주지 않을 것
  - 깨어난 스레드들은 기다리던 조건이 충족되었는지 확인하여 충족되지 않았다면 다시 대기할 것
  
- 모든 스레드가 같은 조건을 기다리고 조건이 한 번 충족될 때마다 단 하나의 스레드만 혜택을 받을 수 있다면 notifyAll 대신 notify를 사용해 최적화 가능
  - 하지만 만족한다하더라도 notifyAll을 사용해야 함
    - 외부로 공개된 객체에 대해 실수로 혹은 악의적으로 notify를 호출하는 상황에 대비하기 위해 wait를 반복문 안에서 호출했듯, notify 대신 notifyAll을 사용하면 관련없는 스레드가 실수로 혹은 악의적으로 wait를 호출하는 공격으로부터 보호할 수 있음
  - 잘못하면 중요한 notify를 삼켜버린다면 꼭 깨어났어야 할 스레드들이 영원히 대기하게 될 수 있음
  
## 정리

- wait와 notify를 직접 사용하는 것을 동시성 '어셈블리 언어'로 프로그래밍하는 것에 비유할 수 있음
- concurrent 패키지는 고수준 언어에 비유할 수 있음
- 코드를 새로 작성한다면 wait와 notify를 쓸 이유가 거의없다.
- 레거시 코드를 유지보수해야한다면 wait는 표준 관용구에 따라 while문 안에서 호출
- 일반적으로 notify 와 notifyAll을 사용
- 혹시라도 notify를 사용한다면 응답 불가 상태에 빠지지 않도록 각별히 유의하자.


  