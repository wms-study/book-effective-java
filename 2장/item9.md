
# try-finally 보다는 try-with-resources를 사용하라

> 꼭 회수해야 하는 자원을 다룰 때는 try-finally 말고, try-with-resources를 사용하자.
> 코드를 짧고 분명하게, 만들어지는 예외 정보도 유용하다.

전통적으로 자원이 제대로 닫힘을 보장하는 수단으로 try-finally가 쓰였다.  

```java
public class TopLine {
    // 코드 9-1 try-finally - 더 이상 자원을 회수하는 최선의 방책이 아니다! (47쪽)
    static String firstLineOfFile(String path) throws IOException {
        BufferedReader br = new BufferedReader(new FileReader(path));
        try {
            return br.readLine();
        } finally {
            br.close();
        }
    }

    public static void main(String[] args) throws IOException {
        String path = args[0];
        System.out.println(firstLineOfFile(path));
    }
}
```
만약 자원을 하나 더 사용한다면 어떨까?

```java
public class Copy {
    private static final int BUFFER_SIZE = 8 * 1024;

    // 코드 9-2 자원이 둘 이상이면 try-finally 방식은 너무 지저분하다! (47쪽)
    static void copy(String src, String dst) throws IOException {
        InputStream in = new FileInputStream(src);
        try {
            OutputStream out = new FileOutputStream(dst);
            try {
                byte[] buf = new byte[BUFFER_SIZE];
                int n;
                while ((n = in.read(buf)) >= 0)
                    out.write(buf, 0, n);
            } finally {
                out.close();
            }
        } finally {
            in.close();
        }
    }

    public static void main(String[] args) throws IOException {
        String src = args[0];
        String dst = args[1];
        copy(src, dst);
    }
}
```

## try-finally 문의 문제점
기기의 물리적 문제가 발생하면 firstLineOfFile 메서드 안의 readLine 메서드가 예외를 던지고, 같은 이유로 close 메서드도 실패한다.  
이런 상황이라면 두번째 예외가 첫 번째 예외를 완전히 집어삼켜 버린다.

- 스택 트레이스에 첫번째 예외에 관한 정보가 남지 않는다
- 이로 인해 디버깅이 어려워 진다.

## try-with-resource [[JLS, 14.20.3](https://docs.oracle.com/javase/specs/jls/se8/html/jls-14.html#jls-14.20.3)]
이 구조를 사용하려면 해당 자원이 AutoCloseable 인터페이스를 구현해야 한다.  

```java
public class TopLine {
    // 코드 9-3 try-with-resources - 자원을 회수하는 최선책! (48쪽)
    static String firstLineOfFile(String path) throws IOException {
        try (BufferedReader br = new BufferedReader(
                new FileReader(path))) {
            return br.readLine();
        }
    }

    public static void main(String[] args) throws IOException {
        String path = args[0];
        System.out.println(firstLineOfFile(path));
    }
}
```

```java
public class Copy {
    private static final int BUFFER_SIZE = 8 * 1024;

    // 코드 9-4 복수의 자원을 처리하는 try-with-resources - 짧고 매혹적이다! (49쪽)
    static void copy(String src, String dst) throws IOException {
        try (InputStream   in = new FileInputStream(src);
             OutputStream out = new FileOutputStream(dst)) {
            byte[] buf = new byte[BUFFER_SIZE];
            int n;
            while ((n = in.read(buf)) >= 0)
                out.write(buf, 0, n);
        }
    }

    public static void main(String[] args) throws IOException {
        String src = args[0];
        String dst = args[1];
        copy(src, dst);
    }
}
```

try-with-resources 버전이 짧고 읽기 수월할 뿐 아니라 문제를 진단하기도 훨씬 좋다.  
readLine과 close 호출 양쪽에서 예외가 발생하면,  
close에서 발생한 예외는 숨겨지고 readLine에서 발생한 예외가 기록된다.  

이처럼 프로그래머에게 보여줄 예외 하나만 보존되고 여러 개의 다른 예외가 숨겨질 수도 있다.  
숨겨진 예외들도 그냥 버려지지 않고, 스택 추적 내역에 '숨겨졌다 (suppressed)'를 통해 출력된다.  
또한 Throwable에 추가된 getSuppressed 메서드를 이용하면 프로그램 코드에서 가져올 수도 있다.

```java
public class TopLineWithDefault {
    // 코드 9-5 try-with-resources를 catch 절과 함께 쓰는 모습 (49쪽)
    static String firstLineOfFile(String path, String defaultVal) {
        try (BufferedReader br = new BufferedReader(
                new FileReader(path))) {
            return br.readLine();
        } catch (IOException e) {
            return defaultVal;
        }
    }

    public static void main(String[] args) throws IOException {
        String path = args[0];
        System.out.println(firstLineOfFile(path, "Toppy McTopFace"));
    }
}
```

추가적으로 try-with-resources에서도 catch 절을 쓸 수 있다.  
이 덕분에 try 문을 중첩하지 않고도 다수의 예외를 처리할 수 있다.  

