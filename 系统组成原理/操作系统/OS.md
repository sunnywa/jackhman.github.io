### Thread vs Quasar

> 纤程：用户空间之间的线程，启动的线程相对较多。用户空间内的线程之间切换相对于内核空间的线程要轻。
>
> 线程：内核空间之间的线程，启动的线程相对较少。

```java
public class HelloFiber {
    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        for (int k = 0; k < 10000; k++) {
            Fiber<Void> fiber = new Fiber<>((SuspendableRunnable) () -> {

                calc();
            });
            fiber.start();

/*
            Thread thread = new Thread(new Runnable() {
                @Override
                public void run() {
                    calc();
                }
            });
            thread.start();
*/
        }


        long end = System.currentTimeMillis();
        System.out.println(end - start);

    }

    private static void calc() {
        int result = 0;
        for (int m = 0; m < 10000; m++) {
            for (int i = 0; i < 200; i++) result += i;
        }
    }
}
```

在VM配置中:代理配置

> class和JVM中间链接着一个代理类agent,对class内部做了一些改动,一个fiber会自动生成一个对应的栈,整个的quasar来管理这个栈之间的切换。

```shell
-javaagent:D:\m2\repository\co\paralleluniverse\quasar-core\0.7.0\quasar-core-0.7.0.jar
```

