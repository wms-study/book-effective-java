
# [Item40] @Override 애너테이션을 일관되게 사용하라

@Override - 상위 타입의 메서드를 재정의했음을 뜻한다.  
이 애너테이션을 일관되게 사용하면 여러 가지 악명 높은 버그들을 예방해준다.  

```java
// 코드 40-1 영어 알파벳 2개로 구성된 문자열(바이그램)을 표현하는 클래스 - 버그를 찾아보자. (246쪽)
public class Bigram {
    private final char first;
    private final char second;

    public Bigram(char first, char second) {
        this.first  = first;
        this.second = second;
    }

    public boolean equals(Bigram b) {
        return b.first == first && b.second == second;
    }

    public int hashCode() {
        return 31 * first + second;
    }

    public static void main(String[] args) {
        Set<Bigram> s = new HashSet<>();
        for (int i = 0; i < 10; i++)
            for (char ch = 'a'; ch <= 'z'; ch++)
                s.add(new Bigram(ch, ch));
        System.out.println(s.size());
    }
}
```

바이그램 26개를 10번 반복해 집합에 추가한뒤 크기를 출력한다.  
Set은 중복을 허용하지 않으니 26이 출력될 것 같지만, 260이 출력된다.  

여기서 문제점은 equals를 재정의(overriding)한게 아니라 다중정의(overloading)해버렸다.

Object.equals를 재정의 한다는 의도를 명시해야한다.
```java
    @Override
    public boolean equals(Bigram b) {
        return b.first == first && b.second == second;
    }
```

이처럼 @Override 에너테이션을 달고 컴파일하면 컴파일 오류가 발생한다.  
잘못한 부분을 명확히 알려주므로 곧장 올바르게 수정할 수 있다. 

```java
    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Bigram))
            return false;
        Bigram b = (Bigram) o;
        return b.first == first && b.second == second;
    }
```

**상위 클래스의 메서드를 재정의하려는 모든 메서드에 @Override 애너테이션을 달자**

한가지 예외적으로 구체 클래스에서 상위 클래스의 추상 메서드를 재정의할 때는 굳이 @Override를 달지 않아도 된다.  
구체 클래스인데 아직 구현하지 않은 추상 메서드가 남아 있다면 컴파일러가 알려주기 때문이다.  





