
# [아이템 76] 가능한 한 실패 원자적으로 만들라

**호출된 메서드가 실패하더라도 해당 객체는 메서드 호출 전 상태를 유지해야 한다.**  
이러한 특성을 실패 원자적(failure-atomic)이라고 한다.  


## 실패 원자성을 얻는 방법

1. 불변 객체 메서드
- 불변 객체는 태생적으로 실패 원자적이다.
- 기존 객체가 불안정한 상태에 빠지는 일은 결코 없다.

2. 가변 객체 메서드
- 작업 수행에 앞서 매개변수의 유효성을 검사
- 객체 내부 상태를 변경하기 전에 잠재적 예외의 가능성을 검증

```java
package java.util;

public class Stack<E> extends Vector<E> {
    private static final long serialVersionUID = 1224463164541339165L;

    public Stack() {
    }

    public synchronized E peek() {
        int len = this.size();
        if (len == 0) {
            throw new EmptyStackException();
        } else {
            return this.elementAt(len - 1);
        }
    }
}
```

3. 객체의 임시 복사본으로 작업 수행
- 객체의 임시 복사본에서 작업을 수행한 다음, 작업이 성공적으로 완료되면 원래 객체와 교체  

4. 실패를 가로채 복구
- 작업 도중 발생하는 실패를 가로채는 복구 코드를 작성하여 작업 전 상태로 되돌리는 방법  


실패 원자성은 일반적으로 권장되나 항상 달성할 수 있는 것은 아니다.  
- ConcurrentModificationException으로 인해서 객체의 일관성이 깨졌을 경우 

메서드 명세에 기술한 예외라면 예외가 발생하더라도 객체의 상태는 메서드 호출 전과 똑같이 유지돼야 한다는 것이 기본 규칙이다.  
이 규칙을 지키지 못한다면 실패 시의 객체 상태를 API 설명에 명시해야 한다.  
~~이것이 이상적이나, 아쉽게도 지금의 API 문서 상당 부분이 잘 지키지 않고 있다.~~
