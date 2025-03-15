## 游戏框架搭建 笔记2

### 在原游戏例子中引入Command

目前，使用的关系图如下：

<img src=".\StructureDesignPic\image-12.png" width="50%" height="50%"/>

现在操作数据的部分（交互逻辑）只有`KilledCount++`，只需要实现一个`KillEnemyCommand`命令即可。代码如下：

```csharp
// KillEnemyCommand.cs
namespace FrameworkDesign.Example
{
    public struct KillEnemyCommand : ICommand
    {
        public void Execute()
        {
            GameModel.KillCount.Value++;
        }
    }
}
```

然后在`Enemy`中也使用这种命令的形式完成交互逻辑：

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

            new KillEnemyCommand().Execute();
        }
    }
}
```

此时关系图变为如下形式，命令和模型都属于系统层：

<img src=".\StructureDesign2Pic\image.png" width="50%" height="50%"/>

目前系统层有如下元素：Command/BindableProperty/Model，其中BindableProperty相当于数据加事件，而数据内容由Model负责，因此也可以说目前系统层元素包括:Command/Event/Model

实际上，这个例子中的`GameStartEvent`和`GamePassEvent`也应该属于底层（事件应该属于系统层）：

<img src=".\StructureDesign2Pic\image-1.png" width="50%" height="50%"/>

实际上这两个事件是游戏的状态变更事件：未开始状态 - 游戏中状态 - 游戏结束状态。这样一个状态并没有被用代码储存在Model，但其实应该存在，可以在`GameModel`中用一个枚举表示

应该规定，表现层只能向系统层发送Command或者进行数据查询，事件只能由底层系统层向表现层发送

因此，这两个事件也可以看作对Model中的状态变量做更改的交互逻辑。因此也应该创建两个Command进行实现：`StartGameCommand`和`PassGameCommand`：

```csharp
// StartGameCommand.cs
namespace FrameworkDesign.Example
{
    public class StartGameCommand : ICommand
    {
        public void Execute()
        {
            GameStartEvent.Trigger();
        }
    }
}
// PassGameCommand同理
```

其实这里触发的事件是状态数据变更事件，和之前接触到的BindableProperty的数据变更事件是一样的事件

然后在`GameStartPanel`中应用游戏开始命令：

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

                    new StartGameCommand().Execute();
                });
        }
    }
}
```

以及在`Game`中应用游戏通过命令：

```csharp
// Game.cs
private void OnEnemyKilled(int killCount)
{
    if (killCount == 10)
    {
        new PassGameCommand().Execute();
    }
}
// 其他略
```

但是经过思考，上面这段判断游戏结束的逻辑放在底层应该是更合理的，因为表现层应该只负责表现和接收用户操作。由于每次击杀一个敌人，就会进行一个判断，所以可以放到`EnemyKillCommand`中进行判断：

```csharp
// EnemyKillCommand.cs
namespace FrameworkDesign.Example
{
    public struct KillEnemyCommand : ICommand
    {
        public void Execute()
        {
            GameModel.KillCount.Value++;

            if (GameModel.KillCount.Value == 10)
            {
                new PassGameCommand().Execute();
            }
        }
    }
}
```

至此，`Game`节点不再需要管理和击杀敌人相关的问题了，可以删除所有相关代码：

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

        private void OnGameStart()
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

然后`PassGameCommand`也不再需要了，只需要使用`GamePassEvent`即可：

```csharp
// EnemyKillCommand.cs
namespace FrameworkDesign.Example
{
    public struct KillEnemyCommand : ICommand
    {
        public void Execute()
        {
            GameModel.KillCount.Value++;

            if (GameModel.KillCount.Value == 10)
            {
                GamePassEvent.Trigger();
            }
        }
    }
}
```

再次更新结构图：

<img src=".\StructureDesign2Pic\image-2.png" width="50%" height="50%"/>

总结规律：

- 事件由系统层向表现层发送
- 表现层只能用Command改变底层系统层的状态（数据）
- 表现层可以直接查询数据

### 模块化优化-引入单例

如何进行底层系统层模块化，需要从两个角度分析：

- 模块对象如何获取
- 如何增加一个模块

目前只有Model可以当作一个代码模块，命令算作操作，可以扩展但对象不可获取，而Model是共享的，会被很多地方引用，且自身有状态，因此可以作为一个代码模块

举例，目前的计数器的模块如下：

```csharp
public static class CounterModel
{
    public static BindableProperty<int> Count = new BindableProperty<int>() { Value = 0 };
}
```

此处是利用静态类当作模块。静态类可以直接获取；扩展一个静态类就是增加一个`static`关键字

静态类实现模块化的方式非常简便，但是随着开发会出现模块间的互相引用关系混乱、随机、没有规律的问题

静态类没有限制别的地方对它的访问，同时实现一个静态类只需要增加静态关键字，而使用静态关键字的类是很多的，难以快速分辨某个类是否是一个模块

对于静态类没有访问限制：可以稍微增加限制；同时可以增加一些识别度。使用单例模式解决即可，其无法通过简单引用获取，而需要通过`ClassName.Instance`，增加了模块获取的难度，同时比较容易识别

实现一个简单的单例工具类（泛型单例）：

```csharp
// Singleton.cs
using System;
using System.Reflection;

namespace FrameworkDesign
{
    public class Singleton<T> where T : Singleton<T>
    {
        private static T mInstance;

        public static T Instance
        {
            get
            {
                if (mInstance == null)
                {
                    var type = typeof(T);
                    var ctors = type.GetConstructors(BindingFlags.Instance | BindingFlags.NonPublic);
                    var ctor = Array.Find(ctors, c => c.GetParameters().Length == 0);

                    if (ctor == null)
                    {
                        throw new Exception("Non Public Constructor Not Found in " + type.Name);
                    }

                    mInstance = ctor.Invoke(null) as T;
                }

                return mInstance;
            }
        }
    }
}
```

此处创建新的`mInstance`运用了反射相关知识，后面再补

现在尝试在计数器APP中使用（别的位置对`CounterModel`的引用都需要改为`CounterModel.Instance`）：

```csharp
// CounterViewController.cs
public class CounterModel : Singleton<CounterModel>
{
    private CounterModel() { } // 这样构造的单例模式都需要一个私人构造方法
    public BindableProperty<int> Count = new BindableProperty<int>() { Value = 0 };
}
```

同样在游戏例子中实现单例模式：

```csharp
// GameModel.cs
namespace FrameworkDesign.Example
{
    public class GameModel : Singleton<GameModel>
    {
        private GameModel() { }

        public BindableProperty<int> KillCount = new BindableProperty<int>() { Value = 0 };

        public BindableProperty<int> Gold = new BindableProperty<int>() { Value = 0 };

        public BindableProperty<int> Score = new BindableProperty<int>() { Value = 0 };

        public BindableProperty<int> BestScore = new BindableProperty<int>() { Value = 0 };
    }
}
```

但是这种模式的单例，仍然没有访问限制，需要继续优化

### IOC容器

单例类的一个问题是：由于所有单例类都继承自`Singleton`类，导致如果单例类之间存在层级的差别或者调用的关系，这种关系难以通过代码展示

IOC容器可以理解为一个字典，这个字典以Type为Key，以对象Instance为value。实现一个简单的IOC如下：

```csharp
// IOCContainer.cs
using System;
using System.Collections.Generic;

namespace FrameworkDesign
{
    public class IOCContainer
    {
        Dictionary<Type, object> mInstances = new Dictionary<Type, object>();

        public void Register<T>(T instance)
        {
            var key = typeof(T);
            
            if (mInstances.ContainsKey(key))
            {
                mInstances[key] = instance;
            }
            else
            {
                mInstances.Add(key, instance);
            }
        }

        public T Get<T>() where T : class
        {
            var key = typeof(T);

            if (mInstances.TryGetValue(key, out var retInstance))
            {
                return retInstance as T;
            }

            return null;
        }
    }
}
```

一个简单的应用：

```csharp
using UnityEngine;

namespace FrameworkDesign.Example
{
    public class IOCExample : MonoBehaviour
    {
        void Start()
        {
            var container = new IOCContainer(); // 创建容器

            container.Register(new BluetoothManager()); // 注册实例

            var bluetoothManager = container.Get<BluetoothManager>(); // 获取实例

            bluetoothManager.Connect(); // 执行操作
        }
    }

    public class BluetoothManager
    {
        public void Connect()
        {
            Debug.Log("successfully connect!");
        }
    }
}
```

对计数器使用IOC容器：

```csharp
using FrameworkDesign;

namespace CounterApp
{
    public class CounterApp
    {
        private static IOCContainer mCountainer;

        static void MakeSureCountainer()
        {
            if (mCountainer == null)
            {
                mCountainer = new IOCContainer();
                Init();
            }
        }

        static void Init()
        {
            mCountainer.Register(new CounterModel());
        }

        public static T Get<T>() where T : class
        {
            MakeSureCountainer();

            return mCountainer.Get<T>();
        }
    }
}
```

更改计数器的模板引用代码：

```csharp
using System;
using UnityEngine;
using UnityEngine.UI;
using FrameworkDesign;

namespace CounterApp
{
    public class CounterViewController : MonoBehaviour
    {
        private CounterModel mCounterModel;

        private void Start()
        {
            mCounterModel = CounterApp.Get<CounterModel>();

            mCounterModel.Count.OnValueChanged += OnCountChanged;

            OnCountChanged(mCounterModel.Count.Value);

            transform.Find("BtnAdd").GetComponent<Button>().onClick.AddListener(() =>
            {
                new AddCountCommand().Execute();
            });

            transform.Find("BtnSub").GetComponent<Button>().onClick.AddListener(() =>
            {
                new SubCountCommand().Execute();
            });
        }

        private void OnCountChanged(int newCount)
        {
            transform.Find("CountText").GetComponent<Text>().text = newCount.ToString();
        }

        private void OnDestroy()
        {
            mCounterModel.Count.OnValueChanged -= OnCountChanged;

            mCounterModel = null;
        }
    }

    public class CounterModel
    {
        public BindableProperty<int> Count = new BindableProperty<int>() { Value = 0 };
    }
}
```

更改加减指令：

```csharp
using FrameworkDesign;

namespace CounterApp
{
    public struct AddCountCommand : ICommand
    {
        public void Execute()
        {
            CounterApp.Get<CounterModel>().Count.Value++;
        }
    }
}
```

现在在例子游戏中加入IOC容器：

```csharp
using UnityEngine;

namespace FrameworkDesign.Example
{
    public class PointGame
    {
        private static IOCContainer mCountainer;

        static void MakeSureContainer()
        {
            if (mCountainer == null)
            {
                mCountainer = new IOCContainer();
                Init();
            }
        }

        static void Init()
        {
            mCountainer.Register(new GameModel());
        }

        public static T Get<T>() where T : class
        {
            MakeSureContainer();

            return mCountainer.Get<T>();
        }
    }
}
```

更改模板，取消单例模式：

```csharp
namespace FrameworkDesign.Example
{
    public class GameModel
    {

        public BindableProperty<int> KillCount = new BindableProperty<int>() { Value = 0 };

        public BindableProperty<int> Gold = new BindableProperty<int>() { Value = 0 };

        public BindableProperty<int> Score = new BindableProperty<int>() { Value = 0 };

        public BindableProperty<int> BestScore = new BindableProperty<int>() { Value = 0 };
    }
}
```

更新在命令中的模板调用：

```csharp
namespace FrameworkDesign.Example
{
    public struct KillEnemyCommand : ICommand
    {
        public void Execute()
        {
            var gameModel = PointGame.Get<GameModel>();

            gameModel.KillCount.Value++;

            if (gameModel.KillCount.Value == 10)
            {
                GamePassEvent.Trigger();
            }
        }
    }
}
```

然而，在`CounterApp`和`PointGame`中对IOC容器的代码编写是几乎一致的，因此可以整合到样板代码中：

```csharp
namespace FrameworkDesign
{
    public abstract class Architecture<T> where T : Architecture<T>, new()
    {
        private static T mArchitecture;

        static void MakeSureArchitecture()
        {
            if (mArchitecture == null)
            {
                mArchitecture = new T();
                mArchitecture.Init();
            }
        }

        protected abstract void Init();

        private IOCContainer mCountainer = new IOCContainer();

        public static T Get<T>() where T : class
        {
            MakeSureArchitecture();

            return mArchitecture.mCountainer.Get<T>();
        }

        public void Register<T>(T instance)
        {
            MakeSureArchitecture();

            mArchitecture.mCountainer.Register<T>(instance);
        }
    }
}
```

在计数器中应用：

```csharp
using FrameworkDesign;

namespace CounterApp
{
    public class CounterApp : Architecture<CounterApp>
    {
        protected override void Init()
        {
            Register(new CounterModel());
        }
    }
}
```

在例子游戏中应用：

```csharp
namespace FrameworkDesign.Example
{
    public class PointGame : Architecture<PointGame>
    {
        protected override void Init()
        {
            Register(new GameModel());
        }
    }
}
```

这里也体现了一个设计代码的思路：当使用继承关系进行代码提取时，需要思考子类要做什么（继承`Init`），父类要做什么（提供`Get`和`Register`）

使用这种模式的好处是：为访问模块提供了限制（这个限制可以在`Architecture`中实现），同时可以使用一个主体类完成所有模块的注册工作，让项目底层模块一目了然