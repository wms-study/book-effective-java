
# [아이템 73] 추상화 수준에 맞는 예외를 던지라

상위 계층에서는 저수준의 예외를 잡아 자신의 추상화 수준에 맞는 예외로 바꿔 던져야 한다.  
이를 예외 번역(exception translation)이라 한다.  

```java
try {
    // 저수준 추상화를 이용한다.
} catch(LowerLevelException e) {
    // 추상화 수준에 맞게 번역한다.
    throw new HigherLevelException();
}
```

저수준 예외가 디버깅에 도움이 된다면 예외 연쇄(exception chaining)를 사용하는게 좋다.  
근본 원인(cause)인 저수준 예외를 고수준 예외에 실어 보내는 방식.

```java
try {
    // 저수준 추상화를 이용한다.
} catch(LowerLevelException cause) {
    // 저수준 예외를 고수준 예외에 실어 보낸다.
    throw new HigherLevelException(cause);
}
```

또는 Throwable의 initCause 메서드를 이용해 '원인'을 직접 못박을 수 있다.

```java
public class Throwable implements Serializable {
    private Throwable cause = this;

    public synchronized Throwable initCause(Throwable cause) {
        if (this.cause != this) {
            throw new IllegalStateException("Can't overwrite cause with " + Objects.toString(cause, "a null"), this);
        } else if (cause == this) {
            throw new IllegalArgumentException("Self-causation not permitted", this);
        } else {
            this.cause = cause;
            return this;
        }
    }
}
```
