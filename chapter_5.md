# 第5章：搭积木的乐趣——建造者模式 (Builder)

## 1. 小剧场：一长串 true/false 的汉堡惨案

周五下午，小白果然被昨天那个汉堡套餐问题折磨得不轻。他写出了这样一个构造方法：

```java
public class Burger {
    public Burger(String bread, String meat, boolean cheese,
                  boolean bacon, String sauce, int pickles,
                  boolean lettuce, boolean tomato) {
        // ...
    }
}

// 下单的时候……
Burger burger = new Burger("全麦", "牛肉", true, false, "番茄酱", 2, true, false);
```

**王哥**凑过来一看，眉头一皱：“小白，我问你，这行代码里第三个 `true` 是加芝士还是加培根？第二个 `false` 又是啥？”

**小白**（数着手指）：“呃……第三个是芝士，第四个 `false` 是不要培根……等等，我自己都快数晕了。”

**王哥**：“这就是经典的**'伸缩构造器'反模式（Telescoping Constructor）**。参数一多，鬼才记得住顺序。更要命的是，如果用户只想要个最朴素的汉堡，难道还得传一堆 `false` 和 `null` 进去？”

**小白**：“那我搞十几个重载构造方法？`Burger(bread)`、`Burger(bread, meat)`、`Burger(bread, meat, cheese)`……”

**王哥**（扶额）：“打住打住，那你能写出几十个重载，组合爆炸。你想想，你去汉堡店点餐，是一口气把所有要求喊出来，还是**一项一项地跟店员说**：'要全麦面包，加牛肉，加块芝士，多放点酸黄瓜，好了，给我打包'？”

**小白**：“当然是一项一项说啊，这样清楚。”

**王哥**：“这就是今天的主角——**建造者模式（Builder）**。它把一个复杂对象的构建过程，拆成一步一步的'点单'，最后再'啪'地一下组装出成品。”

---

## 2. 核心概念：像点菜一样造对象

**王哥**：“建造者模式的核心，是给对象配一个专门的'建造者'。你通过建造者**链式地**一步步设置属性，每一步都清清楚楚，最后调一个 `build()` 收工。”

### 1) 改造前 vs 改造后

```java
public class Burger {
    private final String bread;   // 必选
    private final String meat;    // 必选
    private final boolean cheese; // 可选
    private final boolean bacon;  // 可选
    private final int pickles;    // 可选

    // 构造器设为私有，强制大家走 Builder
    private Burger(Builder builder) {
        this.bread = builder.bread;
        this.meat = builder.meat;
        this.cheese = builder.cheese;
        this.bacon = builder.bacon;
        this.pickles = builder.pickles;
    }

    // 静态内部类：建造者
    public static class Builder {
        private String bread;
        private String meat;
        private boolean cheese = false; // 可选项给默认值
        private boolean bacon = false;
        private int pickles = 0;

        public Builder(String bread, String meat) { // 必选项放构造器
            this.bread = bread;
            this.meat = meat;
        }

        // 每个可选项一个方法，返回 this 以便链式调用
        public Builder addCheese() { this.cheese = true; return this; }
        public Builder addBacon() { this.bacon = true; return this; }
        public Builder pickles(int n) { this.pickles = n; return this; }

        // 最后一步：组装成品
        public Burger build() { return new Burger(this); }
    }
}
```

**王哥**：“现在再看看下单的代码，是不是像说话一样自然？”

```java
// ✅ 链式建造，每一步都清清楚楚，自带说明书
Burger burger = new Burger.Builder("全麦", "牛肉")
        .addCheese()       // 加芝士
        .pickles(2)        // 来两片酸黄瓜
        .build();          // 收工！
```

**小白**（眼睛一亮）：“太直观了！我想加什么就 `.addXxx()`，不想加的根本不用写，再也没有那一长串看不懂的 `true, false, null` 了！”

```mermaid
classDiagram
    class Burger {
        -bread: String
        -meat: String
        -cheese: boolean
        -Burger(Builder)
    }
    class Builder {
        -bread: String
        -meat: String
        -cheese: boolean
        +addCheese() Builder
        +addBacon() Builder
        +pickles(int) Builder
        +build() Burger
    }
    Builder ..> Burger : build() 组装
    Burger +.. Builder : 静态内部类
```

### 2) 建造者的两大隐藏福利

**王哥**：“除了'好读'，建造者还有两个杀手锏：”

**福利一：能造不可变对象**。“你看 `Burger` 的字段全是 `final` 的，构造完就锁死，谁也改不了。这种'不可变对象'在多线程环境下天生安全。”

**福利二：能在 `build()` 里做校验**。“比如规定'汉堡必须有面包'，你可以在 `build()` 里检查，不合规就直接拒绝，保证造出来的对象**永远是合法的**。”

```java
public Burger build() {
    if (bread == null) {
        throw new IllegalStateException("汉堡不能没有面包！");
    }
    return new Burger(this);
}
```

**小白**：“好家伙，这就把'参数校验'集中到一个地方了，对象一旦造出来就保证是对的。”

---

## 3. 模式精讲：工厂 vs 建造者

**小白**：“王哥，我有点懵。工厂模式不也是造对象吗？它俩到底啥区别？”

**王哥**：“好问题。一句话区分——

- **工厂模式**关心的是'**造哪个**'。你要锅还是要铲子，给你一个**整体的成品**，过程你不关心。
- **建造者模式**关心的是'**怎么造**'。产品就一个（汉堡），但它**构造过程很复杂、有很多可选部件**，需要你一步步拼装。

打个比方：你去 4S 店**直接提一辆现成的车**，那是工厂；你**选配置**——天窗要不要、真皮座椅要不要、什么颜色——最后给你定制一辆，那是建造者。”

| 维度 | 工厂模式 | 建造者模式 |
| --- | --- | --- |
| 关注点 | 创建哪种产品 | 如何分步构建一个产品 |
| 产品复杂度 | 通常较简单 | 复杂，部件多、可选项多 |
| 典型标志 | `createXxx()` | 链式 `.setA().setB().build()` |

**王哥**：“实战里见过的 `StringBuilder`、`Lombok` 的 `@Builder`、OkHttp 的 `Request.Builder`、各种 `XxxConfig.Builder`，全是这个套路。”

---

## 4. 课后总结与吐槽

小白把汉堡、订单、HTTP 请求这些'参数巨多'的对象全用建造者重构了一遍，同事直呼代码可读性起飞。

**小白的笔记**：
1. **建造者模式**：把复杂对象的构建拆成链式的分步操作，最后 `build()` 收工。
2. 专治'**伸缩构造器**'——参数又多又杂、还有一堆可选项。
3. 福利：能造**不可变对象**、能在 `build()` 里**集中校验**。
4. 与工厂的区别：工厂管'造哪个'，建造者管'怎么一步步造'。

**王哥**：“建造者是'从零开始一步步搭'。但小白，我再问你一个场景——”

> [!TIP]
> **王哥的思考题**
> “假设你已经费了九牛二虎之力，造好了一个配置极其复杂的'游戏角色'对象——满级、顶级装备、一身好属性，光初始化就花了 3 秒。现在我要 100 个一模一样的角色当 NPC。难道要把那 3 秒的建造过程重复跑 100 遍吗？有没有办法，拿着这个现成的角色当'母版'，直接'咔咔咔'复制出 100 份？”

（小白想起了生物课上的克隆羊多莉……）

---
*下一章，原型模式将告诉小白：与其重新造，不如直接复制。*
