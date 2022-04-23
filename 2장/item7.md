
# 다 쓴 객체 참조를 해제하라
> 메모리 누수는 철저한 코드 리뷰나 힙 프로파일러 같은 디버깅 도구를 동원해야만 발견되기도 한다.
> 그래서 이런 종류의 문제는 예방법을 익혀두는 것이 매우 중요하다.

Java GC를 신뢰하여 메모리 관리에 더 이상 신경 쓰지 않아도 된다고 오해할 수 있는데, 사실이 아니다.

```java
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        return elements[--size];
    }

    /**
     * 원소를 위한 공간을 적어도 하나 이상 확보한다.
     * 배열 크기를 늘려야 할 때마다 대략 두 배씩 늘린다.
     */
    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```
메모리 누수로, 이 스택을 사용하는 프로그램을 오래 실행하면 점차 GC 활동과 메모리 사용량이 늘어나 결국 성능이 저하될 것이다.  
심한 경우 디스크 페이징이나 OutOfMemoryError를 일으켜 프로그램이 예기치 않게 종료되기도 한다.

위 스택에서 메모리 누수는
- 스택이 커졌다가 줄어들었을 때 스택에서 꺼내진 객체들을 가비지 컬렉터가 회수 하지 않는다.
- 스택이 그 객체들의 다 쓴 참조(obsolete reference)를 여전히 가지고 있기 때문
- elements 배열의 '활성 영역' 밖의 참조들이 모두 여기에 해당한다.

객체 참조 하나를 살려두면 가비지 컬렉터는 그 객체뿐 아니라 그 객체가 참조하는 모든 객체를 회수해가지 못한다.
단 몇 개의 객체가 매우 많은 객체를 회수되지 못하게 할 수 있고 잠재적으로 성능에 악영향을 줄 수 있다.

```java
public class Stack {
    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        Object result = elements[--size];
        elements[size] = null; // 다 쓴 참조 해제
        return result;
    }
}
```

## 객체 참조 해제
해당 참조를 다 썼을 때 null 처리(참조 해제)를 하도록한다.  

다 쓴 참조를 null 처리하면 다른 이점도 따라온다.  
**만약 null 처리한 참조를 실수로 사용하려 하면 프로그램은 즉시 NullPointerException을 던지며 종료된다.**

그러나 모든 객체를 일일이 null 처리 하는 것은 바람직하지 않고 코드를 지저분하게 만든다
**객체 참조를 null 처리하는 일은 예외적인 경우여야 한다.**

다 쓴 참조를 해제하는 가장 좋은 방법
- 참조를 담은 변수를 유효 범위(scope) 밖으로 밀어내는 것이다.
- 변수의 범위를 최소가 되게 정의했다면(아이템 57) 이 일은 자연스럽게 이루어진다.



### 메모리 누수에 취약한 이유
- 스택이 자기 메모리를 직접 관리하기 때문
    - 객체 자체가 아니라 객체 참조를 담는 elements 배열로 저장소 풀을 만들어 원소를 관리
    - GC가 보기에는 비활성 영역에서 참조하는 객체도 똑같이 유효한 객체다.
- 자기 메모리를 직접 관리하는 클래스라면 프로그래머는 항시 메모리 누수에 주의해야 한다.
    - 원소를 다 사용한 즉시 원소가 참조한 객체들을 다 null 처리해줘야 한다.

### 캐시 메모리
객체 참조를 캐시에 넣고 캐시를 비우지 않을 경우 메모리 누수가 발생될 수 있다.

방법1
- 외부에서 키(key)를 참조하는 동안만 엔트리가 살아 있는 캐시가 필요한 상황이라면 WeakHashMap을 사용해 캐시를 만든다.
방법2
- 캐시 엔트리의 유효 기간을 정확히 정의하기 어렵기 때문에 시간이 지날수록 엔트리의 가치를 떨어뜨리는 방식을 사용
- ScheduledThreadPoolExecutor 같은 백그라운드 스레드를 활용하거나 캐시에 새 엔트리를 추가할 때 부수작업 수행
- LinkedHashMap은 removeEldestEntry 메서드를 써서 처리
- 더 복잡한 캐시를 만들고 싶다면 java.lang.ref 패키지를 직접 활용

cf) [Java – Collection – Map – WeakHashMap (약한 참조 해시맵)](https://blog.breakingthat.com/2018/08/26/java-collection-map-weakhashmap/)

### 리스너(listener), 콜백(callback)

클라이언트가 콜백을 등록만하고 명확히 해지 하지 않을경우
- 콜백이 지속적으로 누적되며 메모리 누수 발생

방법
- 콜백을 약한 참조(weak reference)로 저장하여 GC가 즉시 수거해가도록 한다.
- ex) WeakHashMap에 키로 저장
