第7章 方法调用和返回 
---
第4章实现了Java虚拟机栈、帧等运行时数据区，为方法的执行打好了基础。第5章实现了一个简单的解释器和150多条指令，已经可以执行单个方法。第6章实现了方法区，为方法调用扫清了障碍。本章将实现方法调用和返回，在此基础上，还会讨论类和对象的初始化。

开始本章之前，还是先把目录结构准备好。复制ch06目录，改名为ch07。修改main.go等文件，把import语句中的ch06全都替换成ch07。本章对目录结构没有太大的调整。
#### 7.1 方法调用概述 
从调用的角度来看，方法可以分为两类：静态方法（或者类方法）和实例方法。静态方法通过类来调用，实例方法则通过对象引用来调用。静态方法是静态绑定的，也就是说，最终调用的是哪个方法在编译期就已经确定。实例方法则支持动态绑定，最终要调用哪个方法可能要推迟到运行期才能知道，本章将详细讨论这一点。 

从实现的角度来看，方法可以分为三类：没有实现（也就是抽象方法）、用Java语言（或者JVM上的其他语言，如Groovy和Scala等）实现和用本地语言（如C或者C++）实现。静态方法和抽象方法是互斥的。在Java 8之前，接口只能包含抽象方法。为了实现Lambda表达式，Java 8放宽了这一限制，在接口中也可以定义静态方法和默认方法。本章不考虑接口的静态方法和默认方法，感兴趣的读者请阅读Java虚拟机规范相关章节。在本书中，我们把Java等语言实现的方法叫作Java方法。本章只讨论Java方法的调用，本地方法调用将在第9章中介绍。 

在Java 7之前，Java虚拟机规范一共提供了4条方法调用指令。其中invokestatic指令用来调用静态方法。invokespecial指令用来调用无须动态绑定的实例方法，包括构造函数、私有方法和通过super关键字调用的超类方法。剩下的情况则属于动态绑定。如果是针对接口类型的引用调用方法，就使用invokeinterface指令，否则使用invokevirtual指令。本章将实现这4条指令。 

为了更好地支持动态类型语言，Java 7增加了一条方法调用指令invokedynamic。本章不讨论这条指令，感兴趣的读者请阅读Java虚拟机规范相关章节。在深入讨论各条方法调用指令的细节之前，先简单了解Java虚拟机是如何调用方法的。 

首先，方法调用指令需要n+1个操作数，其中第1个操作数是uint16索引，在字节码中紧跟在指令操作码的后面。通过这个索引，可以从当前类的运行时常量池中找到一个方法符号引用，解析这个符号引用就可以得到一个方法。注意，这个方法并不一定就是最终要调用的那个方法，所以可能还需要一个查找过程才能找到最终要调用的方法。剩下的n个操作数是要传递给被调用方法的参数，从操作数栈中弹出。将在7.2小节讨论方法符号引用的解析。 

如果要执行的是Java方法（而非本地方法），下一步是给这个方法创建一个新的帧，并把它推到Java虚拟机栈顶。传递参数之后，新的方法就可以开始执行了。将在7.3小节讨论参数传递，本地方法调用则推迟到第9章再讨论。下面是一段伪代码，用于说明Java方法的调用过程。 
```go
func (self *INVOKE_XXX) Execute(frame *rtda.Frame) { 
cp := frame.Method().Class().ConstantPool() 
methodRef := cp.GetConstant(self.Index).(*heap.MethodRef) 
resolved := resolveMethodRef(methodRef) 
checkResolvedMethod(resolved) 
toBeInvoked := findMethodToInvoke(methodRef) 
newFrame := frame.Thread().NewFrame(toBeInvoked) 
frame.Thread().PushFrame(newFrame) 
passArgs(frame, newFrame) 
} 
```
方法的最后一条指令是某个返回指令，这个指令负责把方法的返回值推入前一帧的操作数栈顶，然后把当前帧从Java虚拟机栈中弹出。将在7.4小节讨论返回指令，在7.5小节讨论方法调用指令。
#### 7.2 解析方法符号引用 
非接口方法符号引用和接口方法符号引用的解析规则是不同的，因此本章分开讨论这两种符号引用。Java虚拟机规范的5.4.3.3节和5.4.3.4节详细描述了这两种符号引用的解析规则。由于本书不讨论接口的静态方法和默认方法，所以在本小节中，主要参考Java虚拟机规范第7版编写代码，请读者注意这一点。
##### 7.2.1 非接口方法符号引用 
打开ch07\rtda\heap\cp_methodref.go文件，在其中实现ResolvedMethod（）方法，代码如下：
```go
func (self *MethodRef) ResolvedMethod() *Method { 
if self.method == nil { 
self.resolveMethodRef() 
}
return self.method 
}
```
如果还没有解析过符号引用，调用resolveMethodRef（）方法进行解析，否则直接返回方法指针。resolveMethodRef（）方法的代码如下：
```go
func (self *MethodRef) resolveMethodRef() { 
d := self.cp.class 
c := self.ResolvedClass() 
if c.IsInterface() { 
panic("java.lang.IncompatibleClassChangeError") 
}
method := lookupMethod(c, self.name, self.descriptor) 
if method == nil { 
panic("java.lang.NoSuchMethodError") 
}
if !method.isAccessibleTo(d) { 
panic("java.lang.IllegalAccessError") 
}
self.method = method
}
```
如果类D想通过方法符号引用访问类C的某个方法，先要解析符号引用得到类C。如果C是接口，则抛出IncompatibleClassChangeError异常，否则根据方法名和描述符查找方法。如果找不到对应的方法，则抛出NoSuchMethodError异常，否则检查类D是否有权限访问该方法。如果没有，则抛出IllegalAccessError异常。isAccessibleTo（）方法是在ClassMember结构体中定义的，在第6章就已经实现了。下面看一下lookupMethod（）函数，其代码如下：
```go
func lookupMethod(class *Class, name, descriptor string) *Method { 
method := LookupMethodInClass(class, name, descriptor) 
if method == nil { 
method = lookupMethodInInterfaces(class.interfaces, name, descriptor) 
}
return method 
}
```
先从C的继承层次中找，如果找不到，就去C的接口中找。LookupMethodInClass（）函数在很多地方都要用到，所以在ch07\rtda\heap\method_lookup.go文件中实现它，代码如下：
```go
func LookupMethodInClass(class *Class, name, descriptor string) *Method { 
for c := class; c != nil; c = c.superClass { 
for _, method := range c.methods { 
if method.name == name && method.descriptor == descriptor { 
return method 
} 
} 
}
return nil
}
```
lookupMethodInInterfaces（）函数也在method_lookup.go文件中，代码如下：
```go
func lookupMethodInInterfaces(ifaces []*Class, name, descriptor string) *Method { 
for _, iface := range ifaces { 
for _, method := range iface.methods { 
if method.name == name && method.descriptor == descriptor { 
return method 
} 
}
method := lookupMethodInInterfaces(iface.interfaces, name, descriptor) 
if method != nil { 
return method 
} 
}
return nil 
}
```
至此，非接口方法符号引用的解析就介绍完了，下面介绍接口方法符号引用如何解析。
##### 7.2.2 接口方法符号引用 
打开ch07\rtda\heap\cp_interface_methodref.go文件，在其中实现ResolvedInterfaceMethod（）方法，代码如下：
```go
func (self *InterfaceMethodRef) ResolvedInterfaceMethod() *Method { 
if self.method == nil { 
self. resolveInterfaceMethodRef() 
}
return self.method 
}
```
上面的代码和ResolvedMethod（）方法大同小异，不多解释。下面来看resolveInterface-MethodRef（）方法，代码如下：
```go
func (self *InterfaceMethodRef) resolveInterfaceMethodRef() { 
d := self.cp.class 
c := self.ResolvedClass() 
if !c.IsInterface() { 
panic("java.lang.IncompatibleClassChangeError") 
}
method := lookupInterfaceMethod(c, self.name, self.descriptor) 
if method == nil { 
panic("java.lang.NoSuchMethodError") 
}
if !method.isAccessibleTo(d) { 
panic("java.lang.IllegalAccessError") 
}
self.method = method 
}
```
上面的代码和resolveMethodRef（）方法也差不多，但有两处差别，已经用粗体显示。下面来看lookupInterfaceMethod（）函数，代码如下：
```go
func lookupInterfaceMethod(iface *Class, name, descriptor string) *Method { 
for _, method := range iface.methods { 
if method.name == name && method.descriptor == descriptor { 
return method 
} 
}
return lookupMethodInInterfaces(iface.interfaces, name, descriptor) 
} 
```
如果能在接口中找到方法，就返回找到的方法，否则调用lookupMethodInInterfaces()函数在超接口中寻找。 
lookupMethodInInterfaces()函数已经在前一小节介绍。至此，接口方法符号引用的解析也介绍完毕了，下面讨论如何给方法传递参数。
#### 7.3 方法调用和参数传递 
在定位到需要调用的方法之后，Java虚拟机要给这个方法创建一个新的帧并把它推入Java虚拟机栈顶，然后传递参数。这个逻辑对于本章要实现的4条方法调用指令来说基本上相同，为了避免重复代码，在单独的文件中实现这个逻辑。在ch07\instructions\base目录下创建method_invoke_logic.go文件，在其中实现InvokeMethod（）函数，代码如下：
```go
func InvokeMethod(invokerFrame *rtda.Frame, method *heap.Method) { 
thread := invokerFrame.Thread() 
newFrame := thread.NewFrame(method) 
thread.PushFrame(newFrame) 
argSlotSlot := int(method.ArgSlotCount()) 
if argSlotSlot > 0 {
for i := argSlotSlot - 1; i >= 0; i-- { 
slot := invokerFrame.OperandStack().PopSlot() 
newFrame.LocalVars().SetSlot(uint(i), slot) 
}
}
}
```
函数的前三行代码创建新的帧并推入Java虚拟机栈，剩下的代码传递参数。重点讨论参数传递。首先，要确定方法的参数在局部变量表中占用多少位置。注意，这个数量并不一定等于从Java代码中看到的参数个数，原因有两个。第一，long和double类型的参数要占用两个位置。第二，对于实例方法，Java编译器会在参数列表的前面添加一个参数，这个隐藏的参数就是this引用。假设实际的参数占据n个位置，依次把这n个变量从调用者的操作数栈中弹出，放进被调用方法的局部变量表中，参数传递就完成了。 
注意，在代码中，并没有对long和double类型做特别处理。因为操作的是Slot结构体，所以这是没问题的。LocalVars结构体的SetSlot（）方法是新增的，代码如下：
```go
func (self LocalVars) SetSlot(index uint, slot Slot) { 
self[index] = slot 
}
```
如果忽略long和double类型参数，则静态方法的参数传递过程如图7-1所示。
![7-1](/img/7-1.png)
图7-1 静态方法参数传递示意图
实例方法的参数传递过程如图7-2所示。 
![7-2](/img/7-2.png) 
图7-2 实例方法参数传递示意图 

那么那个ArgSlotCount（）方法返回了什么呢？打开ch07\rtda\heap\method.go文件，修改Method结构体，给它添加argSlotCount字段，代码如下： 
```go
type Method struct { 
ClassMember 
maxStack uint 
maxLocals uint 
code []byte 
argSlotCount uint 
} 
```


ArgSlotCount（）只是个Getter方法而已，代码如下：func (self *Method) 
```go
ArgSlotCount() uint { 
    return self.argSlotCount 
}
```
newMethods（）方法也需要修改，在其中计算方法的argSlotCount，代码如下：
```go
func newMethods(class *Class, cfMethods []*classfile.MemberInfo) []*Method { 
methods := make([]*Method, len(cfMethods)) 
for i, cfMethod := range cfMethods { 
methods[i] = &Method{} 
methods[i].class = class 
methods[i].copyMemberInfo(cfMethod) 
methods[i].copyAttributes(cfMethod) 
methods[i].calcArgSlotCount() 
}
return methods 
}
```
下面是calcArgSlotCount（）方法的代码。 
```go
func (self *Method) calcArgSlotCount() { 
parsedDescriptor := parseMethodDescriptor(self.descriptor) 
for _, paramType := range parsedDescriptor.parameterTypes { 
self.argSlotCount++ 
if paramType == "J" || paramType == "D" { 
self.argSlotCount++ 
} 
}
if !self.IsStatic() { 
self.argSlotCount++ 
} 
} 
```
parseMethodDescriptor（）函数分解方法描述符，返回一个MethodDescriptor结构体实例。这个结构体定义在ch06\rtda\heap\method_descriptor.go文件中，代码如下：
```go
package heap 
type MethodDescriptor struct { 
parameterTypes []string 
returnType string 
}
```
parseMethodDescriptor（）函数定义ch06\rtda\heap\method_descriptor_parser.go文件中，为了节约篇幅，这里就不详细介绍这个函数了，感兴趣的读者请阅读随书源代码。
#### 7.4 返回指令 
方法执行完毕之后，需要把结果返回给调用方，这一工作由返回指令完成。返回指令属于控制类指令，一共有6条。其中return指令用于没有返回值的情况，areturn、ireturn、lreturn、freturn和dreturn分别用于返回引用、int、long、float和double类型的值。在ch07/instructions/control目录下创建return.go，在其中定义返回指令，代码如下：
```go
package control 
import "jvmgo/ch07/instructions/base" 
import "jvmgo/ch07/rtda" 
type RETURN struct{ base.NoOperandsInstruction } // Return void from method 
type ARETURN struct{ base.NoOperandsInstruction } // Return reference from method 
type DRETURN struct{ base.NoOperandsInstruction } // Return double from method 
type FRETURN struct{ base.NoOperandsInstruction } // Return float from method 
type IRETURN struct{ base.NoOperandsInstruction } // Return int from method 
type LRETURN struct{ base.NoOperandsInstruction } // Return long from method 
```
6条返回指令都不需要操作数。return指令比较简单，只要把当前帧从Java虚拟机栈中弹出即可，它的Execute（）方法如下： 
```go
func (self *RETURN) Execute(frame *rtda.Frame) { 
frame.Thread().PopFrame() 
} 
```
其他5条返回指令的Execute（）方法都非常相似，为了节约篇幅，下面只给出ireturn指令的代码（有差异的部分已经加粗）：
```go
func (self *IRETURN) Execute(frame *rtda.Frame) { 
    thread := frame.Thread() 
    currentFrame := thread.PopFrame() 
    invokerFrame := thread.TopFrame() 
    retVal := currentFrame.OperandStack().PopInt() 
    invokerFrame.OperandStack().PushInt(retVal) 
}
```
Thread结构体的TopFrame（）方法和CurrentFrame（）代码一样，这里用不同的名称主要是为了避免混淆。方法符号引用解析、参数传递、结果返回等都实现了，下面实现方法调用指令。
#### 7.5 方法调用指令 
与7.2节类似，由于本书不考虑接口的静态方法和默认方法，所以要实现的这4条指令并没有完全满足Java虚拟机规范第8版的规定，请读者留意这一点。下面从较简单的invokestatic指令开始。
##### 7.5.1 invokestatic指令 
在ch07\instructions\references目录下创建invokestatic.go文件，在其中定义invokestatic指令，代码如下：
```go
package references 
import "jvmgo/ch07/instructions/base" 
import "jvmgo/ch07/rtda" 
import "jvmgo/ch07/rtda/class" 
// Invoke a class (static) method 
type INVOKE_STATIC struct{ base.Index16Instruction } 

结构体定义比较简单，直接看Execute（）方法，代码如下：
```go
func (self *INVOKE_STATIC) Execute(frame *rtda.Frame) { 
cp := frame.Method().Class().ConstantPool() 
methodRef := cp.GetConstant(self.Index).(*heap.MethodRef) 
resolvedMethod := methodRef.ResolvedMethod() 
if !resolved.IsStatic() { 
panic("java.lang.IncompatibleClassChangeError") 
}
base.InvokeMethod(frame, resolvedMethod) 
} 
```
假定解析符号引用后得到方法M。M必须是静态方法，否则抛出Incompatible -ClassChangeError异常。M不能是类初始化方法。类初始化方法只能由Java虚拟机调用，不能使用invokestatic指令调用。这一规则由class文件验证器保证，这里不做检查。如果声明M的类还没有被初始化，则要先初始化该类。将在7.8小节讨论类的初始化。
对于invokestatic指令，M就是最终要执行的方法，调用InvokeMethod（）函数执行该方法。
##### 7.5.2 invokespecial指令 
invokespecial指令在第6章就定义好了，代码如下：
```go
// Invoke instance method; special handling for superclass, private, 
// and instance initialization method invocations 
type INVOKE_SPECIAL struct{ base.Index16Instruction } 
```
需要修改Execute（）方法，代码稍微有些复杂，先看第一部分：
```go
func (self *INVOKE_SPECIAL) Execute(frame *rtda.Frame) { 
currentClass := frame.Method().Class() 
cp := currentClass.ConstantPool() 
methodRef := cp.GetConstant(self.Index).(*heap.MethodRef) 
resolvedClass := methodRef.ResolvedClass() 
resolvedMethod := methodRef.ResolvedMethod() 
```
先拿到当前类、当前常量池、方法符号引用，然后解析符号引用，拿到解析后的类和方法。继续看代码： 
```go
if resolvedMethod.Name() == "<init>" && resolvedMethod.Class() != resolvedClass { 
panic("java.lang.NoSuchMethodError") 
}
if resolvedMethod.IsStatic() { 
panic("java.lang.IncompatibleClassChangeError") 
}
```
假定从方法符号引用中解析出来的类是C，方法是M。如果M是构造函数，则声明M的类必须是C，否则抛出NoSuchMethodError异常。如果M是静态方法，则抛出Incompatible ClassChangeError异常。继续看代码： 
```go
ref := frame.OperandStack().GetRefFromTop(resolvedMethod.ArgSlotCount()) 
if ref == nil { 
panic("java.lang.NullPointerException") 
}
```
从操作数栈中弹出this引用，如果该引用是null，抛出NullPointerException异常。注意，在传递参数之前，不能破坏操作数栈的状态。给OperandStack结构体添加一个GetRefFromTop（）方法，该方法返回距离操作数栈顶n个单元格的引用变量。比如GetRefFromTop（0）返回操作数栈顶引用，GetRefFromTop（1）返回从栈顶开始的倒数第二个引用，等等。GetRefFromTop（）方法的代码很简单，在后面给出。继续看Execute（）方法： 
```go
if resolvedMethod.IsProtected() && 
resolvedMethod.Class().IsSuperClassOf(currentClass) && 
resolvedMethod.Class().GetPackageName() != currentClass.GetPackageName() && 
ref.Class() != currentClass && 
!ref.Class().IsSubClassOf(currentClass) { 
panic("java.lang.IllegalAccessError") 
} 
```
上面的判断确保protected方法只能被声明该方法的类或子类调用。如果违反这一规定，则抛出IllegalAccessError异常。接着往下看：
```go
methodToBeInvoked := resolvedMethod 
if currentClass.IsSuper() && 
resolvedClass.IsSuperClassOf(currentClass) && 
resolvedMethod.Name() != "<init>" { 
methodToBeInvoked = heap.LookupMethodInClass(currentClass.SuperClass(), 
methodRef.Name(), methodRef.Descriptor()) 
} 
```

上面这段代码比较难懂，把它翻译成更容易理解的语言：如果调用的中超类中的函数，但不是构造函数，且当前类的ACC_SUPER标志被设置，需要一个额外的过程查找最终要调用的方法；否则前面从方法符号引用中解析出来的方法就是要调用的方法。
```go
if methodToBeInvoked == nil || methodToBeInvoked.IsAbstract() { 
panic("java.lang.AbstractMethodError") 
}
base.InvokeMethod(frame, methodtoBeInvoked) 
} 
```
如果查找过程失败，或者找到的方法是抽象的，抛出AbstractMethodError异常。最后，如果一切正常，就调用方法。这里之所以这么复杂，是因为调用超类的（非构造函数）方法需要特别处理。限于篇幅，这里就不深入讨论了，读者可以阅读Java虚拟机规范，了解类的ACC_SUPER访问标志的用法。OperandStack结构体GetRefFromTop（）方法的代码如下：
```go
func (self *OperandStack) GetRefFromTop(n uint) *heap.Object { 
return self.slots[self.size-1-n].ref 
}
```
##### 7.5.3 invokevirtual指令 
invokevirtual指令也已经在第6章定义好了，代码如下：
```go
// Invoke instance method; dispatch based on class 
type INVOKE_VIRTUAL struct{ base.Index16Instruction } 
```
下面修改Execute（）方法，第一部分代码如下：
```go
func (self *INVOKE_VIRTUAL) Execute(frame *rtda.Frame) { 
currentClass := frame.Method().Class() 
cp := currentClass.ConstantPool() 
methodRef := cp.GetConstant(self.Index).(*heap.MethodRef) 
resolvedMethod := methodRef.ResolvedMethod() 
if resolvedMethod.IsStatic() { 
panic("java.lang.IncompatibleClassChangeError") 
}
ref := frame.OperandStack().GetRefFromTop(resolvedMethod.ArgSlotCount() - 1) 
if ref == nil { 
// hack System.out.println() 
panic("java.lang.NullPointerException") 
}
if resolvedMethod.IsProtected() && 
resolvedMethod.Class().IsSuperClassOf(currentClass) && 
resolvedMethod.Class().GetPackageName() != currentClass.GetPackageName() && 
ref.Class() != currentClass && 
!ref.Class().IsSubClassOf(currentClass) { 
panic("java.lang.IllegalAccessError") 
}
```
这部分代码和invokespecial指令基本上差不多，也不多解释了。下面是剩下的代码。 
```go
methodToBeInvoked := heap.LookupMethodInClass(ref.Class(), 
methodRef.Name(), methodRef.Descriptor()) 
if methodToBeInvoked == nil || methodToBeInvoked.IsAbstract() {
panic("java.lang.AbstractMethodError") 
}
base.InvokeMethod(frame, methodToBeInvoked) 
} 
```
从对象的类中查找真正要调用的方法。如果找不到方法，或者找到的是抽象方法，则需要抛出AbstractMethodError异常，否则一切正常，调用方法。注意，仍然要用hack的方式调用System.out.println（）方法，代码如下：
```go
func (self *INVOKE_VIRTUAL) Execute(frame *rtda.Frame) { 
... // 其他代码 
ref := frame.OperandStack().GetRefFromTop(resolvedMethod.ArgSlotCount() - 1) 
if ref == nil { 
// hack! 
if methodRef.Name() == "println" { 
_println(frame.OperandStack(), methodRef.Descriptor()) 
return 
}
panic("java.lang.NullPointerException") 
}
... // 其他代码 
} 

_println（）函数如下：
func _println(stack *rtda.OperandStack, descriptor string) { 
switch descriptor { 
case "(Z)V": fmt.Printf("%v\n", stack.PopInt() != 0) 
case "(C)V": fmt.Printf("%c\n", stack.PopInt()) 
case "(B)V": fmt.Printf("%v\n", stack.PopInt()) 
case "(S)V": fmt.Printf("%v\n", stack.PopInt()) 
case "(I)V": fmt.Printf("%v\n", stack.PopInt()) 
case "(F)V": fmt.Printf("%v\n", stack.PopFloat()) 
case "(J)V": fmt.Printf("%v\n", stack.PopLong())
case "(D)V": fmt.Printf("%v\n", stack.PopDouble()) 
default: panic("println: " + descriptor) 
}
stack.PopRef() 
}
```
##### 7.5.4 invokeinterface指令 
在ch07\instructions\references目录下创建invokeinterface.go文件，在其中定义invokeinterface指令，代码如下：
```go
package references 
import "jvmgo/ch07/instructions/base" 
import "jvmgo/ch07/rtda" 
import "jvmgo/ch07/rtda/heap" 
// Invoke interface method 
type INVOKE_INTERFACE struct { 
index uint
// count uint8 
// zero uint8 
} 
```
注意，和其他三条方法调用指令略有不同，在字节码中，invokeinterface指令的操作码后面跟着4字节而非2字节。前两字节的含义和其他指令相同，是个uint16运行时常量池索引。第3字节的值是给方法传递参数需要的slot数，其含义和给Method结构体定义的argSlotCount字段相同。正如我们所知，这个数是可以根据方法描述符计算出来的，它的存在仅仅是因为历史原因。第4字节是留给Oracle的某些Java虚拟机实现用的，它的值必须是0。该字节的存在是为了保证Java虚拟机可以向后兼容。 
invokeinterface指令的FetchOperands（）方法如下：
```go
func (self *INVOKE_INTERFACE) FetchOperands(reader *base.BytecodeReader) { 
self.index = uint(reader.ReadUint16()) 
reader.ReadUint8() // count 
reader.ReadUint8() // must be 0 
} 
```
下面看Execute（）方法，第一部分代码如下：
```go
func (self *INVOKE_INTERFACE) Execute(frame *rtda.Frame) { 
cp := frame.Method().Class().ConstantPool() 
methodRef := cp.GetConstant(self.index).(*heap.InterfaceMethodRef) 
resolvedMethod := methodRef.ResolvedInterfaceMethod() 
if resolvedMethod.IsStatic() || resolvedMethod.IsPrivate() { 
panic("java.lang.IncompatibleClassChangeError") 
}
```
先从运行时常量池中拿到并解析接口方法符号引用，如果解析后的方法是静态方法或私有方法，则抛出IncompatibleClassChangeError异常。继续看代码： 
```go
ref := frame.OperandStack().GetRefFromTop(resolvedMethod.ArgSlotCount() - 1) 
if ref == nil { 
panic("java.lang.NullPointerException") 
}
if !ref.Class().IsImplements(methodRef.ResolvedClass()) { 
panic("java.lang.IncompatibleClassChangeError") 
}
```
从操作数栈中弹出this引用，如果引用是null，则抛出NullPointerException异常。如果引用所指对象的类没有实现解析出来的接口，则抛出IncompatibleClassChangeError异常。继续看代码：
```go
methodToBeInvoked := heap.LookupMethodInClass(ref.Class(), 
methodRef.Name(), methodRef.Descriptor()) 
if methodToBeInvoked == nil || methodToBeInvoked.IsAbstract() { 
panic("java.lang.AbstractMethodError") 
}
if !methodToBeInvoked.IsPublic() { 
panic("java.lang.IllegalAccessError") 
}
```
查找最终要调用的方法。如果找不到，或者找到的方法是抽象的，则抛出Abstract-MethodError异常。如果找到的方法不是public，则抛出IllegalAccessError异常，否则，一切正常，调用方法：
```go
base.InvokeMethod(frame, methodToBeInvoked) 
} 
```
至此，4条方法调用指令都实现完毕了。再总结一下这4条指令的用途。invokestatic指令调用静态方法，很好理解。invokespecial指令也比较好理解。首先，因为私有方法和构造函数不需要动态绑定，所以invokespecial指令可以加快方法调用速度。其次，使用super关键字调用超类中的方法不能使用invokevirtual指令，否则会陷入无限循环。 

那么为什么要单独定义invokeinterface指令呢？统一使用invokevirtual指令不行吗？答案是，可以，但是可能会影响效率。这两条指令的区别在于：当Java虚拟机通过invokevirtual调用方法时，this引用指向某个类（或其子类）的实例。因为类的继承层次是固定的，所以虚拟机可以使用一种叫作vtable（Virtual Method Table）的技术加速方法查找。但是当通过invokeinterface指令调用接口方法时，因为this引用可以指向任何实现了该接口的类的实例，所以无法使用vtable技术。 

由于篇幅限制，这里就不深入讨论vtable技术了。感兴趣的读者可以阅读相关资料，或者改进我们的代码，给invokevirtual指令增加vtable优化。 

4条方法调用指令和6条返回指令都准备好了，还需要修改ch07\instructions\factor -y.go文件，在其中增加这些指令的case语句。鉴于改动比较简单，这里就不给出代码了。

#### 7.6 改进解释器 
我们的解释器目前只能执行单个方法，现在就扩展它，让它支持方法调用。打开ch07\interpreter.go文件，修改interpret（）方法，代码如下：
```go
func interpret(method *heap.Method, logInst bool) { 
thread := rtda.NewThread() 
frame := thread.NewFrame(method) 
thread.PushFrame(frame) 
defer catchErr(thread) 
loop(thread, logInst) 
} 
```
logInst参数控制是否把指令执行信息打印到控制台。更重要的变化在loop（）函数中，代码如下所示：
```go
func loop(thread *rtda.Thread, logInst bool) {
reader := &base.BytecodeReader{} 
for {
frame := thread.CurrentFrame() 
pc := frame.NextPC() 
thread.SetPC(pc) 
// decode 
reader.Reset(frame.Method().Code(), pc) 
opcode := reader.ReadUint8() 
inst := instructions.NewInstruction(opcode) 
inst.FetchOperands(reader) 
frame.SetNextPC(reader.PC()) 
if (logInst) { 
logInstruction(frame, inst) 
}
// execute 
inst.Execute(frame) 
if thread.IsStackEmpty() { 
break 
}
} 
}
```
在每次循环开始，先拿到当前帧，然后根据pc从当前方法中解码出一条指令。指令执行完毕之后，判断Java虚拟机栈中是否还有帧。如果没有则退出循环；否则继续。Thread结构体的IsStackEmpty（）方法是新增加的，代码在ch07\rtda\thread.go中，如下所示： 
```go
func (self *Thread) IsStackEmpty() bool { 
return self.stack.isEmpty() 
}
```
它只是调用了Stack结构体的isEmpty（）方法，代码在ch07\rtda\jvm_stack.go中，如下所示：
```go
func (self *Stack) isEmpty() bool { 
return self._top == nil 
}
```
回到interpreter.go，如果解释器在执行期间出现了问题，catchErr（）函数会打印出错信息，代码如下：
```go
func catchErr(thread *rtda.Thread) { 
if r := recover(); r != nil { 
logFrames(thread) 
panic(r) 
}
} 
```
logFrames（）函数打印Java虚拟机栈信息，代码如下：
```go
func logFrames(thread *rtda.Thread) { 
for !thread.IsStackEmpty() { 
frame := thread.PopFrame() 
method := frame.Method() 
className := method.Class().Name() 
fmt.Printf(">> pc:%4d %v.%v%v \n", 
frame.NextPC(), className, method.Name(), method.Descriptor()) 
} 
} 
```
logInstruction（）函数在方法执行过程中打印指令信息，代码如下：
```go
func logInstruction(frame *rtda.Frame, inst base.Instruction) { 
method := frame.Method() 
className := method.Class().Name() 
methodName := method.Name() 
pc := frame.Thread().PC() 
fmt.Printf("%v.%v() #%2d %T %v\n", className, methodName, pc, inst, inst) 
}
```
解释器改造完毕，下面测试方法调用。
#### 7.7 测试方法调用 
先改造命令行工具，给它增加两个选项。java命令提供了-verbose：class（简写为-verbose）选项，可以控制是否把类加载信息输出到控制台。也增加这样一个选项，另外参照这个选项增加一个-verbose：inst选项，用来控制是否把指令执行信息输出到控制台。 
打开ch07\cmd.go文件，修改Cmd结构体如下：
```go
type Cmd struct { 
helpFlag bool 
versionFlag bool 
verboseClassFlag bool 
verboseInstFlag bool 
cpOption string 
XjreOption string 
class string 
args []string 
}
```
parseCmd（）函数也需要修改，改动比较简单，这里就不给出代码了。下面修改ch07\main.go文件，其他地方不变，只需要修改startJVM（）函数，代码如下：
```go
func startJVM(cmd *Cmd) { 
cp := classpath.Parse(cmd.XjreOption, cmd.cpOption) 
classLoader := heap.NewClassLoader(cp, cmd.verboseClassFlag) 
className := strings.Replace(cmd.class, ".", "/", -1) 
mainClass := classLoader.LoadClass(className) 
mainMethod := mainClass.GetMainMethod()
if mainMethod != nil { 
interpret(mainMethod, cmd.verboseInstFlag) 
} else { 
fmt.Printf("Main method not found in class %s\n", cmd.class) 
} 
} 
```
然后修改ch07\rtda\heap\class_loader.go文件，给ClassLoader结构体添加verboseFlag字段，代码如下：
```go
type ClassLoader struct { 
cp *classpath.Classpath 
verboseFlag bool 
classMap map[string]*Class 
} 
```
NewClassLoader（）函数要相应修改，改动如下：
```go
func NewClassLoader(cp *classpath.Classpath, verboseFlag bool) *ClassLoader { 
return &ClassLoader{ 
cp: cp, 
verboseFlag: verboseFlag, 
classMap: make(map[string]*Class), 
} 
}
```
loadNonArrayClass（）函数也要修改，改动如下：
```go
func (self *ClassLoader) loadNonArrayClass(name string) *Class { 
data, entry := self.readClass(name) 
class := self.defineClass(data) 
link(class) 
if self.verboseFlag { 
fmt.Printf("[Loaded %s from %s]\n", name, entry) 
}
return class 
}
```
一切都准备就绪，打开命令行窗口，执行下面的命令编译本章代码： 
```shell script
go install jvmgo\ch07 
```
命令执行完毕后，在D：\go\workspace\bin目录下出现ch07.exe文件。下面这个类演示了各种情况下，4种方法调用命令的使用。 
```java
package jvmgo.book.ch07; 
public class InvokeDemo implements Runnable { 
public static void main(String[] args) { 
new InvokeDemo().test(); 
}
public void test() { 
InvokeDemo.staticMethod(); // invokestatic 
InvokeDemo demo = new InvokeDemo(); // invokespecial 
demo.instanceMethod(); // invokespecial 
super.equals(null); // invokespecial 
this.run(); // invokevirtual 
((Runnable) demo).run(); // invokeinterface 
}
public static void staticMethod() {} 
private void instanceMethod() {} 
@Override public void run() {} 
}
```
用javac编译InvokeDemo类，然后用ch07.exe执行InvokeDemo程序，可以看到程序正常执行（没有任何输出），如图7-3所示。
![7-3](/img/7-3.png)
图7-3 InvokeDemo执行结果 

InvokeDemo只是演示，下面看一个稍微复杂一些的例子。
```java
package jvmgo.book.ch07; 
public class FibonacciTest { 
public static void main(String[] args) { 
long x = fibonacci(30); 
System.out.println(x); 
}
private static long fibonacci(long n) { 
if (n <= 1) { return n; } 
return fibonacci(n - 1) + fibonacci(n - 2); 
} 
}
```
FibonacciTest类演示了斐波那契数列的计算，用javac编译它，然后用ch07.exe执行，结果如图7-4所示。 
![7-4](/img/7-4.png) 
图7-4 FibonacciTest执行结果 
几秒钟停顿之后，控制台上打印出了832040。我们的Java虚拟机终于可以执行复杂计算了。方法调用指令就测试到这里，下面在本章的最后，讨论类的初始化。
#### 7.8 类初始化 
第6章实现了一个简化版的类加载器，可以把类加载到方法区中。但是因为当时还没有实现方法调用，所以没有办法初始化类。现在可以把这个逻辑补上了。我们已经知道，类初始化就是执行类的初始化方法（`<clinit>`）。类的初始化在下列情况下触发： 
- 执行new指令创建类实例，但类还没有被初始化。 
- 执行putstatic、getstatic指令存取类的静态变量，但声明该字段的类还没有被初始化。 
- 执行invokestatic调用类的静态方法，但声明该方法的类还没有被初始化。 
- 当初始化一个类时，如果类的超类还没有被初始化，要先初始化类的超类。 
- 执行某些反射操作时。 
为了判断类是否已经初始化，需要给Class结构体添加一个字段：
```go
type Class struct { 
... // 其他字段 
initStarted bool 
}
``` 
类的初始化其实分为几个阶段，但由于我们的类加载器还不够完善，所以先使用一个简单的布尔状态就足够了。initStarted字段表示类的`<clinit>`方法是否已经开始执行。接下来给Class结构体添加两个方法，代码如下：
```go
func (self *Class) InitStarted() bool { 
return self.initStarted 
}
func (self *Class) StartInit() { 
self.initStarted = true 
}
```
InitStarted（）是Getter方法，返回initStarted字段值。StartInit（）方法把initStarted字段设置成true。下面修改new指令，代码如下：
```go
func (self *NEW) Execute(frame *rtda.Frame) { 
cp := frame.Method().Class().ConstantPool() 
classRef := cp.GetConstant(self.Index).(*heap.ClassRef) 
class := classRef.ResolvedClass() 
if !class.InitStarted() { 
frame.RevertNextPC() 
base.InitClass(frame.Thread(), class) 
return 
}
... // 其他代码 
} 
```
putstatic和getstatic指令改动类似，以putstatic指令为例，代码如下：
```go
func (self *PUT_STATIC) Execute(frame *rtda.Frame) { 
... // 其他代码 
field := fieldRef.ResolvedField() 
class := field.Class() 
if !class.InitStarted() { 
frame.RevertNextPC() 
base.InitClass(frame.Thread(), class) 
return 
}
... // 其他代码 
}
```
invokestatic指令也需要修改，改动如下：
```go
func (self *INVOKE_STATIC) Execute(frame *rtda.Frame) { 
... // 其他代码 
class := resolvedMethod.Class() 
if !class.InitStarted() { 
frame.RevertNextPC() 
base.InitClass(frame.Thread(), class) 
return 
}
base.InvokeMethod(frame, resolvedMethod) 
}
```
4条指令都修改完毕了，但是新增加的代码做了些什么？先判断类的初始化是否已经开始，如果还没有，则需要调用类的初始化方法，并终止指令执行。但是由于此时指令已经执行到了一半，也就是说当前帧的nextPC字段已经指向下一条指令，所以需要修改nextPC，让它重新指向当前指令。Frame结构体的RevertNextPC（）方法做了这样的操作，代码如下：
```go
func (self *Frame) RevertNextPC() { 
self.nextPC = self.thread.pc 
}
nextPC调整好之后，下一步查找并调用类的初始化方法。这个逻辑是通用的，在ch07 \instructions\base\class_init_logic.go文件中实现它，代码如下：
```go
func InitClass(thread *rtda.Thread, class *heap.Class) { 
class.StartInit() 
scheduleClinit(thread, class) 
initSuperClass(thread, class) 
}
```
InitClass（）函数先调用StartInit（）方法把类的initStarted状态设置成true以免进入死循环，然后调用scheduleClinit（）函数准备执行类的初始化方法，代码如下：
```go
func scheduleClinit(thread *rtda.Thread, class *heap.Class) { 
clinit := class.GetClinitMethod() 
if clinit != nil { 
// exec <clinit> 
newFrame := thread.NewFrame(clinit) 
thread.PushFrame(newFrame) 
} 
}
```
类初始化方法没有参数，所以不需要传递参数。Class结构体的GetClinitMethod（）方法如下：
```go
func (self *Class) GetClinitMethod() *Method { 
return self.getStaticMethod("<clinit>", "()V") 
}
```
注意，这里有意使用了scheduleClinit这个函数名而非invokeClinit，因为有可能要先执行超类的初始化方法，如函数initSuperClass（）所示。
```go
func initSuperClass(thread *rtda.Thread, class *heap.Class) { 
if !class.IsInterface() { 
superClass := class.SuperClass() 
if superClass != nil && !superClass.InitStarted() { 
InitClass(thread, superClass) 
} 
} 
}
```
如果超类的初始化还没有开始，就递归调用InitClass（）函数执行超类的初始化方法，这样可以保证超类的初始化方法对应的帧在子类上面，使超类初始化方法先于子类执行。 
类的初始化逻辑写完了，由于篇幅限制，这里就不进行测试了。读者可以参考随书Java示例代码，或者自行编写Java程序进行测试。不过在进行测试之前，还需要增加一个小小的hack。由于目前还不支持本地方法调用，而Java类库中的很多类都要注册本地方法，比如Object类就有一个registerNatives（）本地方法，用于注册其他方法，代码如下： 
```java
// java.lang.Object 
public class Object { 
private static native void registerNatives(); 
static { 
registerNatives(); 
}
... // 其他代码 
} 
```
由于Object类是其他所有类的超类，所以这会导致Java虚拟机崩溃。解决办法是修改InvokeMethod（）函数（代码在ch07\instructions\base\method_invoke_logic.go文件中），让它跳过所有registerNatives（）方法，改动如下：
```go
package base 
import "fmt" 
import "jvmgo/ch07/rtda"import "jvmgo/ch07/rtda/heap" 
func InvokeMethod(invokerFrame *rtda.Frame, method *heap.Method) { 
... // 前面的代码不变，下面是 
hack!
if method.IsNative() { 
if method.Name() == "registerNatives" { 	
thread.PopFrame() 
} else { 
panic(fmt.Sprintf("native method: %v.%v%v\n", 
method.Class().Name(), method.Name(), method.Descriptor())) 
} 
} 
}
```
如果遇到其他本地方法，直接调用panic（）函数终止程序执行即可。将在第9章讨论本地方法调用。
#### 7.9 本章小结 
本章讨论了方法调用和返回，并且实现了类初始化逻辑。如果说在前面几章里，我们的Java虚拟机还是个小baby只会爬的话，到了本章结尾，它已经可以满地跑了。下一章将讨论数组和字符串，届时，我们的小baby就有更多的玩具可以玩耍了。

