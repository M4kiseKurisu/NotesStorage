### 命名空间

- 可以将命名空间视作一个字符串（其中可以包含点），可通过加在类名、类型名前面并通过点分隔
- 命名空间是共享命名空间名的一组类和类型

命名格式：
```csharp
namespace NamespaceName { /*Declarations...*/ }
```

命名空间内类的定义举例：
```csharp
namespace MyCorp.SuperLib 
{
    public class SquareWidget
    {
        public double SideLength = 0;
        public double Area
        {
            get {return SideLength * SideLength; }
        }
    }
}
```

如果想使用上述命名空间内的类，其使用方式举例为：`MyCorp.SuperLib.SquareWidget sq = new MyCorp.SuperLib.SquareWidget();`

不同命名空间内的相同类名、类型名的指代可以是不同的，这也是命名空间的作用；但是同一个命名空间内，每个类型名要有别于其他所有类型

命名空间的其他特性：

- 同一命名空间是可以在不同源文件中增加声明内容的（命名空间不封闭）
- 命名空间是可以嵌套的，例如上面的`MyCorp.SuperLib`也可以使用嵌套方式进行定义

由于命名空间（尤其存在嵌套时）较长，为了缩短类型名长度，可以使用using命名空间指令（如`using NamespaceName`），这样在使用类型名时就可以省略其所属命名空间前缀，简化代码

### 委托

简单理解：委托是一个持有一个或多个方法的对象。当执行委托时，委托会执行其所持有的方法（也可以理解为C中的函数指针）

- 使用委托的方法：声明委托类型 -> 声明委托类型的变量 -> 创建委托实例并将其引用赋值给变量，添加第一个方法 -> 调用委托对象

可以把委托看作包含有序方法列表（称为调用列表）的对象，且这些方法具有相同的签名（名称、参数列表）及返回类型。调用列表中的方法可以是实例方法也可以是静态方法，但在调用委托的时候，会执行调用列表中的所有方法

#### 声明委托类型

- `delegate void/*返回类型*/ MyDel( int x )/*签名*/`

在声明时只用给出调用列表中方法的签名和返回类型，不需要直接给出方法主体

#### 创建委托对象

首先，需要声明委托类型的变量：`MyDel delVar`

委托实例的创建及委托变量的赋值有两种方式：

- `delVar = new MyDel( myInstObj.MyM1/*实例或静态方法*/ );`
- `delVar = myInstObj.MyM1;`

当需要更换委托中的方法时，直接使用`delVar = myInstObj.MyM2;`即可创建新委托实例并更换，旧委托实例会被回收

#### 委托的组合和方法增删

组合委托：可以将多个委托对象中的方法聚集起来，组合后的委托中含有所有组合前委托的方法。代码如下：

```csharp
MyDel delA = myInstObj.MyM1;
MyDel delB = SClass.OtherM2;
MyDel delC = delA + delB;
```

为一个委托增删方法使用简单的`+=`和`-=`即可完成，方法视作操作数：

```csharp
delVar += SClass.OtherM2;
delVar -= myInstObj.MyM1;
```

#### 委托调用

委托的调用有两种方式，一种是和调用方法一样调用；一种是使用`Invoke`方法：

```csharp
/*此处的委托要求的参数类型为整形*/
if (delVar != null) { delVar(55); } // 方法一
delVar?.Invoke(55); // 方法二
```

注意，调用时要求委托不能为空，因此需要使用`if`或空条件运算符及`Invoke`方法进行判别

委托调用时，参数会作用于调用列表的每一个方法，且如果一个方法多次出现时，每次遇到该方法都会进行调用

当委托有返回值时，调用列表中最后一个方法返回的值就是委托调用返回的值，其余方法返回值会被忽略

#### 匿名方法

如果不想在外部定义一个方法而只想使用一个一次性方法，可以采取这种写法：

- `delegate ( Parameters/*参数列表*/ ) { ImplementationCode/*语句块*/} `

匿名方法不会显式的声明返回值，但是其语句块需要返回一个和生命的委托类型的返回类型相同类型的值。举例如下：

```csharp
delegate int OtherDel(int InParam);
static void Main()
{
    OtherDel del = delegate(int x) 
        {
            return x + 20;
        }
}
```

#### Lambda表达式

创建匿名方法的时候可以使用Lambda表达式简化匿名方法对应委托的创建。例子如下：

```csharp
delegate int MyDel(int x);
MyDel del = delegate(int x) { return x + 3; } // 标准写法
MyDel del = (x) => { return x + 1; } // Lambda表达式
MyDel del = x => x + 1; // 更简单的Lambda表达式
```

因此，一个无参数空返回值的匿名方法委托，就可以被Lambda表达式写成`Mydel del = () => { /*代码块*/ }`


### 事件

发布者/订阅者模式：发布者定义了一系列程序其他部分可能感兴趣的事件，其他类可以“注册”，便于在这些事件发生时收到发布者通知。这些订阅这类通过向发布者提供一个方法来“注册”以获得通知。当事件发生时发布者“触发事件”，执行订阅者提交的所有事件。

<img src=".\CSharpStudyPic\image.png"/>

- 发布者：发布某个事件的类或结构，其他类可以在事件发生时得到通知
- 订阅者：注册并在事件发生时得到通知的类、结构
- 事件处理程序：订阅者注册到事件的方法，在发布者触发事件时执行
- 触发事件：当事件被触发，所有注册到它的方法都被调用

### 泛型

泛型能够让多个类型共享一组代码，允许声明类型参数化的代码。如果说类型不是对象而是对象的模板，那么泛型类型也不是类型，而是类型的模板

- C#提供了类、结构、接口、委托和方法泛型

#### 泛型声明和创建

在类名后增加一组尖括号，括号内用占位符字符串表示需要提供的类型（类型参数）。主体中使用类型参数代替类型。举例：

```csharp
class SomeClass <T1, T2>
{
    public T1 SomeVar;
    public T2 OtherVar;
}
```

声明之后，只要在尖括号中使用真实类型替代类型参数就可以产生构造类型：`SomeClass<short, int>`

之后就可以将这个构造类型当作一个具体的类型进行使用：

```csharp
SomeClass<short, int> mySc1 = new SomeClass<short, int>();
var mySc1 = new SomeClass<short, int>();
```

#### where子句

where子句用来列出约束：

- 每一个有约束的类型参数都有自己的where子句
- 如果形参有多个约束，它们在where子句中使用逗号分隔

where子句的语法为：`where TypeParam/*类型参数*/ : constraint, constraint, .../*约束列表*/`

举例如下：

```csharp
class MyClass<T1, T2, T3>
              where T2 : Customer // 约束T2参数只能是Customer及其派生类
              where T3 : IComparable // 约束T3参数只能是IComparable及其派生类
{
    // 类内容
}
```