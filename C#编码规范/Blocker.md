## C#编码规范(Blocker级别)

明确禁止且可能会造成严重后果的注意事项

### 1.正确合理的使用"IDisposable" 

1. 对于非托管资源，例如Bitmap，FileStream，Socket，不再使用的时候，要立即对它们进行处理
2. 具有IDisposable成员的类和含有Dispose命名方法的类，应该实现"IDisposable"接口
3. 不要忘记基类的Dispose清理工作
4. 继承自IDisposable的类，必须有析构函数，并且在析构函数中进行资源释放

注意：类各自的资源应该放到各自的Dispose中进行清理，否则可能引起内存泄漏

错误的例子
```
public class ResourceHolder
{
  private FileStream fs; //非托管资源

  public void OpenResource(string path)
  {
    this.fs = new FileStream(path, FileMode.Open);
  }

  public void WriteToFile(string path, string text)
  {
    var fs = new FileStream(path, FileMode.Open); 
    var bytes = Encoding.UTF8.GetBytes(text);
    fs.Write(bytes, 0, bytes.Length);
  }
}
```
正确的例子
```
public class ResourceHolder : IDisposable
{
  private FileStream fs;

  public void OpenResource(string path)
  {
    this.fs = new FileStream(path, FileMode.Open);
  }

  public void Dispose()
  {
    this.fs.Dispose();
  }

  public void WriteToFile(string path, string text)
  {
    using (var fs = new FileStream(path, FileMode.Open))
    {
      var bytes = Encoding.UTF8.GetBytes(text);
      fs.Write(bytes, 0, bytes.Length);
    }
  }

  ~ResourceHolder()
  {
    Dispose();
  }
}
```
### 2.不要重载引用类型运算符 operator==
重载引用类型的operator==操作去实现两个对象的比较判断，可能会导致调用的混乱，同时也会模糊引用类型比较的本身含义，请用其他方式实现引用类型的自定义比较

错误的例子
```
public static bool operator== (MyType x, MyType y)
{
    //
}
```
### 3.不要使用"SafeHandle.DangerousGetHandle"
使用 DangerousGetHandle 方法可能会带来安全风险，因为如果句柄已使用 SetHandleAsInvalid标记为无效，DangerousGetHandle 仍将返回原始的、可能过时的句柄值。 还可以随时回收返回的句柄。 最重要的是，这意味着句柄可能突然停止工作。 在最糟糕的情况下，如果句柄表示的句柄或资源公开给不受信任的代码，这可能会导致重复使用或返回的句柄上出现回收安全攻击。 例如，不受信任的调用方可以查询刚刚返回的句柄上的数据，并接收完全无关资源的信息。 有关使用 DangerousGetHandle methodsafely 的详细信息，请参阅 DangerousAddRef 和 DangerousRelease 方法。

错误的例子
```
static void Main(string[] args)
{
    System.Reflection.FieldInfo fieldInfo = ...;
    SafeHandle handle = (SafeHandle)fieldInfo.GetValue(rKey);
    IntPtr dangerousHandle = handle.DangerousGetHandle();  
}
```

### 4.不要在“async”方法中阻塞线程
由于在同一个Web Request 或 Winform WPF 的GUI操作上下文中，只允许绑定到一个线程，当在异步方法中同步等待，且互相调用，则会导致上下文被阻塞产生死锁

错误的例子
```
public static class DeadlockDemo
{
    private static async Task DelayAsync()
    {
        await Task.Delay(1000);//异步返回，控制权交给当前上下文，而此时上下文正被同步方法阻塞
    }

    public static void Test()
    {
        var delayTask = DelayAsync();
        // 阻塞当前上下文，等待异步任务完成
        delayTask.Wait(); 
    }
}
```
正确的例子
```
public static class DeadlockDemo
{
    private static async Task DelayAsync()
    {
        await Task.Delay(1000);
    }

    public static async Task TestAsync()
    {
        var delayTask = DelayAsync();
        await delayTask;
    }
}
```

### 5.正确使用ExportAttribute接口
在接口MEF属性编程中，ExportAttibute用来导出接口类型和契约供容器匹配使用，如果类没有继承ExportAttribute中声明的类型，会引发程序异常

错误的例子
```
[Export(typeof(ISomeType))]
public class SomeType // 没有继承接口ISomeType
{
}
```
正确的例子
```
[Export(typeof(ISomeType))]
public class SomeType : ISomeType
{
}
```
### 6.正确规范的使用字符串格式化或拼接
在字符串格式化的静态编码和编译阶段容易忽略格式化的问题，从而引发执行时的异常，尤其注意SQL语句字符的正确性

错误的例子
```
s = string.Format("[0}", arg0);
s = string.Format("{{0}", arg0);
s = string.Format("{0}}", arg0);
s = string.Format("{-1}", arg0);
s = string.Format("{0} {1}", arg0);
```
正确的例子
```
s = string.Format("{0}", 42); 
s = string.Format("{0,10}", 42); 
s = string.Format("{0,-10}", 42); 
s = string.Format("{0:0000}", 42); 
s = string.Format("{2}-{0}-{1}", 1, 2, 3); 
s = string.Format("no format"); 
```
### 7.禁止将敏感信息硬编码
通过反编译工具从已编译程序中提取字符串很容易，禁止硬编码任何敏感数据，建议存储在安全性更高的加密文件或数据库中

错误的例子
```
string username = "admin";
string password = "Password123"; 
string usernamePassword  = "user=admin&password=Password123"; 
string usernamePassword2 = "user=admin&" + "password=" + password; 
```
正确的例子
```
string username = "admin";
string password = GetEncryptedPassword();
string usernamePassword = string.Format("user={0}&password={1}", GetEncryptedUsername(), GetEncryptedPassword())
```
### 8.禁止在析构函数中抛出异常
当对象被标记为GC可回收对象，且被执行销毁时，程序会调用对象析构函数进行释放，如果此时执行抛出异常，将会导致整个程序的立即终止

错误的例子
```
class MyClass
{
    ~MyClass()
    {
        throw new NotImplementedException(); 
    }
}
```
正确的例子
```
class MyClass
{
    ~MyClass()
    {
        // no throw
    }
}
```
### 9.禁止在异常类的构造函数中抛出异常
当初始化异常对象时抛出异常将会导致应用程序的立即终止

错误的例子
```
class MyException: Exception
{
    public void MyException()
    {
         if (bad_thing)
         {
             throw new Exception("A bad thing happened");  
         }
    }
}
```
### 10.谨慎调用Exit方法
调用Environment.Exit(exitCode)或Application.Exit()将终止进程，并向操作系统返回退出代码，这些方法应该非常小心地使用，确保调用能够正确无误

### 11.测试方法的签名要正确
标注为测试的方法应该有正确的签名格式，非公开修饰，异步返回，以及泛型参数都会导致测试方法不会被执行到

错误的例子
```
[TestMethod]
void TestNullArg()  // 非公开方法
{  /* ... */  }

[TestMethod]
public async void MyIgnoredTestMethod()  // 异步方法
{ /* ... */ }

[TestMethod]
public void MyIgnoredGenericTestMethod<T>(T foo)  // 泛型参数
{ /* ... */ }
```
正确的例子
```
[TestMethod]
public void TestNullArg()
{  /* ... */  }
```
### 12.禁止类的递归继承
我们一般递归用在方法处理逻辑上，然而类的递归继承能够被正常编译，但无法被执行，禁止使用递归继承

错误的例子
```
class C1<T>
{
}
class C2<T> : C1<C2<C2<T>>> 
{
}

...
var c2 = new C2<int>();
```
