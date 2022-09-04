# item 78 공유 중인 가변 데이터는 동기화에 사용하라

---

### synchronized 키워드

- 해당 메서드나 블록을 한번에 한 스레드씩 수행하도록 보장
  - 베타적 실행
    - 즉, 한 스레드가 변경하는 중이라서 상태가 일관되지 않은 순간의 객체를 다른 스레드가 보지 못하게 막는 용도
    - 한 객체가 일관된 상태를 가지고 생성 & 객체에 접근하는 메서드는 해당 객체에 락(lock)을 건다.
    - 락을 건 메서드는 객체의 상태를 확인하고 필요하면 수정
    - 즉, 객체를 하나의 일관된 상태에서 다른 일관된 상태로 변화시킨다.
    - 동기화를 제대로 사용하면 어떤 메서드도 객체의 상태가 일관되지 않은 상태를 볼 수 없다.
  - 동기화 없이는 한 스레드가 만든 변화를 다른 스레드에서 확인하지 못할 수 있음
    
### 동기화 역할

- 일관성이 깨진 상태를 볼 수 없게 함
- 동기화된 메서드나 블록에 들어간 스레드가 같은 락의 보호 하에 수행된 모든 이전 수정의 최종결과를 보게 함

### 언어 명세상 long & double 외의 변수를 일고 쓰는 동작은 원자적(atomic)이다.

- 원자적?
  - 여러 스레드가 같은 변수를 동기화 없이 수정하는 중이라도 항상 어떤 스레드가 정상적으로 저장한 값을 온전히 읽어 옴을 보장한다는 뜻
- 그렇다면 원자적일 때는 동기화하지 말아야겠다?
  - 아주 위험한 발상
    - 자바 언어 명세는 스레드가 필드를 읽을 때 항상 '수정이 완전히 반영된' 값을 얻는다고 보장하지만,
    - 한 스레드가 저장한 값이 다른 스레드에게 '보이는가'는 보장하지 않음
  
## 따라서 동기화는 배타적 실행뿐 아니라 스레드 사이의 안정적인 통신에 꼭 필요

주관적 해석
- 배타적 실행 -> 락
- 스레드 사이의 안정적 통신 -> 한 스레드가 저장한 값이 다른 스레드에 보이는가

책 설명
- 한 스레드가 만든 변화가 다른 스레드에게 언제 어떻게 보이는지를 규정한 자바의 메모리 모델 떄문에 필요

### 공유 중인 가변 데이터를 원자적으로 읽고 쓸 수 있을 지라도 동기화에 실패하면 처참한 결과로 이어짐

- IF 다른 스레드를 멈추는 작업이 있을 때
  - Thread stop 사용하지 말자
    - [Thread.stop 메서드는 안전하지 않아](https://docs.oracle.com/javase/6/docs/technotes/guides/concurrency/threadPrimitiveDeprecation.html) 이미 오래전에 사용 자제(Deprecated) API로 지정 (데이터 훼손 될 수 있음)
    
- 스레드를 멈추는 올바른 방법
  - 첫 번째 스레드는 자신의 boolean 필드를 폴링하면서 그 값이 true가 되면 멈춘다.
  - 필드는 false로 초기화 해놓고 다른 스레드에서 이 스레드(첫 번째 스레드)를 멈추고자 할 때 true로 변경
  - 만약 boolean 필드를 읽고 쓰는 작업이 원자적이라 생각하여 동기화를 제거한다면?
    ```java
    import java.util.concurrent.TimeUnit;
    
    public class StopThread {
        private static boolean stopRequested;
    
        public static void main(String[] ars) throws InterruptedException {
            Thread backgroundThread = new Thread(() -> {
                int i = 0;
                while(!stopRequested) {
                    System.out.println("멈춤 test start");
                    i++;
                }
    
                System.out.println("멈춤 test end");
              });
            backgroundThread.start();
    
            TimeUnit.SECONDS.sleep(1);
            stopRequested = true;
        }
    }
    ```
    - 1초 후에 종료될까? 
      - 메인 스레드가 1초 후 stopRequested를 true로 설정하면 backgroundThread는 반복문을 빠져나올 것처럼 보이지만 그렇지 않다.
      - (놀랍게도 루이스 테스트 때는 1초 후에 멈춤 why?)
  
    - OpenJDK 서버 VM이 실제로 적용하는 끌어올리기(hoisting) 최적화 기법에 의해 빠져 나올 수 없는 상태가 됨
    ```java
    // 원래 코드
    while (!stopRequested)
        i++;
    
    // 최적화 코드
    if (!stopRequested)
      while (true)
        i++;
    ```
    - 프로그램이 응답 불가 상태가 되어 더 이상 진전이 없어진다.
    - stopRequested 필드를 동기화해 접근하면 문제를 해결할 수 있다.
    
    - 1초 후에 종료되게 하는 방법 (stopRequested 동기화)
    ```java
    import java.util.concurrent.TimeUnit;
    
    public class StopThread {
        private static boolean stopRequested;
    
        private static synchronized void requestStop() {
            stopRequested = true;
        }
    
        private static synchronized boolean stopRequested() {
            return stopRequested;
        }
    
        public static void main(String[] ars) throws InterruptedException {
            Thread backgroundThread = new Thread(() -> {
                int i = 0;
                while(!stopRequested()) {
                    System.out.println("멈춤 test start");
                    i++;
                }
    
                System.out.println("멈춤 test end");
              });
            backgroundThread.start();
    
            TimeUnit.SECONDS.sleep(1);
            requestStop();
        }
    }
    ```
  
  
### 쓰기 메서드(requestStop)와 읽기 메서드(stopRequest) 모두를 동기화했음에 주목

- 쓰기만 동기화해서는 충분하지 않다.
  - 쓰기와 읽기 모두가 동기화되지 않으면 동작을 보장하지 않음
  
- 어떤 기기에서는 둘 중 하나만 동기화해도 동작하는 듯 보이지만 겉모습에 속으면 안 된다.
- 예제 두 메서드는 통신 목적으로만 사용된 것
  - 배타적 수행은 아직 안 함
  
- 매번 동기화하는 비용이 크지는 않지만 속도는 더 빠른 대안
  - stopRequested를 volatile(링크 필요)로 선언하면 동기화를 생략할 수 있음
    - 가장 최근에 기록된 값을 읽게 됨을 보장
    - [volatile이란](https://nesoy.github.io/articles/2018-06/Java-volatile)
      - 메인 메모리에 저장하겠다는 것
      
  ```java
    import java.util.concurrent.TimeUnit;
  
    public class StopThread {
    private static volatile boolean stopRequested;
  
    public static void main(String[] args) throws InterruptedException {
        Thread backgroundThread = new Thread(() -> {
            int i = 0;
            while (!stopRequested)
                i++;
        });
        backgroundThread.start();
        
        TimeUnit.SECONDS.sleep(1);
        stopRequested = true;
    }
  }
  ```
  
  - volatile은 주의해서 사용해야 한다.
  ```java
  // 잘못된 예
  public static volatile int nextSerialNumber = 0;
  
  public static int generateSerialNumber() {
    reutn nextSerialNumber++;
  }
  ```
    - 이 메서드는 매번 고유한 값을 반환할 의도로 만들어졌다.
      - 이 메서드의 상태는 nextSerialNumber 단 하나의 필드로 결정되는데 원자적 접근할 수 있고 어떤 값이든 허용
        - 따라서 굳이 동기화하지 않더라도 불변식을 보호할 수 있어 보인다.
          - 하지만 동기화 없이는 올바로 동작하지 않는다.
  - 뭐 때문에?
    - 증가 연산자(++) 떄문
      - 연산자는 코드상으로는 하나지만 실제로는 nextSerialNumber에 두 번 접근한다.
        - 값 읽기 (1번), 새로운 값 저장 (2번)
          - 만약 두 번째 스레드가 이 두 접근 사이를 비집고 들어와 값을 읽는다면 첫 번째 스레드와 똑같은 값을 돌려받게 된다.
            - 배타적 실행하지 않기 때문
  - 프로그램이 잘못된 결과를 계산해내는 이런 오류를 안전 실패(safety failure)라고 한다. 
    - generateSerialNumber 메서드에 synchronized 한정자를 붙이면 해결됨
  - 동기에 호출해도 서로 간섭하지 않고 이전 호출이 변경한 값을 읽게된다는 뜻
  - 메서드에 synchronized를 붙였다면 필드에서는 volatile을 제거해야 한다. (왜?)
  
  - 메서드 더욱 견고하게 하기 위한 방법
    - int -> long
    - nextSerialNumber의 최대값에 도달하면 예외 던지기
    - AtomicLong 사용
      - AtomicLong에는 lock-free 스레드 안전 프로그래밍을 지원하는 클래스들이 담겨있음
  
  - volatile은 동기화 중 통신 쪽만 지원하지만 AtomicLong은 배타적 실행까지 지원함
  ```java
   private static final AtomicLong nextSerialNum = new AtomicLong();
  
    public static long generateSerialNumber() {
        return nextSerialNum.getAndIncrement();
    }
  ```
  
### 근본적으로 막는 것은 애초에 가변 데이터를 공유하지 않는 것

- 불변 데이터만 공유하거나 아무것도 공유하지 말자.
- 가변 데이터는 단일 스레드에서만 쓰도록 하자.

- 위 정책을 받아들였다면 사실을 문서에 남겨 유지보수 과정에서도 지켜지도록하는 것이 중요
  - 사용하려는 프레임워크와 라이브러리를 깊게 이해하는 것도 중요
  - (가변 데이터를 변경하는)외부 코드가 인지 못한 스레드를 수행하는 복병으로 작용하는 경우도 있다.
  
- 한 스레드가 데이터를 다 수정한 후 다른 스레드에 공유할 때는 해당 객체에서 공유하는 부분만 동기화해도 된다.
  - 해당 객체를 다시 수정할 일이 생기기 전까지는 다른 스레드들은 동기화 없이 자유롭게 값을 읽어나갈 수 있음
  - 이러한 것을 사실상 불변(effectively immutable)
    - 다른 스레드에 이런 객체를 건네는 행위를 안전 발행(safe publication)이라 함
  
### 객체를 안전 발행하는 방법 (safe publication)

- 클래스 초기화 과정에서 객체를 정적 필드 
- volatile 필드
- final 필드
- 보통의 락을 통해 접근하는 필드
에 저장
  
- 동시성 컬렉션에 저장 (concurrent package or synchronized collection)


## 핵심 정리

- 여러 스레드가 가변 데이터를 공유한다면 그 데이터를 읽고 쓰는 동작은 반드시 동기화 해야 한다.
- 동기화 하지 않으면 한 스레드가 수행한 변경을 다른 스레드가 보지 못할 수 있다.
- 가변 데이터 동기화에 실패하면 응답 불가 상태 or 안전 실패로 이어질 수 있다.
  - 디버깅 난이도가 가장 높은 문제
  - 간헐 or 특정 타이밍에만 발생, VM에 따라 현상이 달라짐
- 배타적 실행이 필요없고 스레드끼리 통신만 필요하다면 volatile 한정자만으로 동기화 할 수 있다.
  - 단, 올바로 사용하기 까다롭다.