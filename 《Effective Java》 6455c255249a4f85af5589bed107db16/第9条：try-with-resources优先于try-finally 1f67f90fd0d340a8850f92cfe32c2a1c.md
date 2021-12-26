# 第9条：try-with-resources优先于try-finally

Java类库中包括许多必须通过`close`方法来手工关闭的资源。例如`InputStream`、`OutputStream`和`java.sql.Connection`。客户端经常会忽略资源的关闭，造成严重的性能影响。虽然其中有许多资源都是通过终结方法来作为安全网的，但是效果并不理想。

根据经验，`try-finally`语句是确保资源会被适时关闭的最佳方法，就算发生异常或者放回也一样：

```java
static String firstLineOfFile(String path) throws IOException {
        BufferedReader br = new BufferedReader(new FileReader(path));
        try {
            return br.readLine();
        }finally {
            br.close();
        }
    }
```

这样的代码看起来也还好，但是如果添加第二个资源，代码的易读性就会严重下降：

```java
static void copy(String str, String dst) throws IOException{
        FileInputStream in = new FileInputStream(str);
        try {
            FileOutputStream out = new FileOutputStream(dst);
            try{
                byte[] buf = new byte[1024];
                int n;
                while ((n = in.read(buf))>=0)
                    out.write(buf,0,n);
            }finally {
                out.close();
            }
        }finally {
            in.close();
        }
    }
```

即便用`try-finally`语句正确的关闭了资源，如前两段代码所示，它也存在着些许不足。因为在`try`块和`finally`块中的代码，都会抛出异常。例如，在`firstLineOfFile`方法中，如果底层物理设备异常，那么调用`readLine`就会抛出异常，基于同样的原因，调用`close`也会出现异常。在这种情况下第二种异常完全抹除了第一个异常，在堆栈异常的轨迹中，完全没有第一个异常的记录，这样实现的系统中会导致调试变得非常困难，因为通常需要看到第一个异常才能判断问题所在。可以通过编写代码来禁止第二个异常，保留第一个异常，但事实上没有人会这么做，因为实现起来台烦琐了。

当Java 7 引入`try-with-resources`语句时，所有这些问题一下子就全部解决了。要使用这个构造的资源，必须先实现`AutoCloseable`接口，并重新`void`的`close`方法，Java类库和第三方库的许多类的接口，都实现和扩展了`AutoCloseable`接口。如果编写一个类，它代表的是必须被关闭的资源，那么这个类也因该实现`AutoCloseable`。

`try-with-resources` 第一个范例：

```java
static String firstLineOfFile(String path) throws IOException {
        try (BufferedReader br = new BufferedReader(
                new FileReader(path))){
            return br.readLine();
        }
    }
```

`try-with-resources` 第二个范例：

```java
static void copy(String str, String dst) throws IOException{
        try (InputStream in = new FileInputStream(str);
             OutputStream out = new FileOutputStream(dst)){
            byte[] buf = new byte[1024];
            int n;
            while ((n = in.read(buf))>= 0)
                out.write(buf,0,n);
        }
    }
```

使用`try-with-resources` 不仅使代码变得更简介易懂，也更容易进行诊断。

在`try-with-resources` 语句中还可以使用catch字句。就像平常使用`try-finally` 语句一样。这样既可以处理异常，又不需要再嵌套一层代码。

下面这个例子中firstLineOfFile方法没有抛出异常，但是如果它无法打开文件，或者无法读取，就会放回一个默认值：

```java
static String firstLineOfFile(String path) throws IOException {
        try (BufferedReader br = new BufferedReader(
                new FileReader(path))){
            return br.readLine();
        }catch (IOException e){
            return "default";
        }
    }
```

结论：在处理必须关闭的资源时，始终优先考虑使用`try-with-resources` 而不是使用`try-finally` 。这样的代码更加简介、清晰，生产的异常也更有价值。有了try-with-resources语句，在使用必须关闭的资源时，就能更轻松的正确编写代码了。