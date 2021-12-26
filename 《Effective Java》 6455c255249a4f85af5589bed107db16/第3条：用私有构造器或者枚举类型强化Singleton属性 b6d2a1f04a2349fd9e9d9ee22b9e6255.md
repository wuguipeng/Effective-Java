# 第3条：用私有构造器或者枚举类型强化Singleton属性

`Singleton`是指只能被实例化一次的类。`SIngleton`通常被用来代表一个无状态的对象。如函数或者一些本质上唯一的系统组件。使类成为`Singleton`会使它的客户端测试变得十分困难，因为不可能给`Singleton`替换模拟实现，除非实现一个充当其接口。

实现`Singleton`的方式有两种。这两种方法都需要将构造器私有化，并导出公有的静态成员，以便允许客户端能够访问该类的唯一实例。

第一种方法 公有静态成员是一个final域：

```java
public class Elvis{
	**public static final Elvis INSTANCE = new Elvis();**
	private Elvis(){...}
	
	public void leaveTheBuilding(){...}

}
```

私有构造器仅被调用一次，用来实例化公有静态`final`域`Elvis.INSTANCE`。由于缺少公有的或者受保护的构造器，所以保证了Elvis的全局唯一性：一旦`Elvis`类被实例化，将只会存在一个`Elvis`实例。客户端的任何行为都不会改变这一点，但要提醒一点：享有特权的客户端可以使用`AccessibleObject.setAccessible`方法，通过放射机制调用私有构造方法。如果需要抵御这种攻击，可以修改构造器，让它在创建第二次的时候抛出异常。

第二种方法 公有的成员是个静态方法工厂：

```java
public class Elvis { 
  **private static final Elvis INSTANCE = new Elvis();** 
  private Elvis() { ... } 
  **public static Elvis getInstance() { return INSTANCE; }** 
  public void leaveTheBuilding() { ... } 
}
```

对于静态方法`Elvis.getInstance`的所有调用，都会放回同一个对象的引用，所以永远不会创建其他的Elvis实例。

公有域方法的主要优势在于，API很清楚的表明了这是个类是一个`Singleton`：公有的静态域是`final`的，所以该域总是包含相同的对象引用。第二个的优势在于它更简单。

静态工厂方法的优势之一在于，它提供了灵活性：在不改变其 API 的前提下，我们可以改变该类是否应该为单例的想法。工厂方法返回该类的唯一实例，但是，它很容易被修改，比如，改为每个调用该方法的线程返回一个唯一的实例。第二个好处是，如果你的应用程序需要它，可以编写一个泛型单例工厂。使用静态工厂的最后一个优点是，可以通过方法引用作为提供者，例如 Elvis::instance 等同于 `Supplier<Elvis>` 。除非满足以上任意一种优势，否则还是优先考虑公有域的方法。

为了将上述方法中实现的单例类变成是可序列化的，仅仅将 `implements Serializable` 添加到声明中是不够的。为了保证单例模式不被破坏，必须声明所有的实例字段为`transient` ，并提供一个 `readResolve` 方法。否则，每当序列化的实例被反序列化时，就会创建一个新的实例，在我们的例子中，导致出现新的 `Elvis` 实例。为了防止这种情况发生，将如下的 `readResolve` 方法添加到 `Elvis` 类：

```java
// readResolve method to preserve singleton property 
private Object readResolve() { 
  // Return the one true Elvis and let the garbage collector 
  // take care of the Elvis impersonator. 
  return INSTANCE;
}
```

实现一个单例的第三种方法是声明单一元素的枚举类：

```java
// Enum singleton - the preferred approach 
public enum Elvis { 
  INSTANCE; 
  public void leaveTheBuilding() { ... } 
}
```

这种方法在功能上与公有域方法相似，但更加简洁，无偿地提供了序列化机制，绝对防止多次实例化，即使是在面对复杂的序列化或者反射攻击的时候。虽然这种方法还没有广泛采用，但是单元素的枚举类型经常成为实现 `Singleton`的最佳方法。注意，如果 Singleton必须扩展一个超类，而不是扩展Enum 的时候，则不宜使用这个方法(虽然可以声明枚举去实现接口)。