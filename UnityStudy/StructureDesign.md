## 游戏框架搭建 笔记

### 设置游戏例子

- 游戏的基本逻辑：开始游戏 -> 生成10个怪物 -> 点击消除所有怪物 -> 游戏结束

经过构造，游戏场景中的节点树构造（场景结构）如下：

<img src=".\StructureDesignPic\image.png"/>

其中两个Panel分别展示游戏开始和游戏结束，点击游戏开始Panel的子Button就会有效化Enemies父节点，生成10个Enemy

明显的调用过程为：BtnStart在被点击后调用自身（无效化）和Enemies节点（有效化），在组件中实现：

<img src=".\StructureDesignPic\image-1.png"/>

Enemy对象在被点击后调用自身（无效化），使用静态变量统计被点击的怪物数量，当达到10时调用游戏结束界面（有效化），在挂载在每个Enemy对象的`Enemy.cs`脚本中实现（需要联系结束界面节点对象）：

```csharp
// Enemy.cs
namespace FrameworkDesign.Example
{
    public class Enemy : MonoBehaviour
    {
        public GameObject GamePassPanel;

        private static int mKilledEnemyCount = 0;

        private void OnMouseDown()
        {
            Destroy(gameObject);

            mKilledEnemyCount++;

            if (mKilledEnemyCount == 10)
            {
                GamePassPanel.SetActive(true);
            }
        }
    }
}
```

### 整理场景结构

创建UI和Games两个根节点，并将与之相关的节点分别置于两个根节点下，形成了清晰的树结构：

<img src=".\StructureDesignPic\image-2.png"/>

将其转化为树状图则为：

<img src=".\StructureDesignPic\image-3.png" width="50%" height="50%"/>

- 在理想情况下，一个APP/游戏最好只有一棵树

若为不同的节点对象添加互相的引用关系，则树状图如下：

<img src=".\StructureDesignPic\image-4.png" width="50%" height="50%"/>

这种通过简单连线、拖拽实现的项目会出现难以维护（因为拖拽的时候对象间引用关系没有规律可循，因此架构复杂后难以厘清关联逻辑）、难以协作（经常拖拽对象会导致场景文件内容经常更改，使用git管理容易冲突）、难以扩展的问题

### 规范化引用

如上图所示，由于不同对象之间混乱的引用关系，会出现难维护、难协作、难扩展的问题；而解决这个问题的途径只有一个：高内聚、低耦合

- 低耦合：减少对象、类之间的双向引用、循环引用
- 高内聚：同样类型的代码放在一起

在架构层面，只需要关注两个问题：对象之间如何交互（低耦合）、如何实现模块化（高内聚）

- 对象之间的交互方式一般分为三种：1.方法调用、2.委托/回调、3.消息/事件
- 模块化也一般分三种：1.单例、2.IOC、3.分层

#### 方法调用

要成功完成一次方法调用，需要调用方能够获取被调用方。举例如下：

```csharp
public class A
{
    public B B;
    void Start() 
    {
        B.SayHello();
    }
}
```

从这个例子就能看出，只有A引用B对象（能够访问B）才能调用B的`SayHello()`方法：A需要持有B

也正因如此，方法调用会造成至少单向引用的关系，无限制的使用方法调用很有可能造成双向引用、循环引用使代码耦合。例如刚才的引用关系：

<img src=".\StructureDesignPic\image-4.png" width="50%" height="50%"/>

其中包含三条调用关系：

- `BtnStartGame`调用`GameStartPanel`的`SetActive`方法
- `BtnStartGame`调用`Enemies`的`SetActive`方法
- `Enemies`调用`GamePassPanel`的`SetActive`方法

对于第一条：`BtnStartGame`调用`GameStartPanel`的`SetActive`方法，因为`BtnStartGame`是`GameStartPanel`的子节点，其为父子关系，但是父子关系也不能有双向、循环引用。一般来说共识是：父节点可以引用子节点，但是子节点不能引用父节点，因此此条引用应该被禁止

当子节点想要使用父结点中的方法时，可以使用第二种交互方式：委托/回调

#### 委托/回调

尝试用委托方式解决第一条调用关系：为父节点`GameStartPanel`附加脚本`GameStartPanel.cs`，并在其中说明委托关系：

```csharp
// GameStartPanel.cs
using UnityEngine;
using UnityEngine.UI;

namespace FrameworkDesign.Example
{
    public class GameStartPanel : MonoBehaviour
    {
        public GameObject Enemies;

        void Start()
        {
            transform.Find("BtnStart").GetComponent<Button>()
                .onClick.AddListener(() =>
                {
                    gameObject.SetActive(false);
                    Enemies.SetActive(true);
                });
            /* onClick是Button组件的一个事件，使用委托来存储监听器 */
            /* AddListener方法为onClick添加新委托方法，此处是一个用Lambda表达式简写的匿名方法 */
        }
    }
}
```

此时关系图变化如下，此时，`BtnStartGame`不再调用`GameStartPanel`，而是由`GameStartPanel`控制自身消失，显示`Enemies`：

<img src=".\StructureDesignPic\image-5.png" width="50%" height="50%"/>

此时，`BtnStartGame`不再承担任何业务逻辑（转移到`GameStartPanel`），而只是单纯作为UI存在。

总结：父节点可以引用子节点，子节点要调用父节点方法时，可以使用事件或者委托

当然，A也需要持有B才能够注册B的委托（实际上等价于B调用A的方法，B通知A）

注意，在采用委托、回调时尽量避免嵌套，可能造成代码混乱

#### 消息/事件

现在考虑`GameStartPanel`对`Enemies`的调用。因为这两个节点处在不同业务逻辑树下（一个属于游戏逻辑一个属于UI逻辑），此时这种调用叫做跨模块的对象交互

当对不同模块之间的开发进行分工时，跨模块的对象交互不利于分工合作

此时，无论是调用还是委托都无法完成，因为这两个方法都要求至少一方持有另一方。而跨模块对象之间没有任何持有关系

只能使用消息/事件的方式满足要求。增加一个`GameStartEvent.cs`脚本，其中封装一个事件静态类：

```csharp
// GameStartEvent.cs
using System;

namespace FrameworkDesign.Example
{
    /* 此静态类只是封装并维护了一个静态委托`mOnEvent`而已 */
    public static class GameStartEvent
    {
        private static Action mOnEvent; // Action是无参数、返回为空的委托

        public static void Register(Action onEvent)
        {
            mOnEvent += onEvent;
        }

        public static void UnRegister(Action onEvent)
        {
            mOnEvent -= onEvent;
        }

        public static void Trigger()
        {
            mOnEvent?.Invoke();
        }
    }
}
```

此时，可以修改`GameStartPanel.cs`脚本，让其与`Enemies`节点脱钩，而去触发`GameStartEvent`事件：

```csharp
// GameStartPanel.cs
using UnityEngine;
using UnityEngine.UI;

namespace FrameworkDesign.Example
{
    public class GameStartPanel : MonoBehaviour
    {
        void Start()
        {
            transform.Find("BtnStart").GetComponent<Button>()
                .onClick.AddListener(() =>
                {
                    gameObject.SetActive(false);

                    GameStartEvent.Trigger();
                });
        }
    }
}
```

现在的问题是，由于`Enemies`的显示事件的注册需要在其`Awake`或`Start`方法中完成，因此在游戏开始时`Enemies`无法完成注册，需要求助于其更高父节点`Game`。为其挂载脚本`Game.cs`：

```csharp
// Game.cs
using UnityEngine;

namespace FrameworkDesign.Example
{
    public class Game : MonoBehaviour
    {
        private void Awake()
        {
            GameStartEvent.Register(OnGameStart);
        }

        private void OnGameStart() // 触发时要调用的方法：子节点Enemies出现
        {
            transform.Find("Enemies").gameObject.SetActive(true); 
        }

        private void OnDestroy()
        {
            GameStartEvent.UnRegister(OnGameStart);
        }
    }
}
```

修改后，各个节点之间的调用关系如下：

<img src=".\StructureDesignPic\image-6.png" width="50%" height="50%"/>

总结：使用消息/事件进行交互时，A和B不需要存在持有关系

通过节点树可以发现，`Enemy`和`GamePassPanel`之间的关系也属于跨模块通信，可以通过事件完成。首先创建一个`GamePassEvent`事件（创建新`GamePassEvent.cs`脚本，但是内容和`GameStartEvent.cs`相同，此处略）

然后要将原先使用引用的`Enemy.cs`标本改成点击十次后触发`GamePassEvent`事件：

```csharp
//Enemy.cs
using UnityEngine;

namespace FrameworkDesign.Example
{
    public class Enemy : MonoBehaviour
    {
        private static int mKilledEnemyCount = 0;

        private void OnMouseDown()
        {
            Destroy(gameObject);

            mKilledEnemyCount++;

            if (mKilledEnemyCount == 10)
            {
                GamePassEvent.Trigger();
            }
        }
    }
}
```

在点击事件发生后，接收这个事件并弹出`GamePassPanel`的应该是`UI`节点，因为点击时`GamePassPanel`是隐藏的（和刚才接收事件的是`Game`节点同理）。因此创建一个`UI.cs`脚本：

```csharp
//UI.cs
using UnityEngine;

namespace FrameworkDesign.Example
{
    public class UI : MonoBehaviour
    {
        private void Start()
        {
            GamePassEvent.Register(OnGamePass);
        }

        private void OnGamePass()
        {
            transform.Find("Canvas/GamePassPanel").gameObject.SetActive(true);
        }

        private void OnDestroy()
        {
            GamePassEvent.UnRegister(OnGamePass);
        }
    }
}
```

（之所以挂载在`UI`而非`Canvas`节点是因为后者是一个无逻辑的节点，可以忽略）

经过优化后，最终的节点图如下：

<img src=".\StructureDesignPic\image-7.png" width="50%" height="50%"/>

- 总结：父节点调用子节点直接方法调用；子节点通知父节点用委托或事件；跨模块通信用事件

### 表现和数据分离

在上一节中，为了实现事件通信，创建了`GameStartEvent.cs`和`GamePassEvent.cs`，但实际上两段代码除了名字，其内容完全一致（维护一个委托，对其实现方法的注册、删除、触发）。可以进一步优化

当两个类的实现代码完全一样，仅类名或类型不一致，且后续可能会扩展的时候，可以使用泛型+继承进行提取。

创建一个新的类`Event.cs`：

```csharp
//Event.cs
using System;

namespace FrameworkDesign
{
    public class Event<T> where T: Event<T>
    {
        private static Action mOnEvent;

        public static void Register(Action onEvent)
        {
            mOnEvent += onEvent;
        }

        public static void UnRegister(Action onEvent)
        {
            mOnEvent -= onEvent;
        }

        public static void Trigger()
        {
            mOnEvent?.Invoke();
        }
    }
}
```

同时需要让原先的两个脚本继承此脚本：

```csharp
// GameStartEvent.cs, 另一个脚本同理
namespace FrameworkDesign.Example
{
    public class GameStartEvent : Event<GameStartEvent>
    {
       
    }
}
```

（注意：此处使用了静态泛型机制，其核心为CRTP奇异递归模板模式；用于在基类中实现静态泛型，因为静态子类是无法存在继承关系的，这种方法才能让子类兼具继承性和静态属性）

此时，我们的引用关系图变成了：

<img src=".\StructureDesignPic\image-8.png" width="50%" height="50%"/>

### 分离表现和数据

游戏中所需的数据部分：点击10次怪物才能通关，这是一个计数逻辑。但是这个逻辑被写在了`Enemy.cs`中（见上）

这就是一个比较严重的问题：游戏的核心数据和表现的部分没有分离。在未来，多个功能所需数据在大多数情况需要在多个场景、界面间共享，且不仅在空间上，在时间上也要共享（存储）

因此需要将数据的部分抽离，单独放在一个地方进行维护。比较常用的就是MVC的开发架构，先采用其中的一个概念：Model

Model的功能就是管理数据（通过其对象/类进行增删改查数据）、存储数据

在此处，可以建立一个`GameModel`类：

```csharp
// GameModel.cs
using UnityEngine;

namespace FrameworkDesign.Example
{
    public class GameModel
    {
        public static int KillCount = 0;

        public static int Gold = 0;
        public static int Score = 0;
        public static int BestScore = 0;
    }
}
```

然后就可以把`KillCount`全局变量存储的数据用于`Enemy.cs`代码中：

```csharp
// Enemy.cs
using UnityEngine;

namespace FrameworkDesign.Example
{
    public class Enemy : MonoBehaviour
    {
        private void OnMouseDown()
        {
            Destroy(gameObject);

            GameModel.KillCount++;

            if (GameModel.KillCount == 10)
            {
                GamePassEvent.Trigger();
            }
        }
    }
}
```

此时游戏中数据和表现逻辑就分开了。关系图变为（进行了简化）：

<img src=".\StructureDesignPic\image-9.png" width="50%" height="50%"/>

当前的架构还有一个问题：游戏的胜负代码是写在单个敌人`Enemy`的代码中的，但实际上，一个游戏是否通关的判定应该由这个游戏本身的概念进行判定，在这里就出现了“不该在这里实现的逻辑，实现在了这里”的问题

这种规则判断的逻辑，应该实现在游戏概念节点，也就是`Game`中。因此，现在需要将判断逻辑迁移到`Game.cs`中

那么，就需要实现在`Enemy`被点击时，要告诉`Game`类自己被点击了。这是一种子节点向父节点通信的例子。但是由于父子节点是一对十的关系，使用委托实现不太现实（若用委托，则需要保存所有十个子节点的调用，要维护一个`Enemy`数组）

此处，使用事件是一个好选择。增加一个`KilledOneEnemyEvent`事件。代码如下：

```csharp
// KilledOneEnemyEvent.cs
using UnityEngine;

namespace FrameworkDesign.Example
{
    public class KilledOneEnemyEvent : Event<KilledOneEnemyEvent>
    {
        
    }
}
```

因为每次一个怪物被点击，就触发一次这个事件，因此需要更改怪物代码：

```csharp
// Enemy.cs
using UnityEngine;

namespace FrameworkDesign.Example
{
    public class Enemy : MonoBehaviour
    {
        private void OnMouseDown()
        {
            Destroy(gameObject);

            KilledOneEnemyEvent.Trigger();
        }
    }
}
```

并且要在`Game.cs`中监听这个事件。代码如下：

```csharp
// Game.cs
using UnityEngine;

namespace FrameworkDesign.Example
{
    public class Game : MonoBehaviour
    {
        private void Awake()
        {
            GameStartEvent.Register(OnGameStart);
            KilledOneEnemyEvent.Register(OnEnemyKilled);
        }

        private void OnEnemyKilled()
        {
            GameModel.KillCount++;

            if (GameModel.KillCount == 10)
            {
                GamePassEvent.Trigger();
            }
        }

        private void OnGameStart()
        {
            transform.Find("Enemies").gameObject.SetActive(true);
        }

        private void OnDestroy()
        {
            GameStartEvent.UnRegister(OnGameStart);
            KilledOneEnemyEvent.UnRegister(OnEnemyKilled);
        }
    }
}
```

此时，引用图变为：

<img src=".\StructureDesignPic\image-10.png" width="50%" height="50%"/>

至此，整个项目做到了正确的代码放到正确的位置，这一原则叫做单一职责原则。只要代码对业务抽象的合理自然就必然符合这一原则

（补充：对于数据存储，如果是共享的数据就应该放在Model里，如果不是共享的就不需要。共享的数据可以包括配置数据、需要存储的数据、在多个地方都需要访问的数据）

### 交互逻辑和表现逻辑

举例：假如现在有一个计算器APP，在界面上点击加号，界面显示的数字就会递增；点击减号，界面显示的数字就会递减

此时，交互逻辑就是：当点击界面(View)上的加号或者减号时，程序会更新Model中的Count数据以记录

用伪代码表示如下（假设使用`Controller`进行控制）：

```csharp
public class Controller 
{
    void Start()
    {
        View.BtnAdd.onClick.AddListener(() => { // 交互逻辑
            Model.Count++;
        });

        View.BtnSub.onClick.AddListener(() => { // 交互逻辑
            Model.Count--;
        });
    }
}
```

而表现逻辑就是：当Model初始化或变更数据后，从Model中查出数据并显示到View上的，就是表现逻辑（如此例中保存当前数字的数据改变后，应该将递增或递减的数字显示在屏幕上）

伪代码表示如下：

```csharp
public class Controller 
{
    void Start()
    {
        View.CountText.text = Model.Count.ToString(); // 初始化表现逻辑

        View.BtnAdd.onClick.AddListener(() => { 
            Model.Count++; // 交互逻辑
            View.CountText.text = Model.Count.ToString(); // 表现逻辑
        });

        View.BtnSub.onClick.AddListener(() => { 
            Model.Count--; // 交互逻辑
            View.CountText.text = Model.Count.ToString(); // 表现逻辑
        });
    }
}
```

- 简单来说：从View到Model的就是交互逻辑，反之就是表现逻辑

在开发时，经常采用表现View和数据Model分离的思想，只要知道这两者之间有这两种逻辑，想清楚就比较好实现

一般来说Controller会同时负责这两种逻辑，随着项目规模加大，Controller中的代码会越来越臃肿。解决的方法是引入Command概念

根据上述例子创建一个小项目。其节点关系如下：

<img src=".\StructureDesignPic\image-11.png"/>

再创建一个用来管理交互和表现逻辑的`CounterViewController.cs`（将其挂载于`Panel`上）:

```csharp
// CounterViewContrller.cs
using UnityEngine;
using UnityEngine.UI;

namespace CounterApp
{
    public class CounterViewController : MonoBehaviour
    {
        private void Start()
        {
            UpdateView();

            transform.Find("BtnAdd").GetComponent<Button>().onClick.AddListener(() =>
            {
                CounterModel.Count++;
                UpdateView();
            });

            transform.Find("BtnSub").GetComponent<Button>().onClick.AddListener(() =>
            {
                CounterModel.Count--;
                UpdateView();
            });
        }

        void UpdateView()
        {
            transform.Find("CountText").GetComponent<Text>().text = CounterModel.Count.ToString();
        }
    }

    public static class CounterModel
    {
        public static int Count = 0;
    }
}
```

之所以叫做`ViewController`，因为在unity中难以区分View和Controller概念，View可以理解为感兴趣的组件应用，比如此示例中的两个按钮和一个文本框。只需要继承`MonoBehaviour`就可以轻松获取这些组件，因此单独创建`View`脚本比较浪费

可以理解为：一个脚本由于挂载在`GameObject`上，所以它既有View的职责，也有Controller的职责

接下来需要考虑表现逻辑和交互逻辑应该使用怎样的交互方法比较合适。在原先的代码中，表现逻辑是在交互逻辑之后马上进行了调用：

```csharp
transform.Find("BtnAdd").GetComponent<Button>().onClick.AddListener(() =>
{
    CounterModel.Count++; // 交互逻辑
    UpdateView(); // 表现逻辑
});
```

因此此处使用的是单纯的方法调用，在Controller中获取`Model`对象，然后将这个对象的`Count`数据显示在View上

如果采用委托调用方式，View只需要监听`CounterModel`的数据变化就可以。首先更改`CounterModel`：

```csharp
// CounterViewContrller.cs
public static class CounterModel
{
    private static int mCount = 0;

    public static Action<int> OnCountChanged;

    public static int Count
    {
        get => mCount;
        set
        {
            if (value != mCount) { mCount = value; }

            OnCountChanged?.Invoke(value); // 当存储数据发生变化，触发委托
        }
    }
}
```

然后在`ViewController`中需要监听`OnCountChanged`委托：

```csharp
// CounterViewContrller.cs
public class CounterViewController : MonoBehaviour
{
    private void Start()
    {
        CounterModel.OnCountChanged += OnCountChanged;

        OnCountChanged(CounterModel.Count); // 在初始化时要主动调用一次表现逻辑

        transform.Find("BtnAdd").GetComponent<Button>().onClick.AddListener(() =>
        {
            CounterModel.Count++; // 交互逻辑
        });

        transform.Find("BtnSub").GetComponent<Button>().onClick.AddListener(() =>
        {
            CounterModel.Count--; // 交互逻辑
        });
    }

    private void OnCountChanged(int newCount)
    {
        transform.Find("CountText").GetComponent<Text>().text = newCount.ToString();
    }

    private void OnDestroy()
    {
        CounterModel.OnCountChanged -= OnCountChanged;
    }
}
```

此时变成了交互逻辑使用View调用Model，而表现逻辑是在Model内部值改变的时候通过委托通知View改变

还可以通过事件/消息的方式进行交互。首先修改Model内容，每次值改变的时候触发事件，同时需要新定义一个继承之前写的事件泛型的事件：

```csharp
// CounterViewContrller.cs
public static class CounterModel
{
    private static int mCount = 0;

    public static int Count
    {
        get => mCount;
        set
        {
            if (value != mCount) { mCount = value; }

            OnCountChangedEvent.Trigger();
        }
    }
}

public class OnCountChangedEvent : Event<OnCountChangedEvent>
{

}
```

然后同样要再Controller中监听事件：

```csharp
// CounterViewContrller.cs
public class CounterViewController : MonoBehaviour
{
    private void Start()
    {
        OnCountChangedEvent.Register(OnCountChanged);

        OnCountChanged();

        transform.Find("BtnAdd").GetComponent<Button>().onClick.AddListener(() =>
        {
            CounterModel.Count++;
        });

        transform.Find("BtnSub").GetComponent<Button>().onClick.AddListener(() =>
        {
            CounterModel.Count--;
        });
    }

    private void OnCountChanged()
    {
        transform.Find("CountText").GetComponent<Text>().text = CounterModel.Count.ToString();
    }

    private void OnDestroy()
    {
        OnCountChangedEvent.UnRegister(OnCountChanged);
    }
}
```

当然，这里存在着一点缺陷：之前定义的`Event`泛型中的委托的方法不支持携带参数。后续需要修改

通过对比可知，通过委托回调或事件消息的方式可以缩减Controller中的代码量

原先的逻辑问题在于：每次按钮点击之后，都需要Controller主动调用表现逻辑。当有很多数据的时候，会造成很多的代码调用，增加心智负担，同时会让Controller变得臃肿，增加维护成本

因此，表现逻辑的实现，使用委托、事件的方式是优于主动调用的。具体来说，如果是单个数值变化，使用委托更加合适；如果是颗粒度较大的更新，使用事件更加合适

### 可绑定属性BindableProperty

现在，在Model中，任意一个`Count`就要写一个`mCount`变量和一个`OnCountChanged`委托，还要有值比较的代码。因此可以引入可绑定属性 BindableProperty。具体泛型代码如下：

```csharp
// BindableProperty.cs
using System;

namespace FrameworkDesign
{
    public class BindableProperty<T> where T : IEquatable<T> // 要求类型T支持Equal方法（可比较）
    {
        private T mValue = default(T); // 默认为0或null

        public T Value
        {
            get => mValue;
            set
            {
                if (!value.Equals(mValue))  // value是set方法中的隐式参数，代表新赋值
                { 
                    mValue = value;
                    OnValueChanged?.Invoke(mValue);
                }
            }
        }

        public Action<T> OnValueChanged;
    }
}
```

然后尝试使用这个属性。首先修改Model部分的代码：

```csharp
// CounterViewContrller.cs
public static class CounterModel
{
    public static BindableProperty<int> Count = new BindableProperty<int>() { Value = 0 };
}
```

然后再将Controller的代码改回委托形式：

```csharp
// CounterViewContrller.cs
public class CounterViewController : MonoBehaviour
{
    private void Start()
    {
        CounterModel.Count.OnValueChanged += OnCountChanged;

        OnCountChanged(CounterModel.Count.Value);

        transform.Find("BtnAdd").GetComponent<Button>().onClick.AddListener(() =>
        {
            CounterModel.Count.Value++;
        });

        transform.Find("BtnSub").GetComponent<Button>().onClick.AddListener(() =>
        {
            CounterModel.Count.Value--;
        });
    }

    private void OnCountChanged(int newCount)
    {
        transform.Find("CountText").GetComponent<Text>().text = newCount.ToString();
    }

    private void OnDestroy()
    {
        CounterModel.Count.OnValueChanged -= OnCountChanged;
    }
}
```

使用这样的泛型就可以减少样本代码（除了类型一模一样的方法），可以减少代码的编写时间

至此，`CounterApp`变成了一个数据驱动的应用

从宏观分析：数据一般是一个项目的底层，而父子节点是一个自底向上的过程，因此Model和View可以看作底层和上层，子和父的关系。因此，表现逻辑就是一种自底向上的逻辑，这种逻辑自然可以使用委托或者事件

现在尝试将原先的项目也使用BindableProperty进行优化。改变`GameModel`：

```csharp
// GameModel.cs
namespace FrameworkDesign.Example
{
    public class GameModel
    {
        public static BindableProperty<int> KillCount = new BindableProperty<int>() { Value = 0 };

        public static BindableProperty<int> Gold = new BindableProperty<int>() { Value = 0 };
        public static BindableProperty<int> Score = new BindableProperty<int>() { Value = 0 };
        public static BindableProperty<int> BestScore = new BindableProperty<int>() { Value = 0 };
    }
}
```

然后需要改变`Game`类以进行适配：

```csharp
// Game.cs
private void OnEnemyKilled()
{
    GameModel.KillCount.Value++;

    if (GameModel.KillCount.Value == 10)
    {
        GamePassEvent.Trigger();
    }
}
// 其余略
```

再次观察结构图：

<img src=".\StructureDesignPic\image-10.png" width="50%" height="50%"/>

发现其实`KilledOneEnemyEvent`可以省略，改换为`GameModel.KilledCount`的变更事件，在点击Enemy后直接让`GameModel.KilledCount++`即可，代码改变：

```csharp
// Enemy.cs
public class Enemy : MonoBehaviour
{
    private void OnMouseDown()
    {
        Destroy(gameObject);

        GameModel.KillCount.Value++;
    }
}
```

同时，需要在`Game.cs`中监听`GanmeModel.KillCount`的变化：

```csharp
// Game.cs
using UnityEngine;

namespace FrameworkDesign.Example
{
    public class Game : MonoBehaviour
    {
        private void Awake()
        {
            GameStartEvent.Register(OnGameStart);
            GameModel.KillCount.OnValueChanged += OnEnemyKilled;
        }

        private void OnEnemyKilled(int killCount)
        {
            if (killCount == 10)
            {
                GamePassEvent.Trigger();
            }
        }

        private void OnGameStart()
        {
            transform.Find("Enemies").gameObject.SetActive(true);
        }

        private void OnDestroy()
        {
            GameStartEvent.UnRegister(OnGameStart);
            GameModel.KillCount.OnValueChanged -= OnEnemyKilled;
        }
    }
}
```

自此，结构图演化为：

<img src=".\StructureDesignPic\image-12.png" width="50%" height="50%"/>

明显，图中交互逻辑是KilledCount++的部分，而表现逻辑是KilledCountChanged的部分

### 优化交互逻辑：引入Command

目前，计数器的交互逻辑如下：

```csharp
transform.Find("BtnAdd").GetComponent<Button>().onClick.AddListener(() =>
{
    CounterModel.Count.Value++; // 交互逻辑改变Model值后自动调用表现逻辑
});

transform.Find("BtnSub").GetComponent<Button>().onClick.AddListener(() =>
{
    CounterModel.Count.Value--;
});
```

但是正常开发中，交互逻辑中包含的代码量，操作量可能很大。而一个Controller如果包含过多交互逻辑，会变得臃肿。利用`Service`层抽象可以稍微缓解（相当于将所有交互逻辑方法另写在一个类中，而在Controller中只调用接口），但这种方法在方法过多时也会难以维护

更好的解决方法，是引入Command：命令模式。简单理解就是本来应该用一个方法实现的逻辑，改成用一个对象实现，而这个对象中只有一个执行方法

首先定义一个接口`ICommand`：

```csharp
// ICommand.cs
namespace FrameworkDesign
{
    public interface ICommand
    {
        void Execute(); // 继承接口的类需要实现自己的执行方法
    }
}
```

然后要用继承这个接口的新类实现交互逻辑，以增加的交互举例：

```csharp
// AddCountCommand.cs
using FrameworkDesign;

namespace CounterApp
{
    public struct AddCountCommand : ICommand // 用struct性能更好一点，如果使用class，每次需要交互逻辑就要new，性能消耗大
    {
        public void Execute()
        {
            CounterModel.Count.Value++;
        }
    }
}
```

最后更改原本的交互逻辑位置：
```csharp
transform.Find("BtnAdd").GetComponent<Button>().onClick.AddListener(() =>
{
    new AddCountCommand().Execute();
});

transform.Find("BtnSub").GetComponent<Button>().onClick.AddListener(() =>
{
    new SubCountCommand().Execute();
});
```

使用Command符合读写分离原则。Command模式是逻辑的调用和执行是分离的。正常的方法是无法分离的。Command模式空间分离是因为调用的位置和执行的位置在不同文件，时间的分离就是调用之后Command过了一点点时间才被执行

因为Command模式有调用和执行分离的特点，就可以用不同的数据结构组织Command的调用，例如队列和堆栈，就可以实现撤销功能

目前Command可以分担Controller中的交互逻辑职责，分离复杂的交互逻辑代码

### CounterApp的编辑器扩展版本

目前为止，这个例子中的表现层包括UI和`ViewController`，而底层系统层则包含其他内容：`AddCountCommand`和`SubCountCommand`，`Model`和数据改变时的回调方法

由于系统层内容是和UNITY引擎脱钩的（不需要`MonoBehaviour`），因此可以组合其他的表现层实现其他的功能。比如实现一个编辑器的扩展：

```csharp
// EditorCounterApp.cs
using UnityEditor;
using UnityEngine;

namespace CounterApp.Editor
{
    public class EditorCounterApp : EditorWindow
    {
        [MenuItem("EditorCounterApp/Open")]
        static void Open()
        {
            var window = GetWindow<EditorCounterApp>();
            window.position = new Rect(100, 100, 400, 600);
            window.titleContent = new GUIContent(nameof(EditorCounterApp));
            window.Show();
        }

        private void OnGUI()
        {
            if (GUILayout.Button("+"))
            {
                new AddCountCommand().Execute();
            }

            GUILayout.Label(CounterModel.Count.Value.ToString());

            if (GUILayout.Button("-"))
            {
                new SubCountCommand().Execute();
            }
        }
    }
}
```

也就是说，切换表现层非常方便，底层系统比较容易共享。如果交互逻辑直接在Controller当中实现，Controller和View有一定的耦合，这就代表交互逻辑不完全在系统层

同时，直接用Command类对应操作比较易于理解，可读性比较强

而且这套架构比较方便原型设计，因为原型设计时表现层比较简陋，但是如果使用这种架构，在构造原型时使用的系统层在添加美术时完全不需要改变

而且可以用于分工：表现层和系统层。表现层到系统层用COMMAND，而系统层通过事件或者委托通知表现层