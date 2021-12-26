# 第47条：Stream要优先用Collection作为返回类型

许多方法都返回元素的序列，。在Java 8之前，这类方法明显的放回类型是集合接口Collection、Set和List；Iterable；以及数组类型。一般来说，很容易确定要返回这其中哪一种类型。标准接口是一个集合接口。如果某个方法只为for-each循环或者返回序列而存在，无法用它来实现一些Collection方法，那么就用Iterable接口吧。如果返回的元素是基本类型值，或者有严格的性能要求，就使用数组。在Java 8中增加了Stream，本质上导致给序列化返回的方法选择适当返回类型的任务变得更复杂了。

或许你曾听说过，现在Stream是返回元素序列最明显的选择了，但如果第45条所述，Stream并没有淘汰迭代：要编写出优秀的代码必须巧妙地将Stream与迭代结合起来使用。如果一个API值返回一个Stream，那些想要用forEach循环遍历返回序列地用户肯定要失望了。因为Stream接口只在Iterable接口中包含了唯一一个抽象方法，Stream对于该方法地规定也适用于Iterable的。唯一可以让程序员避免使用forEach循环遍历Stream的是Stream无法扩展Iterable接口。

遗憾的是，这个问题还没有适当的解决办法。乍看之下，好像给Stream的iterator方法传入一个方法引用可以解决。这样得到的代码可能优点乱、不清晰，但也不算难以理解：

```java
for(ProcessHandle ph : ProcessHandle.allProcesses()::iterator){
		//Process the process
}
```

遗憾的是，如果想要编译这段代码，就会得到这样一条信息：

```java
Test:java:6: error: method reference not expected here
for(ProcessHandle ph : ProcessHandle.allProcesses()::iterator){
											 ^
```

为了使代码能够进行编译，必须将方法引用转换成适合参数化的Iteralbe：

```java
for(ProcessHandle ph : (Iterable<ProcessHandle>) ProcessHandle.allProcesses()::iterator)
```

这个客户端代码可行，但是实际使用时过于杂乱、不清晰。更好的解决办法是使用适配器方法。JDK没有提供这样的方法，但是编写起来很容易，使用在上述代码中内嵌的相同方法即可。注意，在适配器方法中没有必要进行转换，因为Java的类型引用这里正好派上了用场：

```java
public static <E> Iterable<E> iterableof(Stream<e> stream){
	return stream::iterator;
}
```

有了这个适配器，就可以利用forEach语句遍历任何Stream：

```java
for(ProcessHandle ph : iterableOf(ProcessHandle.allProcesses())){
	//Process the process
}
```

注意，第34条中Anagram程序的Stream版本是使用File.line方法读取词典，而迭代版本中则使用了扫描器（Scanner）。File.lines方法优先于扫描器，因为或者是默默地吞掉了在读取文件过程中遇到的所有异常。最理想的方式是在迭代化版本中也使用了File.lines。这是程序员在特定情况下所做的一种妥协，比如当API只要Stream能访问序列，而它们想通过forEach语句遍历该序列的时候。

反过来说，想要利用Stream pipeline处理序列的程序员，也会被只提供Iterable的API搞得束手无策。同样的，JDK没有提供适配器，但是编写起来也很容易：

```java
public static <E> Stream<E> streamOf(Iterable<E> iterable){
	return StreamSupport.stream(iterabl.spliterator(), false);
}
```

如果在编写一个放回对象的方法时，就知道它只在Stream pipeline中使用，当然就可以放心地返回Stream 了。同样地，当放回序列地方法只在迭代中使用，则应该返回Iterable。但如果是用公共地API返回序列，则应该为那些想要编写Stream pipeline，以及想要编写foreach语句的用户分别提供，除非有足够的理由相信大多数用户都想要使用相同的机制。

Collection接口是Iterable的一个子类型，他又一个stream方法，因此提供了迭代和stream访问。对于公共的、返回序列的方法，Collection或者适当的子类型通常是最佳的返回类型。数组也通过Arrays.asList和Stream.of方法提供了简单的迭代和stream访问。如果放回的序列足够小，容易存储，或许最好返回标准的集合实现，如ArrayList或者HashSet。但千万别在内存中保存巨大的序列，将它作为集合返回即可。

如果返回的序列很大，但是能被准确的表述，可以考虑实现一个专用的集合。假设想要返回一个指定的集合的幂集，其中包括它所有的子集。{a, b, c}的幂集是{{}, {a}, {b}, {c}, {a, b}, {a, c}, {b, c}, {a,, b, c}}。如果集合中有n个元素，它的幂集就有2n个。因此，不必考虑将幂集保存在标准的集合实现中。但是，有了AbstractList的协助，为此实现定制集合就很容易了。

技巧在于，用幂集中每个元素的索引作为位向量，在索引中排第n位，表示员集合中第n位元素存在或不存在。实质上，在二进制数0至2n-1和有n位元素的集合的幂集之间，有一个自然映射。代码如下：

```java
public class PowerSet {
    public static final <E> Collection<Set<E>> of(Set<E> s){
        
        if(src.size() > 30)
            throw new IllegalArgumentException("Set to big " + s);
        return new AbstractList<Set<E>>() {
            @Override
            public Set<E> get(int index) {
                Set<E> result = new HashSet<>();
                for(int i = 0; index != 0; i++, index >>= 1)
                    if((index & 1) == 1)
                        result.add(src.get(i));
                return result;
            }
            
            @Override
            public boolean contains(Object o){
                return o instanceof Set && src.containsAll((Set) o);
            }

            @Override
            public int size() {
                return 1 << src.size();
            }
        };
    } 
}
```

注意，如果输入值集合中超过了30个元素，PowerSet.of会抛出异常。这正是用Collection而不是用Stream或Iterable作为返回类型的缺点：Collection有一个返回int类型的size方法，他限制返回的序列长度位Integer.MAX_VALUE或者$2^{31}-1$。如果集合更大，甚至无限大，Collection规范确实允许size方法返回$2^{31}-1$，但是这并非是令人满意的解决方案。

为了在`AbstractCollection` 上编写一个Collection实现，处理Iterable必需的那一个方法之外，还需要实现两个方法： contains和size。这些方法经常很容易编写出高效的实现。如果不可行，或许是因为没有在迭代发生之前先确定序列的内容，返回Stream或者Iterable，感觉哪一种更自然即可。如果选择，可以尝试着分别用两个方法返回。

实现输入列表的所有子列表的Stream是很简单的，尽管它确实需要优点洞察力。我们把包含列表第一个元素的子列表称作列表的前缀。例如，(a, b, c)的前缀就是(a)、(a, b)和(a, b, c)。同样的，把包含最后一个元素的子列表称作后缀，因此(a, b, c)的后缀就是(a, b, c)、(b, c)和(c)。考验洞察力的是，列表的子列表不过是前缀的后缀（或者说后缀的前缀）和空列表。这一发现直接带来了一个清晰且相当整洁的实现：

```java
public class SubLists {
    public static <E> Stream<List<E>> of(List<E> list){
        return Stream.concat(Stream.of(Collections.emptyList()),
                prefixes(list).flatMap(SubLists::suffixes));
    }
    
    private static <E> Stream<List<E>> prefixes(List<E> list){
        return IntStream.rangeClosed(1, list.size())
                .mapToObj(end -> list.subList(0, end));
    }
    
    private static <E> Stream<List<E>> suffixes(List<E> list){
        return IntStream.range(0, list.size())
                .mapToObj(start -> list.subList(start, list.size()));
    }
}
```

注意，它用Stream.concat方法将空列表添加到返回的Stream。另外还用flatMap方法生成了一个包含所有后缀的Stream。最后，通过映射IntStream.range和intStream.rangeClose返回的连续int值的Stream，生成了前缀和后缀。通俗的讲，这一术语的意思就是指数位整数的标准for循环的Stream版本，因此，这个子列表实现本质上与明显的嵌套式for循环相类似：

```java
for(int start = 0; start < src.size(); start++)
	for(int end = start + 1; end <= src.size(); end++ )
		System.out.println(src.subList(start, end));
```

这个for循环也可以直接翻译成一个Stream。这样得到的结果给比前一个实现更加简洁，但是可读性稍微差了一点。它本质上与第45条中笛卡尔积的Stream代码相类似：

```java
public static <E> Stream<List<E>> of(List<E> list){
	return IntStream.range(0, list.size())
						.mapToObj(start -> IntStream.rangeClosed(star+1, list.size))
						.flatMap(x -> x);
} 
```

像前面的for循环一样，这段代码也没有发出空列表。为了修正这个错误，也应该使用concat，如前一个版本中那样，或者用rangeClosed调用中的(int)Math.signum(start)代替1.

子列表的这些Stream实现都很好，但这两者都需要用户在任何更适合迭代的地方，采用StreamtoIterable适配器，或者用Stream。StreamtoIterable适配器不仅打乱了客户端代码，在循环速度上还降低了2.3倍。专门构建的Collection实现相当烦琐，但是运行速度比基于Stream的实现快了约1.4被。

总而言之，在编写返回一系列元素的方法时，要记住有些用户可能想要当作Stream处理，而其他用户可能想要使用迭代。要尽量两边兼顾。如果可以返回集合，那就返回一个标准的集合，如ArrayList。否则，就要考虑实现一个定制的集合，如幂集范例中所示。如果无法返回集合，就返回Stream或者Iterable，感觉哪一种更自然即可。如果在未来的Java发行版本中，Stream接口声明被修改成了扩展了Iterable接口，就可以放心的返回Stream了，因为它们允许进行Stream处理和迭代。