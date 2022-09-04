# item 83 지연 초기화는 신중히 사용하라

---

### 지연 초기화

- 지연 초기화(lazy initialization)는 필드의 초기화 시점을 그 값이 처음 필요할 때까지 늦추는 기법
  - 값이 전혀 쓰이지 않으면 초기화도 결코 일어나지 않음
    - 정적 필드와 인스턴스 필드에 모두 사용가능
  - 지연 초기화는 주로 최적화 용도로 쓰임
- 클래스와 인스턴스 초기화 때 발생하는 위험한 순환 문제를 해결하는 효과도 있음
- 최선의 조언은 '필요할 때까지는 지연 초기화 하지말라'
- 지연 초기화는 양날의 검
  - 클래스 혹은 인스턴스 생성 시의 초기화 비용은 줄지만 그 대신 지연 초기화하는 필드에 접근하는 비용은 커짐
  - 지연 초기화하려는 필드들 중 결국 초기화가 이루어지는 비율에 따라 실제 초기화에 드는 비용에 따라 초기화된 각 필드를 얼마나 빈번히 호출하느냐에 따라 지연초기화가 성능을 느리게 할 수 있음

### 지연 초기화가 필요한 경우

- 해당 클래스의 인스턴스 중 필드를 사용하는 인스턴스의 비율이 낮은 반면, 그 필드를 초기화하는 비용이 크다면 지연 초기화가 제 역할을 해줄 것
  - 하지만 안타깝게도 정말 그런지 알 수 있는 방법이 측정하는 것임
  

- 멀티스레드 환경에서는 지연 초기화 하기가 까다롭다.
  - 지연 초기화하는 필드를 둘 이상의 스레드가 공유한다면 어떤 형태로든 반드시 동기화해야 함
- 대부분의 상황에서 일반적 초기화가 지연 초기화보다 나음

```java 
// 일반 초기화
private final FieldType field = computeFieldValue();
```

- 지연 초기화가 [초기화 순환성](https://github.com/JavaBookStudy/JavaBook/issues/45) 을 깨뜨릴 것 같으면 synchronized를 단 접근자를 사용

```java
// synchronized 접근자 방식
private FieldType field;

private synchronized FieldType getField() {
    if (field == null)
        field = computeFieldValue();
    
    return field;
}
```

- 이상의 두 관용구는 정적 필드에도 똑같이 적용 (static 추가해서)

- 성능 때문에 정적 필드를 지연 초기화해야 한다면 지연 초기화 홀더 클래스 관용구를 사용하자.

```java
// 정적 필드용 지연 초기화 홀더 클래스 관용구
private static class FieldHolder {
    static final FieldType field = computeFieldValue();
}

private static FieldType getField() {
    return FieldHolder.field;
}
```

- 성능 때문에 인스턴스 필드를 지연 초기화해야 한다면 이중검사(double-check) 관용구를 사용
  - 동기화 비용 없애 줌
  - 필드의 값을 두 번 검사하는 방식
    - 한 번은 동기화 없이 계산하고 두 번째는 동기화 하여 검사
    - 두 번째 검사에서도 초기화되지 않았을 때만 필드를 초기화
    - 초기화된 이후로는 동기화하지 않으므로 필드는 반드시 volatile로 선언한다.
  
```java
// 인스턴스 필드 지연 초기화용 이중검사 관용구
private volatile FieldType field;

private FieldType getField() {
    FieldType result = field;
    if (result != null) { // 첫 번째 검사
        return result;
    }
    
    synchronized(this) { // 두 번째 검사
        if (field == null)
            field = computeFieldValue();
        return field;
    }
}
```

- 정적 필드에는 이중 검사 보다는 지연 초기화 홀더 클래스 방식이 낫다.

- 초기화가 중복해서 일어나도 되는 경우
```java
// 단일검사 관용구 - 초기화 중복 가능한 경우
private volatile FieldType field;

private FieldType getField() {
    FieldType result = field;
    if (result == null)
        field = result = computeFieldValue();
    return result;
}
```

- 모든 스레드가 필드의 값을 다시 계산해도 상관없고 필드 타입이 long과 double을 제외한 다른 기본 타입이라면 단일검사 필드선언에서 volatile 한정자를 없애도 된다.
  - 짜릿한 단일검사(racy single-check) 관용구라 불림
    - 관용구는 어떤 환경에서는 필드 접근 속도를 늘려주지만, 초기화가 스레드당 최대 한번 더 이뤄질 수 있음
      - 보통은 거의 안 쓴다.
  
## 정리

- 대부분의 필드는 지연시키지말고 곧바로 초기화
- 성능 or 위험한 초기화 순환을 막기 위해 지연 초기화를 써야한다면 올바른 지연 초기화 기법을 사용하자.
