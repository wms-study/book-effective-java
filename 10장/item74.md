
# [아이템 74] 메서드가 던지는 모든 예외를 문서화하라

검사 예외는 항상 따로따로 선언하고, 각 예외가 발생하는 상황을 자바독의 @throws 태그를 사용하여 정확히 문서화하자.

```java
// io.netty.util.NetUtil

    /**
     * Converts 4-byte or 16-byte data into an IPv4 or IPv6 string respectively.
     *
     * @throws IllegalArgumentException
     *         if {@code length} is not {@code 4} nor {@code 16}
     */
    public static String bytesToIpAddress(byte[] bytes) {
        return bytesToIpAddress(bytes, 0, bytes.length);
    }
```

Exception이나 Throwable을 던진다고 선언해서는 안 된다.  
- 각 메서드 사용자에게 각 예외에 대처할 수 있는 힌트를 못준다.  
- 같은 맥락에서 발생할 여지가 있는 다른 예외들까지 삼켜버림.
- main은 JVM만이 호출하므로 괜찮다.

비검사 예외도 검사 예외처럼 문서화해두면 좋다.
- 일어날 수 있는 오류들을 알려주면 해당 오류가 나지 않도록 코딩하게 된다.
- 잘 정비된 비검사 예외 문서는 사실상 그 메서드를 성공적으로 수행하기 위한 전제조건이 된다.
- 발생 가능한 비검사 예외를 문서로 남기는 일은 인터페이스 메서드에서 특히 중요하다.  
이 조건이 인터페이스의 일반 규약에 속하게 되어 그 인터페이스를 구현한 모든 구현체가 일관되게 동작하도록 해주기 때문이다.

    ```java
    // io.netty.handler.codec.DefaultHeaders

    public interface NameValidator<K> {
        /**
        * Verify that {@code name} is valid.
        * @param name The name to validate.
        * @throws RuntimeException if {@code name} is not valid.
        */
        void validateName(K name);

        @SuppressWarnings("rawtypes")
        NameValidator NOT_NULL = new NameValidator() {
            @Override
            public void validateName(Object name) {
                checkNotNull(name, "name");
            }
        };
    }
    ```

- 검사 예외 @throws 태그로 문서화하되,  
비검사 예외는 메서드 선언의 throws 목록에 넣지 말자.  
검사, 비검사에 따라 사용자가 해야 할 일이 달라지므로 이 둘을 확실히 구분해주는 게 좋다.

한 클래스에 정의된 많은 메서드가 같은 이유로 같은 예외를 던진다면 클래스 설명에 추가하는 방법도 있다.
