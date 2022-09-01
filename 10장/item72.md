
# [아이템 72] 표준 예외를 사용하라

표준 예외를 사용하여 코드를 재사용하는 것이 좋으며, 자바 라이브러리는 쓰기 충분한 수의 예외를 제공한다.

## 장점

- API가 다른 사람이 익히고 사용하기 쉬워진다.  
익숙해진 규약을 그대로 따르기 때문.
- 예외 클래스 수가 적을수록 메모리 사용량도 줄고 클래스를 적재하는 시간도 적게 걸린다.

## 널리 재사용되는 예외

| 예외                            | 주요 쓰임                                      |
| ------------------------------- | ---------------------------------------------- |
| IllegalArgumentException        | 허용하지 않는 값이 인수로 건네졌을 때          |
| IllegalStateException           | 객체가 메서드를 수행하기 적절하지 않은 상태    |
| NullPointerException            | null을 허용하지 않는 메서드에 null을 건넸을 때 |
| IndexOutOfBoundsException       | 인덱스 범위를 넘어섰을 때                      |
| ConcurrentModificationException | 허용하지 않는 동시 수정이 발견되었을 때        |
| UnsupportedOperationException   | 호출한 메서드를 지원하지 않을 때               |

## 주의점

**Exception, RuntimeException, Throwable, Error는 직접 재사용하지 말자.**  
여러 성격의 예외들을 포괄하는 클래스이므로 안정적으로 테스트할 수 없다.

인수 값이 무엇이었든 어차피 실패했을 거라면 IllegalStateException,  
그렇지 않으면 IllegalArgumentException
