# 第46条：优先选择Stream中无副作用的函数

如果刚接触Stream，可能比较难以掌握其中的窍门。就算只是用Stream pipeline来表达计算就困难重重。当你好不容易成功了，运行程序之后，却可能感受到这么做并没有享受到多大益处。Stream并不是一个API，它是一种基于函数编程的模型。为了获得Stream带来的描述性和速度，有时候还有并行性，必须采用范型以及API。

Stream范型最重要的部分是把计算构造成一些列变型，每一级结果都尽量靠近上一级结果的纯函数。纯函数是指其结果只取决于输入的函数：它不依赖任何可变的状态，也不更新任何状态。为了做到这一点，传入Stream操作的任何函数对象，无论是中间操作还是终止操作，都应该是无副作用的。

有时会看到如下代码片段，它构建了一张表格，显示这些单词在一文本文件中出现的频率：

```java
Map<String, Long> freq = new HashMap<>();
try(Stream<String> words = new Scanner(file).tokens()){
	words.forEach(word -> {
		freq.merge(word.toLowerCase(), 1L, Long::sum);
	});
}
```

上述代码中使用了Stream、Lambda和方法引用，并且得到了正确的答案。但这根本不是Stream代码：只不过是伪装成Stream代码的迭代式代码。它并没有享受到Stream API带来的优势，代码反而更长了一点，可读性也差了点，并且比相应的迭代话代码更难以维护。因为这段代码利用了一个改变外部状态的Lambda，完成了在终止操作的forEach中的所有工作。forEach操作的任务不只是展示由Stream执行的计算结果，这在代码中并非好事，改变状态的Lambda也是如此。下面这段代码才是Stream API的正确使用：

```java
Map<String, Long> freq;
try (Stream<String > words = new Scanner(file).tokens()){
    freq = words.collect(groupingBy(String::toLowerCase, counting()));
}
```

这段代码的作用和前面的代码一样，只不过是正确的使用了Stream API，变得更加简洁、清晰。使用forEach是因为我们对forEach比较熟悉，也知道怎么使用。forEach操作是终止操作中最没有威力的，也是对Stream最不友好的。它显示迭代，因而不适合并行。forEach操作应该只用于报告Stream 计算的结果，而不是执行计算。有时候，也可以将forEach用于其他目的，比如将Stream 计算的结果添加到之前已经存在的集合中去。

改进过的代码使用了一个**收集器**（collector），为了使用Stream，这是必须了解的一个概念。Collectors API由39种方法，其中有些方法带有5个类型参数！但是并不需要将每一个方法都完全搞懂。对于初学者，可以忽略Collector接口，并把收集器当作封装缩减策略的一个黑盒子对象。在这里，缩减的意思是将Stream的元素合并到单个对象中去。收集器产生的对象一般是一个集合（即名称收集器）。

将Stream 的元素集中到一个真正的Collection里去的收集器比较简单。有三个这样的收集器：toList()、toSet()和toCollection(collectionFactory)。它们分别返回一个列表、一个集合和程序员指定的集合类型。了解了这些，就可以编写Stream pipeline，从频率表中提取排名前十的单词列表了：

```java
List<String> topTen = freq.keySet().stream()
			.sorted(**comparing(freq::get).reversed()**)
			.limit(10)
			.collect(toList());
```

注意，这里没有给toList方法配上它的Collectors类。静态导入Collectors的所有成员是惯例也是明知的选择，因为这样可以提升Stream pipeline的可读性。

这段代码中的唯一有技巧的部分是传给sorted的比较器comparing(freq::get).reversed()。comparing方法是一个比较构造器方法，它带有一个键提取器函数。函数读取一个单词，“提取”实际上是一个表查找：有限制的方法引用freq::get在频率表中查找单词，并返回该单词在文件中出现的次数。最后，在比较构造器上调用reversed，按频率高低对单词进行排序。后面的事情就简单了，只要限制Stream 为10个单词，并将它们集中到一个列表中即可。

最简单的映射收集器是toMap(keyMapper, valueMapper)，它带有两个参数，其中一个是将Stream元素映射到键，另一个是将它映射到值。采用第34条中fromString实现中的收集器，将枚举的字符串形式映射到枚举本身：

```java
private static final Map<String, Operation> stringtoEnum = 
													Stream.of(values()).collect(toMap(Object::toString, e -> e));
```

如果Stream中的每个元素都映射到一个唯一的键，那么这个形式简单的toMap还是很完美的。如果多个Stream元素的映射到同一个键，pipeline就会抛出一个IllegalStateException异常将它终止。

toMap更复杂的形式，以及groupingBy方法，提供了更多处理这种冲突的策略。其中一种方法是除了给toMap方法提供了检核值映射器之外，还提供一个**合并函数**（merge function）。合并函数是一个BinaryOperator<V>，这里的V是映射的值类型。合并函数将与键关联的任何其他值与现有值合并起来，因此，假如合并函数是乘法，得到的值就是与该值映射的键关联的所有值的积。

带有三个参数的toMap形式，对于完成从键到与键关联的元素的映射也是非常有用的。假设一个Stream，代表不同的歌唱家的唱片，我们想得到一个从歌唱家到最畅销唱片之间的映射。下面就是这个收集器可以完成的任务。

```java
Map<Artist, Album> topHits = albums.collect(
		**toMap(Albm::artist, a -> a, maxBy(comparing(Album::sales)))**);
```

注意，这个比较器使用了静态工厂方法maxBy，这是从BinaryOperator静态导入的。该方法将Comparator<T>转换成一个BinaryOperator<T>，用于计算指定比较器生产的最大值。在这个例子中，比较器是由比较器狗仔方法comparing返回的，它由一个键提取函数**Album::sales**。这看起来优点绕，但是代码的可读性良好。不严格的说，它的意思是“将唱片的Stream转换成以一个映射，将每个歌唱家映射到销量最佳的唱片”。这就非常接近问题的描述了。

带有三个参数的toMap形式还有另一种用途，即生产一个收集器，当有冲突是强制“保留最后更新”。对于许多Stream而言，结果是不确定的，但如果与映射函数的键关联的所有值都相同，或则都是可接受的，那么下面收集器的行为就正是你所要的：

```java
toMap(keyMapper, valueMapper, (oldVal, newVal) -> new?val)
```

toMap的第三个也是最后一种形式，带有四个参数，这是一个映射工厂，在使用时要指定特殊的映射实现，如EnumMap或则TreeMap。

toMap的前三种版本还有另外的变换形式，命名为toConcurrentMap，能够有效的并行运行，并生成ConcurrentHashMap实例。

除了toMap方法，Collectors API还提供了groupingBy方法，它返回收集器以生成映射，根据分类函数将元素分门别类。分类函数带有一个元素，并返回其所属的类别。这个类别就是元素的映射键。groupingBy方法最简单的版本是只有一个分类器，并返回一个映射，映射值为每个类别中所有元素的列表。下列代码就是在第45条的Anagram程序中用于生成映射的收集器：

```java
words.collect(groupingBy(word -> alphabetize(word)));
```

如果要让groupingBy返回一个收集器，用它生成一个值而不是列表的映射，除了分类器之外，还可以指定一个下游收集器。下游收集器从包含某个类别中所有元素的Stream中生成一个值。这个参数最简单的用法是传入toSet(),结果生成一个映射，这个映射值为元素集合而非列表。

另一种方法是传入toCollection(collectionFactory)，允许创建存放各元素类别的集合。这样就可以自由选择自己想要的任何集合类型了。带有两个参数的groupingBy版本的另一种简单用法是，传入counting()作为下游收集器。这样会生成一个映射，它将每个类别与该类别种的元素数量联合起来，而不是包含元素的集合。这正是在本条目开头处频率表范例中见到的：

```java
Map<String, Long> freq = words.collect(groupingBy(String::toLowerCase, counting()));
```

groupingBy的第三个版本，除了下游收集器之外，还可以指定一个映射工厂。注意，这个方法违背了标准的可伸缩参数列表模式：参数mapFactory要在downStream参数之前，而不是在它之后。groupingBy的这个版本可以控制所包围的映射，以及所有包围的集合，因此，比如可以定义一个收集器，让它返回值为TreeSets的TreeMap。

`groupingByConcurrent` 方法提供了`groupingBy` 所有三种重载的变体。这些变体可以有效的并发运行，生成`ConcurrentHashMap` 实例。还有一种比较少用到的groupingBy变体叫做partitioningBy。除了分类方法之外，它还带有一个断言，并返回一个建委Boolean的映射。这个方法有两个重载，其中一个除了带有断言之外，还带有下游收集器。

counting方法返回的收集器仅作用下游收集器。通过Stream上的count方法，直接就有相同的功能，因此压根没有理由使用collect(counting())。这个属性还有15种Coollectors方法。其中包含9种方法其名称以summing、averaging和summarizing开头。它们还包括reducing、filtering、maping、flatMapping和collectingAndThen方法。大多数程序员都能安全地避开这里大多数方法。从设计角度来看，这些收集器试图部分复制收集器种Stream的功能，以便下游收集器可以成为“ministream”。

目前已经提到了3个Collectors方法。虽然它们都在Collectors中，但是并不包含集合。前两个是minBy和maxBy，它们有一个比较器，并返回有比较器确定的Stream中的最少元素或者最多元素。它们是Stream接口中min和max方法的粗概括，也是`BinaryOperator` 中同名方法返回的二进制操作符，与收集器相类似。

最后一个Collectors方法是joining，它只在CharSequence实例的Stream中操作，例如字符串。它以参数的形式放回一个见到的合并元素的收集器。其中一种参数形式带有一个名为delimiter（分解符）的CharSequence参数，它返回一个链接Stream元素并在相邻元素之间插入分隔符的收集器。如果传入一个逗号作为分割符，收集器就会返回一个用逗号隔开的值字符串。这三种参数形式，除了分隔符之外，还有一个前缀和一个后缀。最终的收集器生成的字符串，就像在打印集合时所得到的那样，如【came， saw， conquered】。

总而言之，编写Stream pipeline的本质是无副作用的函数对象。这适用于传入Stream及相关对象的所有函数对象。终止操作中的forEach应该只用来报告由Stream执行的计算结果，而不是让它执行计算。为了正确的使用Stream，必需了解收集器。最重要的手机工厂时toList、toMap、toSet、groupingBy和joining。