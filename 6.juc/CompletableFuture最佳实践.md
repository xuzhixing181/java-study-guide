# 1、避免阻塞

- 避免阻塞：使用回调方法替代 get() 问题场景：直接使用 get() 方法会阻塞主线程，导致性能下降。

```Java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    try {
        // 模拟耗时操作
        Thread.sleep(5000);
    } catch (InterruptedException e) {
    }
    return "invoke supplyAsync method";
});

// 阻塞主线程等待结果
try {
    System.out.println(future.get());
} catch (InterruptedException | ExecutionException e) {
    throw new RuntimeException(e);
}
```

## **最佳实践**

- 使用回调处理结果

```java
CompletableFuture.supplyAsync(() -> {
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
    }
    return "result sync handle";
}).thenAccept(result -> {
    // 异步处理结果，不阻塞主线程
    System.out.println("接收到结果: " + result);
});

// 主线程可以继续执行其他任务
System.out.println("主线程继续执行...");
```

- 当使用回调方法`thenApply(Function<? super T,? extends U> fn)`替代 `get()` 方法处理 CompletableFuture 结果时，可以通过多种方式获取执行的详细信息（如：上一步任务的返回结果，执行回调的线程信息等）

```java
CompletableFuture.supplyAsync(() -> {
    // 模拟任务执行
    return "origin result";
}).thenApply(result -> {
    System.out.println("接收到结果: " + result);
    System.out.println("当前线程: " + Thread.currentThread().getName());
    return result.toUpperCase();
}).thenAccept(finalResult -> {
    System.out.println("最终结果: " + finalResult);
});
```

- `whenComplete(BiConsumer<? super T, ? super Throwable> action)`获取任务执行结果和异常信息（如：正常结果或异常对象、回调发生的精确时间、完整的异常堆栈信息等）

```Java
CompletableFuture.supplyAsync(() -> {
    if (Math.random() > 0.5) {
        throw new RuntimeException("模拟异常");
    }
    return "成功结果";
}).whenComplete((result, ex) -> {
    System.out.println("回调执行时间: " + Instant.now());
    if (ex != null) {
        System.out.println("任务失败: " + ex.getMessage());
        System.out.println("异常类型: " + ex.getClass().getName());
    } else {
        System.out.println("任务成功: " + result);
    }
});
```



# 2、合理使用线程池

- **问题场景**：默认使用公共的 ForkJoinPool，可能影响其他任务

```java
// 所有任务都使用默认的公共线程池
CompletableFuture.runAsync(() -> {
    // 长时间运行的任务
});
```

## **最佳实践**

- 为不同任务类型（IO密集型或CPU密集型）创建专用线程池

```java
        // 创建专用线程池(推荐使用ThreadPoolExecutor而非Executors来创建线程池,这里如何创建线程池不是重点)
        ExecutorService ioBoundExecutor = (不推荐使用Executors来创建线程池).newFixedThreadPool(10); // I/O密集型
        ExecutorService cpuBoundExecutor = Executors.newWorkStealingPool(); // CPU密集型

        // I/O密集型任务
        CompletableFuture.supplyAsync(() -> {
            // 模拟I/O操作
            try { Thread.sleep(1000); } catch (InterruptedException e) {}
            return "I/O结果";
        }, ioBoundExecutor);

        // CPU密集型任务
        CompletableFuture.supplyAsync(() -> {
            // 计算密集型操作
            return IntStream.range(1, 1000000).sum();
        }, cpuBoundExecutor);
```



# 3、完善的异常处理机制

- **问题场景**：忽略异常处理导致问题难以追踪

```java
CompletableFuture.supplyAsync(() -> {
    if (Math.random() > 0.5) {
        throw new RuntimeException("随机错误");
    }
    return "成功";
}).thenAccept(System.out::println); // 异常会被吞没
```

## 最佳实践

- 使用`handle(BiFunction<? super T, Throwable, ? extends U> fn)`全面处理 执行任务得到的结果和异常

```java
CompletableFuture.supplyAsync(() -> {
    if (Math.random() > 0.5) {
        throw new RuntimeException("随机错误");
    }
    return "成功";
}).handle((result, ex) -> {
    if (ex != null) {
        System.err.println("任务失败: " + ex.getMessage());
        return "默认值";
    }
    return result;
}).thenAccept(result -> {
    System.out.println("最终结果: " + result);
});
```



# 4、资源清理

**问题场景**：异步操作中使用资源未正确关闭

```Java
CompletableFuture.supplyAsync(() -> {
    FileInputStream fis = new FileInputStream("file.txt");
    // 使用文件流但未关闭
    return readContent(fis);
});
```

## 最佳实践

- try-with-resources

```java
CompletableFuture.supplyAsync(() -> {
    try (FileInputStream fis = new FileInputStream("file.txt")) {
        return readContent(fis);
    } catch (IOException e) {
        throw new CompletionException(e);
    }
});
```



## 关于try-with-resources

- 在传统同步代码中，使用try-with-resources关闭资源的常用方式如下：

```java
try (FileInputStream fis = new FileInputStream("file.txt")) {
    // 使用资源
    return processStream(fis);
} // 自动关闭
```

- 当资源操作被包装在 CompletableFuture等异步环境中，还使用如上方式，则可能导致资源在异步线程中未正确关闭，关闭资源的正确方式如下：

```java
CompletableFuture.supplyAsync(() -> {
    try (FileInputStream fis = new FileInputStream("file.txt")) {
        return processStream(fis);
    } catch (IOException e) {
        throw new CompletionException(e); // 必须包装检查异常
    }
});
```

1. `try-with-resources` 确保在任何情况下（正常返回或异常）都会关闭资源
2. 将检查异常包装为 `CompletionException`（必需，因为 Supplier 不能抛出检查异常）

- 多资源管理示例：

```java
CompletableFuture.supplyAsync(() -> {
    try (FileInputStream fis = new FileInputStream("input.txt");
         FileOutputStream fos = new FileOutputStream("output.txt")) {
        // 处理数据
        return transformData(fis, fos);
    } catch (IOException e) {
        throw new CompletionException("文件操作失败", e);
    }
});
```

## **`CompletionException`** 

- 如上所示，Java 的 `CompletableFuture` 异步编程中，**检查异常（Checked Exceptions）需要被包装为 `CompletionException`** ，原因如下：

### 函数式接口的约束

- `CompletableFuture.supplyAsync()` 接收的参数是一个 `Supplier` 函数式接口，其 `get()` 方法 **不声明任何检查型异常**：

  ```java
  @FunctionalInterface
  public interface Supplier<T> {
      T get();
  }
  ```

  当异步任务中可能抛出检查型异常（如 `IOException`）时，必须通过以下方式之一处理：

  - **直接捕获并处理**（但异步任务通常无法就地解决）
  - **包装为非检查异常**（如 `CompletionException`）抛出，由后续流程统一处理



### **统一异常类型**

- 通过将检查异常包装为 `CompletionException`，可以：

  1）**绕过函数式接口的限制**，允许异常传播

  2）**统一异常类型**，方便后续处理

  3）**保留原始异常信息**，便于调试

```java
future.exceptionally(ex -> {
    Throwable rootCause = ex.getCause(); // 获取原始异常
    if (rootCause instanceof IOException) {
        System.err.println("I/O错误: " + rootCause.getMessage());
    }
    return "默认值";
});
```



### 异常处理的一致性

- 在异步链中，所有阶段的异常最终会被包装为 `CompletionException`。主动包装异常可以：

  1）确保异常信息的统一性

  2）避免丢失原始异常栈信息

  3）简化异常处理逻辑（后续可以通过 `exceptionally()` 统一处理）



### **对比直接抛出非检查异常**

- 虽然也可以抛出其他非检查异常（如 `RuntimeException`），但使用 `CompletionException` 的优势在于：

  1）**语义明确**：明确表示这是异步任务的失败

  2）**与 `CompletableFuture` 机制兼容**：`join()`、`get()` 等方法会正确解包 `CompletionException`

  3）**框架友好**：Spring 等框架对 `CompletionException` 有专门处理逻辑

小结：

| 场景                                    | 正确做法                               | 错误做法                          | 结果       |
| :-------------------------------------- | :------------------------------------- | :-------------------------------- | :--------- |
| 抛出检查异常（如 `IOException`）        | 包装为 `CompletionException`           | 直接抛出 `IOException`            | 编译错误   |
| 抛出非检查异常（如 `RuntimeException`） | 直接抛出或包装为 `CompletionException` | 不处理                            | 运行时异常 |
| 需要保留原始异常信息                    | 包装时包含原始异常                     | 直接抛出 `new RuntimeException()` | 丢失上下文 |

------

- 通过合理包装检查异常，可以确保 `CompletableFuture` 的资源清理和异常处理既安全又可维护，符合 Java 异步编程的最佳实践。

