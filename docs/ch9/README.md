第9章 本地方法调用
---

在前面的8章里，我们一直在实现Java虚拟机的基本功能。我们已经知道，要想运行Java程序，除了Java虚拟机之外，还需要Java类库的配合。Java虚拟机和Java类库一起构成了Java运行时环境。Java类库主要用Java语言编写，一些无法用Java语言实现的方法则使用本地语言编写，这些方法叫作本地方法。从本章开始，将陆续实现一些Java类库中的本地方法。 

OpenJDK类库中的本地方法是用JNI（Java Native Interface） [1]编写的，但是要让虚拟机支持JNI规范还需要做大量的工作。由于本书的主要目的是介绍Java虚拟机的工作原理，为了不陷入JNI规范的细节之中，将使用Go语言来实现这些方法。 

开始编写代码之前，还是先把目录结构准备好。复制ch08目录，改名为ch09。修改main.go等文件，把import语句中的ch08全都替换成ch09。在ch09目录下创建native子目录，本章新增的go文件主要都在这个目录（和它的子目录）中。现在，目录结构看起来是下面这个样子：

```shell script
D:\go\workspace\src 
|-jvmgo 
|-ch01 ~ ch08 
|-ch09 
|-classfile 
|-classpath 
|-instructions 
|-native 
|-rtda 
|-cmd.go 
|-interpreter.go 
|-main.go 
```

[1]http://docs.oracle.com/javase/8/docs/technotes/guides/jni/spec/intro.html
#### 9.1 注册和查找本地方法 
在开始实现本地方法之前，先实现一个本地方法注册表，用来注册和查找本地方法。在ch09\native目录下创建registry.go，先在其中定义NativeMethod类型和registry变量，代码如下：
```go
package native 
import "jvmgo/ch09/rtda" 
type NativeMethod func(frame *rtda.Frame) 
var registry = map[string]NativeMethod{} 
```
把本地方法定义成一个函数，参数是Frame结构体指针，没有返回值。这个frame参数就是本地方法的工作空间，也就是连接Java虚拟机和Java类库的桥梁，后面会看到它是如何发挥作用的。registry变量是个哈希表，值是具体的本地方法实现。那么键是什么呢？继续编辑registry.go文件，在其中实现Register（）函数，代码如下：
```go
func Register(className, methodName, methodDescriptor string, method NativeMethod) { 
    key := className + "~" + methodName + "~" + methodDescriptor 
    registry[key] = method 
}
```
类名、方法名和方法描述符加在一起才能唯一确定一个方法，所以把它们的组合作为本地方法注册表的键，Register（）函数把前述三种信息和本地方法实现关联起来。继续编辑registry.go文件，在其中实现FindNativeMethod（）方法，代码如下：
```go
func FindNativeMethod(className, methodName, methodDescriptor string) NativeMethod { 
key := className + "~" + methodName + "~" + methodDescriptor 
if method, ok := registry[key]; ok { 
return method 
}
if methodDescriptor == "()V" && methodName == "registerNatives" { 
return emptyNativeMethod 
}
return nil 
}
```
FindNativeMethod（）方法根据类名、方法名和方法描述符查找本地方法实现，如果找不到，则返回nil。第7章结尾提到过，jva.lang.Object等类是通过一个叫作registerNatives（）的本地方法来注册其他本地方法的。在本章和后面的章节中，将自己注册所有的本地方法实现。所以像registerNatives（）这样的方法就没有太大的用处。为了避免重复代码，这里统一处理，如果遇到这样的本地方法，就返回一个空的实现，代码如下：
```go
func emptyNativeMethod(frame *rtda.Frame) { 
// do nothing 
}
```
本地方法注册表准备好了，下面介绍如何调用本地方法。
#### 9.2 调用本地方法 
第7章用一段hack代码来跳过本地方法的执行。现在，终于可以把这段代码删除了！编辑ch09\instructions\base\method_invoke_logic.go，把fmt包的导入语句和Invoke -Method（）函数中的hack代码删除。为了节约篇幅，这里就不给出代码了。 
Java虚拟机规范并没有规定如何实现和调用本地方法，这给了我们充分的空间来发挥自己的想象力。读者很快就会看到，我们将利用Java虚拟机栈执行本地方法，所以除了删除上面的InvokeMethod（）函数中的hack代码之外，不用做任何修改。
但是，本地方法并没有字节码，如何利用Java虚拟机栈来执行呢？Java虚拟机规范预留了两条指令，操作码分别是0xFE和0xFF。下面将使用0xFE指令来达到这个目的。打开ch09\rtda\heap\method.go文件，修改newMethods（）函数，改动如下：
```go
func newMethods(class *Class, cfMethods []*classfile.MemberInfo) []*Method { 
methods := make([]*Method, len(cfMethods)) 
for i, cfMethod := range cfMethods { 
methods[i] = newMethod(class, cfMethod) 
}
return methods 
}
```
为了避免newMethods（）函数变得太长，我们抽取出一个newMethod（）函数，代码如下：
```go
func newMethod(class *Class, cfMethod *classfile.MemberInfo) *Method { 
method := &Method{} 
method.class = class 
method.copyMemberInfo(cfMethod) 
method.copyAttributes(cfMethod) 
md := parseMethodDescriptor(method.descriptor) 
method.calcArgSlotCount(md.parameterTypes) 
if method.IsNative() { 
method.injectCodeAttribute(md.returnType) 
}
return method 
}
```
粗体部分需要解释一下：先计算argSlotCount字段，如果是本地 
方法，则注入字节码和其他信息。继续编辑method.go文件，添加 
injectCodeAttribute（）方法，代码如下：
```go
func (self *Method) injectCodeAttribute(returnType string) { 
self.maxStack = 4 
self.maxLocals = self.argSlotCount 
switch returnType[0] {
case 'V': self.code = []byte{0xfe, 0xb1} // return 
case 'D': self.code = []byte{0xfe, 0xaf} // dreturn 
case 'F': self.code = []byte{0xfe, 0xae} // freturn 
case 'J': self.code = []byte{0xfe, 0xad} // lreturn 
case 'L', '[': self.code = []byte{0xfe, 0xb0} // areturn 
default: self.code = []byte{0xfe, 0xac} // ireturn 
}
}
```
本地方法在class文件中没有Code属性，所以需要给maxStack和maxLocals字段赋值。本地方法帧的操作数栈至少要能容纳返回值，为了简化代码，暂时给maxStack字段赋值为4。因为本地方法帧的局部变量表只用来存放参数值，所以把argSlotCount赋给maxLocals字段刚好。至于code字段，也就是本地方法的字节码，第一条指令都是0xFE，第二条指令则根据函数的返回值选择相应的返回指令。
另外，由于把方法描述符的解析挪到了newMethod（）函数中，所以calcArgSlotCount()方法也稍微有些变化（增加了一个参数），变动如下： 
```go
func (self *Method) calcArgSlotCount(paramTypes []string) { 
for _, paramType := range paramTypes { 
... // 其他代码不变 
}
```
下面我们来实现0xFE指令。在ch09\instructions目录下创建reserved子目录，然后在该目录下创建invokenative.go文件，在其中定义0xFE（后面称之为invokenative）指令，代码如下：
```go
package reserved 
import "jvmgo/ch09/instructions/base" 
import "jvmgo/ch09/rtda" 
import "jvmgo/ch09/native"
type INVOKE_NATIVE struct{ base.NoOperandsInstruction } 
```
这个指令不需要操作数，Execute（）方法的代码如下：
```go
func (self *INVOKE_NATIVE) Execute(frame *rtda.Frame) { 
method := frame.Method() 
className := method.Class().Name() 
methodName := method.Name() 
methodDescriptor := method.Descriptor() 
nativeMethod := native.FindNativeMethod(className, methodName, methodDescriptor) 
if nativeMethod == nil { 
methodInfo := className + "." + methodName + methodDescriptor 
panic("java.lang.UnsatisfiedLinkError: " + methodInfo) 
}
nativeMethod(frame) 
}
```
根据类名、方法名和方法描述符从本地方法注册表中查找本地方法实现，如果找不到，则抛出UnsatisfiedLinkError异常，否则直接调用本地方法。最后，还需要修改instruction -s\factory.go文件，在其中添加invokenative指令的case语句，这里就不给出代码了。 
现在，万事俱备，只欠实现本地方法！接下来，我们将实现Object和String等类的一些本地方法。在后面几章中，还会实现更多的本地方法。
#### 9.3 反射 
Java的反射机制十分强大，本节讨论的内容只是冰山一角。
##### 9.3.1 类和对象之间的关系 
在Java中，类也表现为普通的对象，它的类是java.lang.Class。听起来有点像鸡生蛋还是蛋生鸡的问题：类也是对象，而对象又是类的实例。那么在Java虚拟机内部，究竟是先有类还是先有对象呢？下面就来一探究竟。 
如前所述，Java有强大的反射能力。可以在运行期间获取类的各种信息、存取静态和实例变量、调用方法，等等。要想运用这种能力，获取类对象 [1] 是第一步。在Java语言中，有两种方式可以获得类对象引用：使用类字面值和调用对象的getClass（）方法。下面的Java代码演示了这两种方式。 
```java
System.out.println(String.class); 
System.out.println("abc".getClass()); 
```
在第6章中，通过Object结构体的class字段建立了类和对象之间的单向关系。现在把这个关系补充完整，让它成为双向的。打开ch09\rtda\heap\class.go文件，修改Class结构体，添加jClass字段，改动如下：
```go
type Class struct { 
... // 其他字段 
jClass *Object // java.lang.Class实例 
}
```
通过jClass字段，每个Class结构体实例都与一个类对象关联。另外需要给jClass字段定义Getter方法，代码比较简单，就不给出了。下面打开ch09\rtda\heap\object.go文件，修改Object结构体，添加extra字段，改动如下： 
```go
type Object struct { 
class *Class 
data interface{} 
extra interface{} 
}
```
extra字段用来记录Object结构体实例的额外信息。同样给它定义Getter和Setter方法，这里就不给出代码了。这个字段之所以是interface{}类型，是因为它在后面几章还会有其他用途。本章，只用它来记录类对象对应的Class结构体指针。 

如果读者读到这里感觉有些吃力，请不要怀疑自己的理解能力，一定是笔者表达得不够好。另外，笔者在写这一节时，自己也是犯了很多次迷糊的。为了帮助大家更好地理解类和对象之间的关系，让我们想象这样一个极简化的Java虚拟机运行时状态：方法区中只加载了两个类，java.lang.Object和java.lang.Class；堆中只通过new指令分配了一个对象。此时Java虚拟机的内存状态如图9-1所示。
![9-1](/img/9-1.png) 
 							图9-1 类和对象关系图 

图9-1只画出了Class和Object结构体的必要字段，并且刻意分开了堆和方法区。在方法区中，class1和class2分别是java.lang.Object和java.lang.Class类的数据。在堆中，object1和object2分别是java.lang.Object和java.lang.Class的类对象。object3是单独的java.lang.Object实例。虽然已经简化到了极点，但仍然有8条箭头，希望有密集恐惧症的读者不要被吓倒。 
[1] 在本书中，类对象特指java.lang.Class类的实例；对象泛指任何类的实例。
##### 9.3.2 修改类加载器 
Class和Object结构体准备好了，接下来修改类加载器，让每一个加载到方法区中的类都有一个类对象与之相关联。打开ch09\rtda\heap\class_loader.go文件，修改NewClassLoader（）函数，改动如下： 
```go
func NewClassLoader(cp *classpath.Classpath, verboseFlag bool) *ClassLoader { 
loader := &ClassLoader{ 
cp: cp, 
verboseFlag: verboseFlag, 
classMap: make(map[string]*Class), 
}
loader.loadBasicClasses() 
return loader 
}
```
在返回ClassLoader结构体实例之前，先调用loadBasicClasses（）函数。这个函数也要添加到class_loader.go文件中，代码如下：
```go
func (self *ClassLoader) loadBasicClasses() { 
jlClassClass := self.LoadClass("java/lang/Class") 
for _, class := range self.classMap { 
if class.jClass == nil { 
class.jClass = jlClassClass.NewObject() 
class.jClass.extra = class 
} 
} 
}
```
loadBasicClasses（）函数先加载java.lang.Class类，这又会触发java.lang.Object等类和接口的加载。然后遍历classMap，给已经加载的每一个类关联类对象。好啦，问题已经解决了一半。下面修改LoadClass（）方法，解决另一半问题。改动较大，代码如下：
```go
func (self *ClassLoader) LoadClass(name string) *Class { 
if class, ok := self.classMap[name]; ok { 
return class // already loaded 
}
var class *Class 
if name[0] == '[' { // array class 
class = self.loadArrayClass(name) 
} else { 
class = self.loadNonArrayClass(name) 
}
if jlClassClass, ok := self.classMap["java/lang/Class"]; ok { 
class.jClass = jlClassClass.NewObject() 
class.jClass.extra = class 
}
return class 
}
```
主要的变动是粗体部分。在类加载完之后，看java.lang.Class是否已经加载。如果是，则给类关联类对象。这样，在loadBasicClasses（）和LoadClass（）方法的配合之下，所有加载到方法区的类都设置好了jClass字段。
##### 9.3.3 基本类型的类 
void和基本类型也有对应的类对象，但只能通过字面值来访问，如下面的Java代码所示。 
```java
System.out.println(void.class); 
System.out.println(boolean.class); 
System.out.println(byte.class); 
System.out.println(char.class); 
System.out.println(short.class); 
System.out.println(int.class); 
System.out.println(long.class); 
System.out.println(float.class); 
System.out.println(double.class); 
```
和数组类一样，基本类型的类也是由Java虚拟机在运行期间生成的。继续编辑class_loader.go文件，修改NewClassLoader（）函数，在其中添加如下一行代码： 
```go
func NewClassLoader(cp *classpath.Classpath, verboseFlag bool) *ClassLoader { 
... // 前面的代码不变 
loader.loadBasicClasses() 
loader.loadPrimitiveClasses() 
return loader 
} 
```
loadPrimitiveClasses（）方法加载void和基本类型的类，代码如下：
```go
func (self *ClassLoader) loadPrimitiveClasses() { 
for primitiveType, _ := range primitiveTypes { 
self.loadPrimitiveClass(primitiveType) // primitiveType是 void、int、 float等 
} 
}
```
生成void和基本类型类的代码在loadPrimitiveClass（）方法中，代码如下：
```go
func (self *ClassLoader) loadPrimitiveClass(className string) { 
class := &Class{ 
accessFlags: ACC_PUBLIC, 
name: className, 
loader: self, 
initStarted: true, 
}
class.jClass = self.classMap["java/lang/Class"].NewObject() 
class.jClass.extra = class 
self.classMap[className] = class 
}
```
这里有三点需要说明。第一，void和基本类型的类名就是void、int、float等。第二，基本类型的类没有超类，也没有实现任何接口。 
第三，非基本类型的类对象是通过ldc指令加载到操作数栈中的，将在9.3.4节修改ldc指令，让它支持类对象。而基本类型的类对象，虽然在Java代码中看起来是通过字面量获取的，但是编译之后的指令并不是ldc，而是getstatic。每个基本类型都有一个包装类，包装类中有一个静态常量，叫作TYPE，其中存放的就是基本类型的类。例如java.lang. Integer类，代码如下：
```java
public final class Integer extends Number implements Comparable<Integer> { 
... // 其他代码 
@SuppressWarnings("unchecked") 
public static final Class<Integer> TYPE 
= (Class<Integer>) Class.getPrimitiveClass("int"); 
... // 其他代码 
} 
```
也就是说，基本类型的类是通过getstatic指令访问相应包装类的TYPE字段加载到操作数栈中的。Class.getPrimitiveClass（）是个本地方法，将在9.3.5节实现它。包装类将在9.7小节详细讨论。
##### 9.3.4 修改ldc指令 
和基本类型、字符串字面值一样，类对象字面值也是由ldc指令加载的。本节修改ldc指令，让它可以加载类对象。打开ch09\instructions\constants\ldc.go文件，修改_ldc()函数，改动如下：
```go
func _ldc(frame *rtda.Frame, index uint) { 
stack := frame.OperandStack() 
class := frame.Method().Class() 
c := class.ConstantPool().GetConstant(index) 
switch c.(type) { 
case int32: ... 
case float32: ... 
case string: ... 
case *heap.ClassRef: 
classRef := c.(*heap.ClassRef) 
classObj := classRef.ResolvedClass().JClass() 
stack.PushRef(classObj) 
default: ... 
} 
}
```
以上只是增加了一个case语句，其他地方没什么变化。如果运行时，常量池中的常量是类引用，则解析类引用，然后把类的类对象推入操作数栈顶。
##### 9.3.5 通过反射获取类名 
为了支持通过反射获取类名，本小节将实现以下4个本地方法： 
::: tip
·java.lang.Object.getClass（） 
·java.lang.Class.getPrimitiveClass（） 
·java.lang.Class.getName0（） 
·java.lang.Class.desiredAssertionStatus0（）
::: 
Object.getClass（）就不用多说了，它返回对象的类对象引用。Class.getPrimitiveClass（）在9.3.3节提到过，基本类型的包装类在初始化时会调用这个方法给TPYE字段赋值。Character类是基本类型char的包装类，它在初始化时会调用Class.desiredAssertionStatus0（）方法，所以这个方法也需要实现。最后，之所以要实现getName0（）方法，是因为Class.getName（）方法是依赖这个本地方法工作的，该方法的代码如下：
```java
// java.lang.Class
public String getName() { 
String name = this.name; 
if (name == null) 
this.name = name = getName0(); 
return name; 
}
```
在ch09\native目录下创建java子目录，在java子目录下创建lang子目录，然后在lang目录中创建Object.go文件，在其中注册getClass（）本地方法，代码如下：
```go
package lang 
import "jvmgo/ch09/native" 
import "jvmgo/ch09/rtda" 
func init() { 
native.Register("java/lang/Object", "getClass", "()Ljava/lang/Class;", getClass) 
} 
```
继续编辑Object.go，实现getClass（）函数，代码如下：
```go
// public final native Class<?> getClass(); 
func getClass(frame *rtda.Frame) { 
this := frame.LocalVars().GetThis() 
class := this.Class().JClass() 
frame.OperandStack().PushRef(class) 
}
```
这是实现的第一个本地方法，所以有必要详细解释一下。首先，从局部变量表中拿到this引用。GetThis（）方法其实就是调用GetRef（0），不过为了提高代码的可读性，给LocalVars结构体添加了这个方法。有了this引用后，通过Class（）方法拿到它的Class结构体指针，进而又通过JClass（）方法拿到它的类对象。最后，把类对象推入操作数栈顶。这样，只用了3行代码，Object.getClass（）方法就实现好了。 
在ch09\native\java\lang目录下创建Class.go文件，在其中注册3个本地方法，代码如下：
```go
package lang 
import "jvmgo/ch09/native" 
import "jvmgo/ch09/rtda" 
import "jvmgo/ch09/rtda/heap" 
func init() { 
native.Register("java/lang/Class", "getPrimitiveClass", 
"(Ljava/lang/String;)Ljava/lang/Class;", getPrimitiveClass) 
native.Register("java/lang/Class", "getName0", "()Ljava/lang/String;", getName0) 
native.Register("java/lang/Class", "desiredAssertionStatus0", 
"(Ljava/lang/Class;)Z", desiredAssertionStatus0) 
}
```
先实现getPrimitiveClass（）方法，代码如下： 
```go
// static native Class<?> getPrimitiveClass(String name); 
func getPrimitiveClass(frame *rtda.Frame) { 
nameObj := frame.LocalVars().GetRef(0) 
name := heap.GoString(nameObj) 
loader := frame.Method().Class().Loader() 
class := loader.LoadClass(name).JClass() 
frame.OperandStack().PushRef(class) 
} 
```
getPrimitiveClass（）是静态方法。先从局部变量表中拿到类名，这是个Java字符串，需要把它转成Go字符串。基本类型的类已经加载到了方法区中，直接调用类加载器的Load -Class（）方法获取即可。最后，把类对象引用推入操作数栈顶。下面实现getName0（）方法，代码如下：
```go 
// private native String getName0(); 
func getName0(frame *rtda.Frame) { 
this := frame.LocalVars().GetThis() 
class := this.Extra().(*heap.Class) 
name := class.JavaName() 
nameObj := heap.JString(class.Loader(), name) 
frame.OperandStack().PushRef(nameObj) 
} 
```
首先从局部变量表中拿到this引用，这是一个类对象引用，通过Extra（）方法可以获得与之对应的Class结构体指针。然后拿到类名，转成Java字符串并推入操作数栈顶。注意这里需要的是java.lang.Object这样的类名，而非java/lang/Object。Class结构体的JavaName（）方法返回转换后的类名，代码如下：
```go
func (self *Class) JavaName() string { 
return strings.Replace(self.name, "/", ".", -1) 
}
```
本书不讨论断言。desiredAssertionStatus0（）方法把false推入操作数栈顶，代码如下：
```go
// private static native boolean desiredAssertionStatus0(Class<?> clazz);
func desiredAssertionStatus0(frame *rtda.Frame) { 
frame.OperandStack().PushBoolean(false) 
}
```
4个本地方法都实现好了，而且也已经在init（）函数中注册，那么可以进行测试了吗？还不行，因为init（）函数还没有机会执行。
编辑ch09\instructions\reserved\invokenative.go文件，在其中导入lang包，代码如下：
```go
package reserved 
import "jvmgo/ch09/instructions/base" 
import "jvmgo/ch09/rtda" 
import "jvmgo/ch09/native" 
import _ "jvmgo/ch09/native/java/lang" 
```
如果没有任何包依赖lang包，它就不会被编译进可执行文件，上面的本地方法也就不会被注册。所以需要一个地方导入lang包，把它放在invokenative.go文件中。由于没有显示使用lang中的变量或函数，所以必须在包名前面加上下划线，否则无法通过编译。这个技术在Go语言中叫作“import for side effect”。 [1] 
[1]https://golang.org/doc/effective_go.html#blank_import
##### 9.3.6 测试本节代码 
打开命令行窗口，执行下面的命令编译本章代码：
```shell script
go install jvmgo\ch09 
```
命令执行完毕后，在D：\go\workspace\bin目录下出现ch09.exe文件。用ch09.exe运行下面的Java程序：
```java
package jvmgo.book.ch09; 
public class GetClassTest { 
public static void main(String[] args) { 
System.out.println(void.class.getName()); // void 
System.out.println(boolean.class.getName()); // boolean 
System.out.println(byte.class.getName()); // byte 
System.out.println(char.class.getName()); // char 
System.out.println(short.class.getName()); // short 
System.out.println(int.class.getName()); // int 
System.out.println(long.class.getName()); // long 
System.out.println(float.class.getName()); // float 
System.out.println(double.class.getName()); // double 
System.out.println(Object.class.getName()); // java.lang.Object 
System.out.println(int[].class.getName()); // [I 
System.out.println(int[][].class.getName()); // [[I 
System.out.println(Object[].class.getName()); // [Ljava.lang.Object; 
System.out.println(Object[][].class.getName()); // [[Ljava.lang.Object; 
System.out.println(Runnable.class.getName()); // java.lang.Runnable 
System.out.println("abc".getClass().getName()); // java.lang.String 
System.out.println(new double[0].getClass().getName()); // [D 
System.out.println(new String[0].getClass().getName());//[Ljava.lang.String; 
} 
}
```
运行结果如图9-2所示。
![9-2](/img/9-2.png)   
图9-2 GetClassTest程序执行结果
#### 9.4 字符串拼接和String.intern（）方法 
##### 9.4.1 Java类库 
在Java语言中，通过加号来拼接字符串。作为优化，javac编辑器会把字符串拼接操作转换成StringBuilder的使用。例如下面这段Java代码：
```java
String hello = "hello,"; 
String world = "world!"; 
String str = hello + world; 
System.out.println(str); 
```
很可能会被javac优化为下面这样： 
```java
String str = new tringBuilder().append("hello,").append("world!").toString(); 
System.out.println(str); 
```
为了运行上面的代码，本节将实现以下3个本地方法：
::: tip 
·System.arrayCopy（） 
·Float.floatToRawIntBits（）
·Double.doubleToRawLongBits（）
::: 
这些方法是在哪里使用的呢?StringBuilder.append()方法只是调用了超类的append()方法，代码如下：
```java
// java.lang.StringBuilder 
@Override 
public StringBuilder append(String str) { 
super.append(str); // 调用 
AbstractStringBuilder.append() 
return this; 
}
```
AbstractStringBuilder.append（）方法调用了String.getChars（）方法获取字符数组，代码如下：
```java
// java.lang.AbstractStringBuilder 
public AbstractStringBuilder append(String str) { 
if (str == null) return appendNull(); 
int len = str.length(); 
ensureCapacityInternal(count + len); 
str.getChars(0, len, value, count); 
count += len; 
return this; 
}
```
String.getChars（）方法调用了System.arraycopy（）方法拷贝数组，代码如下：
```java
// java.lang.String 
public void getChars(int srcBegin, int srcEnd, char dst[], int dstBegin) { 
... // 其他代码
System.arraycopy(value, srcBegin, dst, dstBegin, srcEnd - srcBegin); 
} 
```
StringBuilder.toString（）方法的代码如下：
```java
// java.lang.StringBuilder 
@Override 
public String toString() { 
// Create a copy, don't share the array 
return new String(value, 0, count); 
} 
```
它调用了String的构造函数，这个构造函数调用了Arrays.copyOfRange（）方法，代码如下：
```java
// java.lang.String 
public String(char value[], int offset, int count) { 
... // 其他代码 
this.value = Arrays.copyOfRange(value, offset, offset+count); 
}
```
Arrays.copyOfRange（）调用了Math.min（）方法，代码如下： 
```java
// java.util.Arrays 
public static char[] copyOfRange(char[] original, int from, int to) { 
int newLength = to - from; 
if (newLength < 0) throw new IllegalArgumentException(from + " > " + to); 
char[] copy = new char[newLength]; 
System.arraycopy(original, from, copy, 0,Math.min(original.length - from, newLength)); 
return copy; 
}
```
Math类在初始化时需要调用Float.floatToRawIntBits（）和Double.doubleToRawLong -Bits（）方法，代码如下：
```java
package java.lang; 
public final class Math { 
// Use raw bit-wise conversions on guaranteed non-NaN arguments. 
private static long negativeZeroFloatBits = Float.floatToRawIntBits(-0.0f); 
private static long negativeZeroDoubleBits = Double.doubleToRawLongBits(-0.0d); 
} 
```
Java类库介绍完了，下面实现本地方法。
##### 9.4.2 System.arraycopy（）方法 
在ch09\native\java\lang目录下创建System.go文件，在其中注册arraycopy（）方法，代码如下：
```go
package lang 
import "jvmgo/ch09/native" 
import "jvmgo/ch09/rtda" 
import "jvmgo/ch09/rtda/heap" 
func init() { 
    native.Register("java/lang/System", "arraycopy", "(Ljava/lang/Object;ILjava/lang/Object;II)V", arraycopy) 
}
```
继续编辑System.go，实现arraycopy（）方法。代码稍微有些复杂，先来看第一部分。
 ```go
// public static native void arraycopy( 
// Object src, int srcPos, Object dest, int destPos, int length) 
func arraycopy(frame *rtda.Frame) { 
    vars := frame.LocalVars() 
    src := vars.GetRef(0) 
    srcPos := vars.GetInt(1) 
    dest := vars.GetRef(2) 
    destPos := vars.GetInt(3) 
    length := vars.GetInt(4) 
```
先从局部变量表中拿到5个参数。源数组和目标数组都不能是null，否则按照System类的Javadoc应该抛出NullPointerException异常，代码如下：
```go
if src == nil || dest == nil { 
    panic("java.lang.NullPointerException") 
}
``` 
源数组和目标数组必须兼容才能拷贝，否则应该抛出ArrayStoreExceptio异常，代码如下： 
ArrayStoreExceptio异常，代码如下： 
```go
if !checkArrayCopy(src, dest) { 
    panic("java.lang.ArrayStoreException") 
}
```
checkArrayCopy（）函数的代码稍后给出。接下来检查srcPos、destPos和length参数，如果有问题则抛出IndexOutOfBoundsException异常，代码如下：
```go
if srcPos < 0 || destPos < 0 || length < 0 || 
    srcPos+length > src.ArrayLength() || 
    destPos+length > dest.ArrayLength() { 
    panic("java.lang.IndexOutOfBoundsException") 
}
```
最后，参数合法，调用ArrayCopy（）函数进行数组拷贝，代码如下：
```go
heap.ArrayCopy(src, dest, srcPos, destPos, length) 
}
```
checkArrayCopy（）函数首先确保src和dest都是数组，然后检查数组类型。如果两者都是引用数组，则可以拷贝，否则两者必须是相同类型的基本类型数组，代码如下：
```go
func checkArrayCopy(src, dest *heap.Object) bool { 
srcClass := src.Class() 
destClass := dest.Class() 
if !srcClass.IsArray() || !destClass.IsArray() { 
return false 
}
if srcClass.ComponentClass().IsPrimitive() || 
destClass.ComponentClass().IsPrimitive() { 
return srcClass == destClass 
}
return true 
}
```
Class结构体的IsPrimitive（）函数判断类是否是基本类型的类， 代码如下：
```go
func (self *Class) IsPrimitive() bool { 
_, ok := primitiveTypes[self.name] 
return ok
}
``` 
真正的数组拷贝逻辑在ch09\rtda\heap\array_object.go文件中，代码如下：
```go
func ArrayCopy(src, dst *Object, srcPos, dstPos, length int32) { 
switch src.data.(type) { 
case ... 
case []int32: 
_src := src.data.([]int32)[srcPos : srcPos+length]_dst := dst.data.([]int32)[dstPos : dstPos+length] 
copy(_dst, _src) 
case []*Object: 
_src := src.data.([]*Object)[srcPos : srcPos+length] 
_dst := dst.data.([]*Object)[dstPos : dstPos+length] 
copy(_dst, _src) 
} 
}
```
利用Go的内置函数copy（）进行slice拷贝。为了节约篇幅，上面的代码只给出了int数组和对象数组的case语句，其他情况代码大同小异。
##### 9.4.3 Float.floatToRawIntBits（）和 
Double.doubleToRawLongBits（）方法 
Float.floatToRawIntBits（）和Double.doubleToRawLongBits（）返回浮点数的编码，这两个方法大同小异，以Float为例进行介绍。在ch09\native\java\lang目录下创建Float.go文件，在其中注册floatToRawIntBits（）本地方法，代码如下：
```go
package lang 
import "math" 
import "jvmgo/ch09/native" 
import "jvmgo/ch09/rtda" 
func init() { 
native.Register("java/lang/Float", "floatToRawIntBits", "(F)I", floatToRawIntBits) 
}
```
Go语言的math包提供了一个类似函数：Float32bits（），正好可以用来实现floatToRaw-IntBits（）方法，代码如下：
```go
// public static native int floatToRawIntBits(float value); 
func floatToRawIntBits(frame *rtda.Frame) { 
value := frame.LocalVars().GetFloat(0) 
bits := math.Float32bits(value) 
frame.OperandStack().PushInt(int32(bits)) 
}
```
##### 9.4.4 String.intern（）方法 
第8章讨论字符串时，实现了字符串池，但它只能在虚拟机内部使用。下面实现String类的intern（）方法，让Java类库也可以使用它。在ch09\native\java\lang目录下创建String.go，在其中注册intern（）方法，代码如下：
```go
package lang 
import "jvmgo/ch09/native" 
import "jvmgo/ch09/rtda" 
import "jvmgo/ch09/rtda/heap" 
func init() { 
    native.Register("java/lang/String", "intern", "()Ljava/lang/String;", intern) 
}
```
继续编辑String.go文件，实现intern（）方法，代码如下：
```go
// public native String intern(); 
func intern(frame *rtda.Frame) { 
this := frame.LocalVars().GetThis() 
interned := heap.InternString(this) 
frame.OperandStack().PushRef(interned) 
}
```
如果字符串还没有入池，把它放入并返回该字符串，否则找到已入池字符串并返回。这个逻辑在InternString（）函数中（ch09\rtda\heap\string_pool.go），代码如下：
```go
func InternString(jStr *Object) *Object { 
goStr := GoString(jStr) 
if internedStr, ok := internedStrings[goStr]; ok { 
return internedStr 
}
internedStrings[goStr] = jStr 
return jStr 
}
```
字符串相关的本地方法都实现好了，下面我们进行测试。
##### 9.4.5 测试本节代码 
下面的Java程序对字符串拼接和入池进行了测试。 

```java
package jvmgo.book.ch09; 
public class StringTest {
public static void main(String[] args) { 
String s1 = "abc1"; 
String s2 = "abc1"; 
System.out.println(s1 == s2); // true 
int x = 1; 
String s3 = "abc" + x; 
System.out.println(s1 == s3); // false 
s3 = s3.intern(); 
System.out.println(s1 == s3); // true 
} 
}
```
重新编译本章代码，然后测试StringTest程序，结果如图9-3所示。 
![9-3](/img/9-3.png)    
图9-3 StringTest程序执行结果
#### 9.5 Object.hashCode（）、equals（）和toString（） 
Object类有3个非常重要的方法：hashCode（）返回对象的哈希码；equals（）用来比较两个对象是否“相同”；toString（）返回对象的字符串表示。hashCode（）是个本地方法，equals（）和toString（）则是用Java写的，它们的代码如下：
```java
package java.lang; 
public class Object { 
... // 其他代码省略 
public native int hashCode(); 
public boolean equals(Object obj) { 
return (this == obj); 
}
public String toString() { 
return getClass().getName() + "@" + Integer.toHexString(hashCode()); 
} 
}
```
下面实现hashCode（）方法。打开ch09\native\java\lang\Object.go，导入unsafe包并注册hashCode（）方法，代码如下：
```go
package lang 
import "unsafe" 
import "jvmgo/ch09/native" 
import "jvmgo/ch09/rtda" 
func init() { 
native.Register("java/lang/Object", "getClass", "()Ljava/lang/Class;", getClass) 
native.Register("java/lang/Object", "hashCode", "()I", hashCode) 
}
```
继续编辑Object.go，实现hashCode（）方法，代码如下：
```go
// public native int hashCode(); 
func hashCode(frame *rtda.Frame) { 
this := frame.LocalVars().GetThis() 
hash := int32(uintptr(unsafe.Pointer(this))) 
frame.OperandStack().PushInt(hash) 
}
```
把对象引用（Object结构体指针）转换成uintptr类型，然后强制转换成int32推入操作数栈顶。

本节只实现这一个本地方法。重新编译本章代码，然后测试下面的Java程序： 
```java
package jvmgo.book.ch09; 
public class ObjectTest { 
public static void main(String[] args) { 
Object obj1 = new ObjectTest(); 
Object obj2 = new ObjectTest(); 
System.out.println(obj1.hashCode()); 
System.out.println(obj1.toString()); 
System.out.println(obj1.equals(obj2)); 
System.out.println(obj1.equals(obj1)); 
} 
}
```
ObjectTest程序执行结果如图9-4所示。
![9-4](/img/9-4.png)   
图9-4 ObjectTest程序执行结果
#### 9.6 Object.clone（） 
Object类提供了clone（）方法，用来支持对象克隆。这也是一个本地方法，代码如下：
```java 
// java.lang.Object 
protected native Object clone() throws CloneNotSupportedException; 
```
本节实现这个方法。在ch09\native\java\lang\Object.go文件中注册clone（）方法，代码如下：
```go
func init() { 
    native.Register(jlObject, "getClass", "()Ljava/lang/Class;", getClass) 
    native.Register(jlObject, "hashCode", "()I", hashCode) 
    native.Register(jlObject, "clone", "()Ljava/lang/Object;", clone) 
}

继续编辑Object.go，实现clone（）方法，代码如下：
```go
func clone(frame *rtda.Frame) { 
    this := frame.LocalVars().GetThis() 
    cloneable := this.Class().Loader().LoadClass("java/lang/Cloneable") 
    if !this.Class().IsImplements(cloneable) { 
        panic("java.lang.CloneNotSupportedException") 
    }
    frame.OperandStack().PushRef(this.Clone()) 
} 
如果类没有实现Cloneable接口，则抛出CloneNotSupportedException异常，否则调用Object结构体的Clone（）方法克隆对象，然后把对象副本引用推入操作数栈顶。Clone（）实现稍微有些长，把它放在ch09\rtda\heap\object_clone.go文件中，代码如下：
```go
func (self *Object) Clone() *Object { 
return &Object{ 
class: self.class, 
data: self.cloneData(), 
} 
}
```
数据克隆逻辑在cloneData（）函数中，代码如下：
```go
func (self *Object) cloneData() interface{} { 
switch self.data.(type) { 
case []int8: ... 
case []int16: ... 
case []uint16: ... 
case []int32: ... 
case []int64: ... 
case []float32: ... 
case []float64: ... 
case []*Object: 
elements := self.data.([]*Object) 
elements2 := make([]*Object, len(elements)) 
copy(elements2, elements) 
return elements2 
default: // []Slot 
slots := self.data.(Slots) 
slots2 := newSlots(uint(len(slots))) 
copy(slots2, slots) 
return slots2 
} 
} 
```
注意，数组也实现了Cloneable接口，所以上面代码中的case语句针对各种数组进行处理。因为代码都大同小异，所以只给出了引用数组的case语句。default语句对普通对象进行克隆。 
重新编译本章代码，测试下面的Java程序。 
```java
package jvmgo.book.ch09; 
public class CloneTest implements Cloneable { 
private double pi = 3.14; 
@Override 
public CloneTest clone() { 
try {
return (CloneTest) super.clone(); 
} catch (CloneNotSupportedException e) { 
throw new RuntimeException(e); 
} 
}
public static void main(String[] args) { 
CloneTest obj1 = new CloneTest(); 
CloneTest obj2 = obj1.clone(); 
obj1.pi = 3.1415926; 
System.out.println(obj1.pi); 
System.out.println(obj2.pi); 
} 
}
```
CloneTest程序执行结果如图9-5所示。 
![9-5](/img/9-5.png)  
图9-5 CloneTest程序执行结果
#### 9.7 自动装箱和拆箱 
前面讨论过，为了更好地融入Java的对象系统，每种基本类型都有一个包装类与之对应。从Java 5开始，Java语法增加了自动装箱和拆箱（autoboxing/unboxing）能力，可以在必要时把基本类型转换成包装类型或者反之。这个增强完全是由编译器完成的，Java虚拟机没有做任何调整。 
以int类型为例，它的包装类是java.lang.Integer。它提供了2个方法来帮助编译器在int变量和Integer对象之间转换：静态方法value（）把int变量包装成Integer对象；实例方法intValue（）返回被包装的int变量。这两个方法的代码如下：
```java
package java.lang; 
public final class Integer extends Number implements Comparable<Integer> { 
    ... // 其他代码省略 
    private final int value; 
    public static Integer valueOf(int i) { 
        if (i >= IntegerCache.low && i <= IntegerCache.high) 
            return IntegerCache.cache[i + (-IntegerCache.low)]; 
        return new Integer(i); 
    }
    public int intValue() { 
        return value; 
    } 
}
```
由上面的代码可知，Integer.valueOf（）方法并不是每次都创建Integer（）对象，而是维护了一个缓存池IntegerCache。对于比较小（默认是–128~127）的int变量，在IntegerCache初始化时就预先加载到了池中，需要用时直接从池里取即可。IntegerCache是Integer类的内部类，为了便于参考，下面给出它的完整代码。 
```go
private static class IntegerCache { 
    static final int low = -128; 
    static final int high; 
    static final Integer cache[]; 
    static { 
        int h = 127; // high value may be configured by property 
        String integerCacheHighPropValue = 
        sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high"); 
        if (integerCacheHighPropValue != null) { 
            try {
                int i = parseInt(integerCacheHighPropValue); 
                i = Math.max(i, 127); 
                // Maximum array size is Integer.MAX_VALUE 
                h = Math.min(i, Integer.MAX_VALUE - (-low) -1); 
            } catch( NumberFormatException nfe) { 
                // If the property cannot be parsed into an int, ignore it. 
            } 
        }
        high = h; 
        cache = new Integer[(high - low) + 1]; 
        int j = low; 
        for(int k = 0; k < cache.length; k++) 
            cache[k] = new Integer(j++); 
        // range [-128, 127] must be interned (JLS7 5.1.7) 
        assert IntegerCache.high >= 127; 
    }
    private IntegerCache() {} 
    }
```
具体细节就不解释了，需要说明的是IntegerCache在初始化时需要确定缓存池中Inte -ger对象的上限值，为此它调用了sun.misc.VM类的getSavedProperty（）方法。要想让VM正确初始化需要做很多工作，这个工作推迟到第11章进行。这里先用一个hack让VM.get -SavedProperty（）方法返回非null值，以便IntegerCache可以正常初始化。 
在ch09\native目录下创建sun\misc子目录，在其中创建VM.go文件，然后在VM.go文件中注册initialize（）方法，代码如下：
```go
package misc 
import "jvmgo/ch09/instructions/base" 
import "jvmgo/ch09/native" 
import "jvmgo/ch09/rtda" 
import "jvmgo/ch09/rtda/heap" 
func init() { 
    native.Register("sun/misc/VM", "initialize", "()V", initialize) 
}
```
initialize（）方法的实现如下： 
```go
// private static native void initialize(); 
func initialize(frame *rtda.Frame) { 
    vmClass := frame.Method().Class() 
    savedProps := vmClass.GetRefVar("savedProps", "Ljava/util/Properties;") 
    key := heap.JString(vmClass.Loader(), "foo") 
    val := heap.JString(vmClass.Loader(), "bar") 
    frame.OperandStack().PushRef(savedProps) 
    frame.OperandStack().PushRef(key) 
    frame.OperandStack().PushRef(val) 
    propsClass := vmClass.Loader().LoadClass("java/util/Properties") 
    setPropMethod := propsClass.GetInstanceMethod("setProperty", "(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/Object;") 
    base.InvokeMethod(frame, setPropMethod) 
}
```
上面的hack代码可能有些不好理解，但翻译成等价的Java代码后只有一句话： 
```go
private static native void initialize() { 
    VM.savedProps.setProperty("foo", "bar") 
}
```
最后，需要修改invokenative.go，在其中导入misc包，改动如下：
```go
package reserved 
import "jvmgo/ch09/instructions/base" 
import "jvmgo/ch09/rtda" 
import "jvmgo/ch09/native" 
import _ "jvmgo/ch09/native/java/lang" 
import _ "jvmgo/ch09/native/sun/misc" 
```
重新编译本章代码，然后测试下面的Java程序：
```java
package jvmgo.book.ch09; 
import java.util.ArrayList; 
import java.util.List; 
public class BoxTest { 
    public static void main(String[] args) { 
        List<Integer> list = new ArrayList<>(); 
        list.add(1); 
        list.add(2); 
        list.add(3); 
        System.out.println(list.toString()); 
        for (int x : list) { 
            System.out.println(x); 
        } 
    } 
}
```
BoxTest程序的执行结果如图9-6所示。
![9-6](/img/9-6.png) 
图9-6 BoxTest程序执行结果
#### 9.8 本章小结 
本章主要讨论了本地方法调用，以及Java类库中一些最基本的类。前几章基本上都是围绕Java虚拟机本身如何工作而展开讨论。通过本章的学习，读者应该对Java虚拟机和Java类库如何配合工作有了初步的了解。下一章将讨论异常处理。

