第4章 运行时数据区 
----
第1章编写了命令行工具，第2章和第3章讨论了如何搜索和解析class文件。读者也许有些着急了，为什么读到第4章后连Java虚拟机的影子都还没有看到？别着急，本章就来讨论并初步实现运行时数据区（run-time data area），为下一章编写字节码解释器做准备。 

在开始阅读本章之前，还是先准备好目录结构。复制ch03目录，改名为ch04。修改main.go等源文件，把import语句中的ch03全都改成ch04，然后在ch04目录下创建rtda [1] 子目录。现在我们的目录结构应该如下所示：
```
D:\go\workspace\src 
  |-jvmgo 
  |-ch01 ~ ch03 
  |-ch04 
    |-classfile 
    |-classpath 
    |-rtda 
    |-cmd.go 
    |-main.go 
```

[1]:  rtda是run-time data area的首字母缩写。
#### 4.1 运行时数据区概述 
在运行Java程序时，Java虚拟机需要使用内存来存放各式各样的数据。Java虚拟机规范把这些内存区域叫作运行时数据区。运行时数据区可以分为两类：一类是多线程共享的，另一类则是线程私有的。多线程共享的运行时数据区需要在Java虚拟机启动时创建好，在Java虚拟机退出时销毁。线程私有的运行时数据区则在创建线程时才创建，线程退出时销毁。

多线程共享的内存区域主要存放两类数据：类数据和类实例（也就是对象）。对象数据存放在堆（Heap）中，类数据存放在方法区（Method Area）中。堆由垃圾收集器定期清理，所以程序员不需要关心对象空间的释放。类数据包括字段和方法信息、方法的字节码、运行时常量池，等等。从逻辑上来讲，方法区其实也是堆的一部分。 

线程私有的运行时数据区用于辅助执行Java字节码。每个线程都有自己的pc寄存器（Program Counter）和Java虚拟机栈（JVMStack）。Java虚拟机栈又由栈帧（Stack Frame，后面简称帧）构成，帧中保存方法执行的状态，包括局部变量表（Local Variable）和操作数栈（Operand Stack）等。在任一时刻，某一线程肯定是在执行某个方法。这个方法叫作该线程的当前方法；执行该方法的帧叫作线程的当前帧；声明该方法的类叫作当前类。如果当前方法是Java方法，则pc寄存器中存放当前正在执行的Java虚拟机指令的地址，否则，当前方法是本地方法，pc寄存器中的值没有明确定义。

根据以上描述，可以大致勾勒出运行时数据区的逻辑结构，如图4-1所示。 
![4-1](/img/4-1.png)
图4-1 运行时数据区示意图

Java虚拟机规范对于运行时数据区的规定是相当宽松的。以堆为例：堆可以是连续空间，也可以不连续。堆的大小可以固定，也可以在运行时按需扩展 [1] 。虚拟机实现者可以使用任何垃圾回收算法管理堆，甚至完全不进行垃圾收集也是可以的。由于Go本身也有 

垃圾回收功能，所以可以直接使用Go的堆和垃圾收集器，这大大简化了我们的工作。 

本章将初步实现线程私有的运行时数据区，为第5章介绍指令集打下基础。方法区和运行时常量池将在第6章详细介绍。 

[1] java命令提供了-Xms和-Xmx两个非标准选项，用来调整堆的初始大小和最大大小。java命令的详细介绍请参考第1章内容。
#### 4.2 数据类型 

Java虚拟机可以操作两类数据：基本类型（primitive type）和引用类型（reference type）。基本类型的变量存放的就是数据本身，引用类型的变量存放的是对象引用，真正的对象数据是在堆里分配的。这里所说的变量包括类变量（静态字段）、实例变量（非静态字段）、数组元素、方法的参数和局部变量，等等。 

基本类型可以进一步分为布尔类型（boolean type）和数字类型（numeric type） [1] ，数字类型又可以分为整数类型（integral type）和浮点数类型（floating-point type）。引用类型可以进一步分为3种：类类型、接口类型和数组类型。类类型引用指向类实例，数组类型引用指向数组实例，接口类型引用指向实现了该接口的类或数组实例。引用类型有一个特殊的值——null，表示该引用不指向任何对象。 

Go语言提供了非常丰富的数据类型，包括各种整数和两种精度的浮点数。Java和Go的浮点数都采用IEEE 754规范 [2] 。对于基本类型，可以直接在Go和Java之间建立映射关系。对于引用类型，自然的选择是使用指针。Go提供了nil，表示空指针，正好可以用来表示null。由于要到第6章才开始实现类和对象，所以本章先定义一个临时的结构体，用它来表示对象。在ch04\rtda目录下创建object.go，在其中定义Object结构体，代码如下：
```go
package rtda 
type Object struct { 
    // todo 
}
```
 表4-1对Java虚拟机支持的类型进行了总结。
 
表4-1 Java虚拟机数据类型 

[1]: 还有一种基本类型是returnAddress，它和jsr、ret、ret_w指令一起，用来实现finally子句。不过从Java 6开始，Oracle的Java编译器已经不再使用这三条指令了。详细情况请参考第10章内容。 
[2]: 本书不讨论浮点数细节，如在内存中的编码形式等。如果读者需要了解这方面的知识，可以阅读Java虚拟机规范或IEEE 754规范的相关章节。
#### 4.3 实现运行时数据区 
前面两节介绍了一些必要的理论，并且定义了Object结构体。本节将实现线程私有的运行时数据区。下面先从线程开始。
##### 4.3.1 线程 
在ch04\jvm\rtda目录下创建thread.go文件，在其中定义Thread结构体，代码如下： 
```go
package rtda 
type Thread struct { 
    pc int 
    stack *Stack 
}
func NewThread() *Thread {...} 
func (self *Thread) PC() int { return self.pc } // getter 
func (self *Thread) SetPC(pc int) { self.pc = pc } // setter 
func (self *Thread) PushFrame(frame *Frame) {...} 
func (self *Thread) PopFrame() *Frame {...} 
func (self *Thread) CurrentFrame() *Frame {...} 
```
目前只定义了pc和stack两个字段。pc字段无需解释，stack字段是Stack结构体（Java虚拟机栈）指针。Stack结构体在4.3.2节介绍。 

和堆一样，Java虚拟机规范对Java虚拟机栈的约束也相当宽松。Java虚拟机栈可以是连续的空间，也可以不连续；可以是固定大小，也可以在运行时动态扩展 [1] 。如果Java虚拟机栈有大小限制，且执行线程所需的栈空间超出了这个限制，会导致StackOverflowError异常抛出。如果Java虚拟机栈可以动态扩展，但是内存已经耗尽，会导致OutOfMemoryError异常抛出。NewThread（）函数创建Thread实例的代码如下： 
```go
func NewThread() *Thread { 
    return &Thread{ 
        stack: newStack(1024), 
    } 
}
```
newStack（）函数创建Stack结构体实例，它的参数表示要创建的Stack最多可以容纳多少帧，4.3.2节将给出这个函数的代码。这里暂时将它赋值为1024，感兴趣的读者可以修改我们的命令行工具，添加选项来指定这个参数。 

PushFrame（）和PopFrame（）方法只是调用Stack结构体的相应方法而已，代码如下：
```go 
func (self *Thread) PushFrame(frame *Frame) { 
    self.stack.push(frame) 
}
func (self *Thread) PopFrame() *Frame { 
    return self.stack.pop() 
} 
```
CurrentFrame（）方法返回当前帧，代码如下：
```go
func (self *Thread) CurrentFrame() *Frame { 
    return self.stack.top() 
}
```
[1]: java命令提供了-Xss选项来设置Java虚拟机栈大小。
##### 4.3.2 Java虚拟机栈 
如前所述，Java虚拟机规范对Java虚拟机栈的约束非常宽松。我们用经典的链表（linked list）数据结构来实现Java虚拟机栈，这样栈就可以按需使用内存空间，而且弹出的帧也可以及时被Go的垃圾收集器回收。在ch04\jvm\rtda目录下创建jvm_stack.go文件，在其中定义Stack结构体，代码如下： 
```go
package rtda 
type Stack struct { 
    maxSize uint 
    size uint 
    _top *Frame 
}
func newStack(maxSize uint) *Stack {...} 
func (self *Stack) push(frame *Frame) {...} 
func (self *Stack) pop() *Frame {...} 
func (self *Stack) top() *Frame {...} 
```
maxSize字段保存栈的容量（最多可以容纳多少帧），size字段保存栈的当前大小，_top字段保存栈顶指针。newStack（）函数的代码如下：
```go
func newStack(maxSize uint) *Stack { 
    return &Stack{ 
        maxSize: maxSize, 
    } 
}
```
push（）方法把帧推入栈顶，代码如下：
```go
func (self *Stack) push(frame *Frame) { 
    if self.size >= self.maxSize { 
        panic("java.lang.StackOverflowError") 
    } 
    if self._top != nil { 
        frame.lower = self._top 
    } 
    self._top = frame 
    self.size++ 
} 
```
如果栈已经满了，按照Java虚拟机规范，应该抛出StackOverflowError异常。在第10章才会讨论异常，这里先调用panic（）函数终止程序执行。pop（）方法把栈顶帧弹出，代码如下： 
```go
func (self *Stack) pop() *Frame { 
    if self._top == nil { 
        panic("jvm stack is empty!") 
    }
    top := self._top 
    self._top = top.lower 
    top.lower = nil 
    self.size-- 
    return top 
}
``` 
如果此时栈是空的，肯定是我们的虚拟机有bug，调用panic（）函数终止程序执行即可。top（）方法只是返回栈顶帧，但并不弹出，代码如下： 
```go
func (self *Stack) top() *Frame {
    if self._top == nil { 
        panic("jvm stack is empty!") 
    }
    return self._top 
}
```
##### 4.3.3 帧
在ch04\jvm\rtda目录下创建frame.go文件，在其中定义Frame结构体，代码如下：
```go 
package rtda 
type Frame struct {
    lower *Frame 
    localVars LocalVars 
    operandStack *OperandStack 
}
func newFrame(maxLocals, maxStack uint) *Frame {...} 
```
Frame结构体暂时也比较简单，只有三个字段，后续章节还会继续完善它。lower字段用来实现链表数据结构，localVars字段保存局部变量表指针，operandStack字段保存操作数栈指针。NewFrame（）函数创建Frame实例，代码如下： 
```go
func NewFrame(maxLocals, maxStack uint) *Frame { 
    return &Frame{ 
        localVars: newLocalVars(maxLocals), 
        operandStack: newOperandStack(maxStack), 
    } 
} 
```
执行方法所需的局部变量表大小和操作数栈深度是由编译器预先计算好的，存储在class文件method_info结构的Code属性中，具体可以参考3.4.5节。Thread、Stack和Frame结构体的代码都已经给出了，根据代码，可以画出Java虚拟机栈的链表结构，如图4-2所示。 
![4-2](/img/4-2.png)
图4-2 使用链表实现Java虚拟机栈
##### 4.3.4 局部变量表 
局部变量表是按索引访问的，所以很自然，可以把它想象成一个数组。根据Java虚拟机规范，这个数组的每个元素至少可以容纳一个int或引用值，两个连续的元素可以容纳一个long或double值。那么使用哪种Go语言数据类型来表示这个数组呢？最容易想到的是[]int。Go的int类型因平台而异，在64位系统上是int64，在32位系统上是int32，总之足够容纳Java的int类型。另外它和内置的uintptr类型宽度一样，所以也足够放下一个内存地址。通过unsafe包可以拿到结构体实例的地址，如下所示： 
```go
obj := &Object{} 
ptr := uintptr(unsafe.Pointer(obj)) 
ref := int(ptr) 
```
但遗憾的是，Go的垃圾回收机制并不能有效处理uintptr指针。也就是说，如果一个结构体实例，除了uintptr类型指针保存它的地址之外，其他地方都没有引用这个实例，它就会被当作垃圾回收。
另外一个方案是用[]interface{}类型，这个方案在实现上没有问题，只是写出来的代码可读性太差。第三种方案是定义一个结构体，让它可以同时容纳一个int值和一个引用值。这里将使用第三种方案。在ch04\rtda目录下创建slot.go文件，在其中定义Slot结构体，代码如下： 
```go
package rtda 
type Slot struct { 
    num int32 
    ref *Object 
}
``` 
num字段存放整数，ref字段存放引用，刚好满足我们的需求。下面用它来实现局部变量表。在ch04\rtda目录下创建local_vars.go文件，在其中定义LocalVars类型，代码如下：
```go
package rtda 
import "math" 
type LocalVars []Slot 
```
继续编辑local_vars.go文件，在其中定义newLocalVars（）函数，代码如下： 
```go
func newLocalVars(maxLocals uint) LocalVars { 
    if maxLocals > 0 { 
        return make([]Slot, maxLocals) 
    }
    return nil 
} 
```
newLocalVars（）函数创建LocalVars实例，代码比较简单，这里就不多解释了。 
在第5章大家会看到，操作局部变量表和操作数栈的指令都是隐含类型信息的。下面给LocalVars类型定义一些方法，用来存取不同类型的变量。int变量最简单，直接存取即可。
```go
func (self LocalVars) SetInt(index uint, val int32) { 
    self[index].num = val 
}
func (self LocalVars) GetInt(index uint) int32 { 
    return self[index].num 
}
```
float变量可以先转成int类型，然后按int变量来处理。
```go
func (self LocalVars) SetFloat(index uint, val float32) { 
    bits := math.Float32bits(val) 
    self[index].num = int32(bits) 
}
func (self LocalVars) GetFloat(index uint) float32 { 
    bits := uint32(self[index].num) 
    return math.Float32frombits(bits) 
}
```
long变量则需要拆成两个int变量。 
```go
func (self LocalVars) SetLong(index uint, val int64) { 
    self[index].num = int32(val) 
    self[index+1].num = int32(val >> 32) 
}
func (self LocalVars) GetLong(index uint) int64 { 
    low := uint32(self[index].num)
    high := uint32(self[index+1].num) 
    return int64(high)<<32 | int64(low) 
}
```
double变量可以先转成long类型，然后按照long变量来处理。 
```go
func (self LocalVars) SetDouble(index uint, val float64) { 
    bits := math.Float64bits(val) 
    self.SetLong(index, int64(bits)) 
}
func (self LocalVars) GetDouble(index uint) float64 { 
    bits := uint64(self.GetLong(index)) 
    return math.Float64frombits(bits) 
}
```
最后是引用值，也比较简单，直接存取即可。
```go
func (self LocalVars) SetRef(index uint, ref *Object) { 
    self[index].ref = ref 
}
func (self LocalVars) GetRef(index uint) *Object { 
    return self[index].ref 
}
```
请读者注意，我们并没有真的对boolean、byte、short和char类型定义存取方法，这些类型的值都可以转换成int值类来处理。下面我们来实现操作数栈。
##### 4.3.5 操作数栈 
操作数栈的实现方式和局部变量表类似。在ch04\rtda目录下创建operand_stack.go文件，在其中定义OperandStack结构体，代码如下：
```go
package rtda 
import "math" 
type OperandStack struct { 
    size uint 
    slots []Slot 
}
```
操作数栈的大小是编译器已经确定的，所以可以用[]Slot实现。size字段用于记录栈顶位置。继续编辑operand_stack.go，在其中实现newOperandStack（）函数，代码如下： 
```go
func newOperandStack(maxStack uint) *OperandStack { 
    if maxStack > 0 { 
        return &OperandStack{ 
            slots: make([]Slot, maxStack), 
        } 
    }
    return nil 
}
```
代码也比较简单，在此就不多解释了。和局部变量表类似，需要定义一些方法从操作数栈中弹出，或者往其中推入各种类型的变量。先看最简单的int变量。
```go
func (self *OperandStack) PushInt(val int32) { 
    self.slots[self.size].num = val 
    self.size++ 
}
func (self *OperandStack) PopInt() int32 { 
    self.size-- 
    return self.slots[self.size].num 
}
```
PushInt（）方法往栈顶放一个int变量，然后把size加1。PopInt（）方法则恰好相反，先把size减1，然后返回变量值。float变量还是先转成int类型，然后按int变量处理。
```go
func (self *OperandStack) PushFloat(val float32) { 
    bits := math.Float32bits(val) 
    self.slots[self.size].num = int32(bits) 
    self.size++ 
}
func (self *OperandStack) PopFloat() float32 { 
    self.size-- 
    bits := uint32(self.slots[self.size].num) 
    return math.Float32frombits(bits) 
}
```go 
把long变量推入栈顶时，要拆成两个int变量。弹出时，先弹出两个int变量，然后组装成一个long变量。
```go
func (self *OperandStack) PushLong(val int64) { 
    self.slots[self.size].num = int32(val) 
    self.slots[self.size+1].num = int32(val >> 32) 
    self.size += 2 
}
func (self *OperandStack) PopLong() int64 {self.size -= 2 
    low := uint32(self.slots[self.size].num) 
    high := uint32(self.slots[self.size+1].num) 
    return int64(high)<<32 | int64(low) 
}
```
double变量先转成long类型，然后按long变量处理。
```go
func (self *OperandStack) PushDouble(val float64) { 
    bits := math.Float64bits(val) 
    self.PushLong(int64(bits)) 
}
func (self *OperandStack) PopDouble() float64 { 
    bits := uint64(self.PopLong()) 
    return math.Float64frombits(bits) 
}
```
最后看引用类型，代码如下： 
```go
func (self *OperandStack) PushRef(ref *Object) { 
    self.slots[self.size].ref = ref 
    self.size++ 
}
func (self *OperandStack) PopRef() *Object { 
    self.size-- 
    ref := self.slots[self.size].ref 
    self.slots[self.size].ref = nil 
    return ref 
}
```
PushRef（）方法比较简单，此处不做太多解释。PopRef（）方法需要说明一点：弹出引用后，把Slot结构体的ref字段设置成nil，这样做是为了帮助Go的垃圾收集器回收Object结构体实例。
至此，局部变量表和操作数栈都准备好了。仅通过代码来理解它们可能不是很直观，下面我们将通过一个具体的例子来分析局部变量表和操作数的使用。
##### 4.3.6 局部变量表和操作数栈实例分析 
以圆形的周长公式为例进行分析，下面是Java方法的代码。 
```go
public static float circumference(float r) { 
    float pi = 3.14f; 
    float area = 2 * pi * r; 
    return area; 
}
```
上面的方法会被javac编译器编译成如下字节码：
```
00 ldc #4 
02 fstore_1 
03 fconst_2 
04 fload_1 
05 fmul 
06 fload_0 
07 fmul 
08 fstore_2 
09 fload_2 
10 return
```
下面分析这段字节码的执行。circumference（）方法的局部变量表大小是3，操作数栈深度是2。假设调用方法时，传递给它的参数是1.6f，方法开始执行前，帧的状态如图4-3所示。
![4-3](/img/4-3.png)    
图4-3 circumference（）方法执行示意图（1） 

第一条指令是ldc，它把3.14f推入栈顶，如图4-4所示。
![4-4](/img/4-4.png)  
图4-4 circumference（）方法执行示意图（2） 

注意，上面是局部变量表和操作数栈过去的状态，最下面是当前状态。接着是fstore_1指令，它把栈顶的3.14f弹出，放到#1号局部变量中，如图4-5所示。 

fconst_2指令把2.0f推到栈顶，如图4-6所示。 
![4-5](/img/4-5.png) 
图4-5 circumference（）方法执行示意图（3）
![4-6](/img/4-6.png)     
图4-6 circumference（）方法执行示意图（4） 

fload_1指令把#1号局部变量推入栈顶，如图4-7所示。 
![4-7](/img/4-7.png)   
图4-7 circumference（）方法执行示意图（5） 

fmul指令执行浮点数乘法。它把栈顶的两个浮点数弹出，相乘，然后把结果推入栈顶，如图4-8所示。 

fload_0指令把#0号局部变量推入栈顶，如图4-9所示。

fmul继续乘法计算，如图4-10所示。 

fstore_2指令把操作数栈顶的float值弹出，放入#2号局部变量表，如图4-11所示。 
![4-8](/img/4-8.png) 
图4-8 circumference（）方法执行示意图（6） 
![4-9](/img/4-9.png) 
图4-9 circumference（）方法执行示意图（7）
![4-10](/img/4-10.png)   
图4-10 circumference（）方法执行示意图（8） 
![4-11](/img/4-11.png)    
图4-11 circumference（）方法执行示意图（9） 

fload_2指令把#2号局部变量推入操作数栈顶，如图4-12所示。
![4-12](/img/4-12.png)      
图4-12 circumference（）方法执行示意图（10） 

最后freturn指令把操作数栈顶的float变量弹出，返回给方法调用者，如图4-13所示。 

如果上面的例子理解起来有点困难，请不要担心。这里重点看局部变量表和操作数栈的用法就可以了，后面的章节会详细介绍Java虚拟机指令集。
![4-13](/img/4-13.png)      
图4-13 circumference（）方法执行示意图（11）
#### 4.4 测试本章代码 
本节简单测试局部变量表和操作数栈的用法，在下一章它们才会真正派上用场。打开ch04\main.go文件，修改import语句，代码如下： 
```go
package main 
import "fmt" 
import "jvmgo/ch04/rtda" 
func main() {...} 
func startJVM(cmd *Cmd) {...} 
```
main（）方法不变，修改startJVM方法，代码如下： 
```go
func startJVM(cmd *Cmd) { 
    frame := rtda.NewFrame(100, 100) 
    testLocalVars(frame.LocalVars()) 
    testOperandStack(frame.OperandStack()) 
}
``` 
testLocalVars（）函数测试局部变量，代码如下： 
```go
func testLocalVars(vars rtda.LocalVars) { 
    vars.SetInt(0, 100) 
    vars.SetInt(1, -100) 
    vars.SetLong(2, 2997924580) 
    vars.SetLong(4, -2997924580) 
    vars.SetFloat(6, 3.1415926) 
    vars.SetDouble(7, 2.71828182845) 
    vars.SetRef(9, nil) 
    println(vars.GetInt(0)) 
    println(vars.GetInt(1)) 
    println(vars.GetLong(2)) 
    println(vars.GetLong(4))println(vars.GetFloat(6)) 
    println(vars.GetDouble(7)) 
    println(vars.GetRef(9)) 
}
``` 
testOperandStack（）函数测试操作数栈，代码如下： 
```go
func testOperandStack(ops *rtda.OperandStack) { 
    ops.PushInt(100) 
    ops.PushInt(-100) 
    ops.PushLong(2997924580) 
    ops.PushLong(-2997924580) 
    ops.PushFloat(3.1415926) 
    ops.PushDouble(2.71828182845) 
    ops.PushRef(nil) 
    println(ops.PopRef()) 
    println(ops.PopDouble()) 
    println(ops.PopFloat()) 
    println(ops.PopLong()) 
    println(ops.PopLong()) 
    println(ops.PopInt()) 
    println(ops.PopInt()) 
}
```
打开命令行窗口，执行下面的命令编译本章代码：
```shell script
go install jvmgo\ch04 
```
编译成功后，在D：\go\workspace\bin目录下出现ch04.exe文件。执行ch04.exe，运行结果如图4-14所示。
![4-14](/img/4-14.png)     
图4-14 ch04.exe的运行结果
#### 4.5 本章小结 
本章介绍了运行时数据区，初步实现了Thread、Stack、Frame、OperandStack和LocalVars等线程私有的运行时数据区。下一章将实现字节码解释器，到时候方法就可以在我们的Java虚拟机里运行了。
