
# [Item35] 비트 필드 대신 EnumSet을 사용하라

## 정수 열거 상수를 이용한 비트 필드

```java
public class Text {
    public static final int STYLE_BOLD          = 1 << 0; // 1 -> 00000000 00000000 00000000 00000001
    public static final int STYLE_ITALIC        = 1 << 1; // 2 -> 00000000 00000000 00000000 00000010
    public static final int STYLE_UNDERLINE     = 1 << 2; // 4 -> 00000000 00000000 00000000 00000100
    public static final int STYLE_STRIKETHROUGH = 1 << 3; // 8 -> 00000000 00000000 00000000 00001000
  
  // 매개변수 styles는 0개 이상의 STYLE_ 상수를 비트별 OR한 값이다.
    public void applyStyles(int styles) {}
  
  // 00000000 00000000 00000000 00001001 -> 9
  // STYLE_BOLD | STYLE_STRIKETHROUGH
}
```

열거한 값들이 집합으로 사용될 경우, 예전에는 각 상수에 서로 다른 2의 거듭제곱 값을 할당한 정수 열거 패턴을 사용해왔다.  
다음과 같은 식으로 비트별 OR를 사용해 여러 상수를 하나의 집합으로 모을 수 있으며, 이렇게 만들어진 집합을 비트 필드(bit field)라 한다.

`text.applyStyles(STYLE_BOLD | STYLE_STRIKETHROUGH);`

비트 필드를 사용하면 비트별 연산을 사용해 합집합과 교집합 같은 집합 연산을 효율적으로 수행할 수 있다.  
하지만 정수 열거 상수의 단점을 그대로 지니며 추가로 다음과 같은 문제까지 안고 있다.
- 비트 필드 값이 그대로 출력되면 단순한 정수 열거 상수를 출력할 때보다 해석하기 훨씬 어렵다.
- 비트 필드 하나에 녹아 있는 모든 원소를 순회하기도 까다롭다.
- 최대 몇 비트가 필요한지를 API 작성시 미리 예측하여 적절한 타입(보통은 int나 long)을 선택해야 한다.

## EnumSet 클래스를 사용한 비트 필드

EnumSet 클래스는 열거 타입 상수의 값으로 구성된 집합을 효과적으로 표현해준다.  
Set 인터페이스를 완벽히 구현하며, 타입 안전하고, 다른 어떤 Set 구현체와도 함께 사용할 수 있다.

```java
// 코드 36-2 EnumSet - 비트 필드를 대체하는 현대적 기법 (224쪽)
public class Text {
    public enum Style {BOLD, ITALIC, UNDERLINE, STRIKETHROUGH}

    // 어떤 Set을 넘겨도 되나, EnumSet이 가장 좋다.
    public void applyStyles(Set<Style> styles) {
        System.out.printf("Applying styles %s to text%n",
                Objects.requireNonNull(styles));
    }

    // 사용 예
    public static void main(String[] args) {
        Text text = new Text();
        text.applyStyles(EnumSet.of(Style.BOLD, Style.ITALIC));
    }
}
```

#### EnumSet의 단점
- 불변 EnumSet을 만들 수 없다.
- 이로인해서 Collections.unmodifiableSet으로 EnumSet을 감싸 불변으로 사용하자.
