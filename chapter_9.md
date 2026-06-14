# 第9章：替身演员——代理模式 (Proxy)

## 1. 小剧场：核心方法被"杂活"塞满了

周四，小白在维护一个数据库查询服务。一开始，它干干净净，只负责查询这一件事：

```java
// 接口：定义"能查询"这个能力
public interface DataQuery {
    String query(String sql);
}

// 真正干活的查询服务，此刻还很纯净
public class RealDataQuery implements DataQuery {
    public String query(String sql) {
        System.out.println("执行查询：" + sql);
        return "查询结果";
    }
}
```

可是安全部门下了死命令：**所有查询操作，之前都要校验权限，前后都要记日志**。小白图省事，直接把这些杂活全塞进了 `query` 方法里：

```java
// 小白的"被污染"版本：核心方法里混进了一堆门卫工作
public class RealDataQuery implements DataQuery {
    public String query(String sql) {
        // ① 权限校验（本不属于"查询"该管的事）
        if (!checkPermission()) throw new RuntimeException("无权限");
        // ② 记一条日志
        System.out.println("[日志] 有人在查询：" + sql);
        // ③ 这才是真正的业务
        System.out.println("执行查询：" + sql);
        // ④ 再记一条日志
        System.out.println("[日志] 查询完成");
        return "查询结果";
    }
}
```

**王哥**：“小白，你看这个 `query` 方法——本来它只该干'查询'一件事，现在却被'权限校验''记日志'这些**杂活**塞得乌烟瘴气。这就违背了第1章讲的**单一职责原则**。而且，下次再来个'限流''统计耗时'，你是不是还得往里塞？这方法迟早变成一坨没人敢碰的代码。”

**小白**：“可这些校验和日志确实得做啊，总不能不做吧？”

**王哥**：“当然要做，但**不该让它自己做**。还记得上一章我留的思考题吗？明星和经纪人。”

**小白**：“记得。粉丝想见明星，得先过经纪人那关。”

**王哥**：“对。`RealDataQuery` 就是那个**明星**，它只该专心'演戏'（查询）。'权限校验''记日志'这些**门卫工作，应该交给一个经纪人**。我们造一个'替身'——它对外长得和真正的查询服务**一模一样**（都实现 `DataQuery` 接口），但你调用它时，它会先帮你把权限、日志这些杂事办了，再把真正的请求**转交**给幕后的明星。这个替身，就叫**代理（Proxy）**。”

---

## 2. 核心概念：找个"替身"来把关

**王哥**：“代理模式只有三个角色，记住这张图你就懂一大半了：

- **接口（DataQuery）**：明星和替身都遵守的'戏约'，保证两者对外长得一样。
- **真实对象（RealDataQuery）**：真正干活的'明星'。
- **代理对象（Proxy）**：'经纪人'，它内部**握着**明星，调用前后帮忙把关，中间把活转交给明星。

关键就一句话：**代理和真实对象实现同一个接口，代理内部持有真实对象**。这样一来，调用方拿到代理，用起来和拿到真货**没有任何区别**，但实际上每次调用都被经纪人'过了一道手'。”

### 1) 静态代理：手写一个替身

**王哥**：“咱们先用最朴素的方式手写一个代理类，你就明白它怎么'把关'了。”

```java
// 代理类：对外是个 DataQuery，内部握着真正的 RealDataQuery
public class DataQueryProxy implements DataQuery {
    private RealDataQuery real;  // ← 经纪人手里握着的"明星"
    private String user;

    public DataQueryProxy(String user) {
        this.user = user;
        this.real = new RealDataQuery();  // 创建（或拿到）真正的明星
    }

    @Override
    public String query(String sql) {
        // —— 转交之前：经纪人先把关 ——
        if (!checkPermission(user)) {
            throw new RuntimeException(user + " 没有查询权限！");
        }
        System.out.println("[日志] " + user + " 开始查询：" + sql);

        // —— 把真正的活，转交给明星 ——
        String result = real.query(sql);

        // —— 转交之后：经纪人收尾 ——
        System.out.println("[日志] 查询完成");
        return result;
    }

    private boolean checkPermission(String user) {
        return "admin".equals(user);  // 简单起见：只有 admin 有权限
    }
}
```

**王哥**：“你品品这个 `query` 方法的结构——它就是个'三明治'：**把关 → 转交给明星 → 收尾**。中间那句 `real.query(sql)` 才是真正的业务，前后都是经纪人加的料。”

现在，`RealDataQuery` 可以回归纯净，**只保留最开始那个干净版本**，门卫工作全被代理接管了。调用方这样用：

```java
// 调用方拿到的是"经纪人"，但用起来和真货一模一样
DataQuery query = new DataQueryProxy("admin");
query.query("SELECT * FROM users");
// 自动完成：权限校验 → 记日志 → 真正查询 → 再记日志
```

```mermaid
classDiagram
    class DataQuery {
        <<interface>>
        +query(String) String
    }
    class RealDataQuery {
        +query(String) String
    }
    class DataQueryProxy {
        -real: RealDataQuery
        -user: String
        +query(String) String
    }
    DataQuery <|.. RealDataQuery
    DataQuery <|.. DataQueryProxy
    DataQueryProxy --> RealDataQuery : 把关后转交
```

**小白**：“清爽了！明星专心演戏，经纪人专心把关，各司其职。调用方还以为自己直接在用查询服务，其实中间隔了个经纪人。”

---

## 3. 进阶：从"手写替身"到"万能替身"

### 1) 静态代理的烦恼

**王哥**：“静态代理好懂，但有个要命的问题——**每个明星，你都得手写一个专属经纪人**。”

**小白**：“啥意思？”

**王哥**：“你现在给 `DataQuery` 写了个 `DataQueryProxy`。可你们系统里还有 `UserService`、`OrderService`、`PayService`……上百个服务。如果每个都要加'日志'，你是不是得**手写上百个代理类**？而且每个代理类里的日志代码几乎一模一样，复制粘贴一百遍。”

**小白**（皱眉）：“那确实蠢。有没有办法写**一个**经纪人，让它能代理**所有**明星？”

**王哥**：“问得好。这正是 Java 的'**动态代理**'要解决的。但它要用到一个新东西——**反射**。我先花点功夫给你讲清楚它的思路，你别急。”

### 2) 难点在哪：经纪人怎么知道要转交哪个方法？

**王哥**：“想象一个'**万能经纪人**'，它要能代理任何明星。难点是——**它在写代码的时候，根本不知道未来要代理的明星有哪些方法**。`DataQuery` 有 `query`，`OrderService` 有 `createOrder`、`cancel`……方法名千奇百怪。

静态代理里，因为你知道是 `DataQuery`，所以能直接写 `real.query(sql)`。但万能经纪人不知道，它没法提前写死任何一个方法名。

Java 的解法很巧妙：**不管你调用代理上的哪个方法，Java 都把这次调用'打包'成统一的形式，塞进一个固定的入口方法里，让你统一处理**。这个统一入口，就是 `InvocationHandler` 接口里的 `invoke` 方法。”

### 3) 看懂 invoke：所有调用的"总闸门"

```java
// 万能经纪人的"处理逻辑"：所有方法调用都会流进这个 invoke
public class LogProxyHandler implements InvocationHandler {
    private Object target;  // 真正的明星（注意类型是 Object，任意对象都行）

    public LogProxyHandler(Object target) {
        this.target = target;
    }

    // 关键方法：你在代理上调用任何方法，最终都会跑进这里
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        // —— 把关：加日志 ——
        System.out.println("[日志] 调用方法：" + method.getName());

        // —— 转交给明星：用反射"动态地"调用那个方法 ——
        Object result = method.invoke(target, args);

        // —— 收尾 ——
        System.out.println("[日志] 方法执行完毕");
        return result;
    }
}
```

**王哥**：“别被吓到，我把 `invoke` 的三个参数挨个翻译给你：

- `Object proxy`：生成出来的那个代理对象本身（一般用不到，先无视）。
- `Method method`：**调用方刚才调的是哪个方法**。比如调了 `query`，这里 `method` 就代表 `query` 这个方法。`method.getName()` 就能拿到字符串 `"query"`。
- `Object[] args`：**调用时传的参数**。比如 `query("SELECT ...")`，那 `args` 就是 `["SELECT ..."]`。

所以这个 `invoke` 的意思是：'**不管你调了什么方法、传了什么参数，都先到我这儿报到。我加完日志，再用 `method.invoke(target, args)` 替你去真正调用明星身上那个方法。**'

那句 `method.invoke(target, args)` 是整段的灵魂——它是**反射**：'在 `target`（明星）这个对象上，把 `method` 这个方法，用 `args` 这组参数，调一遍'。因为用的是 `method` 变量而不是写死的方法名，所以**它对 `query`、`createOrder`、任何方法都通用**。这就是万能的来源。”

### 4) 让 Java 帮你"凭空造出"代理对象

**王哥**：“处理逻辑有了，最后一步——谁来生成那个'对外长得像明星'的代理对象？答案是 Java 自带的 `Proxy.newProxyInstance`，它能在**程序运行时**帮你凭空造一个：”

```java
RealDataQuery realObject = new RealDataQuery();  // 真正的明星

// 让 Java 运行时动态生成一个代理对象
DataQuery proxy = (DataQuery) Proxy.newProxyInstance(
        realObject.getClass().getClassLoader(),     // 参数1：用哪个类加载器（照抄即可）
        new Class[]{ DataQuery.class },              // 参数2：这个代理要"伪装"成哪些接口
        new LogProxyHandler(realObject));            // 参数3：调用时用哪套处理逻辑（上面那个）

// 像用真货一样用它
proxy.query("SELECT * FROM users");
```

**王哥**：“翻译一下这三个参数：

- 参数2 告诉 Java：'我要的代理，得能**冒充 `DataQuery` 接口**'，于是生成的对象就能强转成 `DataQuery`。
- 参数3 告诉 Java：'**当有人调用这个代理时，请把调用都转给我这个 `LogProxyHandler` 去处理**'。

于是当你执行 `proxy.query(...)` 时，幕后发生的事是：Java 不会直接调用，而是**自动跳进 `LogProxyHandler.invoke`**，把 `query` 方法和参数打包传进去——也就回到了上一步我们写的那段逻辑。”

**小白**（恍然大悟）：“我串起来了！`Proxy.newProxyInstance` 负责**造一个假明星**，`InvocationHandler.invoke` 负责**定义这个假明星被调用时该干嘛**。而且因为 `invoke` 里用的是反射、不写死方法名，**同一个 `LogProxyHandler` 能套给任何接口**——一个经纪人代理所有明星，不用再手写上百个代理类了！”

**王哥**：“正是。而且——这就是 **Spring AOP（面向切面编程）的底层原理**。你平时打的那些 `@Transactional`（事务）、`@Cacheable`（缓存）注解，背后全是 Spring 在运行时给你的 Bean **悄悄包了一层动态代理**，在方法前后帮你开事务、查缓存。你以为在直接调自己的 Service，其实调的是个代理。”

---

## 4. 模式精讲：代理 vs 装饰器，以及代理的种类

**小白**：“王哥，代理和上一章的装饰器，结构几乎一样啊——都是实现同一接口、内部包一个对象。到底咋区分？”

**王哥**：“结构确实是双胞胎，区别在**目的**和**谁说了算**：

| 模式 | 被包的对象怎么来 | 核心目的 |
| --- | --- | --- |
| **装饰器** | **外部传进来**，你想包几层包几层 | **增强**功能，强调自由叠加 |
| **代理** | 通常代理**自己内部创建/掌控** | **控制访问**，强调'把关' |

一句话：装饰器是'**我主动给你叠 buff**'，由调用方决定叠几层；代理是'**想用它？先过我这关**'，由代理决定你能不能用、用之前先干点啥。”

**王哥**：“代理按'把关的目的'不同，还分好几种，你了解个名字就行：

- **保护代理**：做权限控制（咱们刚写的就是）。
- **虚拟代理**：真实对象创建成本高（比如加载大图），先用代理顶着，等真要用了再创建——也就是'懒加载'。
- **远程代理**：真实对象在另一台服务器上，代理负责帮你做网络通信——这就是 RPC 的本质。
- **缓存代理**：把结果缓存起来，下次同样的请求直接返回，不再麻烦明星。”

---

## 5. 课后总结与吐槽

小白用动态代理给所有查询服务统一加上了权限校验和日志，核心业务代码恢复了清白，安全部门也满意了。

**小白的笔记**：
1. **代理模式**：给真实对象配一个"替身"来**控制访问**，代理和真实对象实现同一接口，调用方用起来无感。
2. **静态代理**：手写代理类，逻辑清晰，但**一个真实类配一个代理类**，数量一多就繁琐。
3. **动态代理**：运行时自动生成代理，**一套 `InvocationHandler` 能代理所有接口**，是 **Spring AOP** 的底层原理。
   - `Proxy.newProxyInstance` 负责"造代理"，`invoke` 负责"定义代理被调用时干嘛"，`method.invoke(target, args)` 用反射把活转交给真实对象。
4. 与装饰器的区别：装饰器重在**增强**（外部主动叠加），代理重在**控制访问**（自己内部把关）。

**王哥**：“适配器、装饰器、代理，这三个'包一层'的兄弟你都见过了。但现在有个新麻烦——不是'包一层'，而是'背后藏着一大堆东西，我只想要一个简单的开关'——”

> [!TIP]
> **王哥的思考题**
> “你想在家看一场电影，得依次干这些事：打开投影仪、放下幕布、打开音响、调暗灯光、启动播放器、选好片源……整整七八个步骤，每次都要手忙脚乱操作一堆设备。有没有办法搞一个'一键观影'的按钮，我一按，它在背后自动帮我把这一长串复杂操作全做了，我根本不需要知道里面那么多设备是怎么联动的？”

（小白看了看自己桌上那堆遥控器，深有同感……）

---
*下一章，外观模式将给小白一个"一键搞定"的总开关。*
