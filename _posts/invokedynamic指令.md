#jvm
[相关阅读： JVM之动态方法调用：invokedynamic](https://ifeve.com/jvm%E4%B9%8B%E5%8A%A8%E6%80%81%E6%96%B9%E6%B3%95%E8%B0%83%E7%94%A8%EF%BC%9Ainvokedynamic/)
- 作为1.0之后唯一新增的指令
- JVM如何支持动态类型语言
	- `invokedynamic`之前
		- Java方法的符号引用在编译时产生，而动态类型语言只有在运行期才能确定方法的接收者
		- 一般是在**预留占位符**，等运行期生成字节码实现具体类型再来适配
		- 缺点：
			- 无法确定方法，只能不停编译每个遇到的方法，并且缓存起来供调用，缓存开销大

---
### MethodHandle-方法句柄
- 例子，查找toString方法
```java
	public MethodHandle getToStringHandle() {
        MethodHandle mh = null;
        MethodType mt = MethodType.methodType(String.class);
        MethodHandles.Lookup lk = MethodHandles.lookup();
        try {
            mh = lk.findVirtual(getClass(), "toString", mt);
        } catch (NoSuchMethodException | IllegalAccessException mhx) {
            throw new AssertionError().initCause(mhx);
        }
        return mh;
    }
```
- 出现背景
	- 原来的方法签名只在反射api `Class[]`出现，很笨重且容易出错
- MethodHandle与反射的区别
	- 更安全
		- 在当前执行方法的上下文中寻找（`MethodHandles.lookup()`），更安全
	- 层次不同：
		- 反射针模拟**代码层**次方法调用，MethodHandle 模拟**字节码层**次
			- MethodHandles.Lookup上的三个方法f`indStatic()`、`findVirtual()`、`findSpecial`()正是为了对应于[[解析#invokestatic 静态绑定|invokestatic]]、[[解析#invokevirtual（动态绑定）|invokevirtual]] & [[解析#invokeinterface（动态绑定）|invokeinterface]]和[[解析#invokespecial（静态绑定）|invokespecial]]这几条字节码指令的执行权限校验行为，而这些底层细节在使用Reflection API时是不需要关心的。
	- 优化性
		- 方法句柄的调用可以优化。
	- 服务范围更广
		- MethodHandle可以不止为java服务，为jvm虚拟机
	- 更轻量
	- 包含信息种类
		- reflect 中的 Method 对象远比 invoke 中的 MethodHandle 对象所包含的信息来得多。


### MethodType
-   用来表示方法的类型签名
-   包括返回值类型以及参数类型
-   不包含接收者类型和方法名
-   是设计来解决反射中的Class[]问题的

>有了这套API，方法签名便可以通过MethodType的实例来表示了，也不再需要为了每一个可能的签名去创建一个新的类型了。一个很简单的工厂方法便能够创建出新的实例：

### BootStrapsMethod-BSM引导方法
- 

### invokedynamic指令
>而对于 invokevirtual和invokeinterface来说，调用目标在**运行时才能确定。**
>然而，选择的目标也是受限于Java语言规范的规则和类型系统的约束的。因此，至少有部分调用信息在编译期是能确定的。
>**invokedynamic更灵活**
- 作用
	- `java.lang.invoke` 包为了虚拟机支持动态语言出现。
	- 将如何查找目标方法的决定权从虚拟机转嫁到具体用户代码中
- 实现流程
	- 以java8 lambda举例
		```java
		class InvokeDynamicExample {
		    public void lambda1() {
		        Runnable runnable = () -> {
		            int i = 1;
		        };
		        runnable.run();
		    }
		}

		```

		```java
		{
		  public void lambda1();
		    Code:
		      stack=1, locals=2, args_size=1
		         0: invokedynamic #2,  0     // 生成动态调用站点
		         5: astore_1
		         6: aload_1
		         7: invokeinterface #3,  1  // 调用 lambda 表达式
		        12: return
		      LocalVariableTable:
		        Start  Length  Slot  Name   Signature
		            0      13     0  this   LInvokeDynamicExample;
		            6       7     1 runnable   Ljava/lang/Runnable;
		
		  private static void lambda$lambda1$0(); // Runnable lambda 表达式默认生成的方法
		    Code:
		      stack=1, locals=1, args_size=0
		         0: iconst_1
		         1: istore_0
		         2: return
		}
		BootstrapMethods:  // 引导方法
		  0: #23 invokestatic   // 调用静态方法 LambdaMetafactory.metafactory() 返回 CallSite 对象
		    Method arguments: // 静态方法关联参数
		      #24 ()V
		      #25 invokestatic InvokeDynamicExample.lambda$lambda1$0:()V
		      #24 ()V
		```

- 编译期间将Lambda表达式内容编译成一个新方法，无关联就静态方法。
	-  上述示例被编译为 `private static void lambda$lambda1$0` 静态方法。
- 执行到invokedynamic时，将调用常量池对应的BSM引导方法，返回一个动态调用站点对象**CallSite**,该对象绑定了要指定的方法句柄
	- 上述示例引导方法为 `#23 LambdaMetafactory.metafactory` ，该方法返回一个动态调用站点对象 CallSite
- 动态调用站点对象 CallSite 上绑定了 `lambda$lambda1$0` 方法相关的信息（参考字节码 #25）。
- 之后执行 `runnable.run();` 代码时，虚拟机则直接调用已经绑定了调用点所链接的 `lambda$lambda1$0` 方法