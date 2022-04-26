
# finalizer와 cleaner 사용을 피하라

> cleaner(자바 8까지는 finalizer)는 안전망 역할이나 중요하지 않은 네이티브 자원 회수용으로만 사용하자.  
> 물론 이런 경우라도 불확실성과 성능 저하에 주의해야 한다.  

finalizer는 예측할 수 없고, 상황에 따라 위험할 수 있어 일반적으로 불필요하다  
오동작, 낮은 성능, 이식성 문제의 원인이 되기도 한다.  
cleaner는 finalizer보다는 덜 위험하지만, 여전히 예측할 수 없고, 느리고, 일반적으로 불필요하다  

## finalizer, cleaner 문제점
### 문제점1
finalizer와 cleaner는 즉시 수행된다는 보장이 없다.  
즉, finalizer와 cleaner로는 제때 실행되어야 하는 작업은 절대 할 수 없다.  
ex) 파일닫기를 finalizer나 cleaner에 맡기면 중대한 오류를 일으킬 수 있다. 시스템이 finalizer나 cleaner 실행을 게을리해서 파일을 계속 열어 둔다면 새로운 파일을 열지 못해 프로그램이 실패할 수 있다.

finalizer나 cleaner를 얼마나 신속히 수행할지는 전적으로 GC 알고리즘에 달렸으며, 이는 GC 구현마다 천차만별이다.  
자바 언어 명세는 finalizer나 cleaner의 수행 시점뿐 아니라 수행 여부조차 보장하지 않는다.  
따라서 프로그램 생애주기와 상관없는, **상태를 영구적으로 수정하는 작업에서는 절대 finalizer나 cleaner에 의존해서는 안 된다.**
ex) 데이터베이스 같은 공유 자원의 영구 락(lock) 해제를 finalizer나 cleaner에 맡겨 놓으면 분산 시스템 전체가 서서히 멈출 것이다.

- System.gc
- System.runFinalization 
메서드는 finalizer와 cleaner가 실행될 가능성을 높여줄 수는 있으나, 보장해주진 않는다.  

- System.runFinalizersOnExit
- Runtime.runFinalizersOnExit
이를 보장해주는 메서드 2개가 있었지만, 이 두 메서드는 심각한 결함 때문에 수십 년간 지탄받아 왔다.(ThreadStop)

### 문제점2
finalizer 동작 중 발생한 예외는 무시되며, 처리할 작업이 남았더라도 그 순간 종료된다.  
경고나 스택 트레이스도 남지 않기 때문에 더욱 위험하다  

### 문제점3
심각한 성능문제도 동반한다.  
AutoCloseable 객체를 생성하고 GC가 수거하기까지  
try-with-resources로 12ns가 걸린 반면, finalizer를 사용하면 550ns가 걸린다.   
finalizer가 GC의 효율을 떨어뜨리기 때문이다.  
cleaner도 클래스의 모든 인스턴스를 수거하는 형태로 사용하면 성능은 finalizer와 비슷하다.  

### 문제점4
finalizer 공격에 노출되어 심각한 보안 문제를 일으킬 수도 있다.  
생성자나 직렬화 과정(readObject와 readResolve 메서드, 12장)에서 예외가 발생하면, 이 생성되다 만 객체에서 악의적인 하위 클래스의 finalizer가 수행될 수 있게 된다.  
이 finalizer는 정적필드에 자신의 참조를 할당하여 GC가 수집하지 못하게 막을 수 있다.  
**객체 생성을 막으려면 생성자에서 예외를 던지는 것만드로 충분하지만, finalizer가 있다면 그렇지 않다.**
**finzl이 아닌 클래스를 finalizer 공격으로부터 방어하려면 아무 일도 하지 않는 finalize 메서드를 만들고 final로 선언하자.**


## 대안
finalizer나 cleaner를 대신해줄 묘안은 무엇인가?  
AutoCloseable을 구현해주고, 클랄이언트에서 인스턴스를 다 쓰고 나면 close 메서드를 호출하면 된다.  
(예외가 발생해도 제대로 종료되도록 try-with-resources를 사용해야 한다.)  
close 메서드에서 이 객체는 더 이상 유효하지 않음을 필드에 기록하고, 다른 메서드는 이 필드를 검사해서 객체가 닫힌 후에 불렸다면 IllegalStateException을 던지는 것이다.  


## 적절항 방법

### 자원의 소유자가 close 메서드를 호출하지 않는 것에 대비한 안전망 역할  
cleaner, finalizer가 즉시 호출되리라는 보장은 없지만. 클라이언트가 자원 회수를 늦게라도 해주는 것이 아예 안 하는 것보다는 낫다.
이런 안전망 역할의 finalizer를 작성할 때는 그럴만한 값어치가 있는지 심사숙고  하자.  

### 네이티브 피어 객체
native peer 객체는 네이티브 메서드를 통해 기능을 위임한 객체  
네이티브 피어는 자바 객체가 아니니 GC는 그 존재를 알지 못한다.  
그 결과 자바 피어를 회수 할 때 네이티브 객체까지 회수하지 못한다.  

성능저하를 감당할 수 있고 네이티브 피어가 심각한 자원을 가지고 있지 않을때 finalizer, cleaner로 처리  
성능저하를 감당할 수 없거나 네이티브 피어가 사용하는 자원을 즉시 회수해야 한다면 close 메서드를 사용할 것

```java
// 코드 8-1 cleaner를 안전망으로 활용하는 AutoCloseable 클래스 (44쪽)
public class Room implements AutoCloseable {
    private static final Cleaner cleaner = Cleaner.create();

    // 청소가 필요한 자원. 절대 Room을 참조해서는 안 된다!
    private static class State implements Runnable {
        int numJunkPiles; // Number of junk piles in this room

        State(int numJunkPiles) {
            this.numJunkPiles = numJunkPiles;
        }

        // close 메서드나 cleaner가 호출한다.
        @Override public void run() {
            System.out.println("Cleaning room");
            numJunkPiles = 0;
        }
    }

    // 방의 상태. cleanable과 공유한다.
    private final State state;

    // cleanable 객체. 수거 대상이 되면 방을 청소한다.
    private final Cleaner.Cleanable cleanable;

    public Room(int numJunkPiles) {
        state = new State(numJunkPiles);
        cleanable = cleaner.register(this, state);
    }

    @Override 
    public void close() {
        cleanable.clean();
    }
}
```

static으로 선언된 중첩 클래스인 State는 cleaner가 방을 청소할 때 수거할 자원들을 담고 있다.  
쓰레기 수를 뜻하는 numJunkPiles 필드가 수거할 자원에 해당한다. 더 현실적으로 만들려면 이 필드는 네이티프 피어를 가리키는 포인터를 담은 final long 변수여야 한다.  

State 인스턴스는 절대로 Room 인스턴스를 참조해서는 안된다. Room 인스턴스를 참조할 경우 순환참조가 생겨 가비지 컬렉터가 Room 인스턴스를 회수해갈 기회가 오지 않는다.  
정적이 아닌 중첩 클래스는 자동으로 바깥 객체의 참조를 갖게 되어 이와 같은 문제가 발생한다.  

```java
public class Adult {
    public static void main(String[] args) {
        try (Room myRoom = new Room(7)) {
            System.out.println("안녕~");
        }
    }
}
```
Room의 cleaner는 단지 안전망으로만 쓰였다.  
클라이언트가 모든 Room 생성을 try-with-resources 블록으로 감쌌다면 자동 청소는 전혀 필요하지 않다.  
"안녕~"을 출력한 후, 이어서 "방 청소"를 출력한다.

```java
public class Teenager {
    public static void main(String[] args) {
        new Room(99);
        System.out.println("Peace out");

//      System.gc();
    }
}
```
"아무렴"에 이어 "방 청소"는 출력 되지 않는다.
System.gc()를 추가하는 것으로 종료전에 "방 청소"를 출력할 수 있지만, 이 방식에 의존해서는 절대 안된다.

cleaner의 동작은 예측할 수 없다.
> System.exit을 호출 할때의 cleaner 동작은 구현하기 나름이다.  
> 청소가 이루어질지는 보장하지 않는다.
