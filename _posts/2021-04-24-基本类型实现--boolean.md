- boolean
	- 在虚拟机中，true =1 ，false=0 ， 所以编译检查，必须是0和1
		- `if(flag)`
			- 相当于：`falg!=0`
		- `if(flag==true)`
			- 相当于`flag==1`
	- 如果通过字节码工具(**asmtools.jar**)修改字节码为其他整数
		- 局部变量
			- long、double 两个数组单元
			- 其他基本类型和引用类型一个数组单元
			- 数组单元，32位4字节，64位8字节
		- 堆上
			- boolean=1 byte，char= short=2 byte
			- 当int 存入 char ，通过掩码截取，只保留低2字节
			- 当加载到操作数栈上，当做int 计算，高位用0填充
		- 
			```java
			//实验类
			public class App{
			 public static void main(String[] args) {
			  boolean flag = true;
			  if (flag) System.out.println("Hello, Java!");
			  if (flag == true) System.out.println("Hello, JVM!");
			 }
			}
			```
		- 编译获取class
			- `javac App.java`
		- 生产jasm字节码文件
			- 
				```bash
				java -jar asmtools.jar jdis App.class > App.jasm.2
				```
		- 修改jasm，将入参true 1改成2
			- 将`iconst_1`->`iconst_2`
		- 重新编译成class
			- 
				```bash
				java -jar asmtools.jar jasm App.jasm.1
				```