## C#编码规范（Critical级别）

非常重要且必须遵循的注意事项

### 1.公共资源禁止用于锁定
共享资源不应用于锁定，因为它会增加死锁的机会。任何其他线程都可以出于另一个不相关的目的而获取（或尝试获取）相同的锁。
相反，应该为对象创建专用的锁实例，以避免死锁或锁争用。

错误的例子
```
public void MyLockingMethod()
{
    lock (this)
    {
        // ...
    }
}
```
正确的例子
```
private readonly object lockObj = new object();

public void MyLockingMethod()
{
    lock (lockObj)
    {
        // ...
    }
}
```
### 2.异步方法中，无需原始线程资源，应该尽量使用ConfigureAwait(false)
ConfigureAwait用于配置当前异步任务await之后是否需要切换到原始同步上下文线程中执行

例如在一个WebRequest中，当调用了异步方法且该方法在await之后读取HttpContext.Current信息，则需要设置ConfigureAwait(true)，否则将会读取到空的HttpContext，而在无需原始线程上下文的情况应该使用ConfigureAwait(false) 以避免可能存在的上下文资源切换和死锁问题

错误的例子
```
var response = await httpClient.GetAsync(url); 
```
正确的例子
```
var response = await httpClient.GetAsync(url).ConfigureAwait(false);
```

### 3.foreach的显式转换
不要在foreach中使用显式转换，这有可能会引发InvalidCastException异常

错误的例子
```
public class Fruit { }
public class Orange : Fruit { }
public class Apple : Fruit { }

class MyTest
{
  public void Test()
  {
    var fruitBasket = new List<Fruit>();
    fruitBasket.Add(new Orange());
    fruitBasket.Add(new Orange());
    // fruitBasket.Add(new Apple());  // InvalidCastException

    foreach (Orange orange in fruitBasket) // InvalidCastException
    {
      ...
    }
  }
}
```
正确的例子
```
var fruitBasket = new List<Orange>();
fruitBasket.Add(new Orange());
fruitBasket.Add(new Orange());

foreach (Orange orange in fruitBasket)
{
  ...
}
```

### 4.避免调用GC.Collect
当程序执行GC.Collect的时候是基于阻塞的操作，继而检查和清理内存中的每个对象，主动调用也无法掌控何时运行及完成，一般此操作损失大于收益，我们应该将精力放在防止内存泄漏上面

错误的例子
```
static void Main(string[] args)
{
  // ...
  GC.Collect(2, GCCollectionMode.Optimized); 
}
```
### 5.使用nameof
在重构的过程中，可能一些命名是会被改变的，要使用nameof读取对象的名称字符，从而降低耦合

错误的例子
```
void DoSomething(int someParameter, string anotherParam)
{
    if (someParameter < 0)
    {
        throw new ArgumentException("Bad argument", "someParameter");  
    }
    if (anotherParam == null)
    {
        throw new Exception("anotherParam should not be null"); 
    }
}
```
正确的例子
```
void DoSomething(int someParameter)
{
    if (someParameter < 0)
    {
        throw new ArgumentException("Bad argument", nameof(someParameter));
    }
    if (anotherParam == null)
    {
        throw new Exception($"{nameof(anotherParam)} should not be null");
    }
}
```

### 6.ValueTask的正确使用
当对ValueTask/ValueTask<TResult>实例执行以下操作时，会引发异常
1.多次等待实例。
2.多次调用AsTask。
3.多次使用.Result或.GetAwaiter().GetResult
4.在操作尚未完成时使用.Result或.GetAwaiter().GetResult()

### 7.不要隐藏基类方法
错误的例子
```
using System;

namespace MyLibrary
{
  class Foo
  {
    internal void SomeMethod(string s1, string s2) { }
  }

  class Bar : Foo
  {
    internal void SomeMethod(string s1, object o2) { } 
  }
}
```
正确的例子
```
using System;

namespace MyLibrary
{
  class Foo
  {
    internal void SomeMethod(string s1, string s2) { }
  }

  class Bar : Foo
  {
    internal void SomeOtherMethod(string s1, object o2) { }
  }
}
```
### 8.委托方法“BeginInvoke”的调用应该与“EndInvoke”配对
BeginInvoke在执行的时候生成的一些资源只有在调用EndInvoke才会被释放，所以异步调用BeginInvoke和EndInvoke要成对使用

错误的例子
```
public delegate string AsyncMethodCaller();

public static void Main()
{
    AsyncExample asyncExample = new AsyncExample();
    AsyncMethodCaller caller = new AsyncMethodCaller(asyncExample.MyMethod);

    IAsyncResult result = caller.BeginInvoke(null, null); 
}
```
正确的例子
```
public delegate string AsyncMethodCaller();

public static void Main()
{
    AsyncExample asyncExample = new AsyncExample();
    AsyncMethodCaller caller = new AsyncMethodCaller(asyncExample.MyMethod);

    IAsyncResult result = caller.BeginInvoke(null, null);

    string returnValue = caller.EndInvoke(out threadId, result);
}
Begi
```
### 9.不要通过反射绕开private访问性
通过反射可以对对象各成员进行操作，然而在这可能带来一些严重的问题
1.私有成员并不是公开API
2.造成内部代码的不稳定
3.被不受信任的代码调用

错误的例子
```
using System.Reflection;

Type dynClass = Type.GetType("MyInternalClass");
BindingFlags bindingAttr = BindingFlags.NonPublic | BindingFlags.Static;
MethodInfo dynMethod = dynClass.GetMethod("mymethod", bindingAttr);
object result = dynMethod.Invoke(dynClass, null);
```

### 10.加密应该更加安全
强密码算法是可抵抗密码分析的密码系统，它们不易受到诸如蛮力攻击之类的知名攻击,建议仅使用由密码社区广泛测试和推广的密码算法。

错误的例子
```
var tripleDES1 = new TripleDESCryptoServiceProvider(); //不合规：三重DES容易受到中间相遇攻击

var simpleDES = new DESCryptoServiceProvider（）; //不符合规定：DES与56位密钥配合使用，可以通过详尽搜索进行攻击

var RC2 = new RC2CryptoServiceProvider（）; //不合规：RC2容易受到相关密钥攻击
```
正确的例子
```
var AES = new AesCryptoServiceProvider();
```

### 11.构造函数不要调用可覆盖方法
类构造的执行顺序是从基类开始调用构造函数，有时候在构造函数中调用可被子类覆盖的方法会导致一些空引用的异常
错误的例子
```
public class Parent
{
  public Parent()
  {
    DoSomething();  
  }

  public virtual void DoSomething()
  {
    ...
  }
}

public class Child : Parent
{
  private string foo;

  public Child(string foo) 
  {
    this.foo = foo;
  }

  public override void DoSomething()
  {
    Console.WriteLine(this.foo.Length);//空引用异常
  }
}
```

### 12.自定义Exception类应该被设置为 public
自定义的Exception是为了我们能够提供更多更精准的自定义信息，然而必须是public才能正常使用，如果抛出非public类的Exception，将会导致最终抛出的是该异常类的public基类，导致自定义的数据丢失

错误的例子
```
internal class MyException : Exception 
{
  // ...
}
```
正确的例子
```
public class MyException : Exception
{
  // ...
}
```

### 13.不要在finaly语句块中抛出异常
在finaly语句块中抛出异常将会导致try catch中的异常抛出被覆盖，从而丢失异常信息

错误的例子
```
try
{
  throw new ArgumentException();
}
finally
{
  /* clean up */
  throw new InvalidOperationException(); //try 中的异常被覆盖
}
```
正确的例子
```
try
{
  //执行时抛出了异常
  throw new ArgumentException();
}
finally
{
  /* clean up */
}
```

### 14.Event字段不要设置为 Virtual
在C#中，对 Event 的支持是由编译器生成 private delegate 和隐式 add remove 等一套driver包装实现的，如果Virtual Event 被多次覆盖，将会导致编译器生成多套新的delegate driver

错误的例子
```
abstract class Car
{
  public virtual event EventHandler OnRefueled; // Noncompliant

  public void Refuel()
  {
    // This OnRefueld will always be null
     if (OnRefueled != null)
     {
       OnRefueled(this, null);
     }
  }
}

class R2 : Car
{
  public override event EventHandler OnRefueled;
}

class Program
{
  static void Main(string[] args)
  {
    var r2 = new R2();
    r2.OnRefueled += new EventHandler((o, a) =>
    {
      Console.WriteLine("This event will never be called");
    });
    r2.Refuel();
  }
}
```
正确的例子
```
abstract class Car
{
  public event EventHandler OnRefueled; // Compliant

  public void Refuel()
  {
    if (OnRefueled != null)
    {
      OnRefueled(this, null);
    }
  }
}

class R2 : Car {}

class Program
{
  static void Main(string[] args)
  {
    var r2 = new R2();
    r2.OnRefueled += new EventHandler((o, a) =>
    {
      Console.WriteLine("This event will be called");
    });
    r2.Refuel();
  }
}
```

### 15.枚举值 0 应该被命名为 None
枚举值的 0 位不应该被使用，而应该被设置为 None，在枚举默认值也是 0 的时候，将会导致我们无法分辨 0 到底是其使用值还是未经赋值的默认值

错误的例子
```
[Flags]
enum FruitType
{
    Void = 0,        
    Banana = 1,
    Orange = 2,
    Strawberry = 4
}
```
正确的例子
```
[Flags]
enum FruitType
{
    None = 0,       
    Banana = 1,
    Orange = 2,
    Strawberry = 4
}
```
### 16.防止SQL脚本注入攻击
对于一般的SQL脚本字符格式化查询，可能会引发注入等安全性漏洞，推荐的安全做法：

1. 避免使用格式化技术手动构建查询，如果仍然要执行此操作，请不要在此构建过程中包括用户输入。
2. 尽量执行参数化查询、准备好的语句或存储过程
3. 正确使用ORM框架（如Hibernate，EntityFramework）可以降低注入风险
4. 避免在存储过程或函数中执行不安全的输入性SQL
5. 检查和清理所有不安全的输入入口
6. 使用权限较低，敏感性更低的数据库账号来减少受到攻击的影响

错误的例子
```
public void Foo(DbContext context, string query, string param)
{
    string sensitiveQuery = string.Concat(query, param);
    context.Database.ExecuteSqlCommand(sensitiveQuery); // 可注入
    context.Query<User>().FromSql(sensitiveQuery); // 可注入

    context.Database.ExecuteSqlCommand($"SELECT * FROM mytable WHERE mycol={value}", param); // 可注入，字符串先求值再执行
    string query = $"SELECT * FROM mytable WHERE mycol={param}";
    context.Database.ExecuteSqlCommand(query); // 可注入，字符串已经被求值出来再执行
}

public void Bar(SqlConnection connection, string param)
{
    SqlCommand command;
    string sensitiveQuery = string.Format("INSERT INTO Users (name) VALUES (\"{0}\")", param);
    command = new SqlCommand(sensitiveQuery); // 可注入

    command.CommandText = sensitiveQuery; // 可注入

    SqlDataAdapter adapter;
    adapter = new SqlDataAdapter(sensitiveQuery, connection); // 可注入
}
```
正确的例子
```
public void Foo(DbContext context, string value)
{
    context.Database.ExecuteSqlCommand("SELECT * FROM mytable"); // 硬编码SQL无问题

    context.Database.ExecuteSqlCommand($"SELECT * FROM mytable WHERE mycol={value}"); // 使用value参数化占位符执行输入

    //EF参数化查询
    using (DbContext dbContext = new DbContext())    
    {    
      var sqlText = "SELECT * FROM mytable WHERE mycol=@Value";    
      var args = new DbParameter[] {    
          new MySqlParameter {ParameterName = "Value", Value = 1}    
      };    
      var data = dbContext.Database.SqlQuery<MyTableEntity>(sqlText, args).ToList();  
    } 
}
```
### 17.不要忽略捕捉到的异常
错误的例子
```
string text = "";
try
{
    text = File.ReadAllText(fileName);
}
catch (Exception exc)
{
}
```
正确的例子
```
string text = "";
try
{
    text = File.ReadAllText(fileName);
}
catch (Exception exc)
{
    logger.Log(exc);
}
```
### 18.使用as类型转换会更好
当我们在进行类型转换而出现错误时会抛出InvalidCastExceptions异常，而使用as运算符只会返回正确的转换值或者null

错误的例子
```
public interface IMyInterface
{ /* ... */ }

public class Implementer : IMyInterface
{ /* ... */ }

public class MyClass
{ /* ... */ }

public static class Program
{
  public static void Main()
  {
    var myclass = new MyClass();
    var x = (IMyInterface) myclass; // InvalidCastException
    var b = myclass is IMyInterface; 

    int? i = null;
    var ii = (int)i; // InvalidOperationException
  }
}
```
正确的例子
```
public interface IMyInterface
{ /* ... */ }

public class Implementer : IMyInterface
{ /* ... */ }

public class MyClass
{ /* ... */ }

public static class Program
{
  public static void Main()
  {
    var myclass = new MyClass();
    var x = myclass as IMyInterface;// x将会为null
    var b = false;

    int? i = null;
    if (i.HasValue)
    {
      var ii = (int)i;
    }
  }
}
```
### 19.禁止嵌套类的成员与外层类静态成员同名
错误的例子
```
class Outer
{
  public static int A;

  public class Inner
  {
    public int A; 
    public int MyProp
    {
      get { return A; }  
    }
  }
}
```
正确的例子
```
class Outer
{
  public static int A;

  public class Inner
  {
    public int InnerA;
    public int MyProp
    {
      get { return InnerA; }
    }
  }
}
```
### 20.非静态成员禁止修改内部静态成员
如果多实例对静态成员的修改，不仅数据会错乱，也会引发并发问题

错误的例子
```
public class MyClass
{
  private static int count = 0;

  public void DoSomething()
  {
    //...
    count++;  // 修改静态成员
  }
}
```

### 21.方法的重载不要改变参数的默认值

错误的例子
```
public class Base
{
  public virtual void Write(int i = 42)
  {
    Console.WriteLine(i);
  }
}

public class Derived : Base
{
  public override void Write(int i = 5) // 不合规
  {
    Console.WriteLine(i);
  }
}
```
### 22.方法属性等成员不应该过于复杂
方法属性成员过于复杂繁琐，请考虑做类对象的拆分，一般阈值控制在10个以内

### 23.非Flags枚举禁止用于位运算
非Flags枚举总是表示一个值，这与Flags的叠加值含义不同，当用非Flags进行运算会导致阅读人员的困惑

错误的例子
```
enum Permissions
{
  None = 0,
  Read = 1,
  Write = 2,
  Execute = 4
}
// ...

var x = Permissions.Read | Permissions.Write; 
```
正确的例子
```
[Flags]
enum Permissions
{
  None = 0,
  Read = 1,
  Write = 2,
  Execute = 4
}
// ...

var x = Permissions.Read | Permissions.Write;
```
### 24.公开常量不应该被修改
错误的例子
```
public class Foo
{
    public const double Version = 1.0; 
}
```
正确的例子
```
public class Foo
{
    public static double Version
    {
      get { return 1.0; }
    }
}
```
### 25.警惕命令行参数的安全性
命令行参数同样会导致用户输入性风险，请警惕

示例
```
namespace MyNamespace
{
    class Program
    {
        static void Main(string[] args) 
        {
            string myarg = args[0];//请进行参数类型的验证
            // ...
        }
    }
}
```
### 26.属性Set访问器 “value” 的使用
在一般的Set访问器中，应该使用“value”关键字赋值，如果属性不能赋值，应该在set访问器抛出异常

错误的例子
```
private int count;
public int Count
{
  get { return count; }
  set { count = 42; } 
}
```
正确的例子
```
private int count;
public int Count
{
  get { return count; }
  set { count = value; }
}
```
或者
```
public int Count
{
  get { return count; }
  set { throw new InvalidOperationException(); }
}
```
### 27."is" 不能和 "this" 同时使用
一般情况下，this和is不会同时使用，唯一可能会出现的情况是在调用父类方法时判断子类的类型去执行逻辑，对于此类代码，违背OOP的原则，我们应将子类的逻辑放入子类代码中，而不是放到父类执行

错误的例子
```
public class Food //基类
{
  public void DoSomething()
  {
    if (this is Pizza)
    {
      // to do something...
    } 
    else if (...
  }
}
```
### 28.子类私有成员不能与父类成员同名
错误的例子
```
public class Fruit
{
  protected Season ripe;
  protected Color flesh;
}

public class Raspberry : Fruit
{
  private bool ripe;
  private static Color FLESH;
}
```
正确的例子
```
public class Fruit
{
  protected Season ripe;
  protected Color flesh;
}

public class Raspberry : Fruit
{
  private bool ripened;
  private static Color FLESH_COLOR;
}
```
### 29.禁止重载带有默认值参数的方法
当另一个没有可选参数的重载方法出现时，除了使方法更难以理解外，还将会引发一些问题

错误的例子
```
public class MyClass
{
  void Print(string[] messages) {...}
  void Print(string[] messages, string delimiter = "\n") {...} 
}

// ...
MyClass myClass = new MyClass();

myClass.Print(new string[3] {"yes", "no", "maybe"});  // 哪个方法会被调用？
```
### 30.禁止使用多维数组作为参数
多维数组结构复杂描述相对困难，缺少作为参数的直观性，请不要使用多维数组作为参数类型

### 31.不要用关键字作为变量标识
错误的例子 
``` 
int await = 42; 
```
正确的例子
``` 
int someOtherName = 42;
```
### 32.异步方法禁止返回void
异步方法返回void可能会导致以下问题
1. 调用方无法知道该异步方法是否执行完成
2. 该异步方法出现的异常无法被原始线程同步上下文捕捉（异步方法在Task线程池的某子线程中抛出，并没有切换到原始同步上下文所在线程）

我们应该将异步方法用Task返回

错误的例子
```
class HttpPrinter
{
  private string content;

  public async void CallNetwork(string url) 
  {
    var client = new HttpClient();
    var response = await client.GetAsync(url);
    content = await response.Content.ReadAsStringAsync();
    Throw new Exception();
  }

  public async Task PrintContent(string url)  
  {
      try
      {
        CallNetwork(url);
        await Task.Delay(1000);//CallNetWork超过1秒仍未执行完成，会导致Content仍然是null
        Console.Write(content);
      }
      catch(Exception ex)
      {
          //CallNetWork 未切换原始同步上下文线程，无法捕捉此异常
      }
  }
}
```
正确的例子
```
class HttpPrinter
{
  private string content;

  public async Task CallNetwork(string url)
  {
    var client = new HttpClient();
    var response = await client.GetAsync(url);
    content = await response.Content.ReadAsStringAsync();
  }

  public async Task PrintContent(string url)
  {
    await CallNetwork(url); // 遵循规范，完成之后切换原始同步上下文执行
    await Task.Delay(1000);
    Console.Write(content); // Content 不会为null
  }
}
```
### 33.Using语句创建的IDispose对象禁止返回
错误的例子
```
public FileStream WriteToFile(string path, string text)
{
  using (var fs = File.Create(path)) // Noncompliant
  {
    var bytes = Encoding.UTF8.GetBytes(text);
    fs.Write(bytes, 0, bytes.Length);
    return fs;
  }
}
```
正确的例子
```
public FileStream WriteToFile(string path, string text)
{
  var fs = File.Create(path);
  var bytes = Encoding.UTF8.GetBytes(text);
  fs.Write(bytes, 0, bytes.Length);
  return fs;
}
```
### 34.禁止在构造函数中使用 this 指针
只有当构造函数完成后，这个对象才是真正有效的，即this才是正确的。而在构造中使用this时，这个对象并没有完全的初始化好。
### 35.ThreadStatic 使用及初始化
基于 ThreadStaticAttribute 线程静态化的字段，在线程内部共享，不同的线程获取各自独立的静态资源，所以标注 ThreadStatic 的字段应该由线程自己初始化，另外非Static 修饰的字段不允许设置为 ThreadStatic

错误的例子
```
public class Foo
{
  [ThreadStatic]
  public static object PerThreadObject = new object(); // 只有初始化线程执行了一次，其他线程获取是null
}
```
正确的例子
```
public class Foo
{
  [ThreadStatic]
  public static object _perThreadObject;
  public static object PerThreadObject
  {
    get
    {
      if (_perThreadObject == null)
      {
        _perThreadObject = new object();
      }
      return _perThreadObject;
    }
  }
}
```
