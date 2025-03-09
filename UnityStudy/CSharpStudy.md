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