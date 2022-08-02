
# [Item37] ordinal 인덱싱 대신 EnumMap을 사용하라

배열이나 리스트에서 원소를 꺼낼 때 ordinal 메서드로 인덱스를 얻는 코드가 있다

```java
// 식물을 아주 단순하게 표현한 클래스 (226쪽)
class Plant {
  enum LifeCycle { ANNUAL, PERENNIAL, BIENNIAL }

  final String name;
  final LifeCycle lifeCycle;

  Plant(String name, LifeCycle lifeCycle) {
    this.name = name;
    this.lifeCycle = lifeCycle;
  }

  @Override public String toString() {
    return name;
  }

  public static void main(String[] args) {
    Plant[] garden = {
            new Plant("바질",    LifeCycle.ANNUAL),
            new Plant("캐러웨이", LifeCycle.BIENNIAL),
            new Plant("딜",      LifeCycle.ANNUAL),
            new Plant("라벤더",   LifeCycle.PERENNIAL),
            new Plant("파슬리",   LifeCycle.BIENNIAL),
            new Plant("로즈마리", LifeCycle.PERENNIAL)
    };

    // 코드 37-3 스트림을 사용한 코드 1 - EnumMap을 사용하지 않는다! (228쪽)
    System.out.println(Arrays.stream(garden)
            .collect(groupingBy(p -> p.lifeCycle)));

    // 코드 37-4 스트림을 사용한 코드 2 - EnumMap을 이용해 데이터와 열거 타입을 매핑했다. (228쪽)
    System.out.println(Arrays.stream(garden)
            .collect(groupingBy(p -> p.lifeCycle,
                    () -> new EnumMap<>(LifeCycle.class), toSet())));
  }
}
```

식물들을 배열 하나로 관리하고, 이들을 생애주기(한해살이, 여러해살이, 두해살이)별로 묶어보자.  
생애주기별로 3개의 집합을 만들고 각 식물을 해당 집합에 넣는다.

## ordinal을 배열 인덱스로 사용

```java
    // 코드 37-1 ordinal()을 배열 인덱스로 사용 - 따라 하지 말 것! (226쪽)
    Set<Plant>[] plantsByLifeCycleArr =
            (Set<Plant>[]) new Set[Plant.LifeCycle.values().length];
    for (int i = 0; i < plantsByLifeCycleArr.length; i++)
      plantsByLifeCycleArr[i] = new HashSet<>();
    
    for (Plant p : garden)
      plantsByLifeCycleArr[p.lifeCycle.ordinal()].add(p);
    
    // 결과 출력
    for (int i = 0; i < plantsByLifeCycleArr.length; i++) {
      System.out.printf("%s: %s%n",
              Plant.LifeCycle.values()[i], plantsByLifeCycleArr[i]);
    }
```

#### 문제점
- 제네릭과 호환되지 않으니 비검사 형변환을 수행해야함
- 배열은 각 인덱스의 의미를 모르니 출력 결과에 직접 레이블을 달아야한다.
- 정확한 정숫값을 사용한다는 것을 직접 보증해야한다.

## EnumMap 사용

```java
    // 코드 37-2 EnumMap을 사용해 데이터와 열거 타입을 매핑한다. (227쪽)
    Map<Plant.LifeCycle, Set<Plant>> plantsByLifeCycle =
            new EnumMap<>(Plant.LifeCycle.class);

    // 생에주기 키 값 세팅 
    for (Plant.LifeCycle lc : Plant.LifeCycle.values())
      plantsByLifeCycle.put(lc, new HashSet<>());
    
    // 식물별 그룹핑
    for (Plant p : garden)
      plantsByLifeCycle.get(p.lifeCycle).add(p);
    
    System.out.println(plantsByLifeCycle);
```

#### 장점
- 안전하지 않은 형변환은 쓰지 않고 
- 맵의 키인 열거 타입이 그 자체로 출력용 문자열을 제공, 직접 레이블을 달 일도 없다.
- 배열 인덱스를 계산하는 과정에서 오류가 날 가능성도 원천봉쇄된다.

EnumMap의 생성자가 받는 키 타입의 Class 객체는 한정적 타입 토큰으로, 런타임 제네릭 타입 정보를 제공한다.  
`Map<Plant.LifeCycle, Set<Plant>> plantsByLifeCycle = new EnumMap<>(Plant.LifeCycle.class);`

## 스트림을 이용한 방법

```java
    // 코드 37-3 스트림을 사용한 코드 1 - EnumMap을 사용하지 않는다! (228쪽)
    System.out.println(Arrays.stream(garden)
            .collect(groupingBy(p -> p.lifeCycle)));
```

이 코드는 EnumMap이 아닌 고유한 맵 구현체를 사용했기 때문에 EnumMap을 써서 얻은 공간과 성능 이점이 사라진다는 문제가 있다.

```java
    // 코드 37-4 스트림을 사용한 코드 2 - EnumMap을 이용해 데이터와 열거 타입을 매핑했다. (228쪽)
    System.out.println(Arrays.stream(garden)
            .collect(groupingBy(p -> p.lifeCycle,
                    () -> new EnumMap<>(LifeCycle.class), toSet())));
```

매개변수 3개짜리 Collectors.groupingBy 메서드를 사용해 mapFactory 매개변수에 원하는 맵 구현체를 명시해 호출 할 수 있다.  

#### 스트림 사용 차이점
- EnumMap 버전은 언제나 식물의 생애주기당 하나씩의 중첩 맵을 만들지만, 스트림 버전에서는 해당 생애주기에 속하는 식물이 있을 때만 만든다.

## 두 열거 타입 매핑 예시

```java
// 코드 37-5 배열등릐 배열의 인덱스에 ordinal()을 사용 - 따라 하지 말 것!
public enum Phase {
    SOLID, LIQUID, GAS;
    
    public enum Transition {
        MELT, FREEZE, BOIL, CONDENSE, SUBLIME, DEPOSIT;
        
        private static final Transition[][] TRANSITIONS = {
                {null, MELT, SUBLIME},
                {FREEZE, null, BOIL},
                {DEPOSIT, CONDENSE, null},
        };

        public static Transition from(Phase from, Phase to) {
            return TRANSITIONS[from.ordinal()][to.ordinal()];
        }
    }
}
```

#### ordinal 사용 문제
컴파일러는 ordinal과 배열 인덱스의 관계를 알 도리가 없다.
- Phase나 Phase.Transition 열거 타입을 수정하면서 상전이 표 TRANSITIONS를 함께 수정하지 않거나 실수로 잘못 수정하면 런타임 오류가 발생.
- 상전이 표의 크기는 상태의 가짓수가 늘어나면 제곱해서 커지며 null로 채워지는 칸도 늘어날 것이다.


```java
// 코드 37-6 중첩 EnumMap으로 데이터와 열거 타입 쌍을 연결했다. (229-231쪽)
public enum Phase {
    SOLID, LIQUID, GAS; //, PLASMA;
    
    public enum Transition {
        MELT(SOLID, LIQUID), FREEZE(LIQUID, SOLID),
        BOIL(LIQUID, GAS), CONDENSE(GAS, LIQUID),
        SUBLIME(SOLID, GAS), DEPOSIT(GAS, SOLID);
//      IONIZE(GAS, PLASMA), DEIONIZE(PLASMA, GAS);
      
        private final Phase from;
        private final Phase to;
        
        Transition(Phase from, Phase to) {
            this.from = from;
            this.to = to;
        }

        // 상전이 맵을 초기화한다.
        private static final Map<Phase, Map<Phase, Transition>>
                m = Stream.of(values()).collect(groupingBy(t -> t.from,
                () -> new EnumMap<>(Phase.class),
                toMap(t -> t.to, t -> t,
                        (x, y) -> y, () -> new EnumMap<>(Phase.class))));
        
        public static Transition from(Phase from, Phase to) {
            return m.get(from).get(to);
        }
    }
}
```

#### EnumMap을 이용한 장점
- 새로운 상태 플라즈마(PLASMA) 이 상태와 연결된 전이 이온화(IONIZE), 탈이온화(DEIONIZE)를 추가한다면 Phase와 Transition만 추가하면 된다.
  - (37-5) 배열과 ordinal을 사용했을 경우 TRANSITIONS 배열을 추가적으로 수정해줘야한다.
- 맵들의 맵이 배열들의 배열로 구현되니 낭비되는 공간과 시간도 거의 없이 명확하고 안전하고 유지보수하기 좋다.
