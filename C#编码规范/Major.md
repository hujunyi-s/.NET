## C#编码规范(Major级别)

必须遵循的编码习惯和注意事项

#### 1.禁止使用 “=+” 代替 "+="
这将会使我们在书写的时候容易产生各种误解

**错误的例子**
```
int target = -5;
int num = 3;

target =- num;  // target = -3，也许我们预期做减法，却执行的赋值
target =+ num; // target = 3 同上
```
**正确的例子**
```
int target = -5;
int num = 3;

target = -num; //target = -3
target += num; //target = -2
```
#### 2.抽象类的构造函数应该为 private 或 protected
由于abstract类无法实例化，因此它们拥有 public 或 internal 构造函数毫无意义。如果在创建扩展类实例时存在基本的初始化逻辑，则可以将其放入构造函数中，但应该将该构造函数设置为 private 或 protected。

**错误的例子**
```
abstract class Base
{
    public Base() 
    {
      //...
    }
}
```
**正确的例子**
```
abstract class Base
{
    protected Base()
    {
      //...
    }
}
```

#### 3.不要在执行for循环的过程中修改循环索引

**错误的例子**
```
class Foo
{
    static void Main()
    {
        for (int i = 1; i <= 5; i++)
        {
            Console.WriteLine(i);
            if (condition)
            {
               i = 20;
           }
        }
    }
}
```
**正确的例子**
```
class Foo
{
    static void Main()
    {
        for (int i = 1; i <= 5; i++)
        {
            Console.WriteLine(i);
        }
    }
}
```
#### 4.禁止使用 goto 语句
goto 是非结构化的流程控制语句，使用起来阅读困难，调试困难，理解困难，禁止使用


#### 5.NaN 不能用于比较
NaN本身甚至没有任何含义不具备比较的理解，判断NaN的最好方式是使用Number.IsNaN()方法

**错误的例子**
```
var a = double.NaN;

if (a == double.NaN) // always false
{
  Console.WriteLine("a is not a number");  
}
if (a != double.NaN)  //  always true
{
  Console.WriteLine("a is not NaN"); 
}
```
**正确的例子**
```
if (double.IsNaN(a))
{
  console.log("a is not a number");
}
```
#### 6.不要使用无参构造函数实例化Guid类
对于这种场景，更多采用静态Guid类

**错误的例子**
```
public void Foo()
{
    var g = new Guid(); 
}
```
**正确的例子**
```
public void Foo(byte[] bytes)
{
    var g1 = Guid.Empty;
    var g2 = Guid.NewGuid();
    var g3 = new Guid(bytes);
}
```
#### 7.Obsolete 标记应该具有说明
**错误的例子**
```
public class Car
{

  [Obsolete]  // Noncompliant
  public void CrankEngine(int turnsOfCrank)
  { ... }
}
```
**正确的例子**
```
public class Car
{

  [Obsolete("此处添加说明")]
  public void CrankEngine(int turnsOfCrank)
  { ... }
}
```

#### 8.平台非托管方法不应该被公开
当调用平台非托管服务时，将它们保持为私有或内部状态可确保对其访问进行控制和正确管理。

**错误的例子**
```
using System;
using System.Runtime.InteropServices;

namespace MyLibrary
{
    public class Foo
    {
        [DllImport("kernel32.dll", CharSet = CharSet.Unicode)]
        public static extern bool RemoveDirectory(string name);  
    }
}
```
**正确的例子**
```
using System;
using System.Runtime.InteropServices;

namespace MyLibrary
{
    public class Foo
    {
        [DllImport("kernel32.dll", CharSet = CharSet.Unicode)]
        private static extern bool RemoveDirectory(string name);
    }
}
```
#### 9.switch case 子句不能包含太多代码行
switch case 子句行数过多，会增加阅读维护的困难，默认行数阈值：3

**错误的例子**
```
switch (myVariable)
{
    case 0: // 
        methodCall1("");
        methodCall2("");
        methodCall3("");
        methodCall4("");
        break;
    case 1:
        ...
}
```
**正确的例子**
```
switch (myVariable)
{
    case 0:
        DoSomething()
        break;
    case 1:
        ...
}
...
private void DoSomething()
{
    methodCall1("");
    methodCall2("");
    methodCall3("");
    methodCall4("");
}
```
#### 10.switch 不应该含有太多 case
当case超过4项时，建议使用字典代替

**错误的例子**
```
public class TooManyCase
{
    public int switchCase(char ch)
    {
        switch(ch) {  
            case 'a':
                return 1;
            case 'b':
            case 'c':
                return 2;
            case 'd':
                return 3;
            case 'e':
                return 4;
            case 'f':
            case 'g':
            case 'h':
                return 5;
            default:
                return 6;
        }
    }
}
```
**正确的例子**
```
using System.Collections.Generic;

public class TooManyCase
{
    Dictionary<char, int> matching = new Dictionary<char, int>()
    {
        {'a', 1}, {'b', 2}, {'c', 2}, {'d', 3},
        {'e', 4}, {'f', 5}, {'g', 5}, {'h', 5}
    };

    public int withDictionary(char ch)
    {
        int value;
        if (this.matching.TryGetValue(ch, out value))
        {
            return value;
        } 
        else 
        {
            return 6;
        }
    }
}
```
#### 11.switch语句不要嵌套使用
嵌套 switch 结构很难理解，因为很容易将内部 switch 的情况误认为属于外部语句。因此，应该避免使用嵌套的switch语句，如果实在不能，则考虑将内部switch移到另一个函数。

#### 12.switch/if else 语句应该包含 default/else子句
**错误的例子**
```
int foo = 42;
switch (foo) // Noncompliant
{
  case 0:
    Console.WriteLine("foo = 0");
    break;
  case 42:
    Console.WriteLine("foo = 42");
    break;
}
```
**正确的例子**
```
int foo = 42;
switch (foo) 
{
  case 0:
    Console.WriteLine("foo = 0");
    break;
  case 42:
    Console.WriteLine("foo = 42");
    break;
  default:
    throw new InvalidOperationException("Unexpected value foo = " + foo);
}
```
#### 13.每个单独的if判断序列，应该有自己独立的行
**错误的例子**
```
if (condition1) {
  // ...
} if (condition2) {  
  //...
}
```
**正确的例子**
```
if (condition1) {
  // ...
} else if (condition2) {
  //...
}

或者

if (condition1) {
  // ...
}

if (condition2) {
  //...
}
```
#### 14.警惕 if switch for while do try 等语句块嵌套太深
 这样的代码很难阅读，理解和重构，应该控制在3层以内

**错误的例子**
```
if (condition1) // 第1层
{
  /* ... */
  if (condition2) // 第2层
  {
    /* ... */
    for(int i = 0; i < 10; i++) // 第3层
    {
      /* ... */
      if (condition4) // 第4层
      {
        if (condition5) // 第5层，我是谁，我在哪，我在干什么？
        {
          /* ... */
        }
        return;
      }
    }
  }
}
```

#### 15.不要省略花括号
尽管没有任何技术上的问题，但是省略花括号可能会增加阅读难度和引起误解

**错误的例子**
```
if (condition)
  ExecuteSomething();
  CheckSomething();
```
**正确的例子**
```
if (condition)
{
  ExecuteSomething();
  CheckSomething();
}
```
#### 16.条件表达式不能太复杂
过于复杂的&&、|| 逻辑判断或条件表达式将会提高阅读难度和维护难度，应当控制在3个以内，复杂条件判断考虑用一个方法函数整合

**错误的例子**
```
if (((condition1 && condition2) || (condition3 && condition4)) && condition5) { ... }
```
**正确的例子**
```
if ((MyFirstCondition() || MySecondCondition()) && MyLastCondition()) { ... }
```
#### 17.Url 应该用 System.Uri 而不是 String
**错误的例子**
```
using System;

namespace MyLibrary
{
   public class Foo
   {
      public void FetchResource(string uriString) { }
      public void FetchResource(Uri uri) { }

      public string ReadResource(string uriString, string name, bool isLocal) { }
      public string ReadResource(Uri uri, string name, bool isLocal) { }

      public void Main() 
      {
        FetchResource("http://www.mysite.com"); //未遵循
        ReadResource("http://www.mysite.com", "foo-resource", true); //未遵循
      }
   }
}
```
**正确的例子**
```
using System;

namespace MyLibrary
{
   public class Foo
   {
      public void FetchResource(string uriString) { }
      public void FetchResource(Uri uri) { }

      public string ReadResource(string uriString, string name, bool isLocal) { }
      public string ReadResource(Uri uri, string name, bool isLocal) { }

      public void Main() 
      {
        FetchResource(new Uri("http://www.mysite.com"));
        ReadResource(new Uri("http://www.mysite.com"), "foo-resource", true);
      }
   }
}
```
#### 18.重写 ToString 方法 返回值不能为 NULL
**错误的例子**
```
public override string ToString ()
{
  if (this.collection.Count == 0)
  {
    return null; 
  }
  else
  {
    // ...
  }
}
```
**正确的例子**
```
public override string ToString ()
{
  if (this.collection.Count == 0)
  {
    return string.Empty;
  }
  else
  {
    // ...
  }
}
```
#### 19.返回集合类型的方法，无数据时要返回空集合而不是 null
**错误的例子**
```
public IEnumerable<Result> GetResults()
{
    return null; 
}
```
**正确的例子**
```
public IEnumerable<Result> GetResults()
{
    return Enumerable.Empty<Result>();
}
```
#### 20.Catch Final 重复的语句块要进行合并
**错误的例子**
```
try
{
  DoTheFirstThing(a, b);
}
catch (InvalidOperationException ex)
{
  HandleException(ex);
}

DoSomeOtherStuff();

try  
{
  DoTheSecondThing();
}
catch (InvalidOperationException ex)
{
  HandleException(ex);
}

try  
{
  DoTheThirdThing(a);
}
catch (InvalidOperationException ex)
{
  LogAndDie(ex);
}
```
**正确的例子**
```
try
{
  DoTheFirstThing(a, b);
  DoSomeOtherStuff();
  DoTheSecondThing();
}
catch (InvalidOperationException ex)
{
  HandleException(ex);
}

try
{
  DoTheThirdThing(a);
}
catch (InvalidOperationException ex)
{
  LogAndDie(ex);
}
```
#### 21.事件的订阅不要使用匿名委托
使用匿名委托来订阅事件的后果会导致事后无法从委托列表中取消订阅

**错误的例子**
```
listView.PreviewTextInput + =（obj，args）=>
        listView_PreviewTextInput（obj，args，listView）;

// ...

listView.PreviewTextInput-=（obj，args）=>
        listView_PreviewTextInput（obj，args，listView）; 
```
**正确的例子**
```
EventHandler func =（obj，args）=> listView_PreviewTextInput（obj，args，listView）;

listView.PreviewTextInput + = func;

// ...

listView.PreviewTextInput-= func;
```
#### 22.引用类型参数的使用应该进行 null 判断 
**错误的例子**
```
public class MyClass
{
    private MyOtherClass other;

    public void Foo(MyOtherClass other)
    {
        this.other = other; 
    }

    public void Bar(MyOtherClass other)
    {
        this.other = other.Clone(); 
    }

    protected void FooBar(MyOtherClass other)
    {
        this.other = other.Clone();
    }
}
```
**正确的例子**
```
public class MyClass
{
    private MyOtherClass other;

    public void Foo(MyOtherClass other)
    {
        this.other = other;
    }

    public void Bar(MyOtherClass other)
    {
        if (other != null)
        {
            this.other = other.Clone();
        }
    }

    protected void FooBar(MyOtherClass other)
    {
        if (other != null)
        {
            this.other = other.Clone();
        }
    }
}
```
#### 23.可空值类型必须进行空判断
可空值类型往往会因为 Null 访问 Value抛出异常，在使用之前务必进行 Null 判断
**错误的例子**
```
int? nullable = null;
...
UseValue(nullable.Value); // Noncompliant
```
**正确的例子**
```
int? nullable = null;
...
if (nullable.HasValue)
{
  UseValue(nullable.Value);
}
```
#### 24.不要在子表达式或断言中求值
子表达式或断言中的求值往往含义很关键，却又难以被阅读发现，使可读性降低，**除 Lambda 条件表达式外**

**错误的例子**
```
if (string.IsNullOrEmpty(result = str.Substring(index, length))) 
{
  //...
}
```
**正确的例子**
```
var result = str.Substring（index，length）;
如果（string.IsNullOrEmpty（result））
{
  // ...
}
```

#### 25.数字类型不能溢出
数字是无限大的，但是容纳数字的类型是有限的，请防止数字溢出

**错误的例子**
```
public int getTheNumber(int val) {
  if (val <= 0) {
      return val;
  }
  int num = int.MaxValue;
  return num + val;  // 溢出iint类型
}
```

#### 26.类不能过多依赖其他的类（Single Responsibility Principle）
基于单一职责原则，我们的类往往被拆分得很细的粒度，但如果不注意控制上层类的依赖关系，往往会形成过度的依赖，对可读性及维护性带来困难，对于类的依赖关系数量以及类的继承树，阈值控制在 5 个以内，超出阈值请考虑拆分

**错误的例子**
```
public class Foo    
{
  private T1 a1;    
  private T2 a2;    
  private T3 a3;    

  public T4 Compute(T5 a, T6 b)    
  {
    T7 result = a.Process(b);    
    return result;
  }
}
```

#### 27.自定义 Attribute 应该设置好适用范围 AttributeUsageAttribute
**错误的例子**
```
using System;

namespace MyLibrary
{

   public sealed class MyAttribute :Attribute 
   {
      string text;

      public MyAttribute(string myText)
      {
         text = myText;
      }
      public string Text
      {
         get
         {
            return text;
         }
      }
   }
}
```
**正确的例子**
```
using System;

namespace MyLibrary
{

   [AttributeUsage(AttributeTargets.Class | AttributeTargets.Enum | AttributeTargets.Interface | AttributeTargets.Delegate)]
   public sealed class MyAttribute :Attribute
   {
      string text;

      public MyAttribute(string myText)
      {
         text = myText;
      }
      public string Text
      {
         get
         {
            return text;
         }
      }
   }
}
```
#### 28.重复的异常应该被正确抛出
**错误的例子**
```
try
{}
catch(ExceptionType1 exc)
{
  Console.WriteLine(exc);
  throw exc; // 新加了一层堆栈跟踪层级
}
catch (ExceptionType2 exc)
{
  throw new Exception("My custom message", exc);  //遵循
}
```
**正确的例子**
```
try
{}
catch(ExceptionType1 exc)
{
  Console.WriteLine(exc);
  throw;
}
catch (ExceptionType2 exc)
{
  throw new Exception("My custom message", exc);
}
```
#### 29.仅在构造函数中修改赋值的字段应该设置为只读
**错误的例子**
```
public class Person
{
    private int _birthYear;  // Noncompliant

    Person(int birthYear)
    {
        _birthYear = birthYear;
    }
}
```
**正确的例子**
```
public class Person
{
    private readonly int _birthYear;

    Person(int birthYear)
    {
        _birthYear = birthYear;
    }
}
```
#### 30.代码行数
代码行数阈值
1. 文件代码行数 ： 1000 行
2. 方法函数代码行数： 80 行


#### 31.没有用到的对象和空置的方法函数应该及时删除

#### 32.禁止使用幻数
每一个逻辑数字应该具有意义，请不要在程序中直接使用幻数

**错误的例子**
```
public static void DoSomething()
{
    for(int i = 0; i < 4; i++) 
    {
        ...
    }
}
```
**正确的例子**
```
private const int NUMBER_OF_CYCLES = 4;

public static void DoSomething()
{
    for(int i = 0; i < NUMBER_OF_CYCLES ; i++)  //Compliant
    {
        ...
    }
}
```
#### 33.方法不应该包含太多参数
我们建议方法的参数最好能够少于 4 个，禁止超过 7 个，参数过多请考虑用参数模型封装

#### 34.不要多次使用OrderBy
多次OrderBy完全没有任何意义，每次OrderBy将会对结果进行重新排序，结果仅代表最后一次OrderBy，所以多次排序请使用ThenBy

**错误的例子**
```
var x = personList
  .OrderBy(person => person.Age)
  .OrderBy(person => person.Name)  
  .ToList();  // 最终只按 Name 排序
```
**正确的例子**
```
var x = personList
  .OrderBy(person => person.Age)
  .ThenBy(person => person.Name)
  .ToList();
```
#### 35.不要嵌套三元运算符
嵌套的三元运算符可读性非常差，请不要嵌套使用

**错误的例子**
```
public string GetTitle(Person p)
{
  return p.Gender == Gender.MALE ? "Mr. " : p.IsMarried ? "Mrs. " : "Miss ";  
}
```
**正确的例子**
```
public string GetTitle(Person p)
{
  if (p.Gender == Gender.MALE)
  {
    return "Mr. ";
  }
  return p.IsMarried ? "Mrs. " : "Miss ";
}
```
#### 36.为了提高可读性，请不要在一行上放置多个语句
**错误的例子**
```
if(someCondition) DoSomething();
```
**正确的例子**
```
if(someCondition)
{
  DoSomething();
}
```
#### 37.default选项应该置于末尾
**错误的例子**
```
switch (param)
{
    case 0:
      DoSomething();
      break;
    default: // default应该置于末尾
      Error();
      break;
    case 1:
      DoSomethingElse();
      break;
}
```
**正确的例子**
```
switch (param)
{
    case 0:
      DoSomething();
      break;
    case 1:
      DoSomethingElse();
      break;
    default:
      Error();
      break;
}
```
