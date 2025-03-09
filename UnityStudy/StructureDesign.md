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
        }
    }
}
```


