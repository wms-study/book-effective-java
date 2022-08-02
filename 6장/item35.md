
# [Item35] ordinal 메서드 대신 인스턴스 필드를 사용하라

열거 타입에서 몇 번째 위치인지를 반환하는 ordinal 메서드 제공한다.

## ordinal 메서드 사용

```java
public enum Ensemble {
  SOLO, DUET, TRIO, QUARTET, QUINTET,
  SEXTET, SEPTET, OCTET, NONET, DECTET;

  public int numberOfMusicians() { return ordinal() + 1; }
}
```

#### 문제점
- 상수 선언 순서를 바꾸는 순간 numberOfMusicians가 오동작한다.
- 이미 사용 중인 정수와 값이 같은 상수는 추가할 방법이 없다.
  - 8중주(octet) 상수가 이미 있으니 똑같이 8명이 연주하는 복4중주(double quartet)는 추가할 수 없다.
- 값을 중간에 비워둘 수도 없다.

## 인스턴스 필드 사용

```java
// 인스턴스 필드에 정수 데이터를 저장하는 열거 타입 (222쪽)
public enum Ensemble {
    SOLO(1), DUET(2), TRIO(3), QUARTET(4), QUINTET(5),
    SEXTET(6), SEPTET(7), OCTET(8), DOUBLE_QUARTET(8),
    NONET(9), DECTET(10), TRIPLE_QUARTET(12);

    private final int numberOfMusicians;
    Ensemble(int size) { this.numberOfMusicians = size; }
    public int numberOfMusicians() { return numberOfMusicians; }
}
```

> 대부분의 프로그래머는 ordinal 메서드를 쓸 일이 없다.
> 이 메서드는 EnumSet과 EnumMap 같이 열거 타입기반의 범용 자료구조에 쓸 목적으로 설계되었다.

**따라서 위와같은 용도가 아니라면 ordinal 메서드는 절대 사용하지 말자.**

