第6章 类和对象 
---
在第4章，我们初步实现了线程私有的运行时数据区，在此基础上，第5章实现了一个简单的解释器和150多条指令。这些指令主要是操作局部变量表和操作数栈、进行数学运算、比较运算和跳转控制等。本章将实现线程共享的运行时数据区，包括方法区和运行时常量池。 

第2章实现了类路径，可以找到class文件，并把数据加载到内存中。第3章实现了class文件解析，可以把class数据解析成一个ClassFile结构体。本章将进一步处理ClassFile结构体，把它加以转换，放进方法区以供后续使用。本章还会初步讨论类和对象的设计，实现一个简单的类加载器，并且实现类和对象相关的部分指令。 

在开始学习本章之前，还是先把目录结构准备好。复制ch05目录，改名为ch06。修改main.go等文件，把import语句中的ch05全都改成ch06，然后在ch06\rtda目录中创建heap子目录。现在目录结构看起来应该是下面这样：
```
D:\go\workspace\src 
|-jvmgo 
|-ch01 ~ ch05 
|-ch06 
|-classfile 
|-classpath 
|-instructions 
|-rtda 
|-heap 
|-cmd.go 
|-interpreter.go 
|-main.go 
```
在第4章中，在rtda\object.go文件中定义了临时的Object结构体。现在可以把object.go移到heap目录下了。注意要修改包名，代码如下：
```go
package heap 
type Object struct { 
    // todo 
}
```
还需要修改slot.go、local_vars.go和operand_stack.go这三个文件，在其中添加heap包的import语句，并把*Object改成*heap.Object。以上改动不大，为了节约篇幅，这里就不给出具体代码了。
#### 6.1 方法区 
第4章简单讨论过方法区，它是运行时数据区的一块逻辑区域，由多个线程共享。方法区主要存放从class文件获取的类信息。此外，类变量也存放在方法区中。当Java虚拟机第一次使用某个类时，它会搜索类路径，找到相应的class文件，然后读取并解析class文件，把相关信息放进方法区。至于方法区到底位于何处，是固定大小还是动态调整，是否参与垃圾回收，以及如何在方法区内存放类数据等，Java虚拟机规范并没有明确规定。
先来看看有哪些信息需要放进方法区。
##### 6.1.1 类信息 
使用结构体来表示将要放进方法区内的类。在ch06\rtda\heap目录下创建class.go文件，在其中定义Class结构体，代码如下：
```go
package heap 
import "jvmgo/ch06/classfile" 
type Class struct { 
    accessFlags uint16 
    name string // thisClassName 
    superClassName string 
    interfaceNames []string 
    constantPool *ConstantPool 
    fields []*Field 
    methods []*Method 
    loader *ClassLoader 
    superClass *Class 
    interfaces []*Class 
    instanceSlotCount uint 
    staticSlotCount uint 
    staticVars *Slots 
} 
```
accessFlags是类的访问标志，总共16比特。字段和方法也有访问标志，但具体标志位的含义可能有所不同。根据Java虚拟机规范，把各个比特位的含义统一定义在heap\access_flags.go文件中，代码如下：
```go
package heap 
const ( 
    ACC_PUBLIC = 0x0001 // class field method 
    ACC_PRIVATE = 0x0002 // field method 
    ACC_PROTECTED = 0x0004 // field method 
    ACC_STATIC = 0x0008 // field method 
    ACC_FINAL = 0x0010 // class field methodACC_SUPER = 0x0020 // class 
    ACC_SYNCHRONIZED = 0x0020 // method 
    ACC_VOLATILE = 0x0040 // field 
    ACC_BRIDGE = 0x0040 // method 
    ACC_TRANSIENT = 0x0080 // field 
    ACC_VARARGS = 0x0080 // method 
    ACC_NATIVE = 0x0100 // method 
    ACC_INTERFACE = 0x0200 // class 
    ACC_ABSTRACT = 0x0400 // class method 
    ACC_STRICT = 0x0800 // method 
    ACC_SYNTHETIC = 0x1000 // class field method 
    ACC_ANNOTATION = 0x2000 // class 
    ACC_ENUM = 0x4000 // class field 
) 
```
回到Class结构体。name、superClassName和interfaceNames字段分别存放类名、超类名和接口名。注意这些类名都是完全限定名，具有java/lang/Object的形式。constantPool字段存放运行时常量池指针，fields和methods字段分别存放字段表和方法表。运行时常量池将在6.2节中详细介绍。 

继续编辑class.go文件，在其中定义newClass()函数，用来把ClassFile结构体转换成Class结构体，代码如下：
```go
func newClass(cf *classfile.ClassFile) *Class { 
    class := &Class{} 
    class.accessFlags = cf.AccessFlags() 
    class.name = cf.ClassName() 
    class.superClassName = cf.SuperClassName() 
    class.interfaceNames = cf.InterfaceNames() 
    class.constantPool = newConstantPool(class, cf.ConstantPool())//见 6.2小节
    class.fields = newFields(class, cf.Fields()) // 见6.1.2小节 
    class.methods = newMethods(class, cf.Methods()) // 见 6.1.3小节 
    return class 
} 
```
newClass()函数又调用了newConstantPool()、newFields()和newMethods()，这三个函数的代码将在后面的小节给出。继续编辑class.go文件，在其中定义8个方法，用来判断某个访问标志是否被设置。这8个方法都很简单，为了节约篇幅，这里只给出IsPublic()方法的代码。
```go
func (self *Class) IsPublic() bool { 
    return 0 != self.accessFlags&ACC_PUBLIC 
}
```
后面将要介绍的Field和Method结构体也有类似的方法，届时也将不再赘述，请读者注意。
##### 6.1.2 字段信息 
字段和方法都属于类的成员，它们有一些相同的信息(访问标志、名字、描述符)。为了避免重复代码，创建一个结构体存放这些信息。在ch06\rtda\heap目录下创建lass_member.go文件，在其中定义ClassMember结构体，代码如下：
```go
package heap 
import "jvmgo/ch06/classfile" 
type ClassMember struct { 
    accessFlags uint16 
    name string 
    descriptor string 
    class *Class 
}
func (self *ClassMember) copyMemberInfo(memberInfo *classfile.MemberInfo) {...} 
```
前面三个字段的含义很明显，这里不多解释。class字段存放Class结构体指针，这样可以通过字段或方法访问到它所属的类。copyMemberInfo()方法从class文件中复制数据，代码如下：
```go
func (self *ClassMember) copyMemberInfo(memberInfo *classfile.MemberInfo) { 
    self.accessFlags = memberInfo.AccessFlags() 
    self.name = memberInfo.Name() 
    self.descriptor = memberInfo.Descriptor() 
}
```
ClassMember定义好了，接下来在ch06\rtda\heap目录下创建field.go文件，在其中定义Field结构体，代码如下：
```go
package heap 
import "jvmgo/ch06/classfile" 
type Field struct { 
    ClassMember 
}
func newFields(class *Class, cfFields []*classfile.MemberInfo) []*Field {...}
```
Field结构体比较简单，目前所有信息都是从ClassMember中继承过来的。newFields()函数根据class文件的字段信息创建字段表，代码如下：
```go
func newFields(class *Class, cfFields []*classfile.MemberInfo) []*Field { 
    fields := make([]*Field, len(cfFields)) 
    for i, cfField := range cfFields { 
        fields[i] = &Field{} 
        fields[i].class = class 
        fields[i].copyMemberInfo(cfField) 
    }
    return fields 
}
```
##### 6.1.3 方法信息 
方法比字段稍微复杂一些，因为方法中有字节码。在ch06\rtda\heap目录下创建method.go文件，在其中定义Method结构体，代码如下：
```go
package heap 
import "jvmgo/ch06/classfile" 
type Method struct {
    ClassMember 
    maxStack uint 
    maxLocals uint 
    code []byte 
}
func newMethods(class *Class, cfMethods []*classfile.MemberInfo) []*Method {...} 
```
maxStack和maxLocals字段分别存放操作数栈和局部变量表大小，这两个值是由Java编译器计算好的。code字段存放方法字节码。newMethods()函数根据class文件中的方法信息创建Method表，代码如下： 
```go
func newMethods(class *Class, cfMethods []*classfile.MemberInfo) []*Method { 
    methods := make([]*Method, len(cfMethods)) 
    for i, cfMethod := range cfMethods { 
        methods[i] = &Method{} 
        methods[i].class = class 
        methods[i].copyMemberInfo(cfMethod) 
        methods[i].copyAttributes(cfMethod) 
    }
    return methods 
}
```
大家还记得吗？maxStack、maxLocals和字节码在class文件中是以属性的形式存储在method_info结构中的。如果读者已经忘记的话，可以参考3.4.5小节。copyAttributes()方法从method_info结构中提取这些信息，代码如下：
```go
func (self *Method) copyAttributes(cfMethod *classfile.MemberInfo) { 
    if codeAttr := cfMethod.CodeAttribute(); codeAttr != nil { 
        self.maxStack = codeAttr.MaxStack() 
        self.maxLocals = codeAttr.MaxLocals() 
        self.code = codeAttr.Code() 
    } 
}
```
到此为止，除了ConstantPool还没有介绍以外，已经定义了4个结构体，这些结构体之间的关系如图6-1所示。 
![6-1](./img/6-1.png)
图6-1 Class结构体关系图
##### 6.1.4 其他信息 
Class结构体还有几个字段没有说明。loader字段存放类加载器指针，superClass和interfaces字段存放类的超类和接口指针，这三个字段将在6.3节介绍。staticSlotCount和instanceSlotCount字段分别存放类变量和实例变量占据的空间大小，staticVars字段存放静态变量，这三个字段将在6.4节介绍。
#### 6.2 运行时常量池 
运行时常量池主要存放两类信息：字面量(literal)和符号引用(symbolic reference)。字面量包括整数、浮点数和字符串字面量；符号引用包括类符号引用、字段符号引用、方法符号引用和接口方法符号引用。在ch06\rtda\heap目录下创建constant_pool.go文件，在其中定义Constant接口和ConstantPool结构体，代码如下：
```go
package heap 
import "fmt" 
import "jvmgo/ch06/classfile" 
type Constant interface{} 
type ConstantPool struct { 
    class *Class 
    consts []Constant 
}
func newConstantPool(class *Class, cfCp classfile.ConstantPool) *ConstantPool {...} 
func (self *ConstantPool) GetConstant(index uint) Constant {...} 
```
GetConstant()方法根据索引返回常量，代码如下： 
```go
func (self *ConstantPool) GetConstant(index uint) Constant { 
    if c := self.consts[index]; c != nil { 
        return c 
    }
    panic(fmt.Sprintf("No constants at index %d", index)) 
} 
```
newConstantPool()函数把class文件中的常量池转换成运行时常量池。这个函数稍微有点复杂，主体代码如下：
```go
func newConstantPool(class *Class, cfCp classfile.ConstantPool) *ConstantPool { 
    cpCount := len(cfCp) 
    consts := make([]Constant, cpCount) 
    rtCp := &ConstantPool{class, consts} 
    for i := 1; i < cpCount; i++ { 
        cpInfo := cfCp[i] 
        switch cpInfo.(type) { 
            ... 
        } 
    }
    return rtCp 
}
```
其实也不难理,核心逻辑就是把[]classfile.ConstantInfo转换成[]heap.Constant。具体常量的转换在switch-case中，我们分几次来看。 

最简单的是int或float型常量，直接取出常量值，放进consts中即可。 
```go
switch cpInfo.(type) { 
    case *classfile.ConstantIntegerInfo: 
        intInfo := cpInfo.(*classfile.ConstantIntegerInfo) 
        consts[i] = intInfo.Value() // int32 
    case *classfile.ConstantFloatInfo: 
        floatInfo := cpInfo.(*classfile.ConstantFloatInfo) 
        consts[i] = floatInfo.Value() // float32 
```

如果是long或double型常量，也是直接提取常量值放进consts 中。但是要注意，这两种类型的常量在常量池中都是占据两个位置，所以索引要特殊处理，代码如下：
```go
    case *classfile.ConstantLongInfo: 
        longInfo := cpInfo.(*classfile.ConstantLongInfo) 
        consts[i] = longInfo.Value() // int64 
        i++ 
    case *classfile.ConstantDoubleInfo: 
        doubleInfo := cpInfo.(*classfile.ConstantDoubleInfo) 
        consts[i] = doubleInfo.Value() // float64 
        i++ 
```

如果是字符串常量，直接取出Go语言字符串，放进consts中，代码如下： 
```go
    case *classfile.ConstantStringInfo: 
        stringInfo := cpInfo.(*classfile.ConstantStringInfo) 
        consts[i] = stringInfo.String() // string 
```

还剩下4种类型的常量需要处理，分别是类、字段、方法和接口方法的符号引用。后面的章节会详细介绍这4种符号引用，下面是剩下的代码。 
```go
    case *classfile.ConstantClassInfo: 
        classInfo := cpInfo.(*classfile.ConstantClassInfo) 
        consts[i] = newClassRef(rtCp, classInfo) // 见6.2.1小节 
    case *classfile.ConstantFieldrefInfo: 
        fieldrefInfo := cpInfo.(*classfile.ConstantFieldrefInfo) 
        consts[i] = newFieldRef(rtCp, fieldrefInfo) // 见6.2.2小节
    case *classfile.ConstantMethodrefInfo: 
        methodrefInfo := cpInfo.(*classfile.ConstantMethodrefInfo) 
        consts[i] = newMethodRef(rtCp, methodrefInfo) // 见 6.2.3小节 
    case *classfile.ConstantInterfaceMethodrefInfo: 
        methodrefInfo := cpInfo.(*classfile.ConstantInterfaceMethodrefInfo) 
        consts[i] = newInterfaceMethodRef(rtCp, methodrefInfo) // 见6.2.4小节 
```

基本类型常量的使用请参考6.4节。
##### 6.2.1 类符号引用 
因为4种类型的符号引用有一些共性，所以仍然使用继承来减少重复代码。在ch06 \rtda\heap目录下创建cp_symref.go文件，在其中定义SymRef结构体，代码如下：
```go
package heap 
// symbolic reference 
type SymRef struct { 
    cp *ConstantPool 
    className string 
    class *Class 
}
```
cp字段存放符号引用所在的运行时常量池指针，这样就可以通过符号引用访问到运行时常量池，进一步又可以访问到类数据。 

className字段存放类的完全限定名。class字段缓存解析后的类结构体指针，这样类符号引用只需要解析一次就可以了，后续可以直接使用缓存值。对于类符号引用，只要有类名，就可以解析符号引用。对于字段，首先要解析类符号引用得到类数据，然后用字段名和描述符查找字段数据。方法符号引用的解析过程和字段符号引用类似。

SymRef定义好了，接下来在ch06\rtda\heap目录下创建cp_classref.go文件，在其中定义ClassRef结构体，代码如下：
```go
package heap 
import "jvmgo/ch06/classfile" 
type ClassRef struct { 
    SymRef 
}
func newClassRef(cp *ConstantPool, 
classInfo *classfile.ConstantClassInfo) *ClassRef {...} 
```
ClassRef继承了SymRef，但是并没有添加任何字段。newClassRef()函数根据class文件中存储的类常量创建ClassRef实例，代码如下：
```go
func newClassRef(cp *ConstantPool, classInfo *classfile.ConstantClassInfo) *ClassRef { 
    ref := &ClassRef{} 
    ref.cp = cp 
    ref.className = classInfo.Name() 
    return ref 
}
```
类符号引用的解析将在6.5.2节讨论。
##### 6.2.2 字段符号引用 
在6.1.2节中，定义了ClassMember结构体来存放字段和方法共有的信息。类似地，本节定义MemberRef结构体来存放字段和方法符号引用共有的信息。在ch06\rtda\heap目录下创建cp_memberref.go文件，在其中定义MemberRef结构体，代码如下：
```go
package heap 
import "jvmgo/ch06/classfile" 
type MemberRef struct { 
    SymRef 
    name string 
    descriptor string 
}
func (self *MemberRef) copyMemberRefInfo(refInfo *classfile.ConstantMemberrefInfo) {...} 
```
读者可能会有疑问：在Java中，我们并不能在同一个类中定义名字相同，但类型不同的两个字段，那么字段符号引用为什么还要存放字段描述符呢？答案是，这只是Java语言的限制，而不是Java虚拟机规范的限制。也就是说，站在Java虚拟机的角度，一个类是完全可以有多个同名字段的，只要它们的类型互不相同就可以。copyMemberRefInfo()方法从class文件内存储的字段或方法常量中提取数据，代码如下：
```go
func (self *MemberRef) copyMemberRefInfo(refInfo *classfile.ConstantMemberrefInfo) { 
    self.className = refInfo.ClassName() 
    self.name, self.descriptor = refInfo.NameAndDescriptor() 
}
```
MemberRef定义好了，接下来在ch06\rtda\heap目录下创建cp_fieldref.go文件，在其中定义FieldRef结构体，代码如下：
```go
package heap 
import "jvmgo/ch06/classfile" 
type FieldRef struct { 
    MemberRef 
    field *Field 
}
func newFieldRef(cp *ConstantPool, 
refInfo *classfile.ConstantFieldrefInfo) *FieldRef {...} 
```
field字段缓存解析后的字段指针，newFieldRef()方法创建FieldRef实例，代码如下： 
```go
func newFieldRef(cp *ConstantPool, refInfo *classfile.ConstantFieldrefInfo) *FieldRef { 
    ref := &FieldRef{} 
    ref.cp = cp 
    ref.copyMemberRefInfo(&refInfo.ConstantMemberrefInfo) 
    return ref 
} 
```
字段符号引用的解析将在6.5.2节讨论。
##### 6.2.3 方法符号引用 
在ch06\rtda\heap目录下创建cp_methodref.go文件，在其中定义MethodRef结构体，代码如下：
```go
package heap 
import "jvmgo/ch06/classfile" 
type MethodRef struct { 
    MemberRef 
    method *Method 
}
func newMethodRef(cp *ConstantPool, refInfo *classfile.ConstantMethodrefInfo) *MethodRef { 
    ref := &MethodRef{} 
    ref.cp = cp 
    ref.copyMemberRefInfo(&refInfo.ConstantMemberrefInfo) 
    return ref 
}
```
上面的代码和字段符号引用大同小异，这里就不多解释了。方法符号引用的解析将在第7章讨论方法调用时详细介绍。
##### 6.2.4 接口方法符号引用 
在ch06\rtda\heap目录下创建cp_interface_methodref.go文件，在其中定义Interface-MethodRef结构体，代码如下：
```go
package heap 
import "jvmgo/ch06/classfile" 
type InterfaceMethodRef struct { 
    MemberRef 
    method *Method
}
func newInterfaceMethodRef(cp *ConstantPool, 
  refInfo *classfile.ConstantInterfaceMethodrefInfo) *InterfaceMethodRef { 
    ref := &InterfaceMethodRef{} 
    ref.cp = cp
    ref.copyMemberRefInfo(&refInfo.ConstantMemberrefInfo) 
    return ref
}
```
代码和前面差不多，也不多解释了。接口方法符号引用的解析同样会在第7章详细介绍。到此为止，所有的符号引用都已经定义好了，它们的继承结构如图6-2所示。
![6-2](./img/6-2.png)
图6-2 符号引用结构体继承关系图
#### 6.3 类加载器 
Java虚拟机的类加载系统十分复杂，本节将初步实现一个简化版的类加载器，后面的章节中还会对它进行扩展。在ch06/rtda/heap目录下创建class_loader.go文件，在其中定义ClassLoader结构体，代码如下：
```go
package heap 
import "fmt" 
import "jvmgo/ch06/classfile" 
import "jvmgo/ch06/classpath" 
type ClassLoader struct { 
    cp *classpath.Classpath 
    classMap map[string]*Class // loaded classes 
}
func NewClassLoader(cp *classpath.Classpath) *ClassLoader {...} 
func (self *ClassLoader) LoadClass(name string) *Class {...} 
```
ClassLoader依赖Classpath来搜索和读取class文件，cp字段保存Classpath指针。classMap字段记录已经加载的类数据，key是类的完全限定名。在前面讨论中，方法区一直只是个抽象的概念，现在可以把classMap字段当作方法区的具体实现。NewClassLoader()函数创建ClassLoader实例，代码比较简单，如下所示：
```go 
func NewClassLoader(cp *classpath.Classpath) *ClassLoader { 
    return &ClassLoader{ 
        cp: cp, 
        classMap: make(map[string]*Class), 
    }
} 
```
LoadClass()方法把类数据加载到方法区，代码如下：
```go
func (self *ClassLoader) LoadClass(name string) *Class { 
    if class, ok := self.classMap[name]; ok { 
        return class // 类已经加载 
    }
    return self.loadNonArrayClass(name) 
}
```
先查找classMap，看类是否已经被加载。如果是，直接返回类数据，否则调用loadNonArrayClass()方法加载类。数组类和普通类有很大的不同，它的数据并不是来自class文件，而是由Java虚拟机在运行期间生成。本章暂不考虑数组类的加载，留到第8章详细讨论。loadNonArrayClass()方法的代码如下：
```go
func (self *ClassLoader) loadNonArrayClass(name string) *Class { 
    data, entry := self.readClass(name) 
    class := self.defineClass(data) 	
    link(class) 
    fmt.Printf("[Loaded %s from %s]\n", name, entry) 
    return class 
}
```
可以看到，类的加载大致可以分为三个步骤：首先找到class文件并把数据读取到内存；然后解析class文件，生成虚拟机可以使用的类数据，并放入方法区；最后进行链接。下面分别讨论这三个步骤。
##### 6.3.1 readClass() 
readClass()方法的代码如下：
```go
func (self *ClassLoader) readClass(name string) ([]byte, classpath.Entry) { 
    data, entry, err := self.cp.ReadClass(name) 
    if err != nil { 
        panic("java.lang.ClassNotFoundException: " + name) 
    }
    return data, entry 
} 
```
readClass()方法只是调用了Classpath的ReadClass()方法，并进行了错误处理。需要解释一下它的返回值。为了打印类加载信息，把最终加载class文件的类路径项也返回给了调用者。
##### 6.3.2 defineClass() 
defineClass()方法的代码如下：
```go
func (self *ClassLoader) defineClass(data []byte) *Class { 
    class := parseClass(data) 
    class.loader = self 
    resolveSuperClass(class) 
    resolveInterfaces(class) 
    self.classMap[class.name] = class 
    return class 
}
```
defineClass()方法首先调用parseClass()函数把class文件数据转换成Class结构体。Class结构体的superClass和interfaces字段存放超类名和直接接口表，这些类名其实都是符号引用。根据Java虚拟机规范的5.3.5节，调用resolveSuperClass()和resolveInterfaces()函数解析这些类符号引用。下面是parseClass()函数的代码。
```go 
func parseClass(data []byte) *Class { 
    cf, err := classfile.Parse(data) 
    if err != nil { 
        panic("java.lang.ClassFormatError") 
    }
    return newClass(cf) // 见6.1.1小节 
}
```
resolveSuperClass()函数的代码如下：
```go
func resolveSuperClass(class *Class) { 
    if class.name != "java/lang/Object" { 
        class.superClass = class.loader.LoadClass(class.superClassName) 
    } 
}
```
再复习一下：除java.lang.Object以外，所有的类都有且仅有一个超类。因此，除非是Object类，否则需要递归调用LoadClass()方法加载它的超类。与此类似，resolveInterfaces()函数递归调用LoadClass()方法加载类的每一个直接接口，代码如下：
```go
func resolveInterfaces(class *Class) { 
    interfaceCount := len(class.interfaceNames) 
    if interfaceCount > 0 { 
        class.interfaces = make([]*Class, interfaceCount) 
        for i, interfaceName := range class.interfaceNames { 
            class.interfaces[i] = class.loader.LoadClass(interfaceName) 
        } 
    } 
}
```
##### 6.3.3 link() 
类的链接分为验证和准备两个必要阶段，link()方法的代码如下：
```go
func link(class *Class) { 
    verify(class) 
    prepare(class) 
} 
```
为了确保安全性，Java虚拟机规范要求在执行类的任何代码之前，对类进行严格的验证。由于篇幅的原因，本书忽略验证过程。Java虚拟机规范4.10节详细介绍了类的验证算法，感兴趣的读者可以尝试自己实现。verify()函数空空如也，代码如下：
```go
func verify(class *Class) { 
    // todo 
}
```
准备阶段主要是给类变量分配空间并给予初始值，prepare()函数推迟到6.4节再介绍。
#### 6.4 对象、实例变量和类变量 
在第4章中，定义了LocalVars结构体，用来表示局部变量表。从逻辑上来看，LocalVars实例就像一个数组，这个数组的每一个元素都足够容纳一个int、float或引用值。要放入double或者long值，需要相邻的两个元素。这个结构体不是正好也可以用来表示类变量和实例变量吗？ 

没错！但是，由于rtda包已经依赖了heap包，而Go语言的包又不能相互依赖，所以heap包中的go文件是无法导入rtda包的，否则Go编译器就会报错。为了解决这个问题，只好容忍一些重复代码的存在。在ch06\rtda\heap目录下创建slots.go文件，把slot.go和local_vars.go文件中的内容拷贝进来，然后在此基础上修改，代码如下：
```go
package heap 
import "math" 
type Slot struct { 
	num int32 
    ref *Object 
}
type Slots []Slot 
```
函数和方法的内容都没什么变化，为了节约篇幅，就不给出详细代码了，下面是列表。 
```
func newSlots(slotCount uint) Slots {...} 
func (self Slots) SetInt(index uint, val int32) {...} 
func (self Slots) GetInt(index uint) int32 {...} 
func (self Slots) SetFloat(index uint, val float32) {...} 
func (self Slots) GetFloat(index uint) float32 {...} 
func (self Slots) SetLong(index uint, val int64) {...} 
func (self Slots) GetLong(index uint) int64 {...} 
func (self Slots) SetDouble(index uint, val float64) {...} 
func (self Slots) GetDouble(index uint) float64 {...} 
func (self Slots) SetRef(index uint, ref *Object) {...} 
func (self Slots) GetRef(index uint) *Object {...} 
```
Slots结构体准备就绪，可以使用了。Class结构体早在6.1节就定义好了，代码如下： 
```
type Class struct { 
    ... 
    staticVars *Slots 
}
```
打开ch06\rtda\heap\object.go文件，给Object结构体添加两个字段，一个存放对象的Class指针，一个存放实例变量，代码如下：
```
type Object struct { 
    class *Class 
    fields Slots 
}
```
接下来的问题是，如何知道静态变量和实例变量需要多少空间，以及哪个字段对应Slots中的哪个位置呢？第一个问题比较好解决，只要数一下类的字段即可。假设某个类有m个静态字段和n个实例字段，那么静态变量和实例变量所需的空间大小就分别是m'和n'。这里要注意两点。首先，类是可以继承的。也就是说，在数实例变量时，要递归地数超类的实例变量；其次，long和double字段都占据两个位置，所以m'>=m，n'>=n。 
第二个问题也不算难，在数字段时，给字段按顺序编上号就可以了。这里有三点需要要注意。首先，静态字段和实例字段要分开编号，否则会混乱。其次，对于实例字段，一定要从继承关系的最顶端，也就是java.lang.Object开始编号，否则也会混乱。最后，编号时也要考虑long和double类型。 
打开field.go文件，给Field结构体加上slotId字段，代码如下： 
```
type Field struct { 
    ClassMember 
    slotId uint 
} 
```
打开class_loader.go文件，在其中定义prepare()函数，代码如下：
```go
func prepare(class *Class) { 
    calcInstanceFieldSlotIds(class)
    calcStaticFieldSlotIds(class) 
    allocAndInitStaticVars(class) 
}
```
calcInstanceFieldSlotIds()函数计算实例字段的个数，同时给它们编号，代码如下：
```go
func calcInstanceFieldSlotIds(class *Class) { 
    slotId := uint(0) 
    if class.superClass != nil { 
        slotId = class.superClass.instanceSlotCount 
    }
    for _, field := range class.fields { 
        if !field.IsStatic() { 
            field.slotId = slotId 
            slotId++ 
            if field.isLongOrDouble() { 
                slotId++ 
            } 
        } 
    }
    class.instanceSlotCount = slotId 
}
```
calcStaticFieldSlotIds()函数计算静态字段的个数，同时给它们编号，代码如下：
```go
func calcStaticFieldSlotIds(class *Class) { 
    slotId := uint(0) 
    for _, field := range class.fields { 
        if field.IsStatic() { 
            field.slotId = slotId 
            slotId++ 
            if field.isLongOrDouble() { 
                slotId++ 
            } 
        } 
    }
    class.staticSlotCount = slotId 
}
```
Field结构体的isLongOrDouble()方法返回字段是否是long或double类型，代码如下：
```go
func (self *Field) isLongOrDouble() bool { 
    return self.descriptor == "J" || self.descriptor == "D" 
}
```
allocAndInitStaticVars()函数给类变量分配空间，然后给它们赋 
予初始值，代码如下：
```go
func allocAndInitStaticVars(class *Class) { 
    class.staticVars = newSlots(class.staticSlotCount) 
    for _, field := range class.fields { 
        if field.IsStatic() && field.IsFinal() { 
            initStaticFinalVar(class, field) 
        } 
    } 
}
```
因为Go语言会保证新创建的Slot结构体有默认值(num字段是0，ref字段是nil)，而浮点数0编码之后和整数0相同，所以不用做任何操作就可以保证静态变量有默认初始值(数字类型是0，引用类型是null)。如果静态变量属于基本类型或String类型，有final修饰符，且它的值在编译期已知，则该值存储在class文件常量池中。initStaticFinalVar()函数从常量池中加载常量值，然后给静态变量赋值，代码如下：
```go
func initStaticFinalVar(class *Class, field *Field) { 
    vars := class.staticVars 
    cp := class.constantPool 
    cpIndex := field.ConstValueIndex() 
    slotId := field.SlotId() 
    if cpIndex > 0 { 
        switch field.Descriptor() { 
            case "Z", "B", "C", "S", "I": 
                val := cp.GetConstant(cpIndex).(int32) 
                vars.SetInt(slotId, val) 
            case "J": 
                val := cp.GetConstant(cpIndex).(int64) 
                vars.SetLong(slotId, val) 
            case "F": 
                val := cp.GetConstant(cpIndex).(float32) 
                vars.SetFloat(slotId, val) 
            case "D": 
                val := cp.GetConstant(cpIndex).(float64) 
                vars.SetDouble(slotId, val) 
            case "Ljava/lang/String;": 
                panic("todo") // 在第8章实现 
        } 
    } 
}
```
字符串常量将在第8章讨论，这里先调用panic()函数终止程序执行。需要给Field结构体添加constValueIndex字段，代码如下： 

```go
type Field struct {
    ClassMember 
    constValueIndex uint 
    slotId uint 
}
```
修改newFields()方法，从字段属性表中读取constValueIndex，代码改动如下： 
```go
func newFields(class *Class, cfFields []*classfile.MemberInfo) []*Field { 
    fields := make([]*Field, len(cfFields)) 
    for i, cfField := range cfFields { 
        fields[i] = &Field{} 
        fields[i].class = class 
        fields[i].copyMemberInfo(cfField) 
        fields[i].copyAttributes(cfField) 
    }
    return fields 
} 
```
copyAttributes()方法的代码如下：
```go
func (self *Field) copyAttributes(cfField *classfile.MemberInfo) { 
    if valAttr := cfField.ConstantValueAttribute(); valAttr != nil { 
        self.constValueIndex = uint(valAttr.ConstantValueIndex()) 
    } 
}
```
MemberInfo结构体的ConstantValueIndex()方法在ch06\classfile\member_info.go文件中，代码如下：
```go
func (self *MemberInfo) ConstantValueAttribute() *ConstantValueAttribute { 
    for _, attrInfo := range self.attributes { 
        switch attrInfo.(type) { 
            case *ConstantValueAttribute: 
                return attrInfo.(*ConstantValueAttribute) 
        } 
    }
    return nil 
}
```
#### 6.5 类和字段符号引用解析 
本节讨论类符号引用和字段符号引用的解析，方法符号引用的解析将在第7章讨论。
##### 6.5.1 类符号引用解析 
打开cp_symref.go文件，在其中定义ResolvedClass()方法，代码如下：
```go
func (self *SymRef) ResolvedClass() *Class { 
    if self.class == nil { 
        self.resolveClassRef() 
    }
    return self.class 
}
```
如果类符号引用已经解析，ResolvedClass()方法直接返回类指针，否则调用resolveClassRef()方法进行解析。Java虚拟机规范5.4.3.1节给出了类符号引用的解析步骤，resolveClassRef()方法就是按照这个步骤编写的(有一些简化)，代码如下：
```go
func (self *SymRef) resolveClassRef() { 
    d := self.cp.class 
    c := d.loader.LoadClass(self.className) 
    if !c.isAccessibleTo(d) { 
        panic("java.lang.IllegalAccessError") 
    }
    self.class = c 
} 
```
通俗地讲，如果类D通过符号引用N引用类C的话，要解析N，先用D的类加载器加载C，然后检查D是否有权限访问C，如果没有，则抛出IllegalAccessError异常。Java虚拟机规范5.4.4节给出了类的访问控制规则，把这个规则翻译成Class结构体的isAccessibleTo()方法，代码如下(在class.go文件中)：
```go
func (self *Class) isAccessibleTo(other *Class) bool { 
    return self.IsPublic() || self.getPackageName() ==	other.getPackageName() 
} 
```
也就是说，如果类D想访问类C，需要满足两个条件之一：C是public，或者C和D在同一个运行时包内。第11章再讨论运行时包，这里先简单按照包名来检查。getPackageName()方法的代码如下(也在class.go文件中)： 
```go
func (self *Class) getPackageName() string { 
    if i := strings.LastIndex(self.name, "/"); i >= 0 { 
        return self.name[:i]
    }
    return "" 
}
``` 
比如类名是java/lang/Object，则它的包名就是java/lang。如果类定义在默认包中，它的包名是空字符串。
##### 6.5.2 字段符号引用解析 
打开cp_fieldref.go文件，在其中定义ResolvedField()方法，代码如下：
```go
func (self *FieldRef) ResolvedField() *Field { 
    if self.field == nil { 
        self.resolveFieldRef() 
    }
    return self.field 
}
```
ResolvedField()方法与ResolvedClass()方法大同小异，就不多解释了。Java虚拟机规范5.4.3.2节给出了字段符号引用的解析步骤，把它翻译成resolveFieldRef()方法，代码如下：
```go
func (self *FieldRef) resolveFieldRef() { 
    d := self.cp.class 
    c := self.ResolvedClass() 
    field := lookupField(c, self.name, self.descriptor) 
    if field == nil { 
        panic("java.lang.NoSuchFieldError") 
    }
    if !field.isAccessibleTo(d) { 
        panic("java.lang.IllegalAccessError") 
    }
    self.field = field 
}
```
如果类D想通过字段符号引用访问类C的某个字段，首先要解析符号引用得到类C，然后根据字段名和描述符查找字段。如果字段查找失败，则虚拟机抛出NoSuchFieldError异常。如果查找成功，但D没有足够的权限访问该字段，则虚拟机抛出IllegalAccessError异常。字段查找步骤在lookupField()函数中，代码如下：
```go
func lookupField(c *Class, name, descriptor string) *Field { 
    for _, field := range c.fields { 
        if field.name == name && field.descriptor == descriptor { 
            return field 
        } 
    }
    for _, iface := range c.interfaces { 
        if field := lookupField(iface, name, descriptor); field != nil { 
            return field 
        } 
    }
    if c.superClass != nil { 
        return lookupField(c.superClass, name, descriptor) 
    }
    return nil
}
```
首先在C的字段中查找。如果找不到，在C的直接接口递归应用这个查找过程。如果还找不到的话，在C的超类中递归应用这个查找过程。如果仍然找不到，则查找失败。Java虚拟机规范5.4.4节也给出了字段的访问控制规则。这个规则同样也适用于方法，所以把它(略做简化)实现成ClassMember结构体的isAccessibleTo()方法，代码如下(在class_member.go文件中)： 
```go
func (self *ClassMember) isAccessibleTo(d *Class) bool { 
    if self.IsPublic() { 
        return true 
    }
    c := self.class 
    if self.IsProtected() { 
        return d == c || d.isSubClassOf(c) || c.getPackageName() == d.getPackageName() 
    }
    if !self.IsPrivate() { 
        return c.getPackageName() == d.getPackageName() 
    }
    return d == c 
}
```
用通俗的语言描述字段访问规则。如果字段是public，则任何类都可以访问。如果字段是protected，则只有子类和同一个包下的类可以访问。如果字段有默认访问权限(非public，非protected，也非privated)，则只有同一个包下的类可以访问。否则，字段是private，只有声明这个字段的类才能访问。
#### 6.6 类和对象相关指令 
本节将实现10条类和对象相关的指令。new指令用来创建类实例；putstatic和getstatic指令用于存取静态变量；putfield和getfield用于存取实例变量；instanceof和checkcast指令用于判断对象是否属于某种类型；ldc系列指令把运行时常量池中的常量推到操作数栈顶。下面的Java代码演示了这些指令的用处。 
```java
public class MyObject { 
    public static int staticVar; 
    public int instanceVar; 
    public static void main(String[] args) { 
        int x = 32768; // ldc 
        MyObject myObj = new MyObject(); // new 
        MyObject.staticVar = x; // putstatic 
        x = MyObject.staticVar; // getstatic 
        myObj.instanceVar = x; // putfield 
        x = myObj.instanceVar; // getfield 
        Object obj = myObj; 
        if (obj instanceof MyObject) { // instanceof 
            myObj = (MyObject) obj; // checkcast 
        } 
    } 
} 
```
上面提到的指令除ldc以外，都属于引用类指令，在ch06\instructions目录下创建references子目录来存放引用类指令。首先实现new指令。
##### 6.6.1 new指令 
注意，new指令专门用来创建类实例。数组由专门的指令创建，在第8章中实现数组和数组相关指令。在ch06\instructions\references目录下创建new.go文件，在其中实现new指令，代码如下：
```go
package references 
import "jvmgo/ch06/instructions/base" 
import "jvmgo/ch06/rtda" 
import "jvmgo/ch06/rtda/heap" 
// Create new object 
type NEW struct{ base.Index16Instruction } 
```
new指令的操作数是一个uint16索引，来自字节码。通过这个索引，可以从当前类的运行时常量池中找到一个类符号引用。解析这个类符号引用，拿到类数据，然后创建对象，并把对象引用推入栈顶，new指令的工作就完成了。Execute()方法的代码如下：
```go
func (self *NEW) Execute(frame *rtda.Frame) { 
    cp := frame.Method().Class().ConstantPool() 
    classRef := cp.GetConstant(self.Index).(*heap.ClassRef) 
    class := classRef.ResolvedClass() 
    if class.IsInterface() || class.IsAbstract() { 
        panic("java.lang.InstantiationError") 
    }
    ref := class.NewObject() 
    frame.OperandStack().PushRef(ref) 
}
```
因为接口和抽象类都不能实例化，所以如果解析后的类是接口或抽象类，按照Java虚拟机规范规定，需要抛出InstantiationError异常。另外，如果解析后的类还没有初始化，则需要先初始化类。在第7章实现方法调用之后会详细讨论类的初始化，这里暂时先忽略。Class结构体的NewObject()方法如下(在class.go文件中)： 
```
func (self *Class) NewObject() *Object { 
    return newObject(self) 
} 
```
这里只是调用了Object结构体的newObject()方法，代码如下(在object.go中)： 
```
func newObject(class *Class) *Object { 
    return &Object{ 
        class: class, 
        fields: newSlots(class.instanceSlotCount), 
    } 
}
```
新创建对象的实例变量都应该赋好初始值，不过并不需要做额外的工作，具体原因前面已经讨论过，此处不再赘述。new指令实现好了，下面看看如何存取类的静态变量。
##### 6.6.2 putstatic和getstatic指令 
在references目录下创建putstatic.go文件，在其中实现putstatic指令，代码如下：
```go
package references 
import "jvmgo/ch06/instructions/base" 
import "jvmgo/ch06/rtda" 
import "jvmgo/ch06/rtda/heap" 
// Set static field in class 
type PUT_STATIC struct{ base.Index16Instruction } 
```
putstatic指令给类的某个静态变量赋值，它需要两个操作数。第一个操作数是uint16索引，来自字节码。通过这个索引可以从当前类的运行时常量池中找到一个字段符号引用，解析这个符号引用就可以知道要给类的哪个静态变量赋值。第二个操作数是要赋给静态变量的值，从操作数栈中弹出。Execute()方法稍微有些复杂，分三部分介绍：
```go
func (self *PUT_STATIC) Execute(frame *rtda.Frame) { 
    currentMethod := frame.Method() 
    currentClass := currentMethod.Class() 
    cp := currentClass.ConstantPool() 
    fieldRef := cp.GetConstant(self.Index).(*heap.FieldRef) 
    field := fieldRef.ResolvedField() 
    class := field.Class() 
```
先拿到当前方法、当前类和当前常量池，然后解析字段符号引用。如果声明字段的类还没有被初始化，则需要先初始化该类，这部分逻辑将在第7章实现。继续看代码：
```go
    if !field.IsStatic() { 
        panic("java.lang.IncompatibleClassChangeError") 
    }
    if field.IsFinal() { 
        if currentClass != class || currentMethod.Name() != "<clinit>" { 
            panic("java.lang.IllegalAccessError") 
        } 
    } 
```
如果解析后的字段是实例字段而非静态字段，则抛出IncompatibleClassChangeError异常。如果是final字段，则实际操作的是静态常量，只能在类初始化方法中给它赋值。否则，会抛出IllegalAccessError异常。类初始化方法由编译器生成，名字是`<clinit>`，具体请看第7章。继续看代码： 
```go
    descriptor := field.Descriptor() 
    slotId := field.SlotId() 
    slots := class.StaticVars() 
    stack := frame.OperandStack() 
    switch descriptor[0] { 
        case 'Z', 'B', 'C', 'S', 'I': slots.SetInt(slotId, stack.PopInt()) 
        case 'F': slots.SetFloat(slotId, stack.PopFloat()) 
        case 'J': slots.SetLong(slotId, stack.PopLong()) 
        case 'D': slots.SetDouble(slotId, stack.PopDouble()) 
        case 'L', '[': slots.SetRef(slotId, stack.PopRef()) 
    } 
}
``` 
根据字段类型从操作数栈中弹出相应的值，然后赋给静态变量。至此，putstatic指令就解释完毕了。getstatic指令和putstatic正好相反，它取出类的某个静态变量值，然后推入栈顶。在references目录下创建getstatic.go文件，在其中实现getstatic指令，代码如下：
```go
package references 
import "jvmgo/ch06/instructions/base" 
import "jvmgo/ch06/rtda" 
import "jvmgo/ch06/rtda/heap" 
// Get static field from class 
type GET_STATIC struct{ base.Index16Instruction } 
```
getstatic指令只需要一个操作数：uint16常量池索引，用法和putstatic一样，代码如下：
```go
func (self *GET_STATIC) Execute(frame *rtda.Frame) { 
    cp := frame.Method().Class().ConstantPool() 
    fieldRef := cp.GetConstant(self.Index).(*heap.FieldRef) 
    field := fieldRef.ResolvedField() 
    class := field.Class() 
    if !field.IsStatic() { 
        panic("java.lang.IncompatibleClassChangeError") 
    }
```
如果解析后的字段不是静态字段，也要抛出IncompatibleClassChangeError异常。如果声明字段的类还没有初始化好，也需要先初始化。getstatic只是读取静态变量的值，自然也就不用管它是否是final了。继续看剩下的代码： 
```go
    descriptor := field.Descriptor()slotId := field.SlotId() 
    slots := class.StaticVars() 
    stack := frame.OperandStack() 
    switch descriptor[0] { 
        case 'Z', 'B', 'C', 'S', 'I': stack.PushInt(slots.GetInt(slotId)) 
        case 'F': stack.PushFloat(slots.GetFloat(slotId)) 
        case 'J': stack.PushLong(slots.GetLong(slotId)) 
        case 'D': stack.PushDouble(slots.GetDouble(slotId)) 
        case 'L', '[': stack.PushRef(slots.GetRef(slotId)) 
    } 
}
```
根据字段类型，从静态变量中取出相应的值，然后推入操作数栈顶。至此，getstatic指令也解释完毕了。下面介绍如何存取对象的实例变量。
##### 6.6.3 putfield和getfield指令 
在references目录下创建putfield.go文件，在其中实现putfield指令，代码如下：
```go
package references 
import "jvmgo/ch06/instructions/base" 
import "jvmgo/ch06/rtda" 
import "jvmgo/ch06/rtda/heap" 
// Set field in object 
type PUT_FIELD struct{ base.Index16Instruction }
``` 
putfield指令给实例变量赋值，它需要三个操作数。前两个操作数是常量池索引和变量值，用法和putstatic一样。第三个操作数是对象引用，从操作数栈中弹出。同样分三次来介绍putfield指令的Execute()方法，第一部分代码如下：
```go
func (self *PUT_FIELD) Execute(frame *rtda.Frame) { 
    currentMethod := frame.Method() 
    currentClass := currentMethod.Class() 
    cp := currentClass.ConstantPool() 
    fieldRef := cp.GetConstant(self.Index).(*heap.FieldRef) 
    field := fieldRef.ResolvedField() 
```

基本上和putstatic一样，这里就不多解释了。看下一部分： 
```go
    if field.IsStatic() { 
        panic("java.lang.IncompatibleClassChangeError") 
    }
    if field.IsFinal() {
        if currentClass != field.Class() || currentMethod.Name() != "<init>" { 
            panic("java.lang.IllegalAccessError") 
        } 
    } 
```

看起来也和putstatic差不多，但有两点不同(在代码中已经加粗)。第一，解析后的字段必须是实例字段，否则抛出IncompatibleClassChangeError。第二，如果是final字段，则只能在构造函数中初始化，否则抛出IllegalAccessError。在第7章会介绍构造函数。下面看剩下的代码： 
```go
    descriptor := field.Descriptor() 
    slotId := field.SlotId() 
    stack := frame.OperandStack() 
    switch descriptor[0] { 
        case 'Z', 'B', 'C', 'S', 'I': 
        val := stack.PopInt() 
        ref := stack.PopRef() 
        if ref == nil { 
            panic("java.lang.NullPointerException") 
        }
        ref.Fields().SetInt(slotId, val) 
        case 'F': ... 
        case 'J': ... 
        case 'D': ... 
        case 'L', '[': ... 
    } 
}
```

先根据字段类型从操作数栈中弹出相应的变量值，然后弹出对象引用。如果引用是null，需要抛出著名的空指针异常(NullPointerException)，否则通过引用给实例变量赋值。其他的case语句和第一个大同小异，为了节约篇幅，省略了详细代码。 
putfield指令解释完毕，下面来看getfield指令。在references目录下创建getfield.go文件，在其中实现getfield指令，代码如下：
```go
package references 
import "jvmgo/ch06/instructions/base" 
import "jvmgo/ch06/rtda" 
import "jvmgo/ch06/rtda/heap" 
// Fetch field from object 
type GET_FIELD struct{ base.Index16Instruction } 
```
getfield指令获取对象的实例变量值，然后推入操作数栈，它需要两个操作数。第一个操作数是uint16索引，用法和前面三个指令一样。第二个操作数是对象引用，用法和putfield一样。下面看看getfield指令的Execute方法()，第一部分代码如下：
```go
func (self *GET_FIELD) Execute(frame *rtda.Frame) { 
    cp := frame.Method().Class().ConstantPool() 
    fieldRef := cp.GetConstant(self.Index).(*heap.FieldRef) 
    field := fieldRef.ResolvedField() 
    if field.IsStatic() { 
        panic("java.lang.IncompatibleClassChangeError") 
    }
```

先是字段符号引用解析。这部分逻辑我们已经很熟悉了，不多解释。下面是第二部分代码： 
```go
    stack := frame.OperandStack()ref := stack.PopRef() 
    if ref == nil { 
        panic("java.lang.NullPointerException") 
    } 
```

弹出对象引用，如果是null，则抛出NullPointerException。剩下的代码如下：
```go
    descriptor := field.Descriptor() 
    slotId := field.SlotId() 
    slots := ref.Fields() 
    switch descriptor[0] { 
        case 'Z', 'B', 'C', 'S', 'I': stack.PushInt(slots.GetInt(slotId)) 
        case 'F': stack.PushFloat(slots.GetFloat(slotId)) 
        case 'J': stack.PushLong(slots.GetLong(slotId)) 
        case 'D': stack.PushDouble(slots.GetDouble(slotId)) 
        case 'L', '[': stack.PushRef(slots.GetRef(slotId)) 
    } 
} 
```

根据字段类型，获取相应的实例变量值，然后推入操作数栈。至此，getfield指令也解释完毕了。下面讨论instanceof和checkcast指令。
##### 6.6.4 instanceof和checkcast指令 
instanceof指令判断对象是否是某个类的实例(或者对象的类是否实现了某个接口)，并把结果推入操作数栈。在references目录下创建instanceof.go文件，在其中实现instanceof指令，代码如下：
```go
package references 
import "jvmgo/ch06/instructions/base" 
import "jvmgo/ch06/rtda" 
import "jvmgo/ch06/rtda/heap" 
// Determine if object is of given type 
type INSTANCE_OF struct{ base.Index16Instruction } 
```
instanceof指令需要两个操作数。第一个操作数是uint16索引，从方法的字节码中获取，通过这个索引可以从当前类的运行时常量池中找到一个类符号引用。第二个操作数是对象引用，从操作数栈中弹出。instanceof指令的Execute()方法如下：
```go
func (self *INSTANCE_OF) Execute(frame *rtda.Frame) { 
    stack := frame.OperandStack() 
    ref := stack.PopRef() 
    if ref == nil { 
        stack.PushInt(0) 
        return 
    }
    cp := frame.Method().Class().ConstantPool() 
    classRef := cp.GetConstant(self.Index).(*heap.ClassRef) 
    class := classRef.ResolvedClass() 
    if ref.IsInstanceOf(class) { 
        stack.PushInt(1) 
    } else { 
        stack.PushInt(0) 
    }
} 
```
先弹出对象引用，如果是null，则把0推入操作数栈。用Java代码解释就是，如果引用obj是null的话，不管ClassYYY是哪种类型，下面这条if判断都是false：
```go
if (obj instanceof ClassYYY) {...} 
```
如果对象引用不是null，则解析类符号引用，判断对象是否是类的实例，然后把判断结果推入操作数栈。Java虚拟机规范给出了具体的判断步骤，我们在Object结构体的IsInstanceOf()方法中实现，稍后给出代码。下面来看checkcast指令。在references目录下创建checkcast.go文件，在其中实现checkcast指令，代码如下：
```go
package references 
import "jvmgo/ch06/instructions/base" 
import "jvmgo/ch06/rtda" 
import "jvmgo/ch06/rtda/heap" 
// Check whether object is of given type 
type CHECK_CAST struct{ base.Index16Instruction } 
```
checkcast指令和instanceof指令很像，区别在于：instanceof指令会改变操作数栈(弹出对象引用，推入判断结果)；checkcast则不改变操作数栈(如果判断失败，直接抛出ClassCastException异常)。checkcast指令的Execute()方法如下： 
```go
func (self *CHECK_CAST) Execute(frame *rtda.Frame) { 
    stack := frame.OperandStack() 
    ref := stack.PopRef() 
    stack.PushRef(ref) 
    if ref == nil { 
        return 
    }
    cp := frame.Method().Class().ConstantPool() 
    classRef := cp.GetConstant(self.Index).(*heap.ClassRef) 
    class := classRef.ResolvedClass() 
    if !ref.IsInstanceOf(class) { 
        panic("java.lang.ClassCastException") 
    } 
} 
```
先从操作数栈中弹出对象引用，再推回去，这样就不会改变操作数栈的状态。如果引用是null，则指令执行结束。也就是说，null引用可以转换成任何类型，否则解析类符号引用，判断对象是否是类的实例。如果是的话，指令执行结束，否则抛出ClassCastException。instanceof和checkcast指令一般都是配合使用的，像下面的Java代码这样：
```go
    if (xxx instanceof ClassYYY) { 
        yyy = (ClassYYY) xxx; 
        // use yyy 
    } 
```
Object结构体的IsInstanceOf()方法的代码如下(在object.go文件中)：
```go
func (self *Object) IsInstanceOf(class *Class) bool { 
    return class.isAssignableFrom(self.class) 
} 
```
真正的逻辑在Class结构体的isAssignableFrom()方法中，这个方法稍微有些复杂，为了避免class.go文件变得过长，把它写在另一个文件中。在ch06\rtda\heap目录下创建class_hierarchy.go文件，在其中定义isAssignableFrom()方法，代码如下：
```go
func (self *Class) isAssignableFrom(other *Class) bool { 
    s, t := other, self 
    if s == t { 
        return true 
    }
    if !t.IsInterface() { 
        return s.isSubClassOf(t) 
    } else { 
        return s.isImplements(t) 
    } 
} 
```
也就是说，在三种情况下，S类型的引用值可以赋值给T类型：S和T是同一类型；T是类且S是T的子类；或者T是接口且S实现了T接口。这是简化版的判断逻辑，因为还没有实现数组，第8章讨论数组时会继续完善这个方法。继续编辑class_hierarchy.go文件，在其中实现isSubClassOf()方法，代码如下：
```go
func (self *Class) isSubClassOf(other *Class) bool { 
    for c := self.superClass; c != nil; c = c.superClass { 
        if c == other {
            return true 
        } 
    }
    return false 
}
``` 
判断S是否是T的子类，实际上也就是判断T是否是S的(直接或间接)超类。继续编辑class_hierarchy.go文件，在其中实现isImplements()方法，代码如下：
```go
func (self *Class) isImplements(iface *Class) bool { 
    for c := self; c != nil; c = c.superClass { 
        for _, i := range c.interfaces { 
            if i == iface || i.isSubInterfaceOf(iface) { 
                return true 
            } 
        } 
    }
    return false 
}
```
判断S是否实现了T接口，就看S或S的(直接或间接)超类是否实现了某个接口T'，T'要么是T，要么是T的子接口。isSubInterfaceOf()方法也在class_hierarchy.go文件中，代码如下：
```go
func (self *Class) isSubInterfaceOf(iface *Class) bool { 
    for _, superInterface := range self.interfaces { 
        if superInterface == iface || superInterface.isSubInterfaceOf(iface) 	{ 
            return true 
        } 
    }
    return false 
}
```

isSubInterfaceOf()方法和isSubClassOf()方法类似，但是用到了递归，这里不多解释了。到此为止，instanceof和checkcast指令就介绍完毕了，下面来看ldc指令。
##### 6.6.5 ldc指令 
ldc系列指令从运行时常量池中加载常量值，并把它推入操作数栈。ldc系列指令属于常量类指令，共3条。其中ldc和ldc_w指令用于加载int、float和字符串常量，java.lang.Class实例或者MethodType和MethodHandle实例。ldc2_w指令用于加载long和double常量。ldc和ldc_w指令的区别仅在于操作数的宽度。 

本章只处理int、float、long和double常量。第8章实现数组和字符串之后，会进一步完善ldc指令，支持字符串常量的加载。第9章还会继续完善ldc指令，支持Class实例的加载。本书不讨论MethodType和MethodHandle，感兴趣的读者请参考Java虚拟机规范的相关章节。 

在ch06\instructions\constants目录下创建ldc.go文件，在其中定义ldc、ldc_w和ldc_2w指令，代码如下：
```go
package constants 
import "jvmgo/ch06/instructions/base" 
import "jvmgo/ch06/rtda" 
type LDC struct{ base.Index8Instruction } 
type LDC_W struct{ base.Index16Instruction } 
type LDC2_W struct{ base.Index16Instruction }
```
ldc和ldc_w指令的逻辑完全一样，在_ldc()函数中实现，代码如下：
```go
func (self *LDC) Execute(frame *rtda.Frame) { 
    _ldc(frame, self.Index) 
}
func (self *LDC_W) Execute(frame *rtda.Frame) { 
    _ldc(frame, self.Index) 
} 
```
_ldc()函数的代码如下：
```go
func _ldc(frame *rtda.Frame, index uint) { 
    stack := frame.OperandStack() 
    cp := frame.Method().Class().ConstantPool() 
    c := cp.GetConstant(index) 
    switch c.(type) { 
        case int32: stack.PushInt(c.(int32)) 
        case float32: stack.PushFloat(c.(float32)) 
        // case string: 在第 8章实现
        // case *heap.ClassRef: 在第 9章实现
        default: panic("todo: ldc!") 
    } 
}
```
先从当前类的运行时常量池中取出常量。如果是int或float常量，则提取出常量值，则推入操作数栈。其他情况还无法处理，暂时调用panic()函数终止程序执行。ldc_2w指令的Execute()方法单独实现，代码如下：
```go
func (self *LDC2_W) Execute(frame *rtda.Frame) { 
    stack := frame.OperandStack() 
    cp := frame.Method().Class().ConstantPool() 
    c := cp.GetConstant(self.Index) 
    switch c.(type) { 
        case int64: stack.PushLong(c.(int64)) 
        case float64: stack.PushDouble(c.(float64)) 
        default: panic("java.lang.ClassFormatError") 
    } 
}
```
代码比较简单，不多解释，这里重点说一下Frame结构体的Method()方法。为了通过frame变量拿到当前类的运行时常量池，给Frame结构体添加了method字段，代码如下： 
```go
type Frame struct { 
    lower *Frame 
    localVars LocalVars 
    operandStack *OperandStack 
    thread *Thread 
    method *heap.Method 
    nextPC int 
} 
```
Method()是Getter方法，就不给出代码了。newFrame()函数有相应变化，代码如下：
```go
func newFrame(thread *Thread, method *heap.Method) *Frame { 
    return &Frame{ 
        thread: thread, 
        method: method, 
        localVars: newLocalVars(method.MaxLocals()), 
        operandStack: newOperandStack(method.MaxStack()),
    } 
}
``` 
到此，类和对象相关的10条指令都实现好了。最后还需要修改ch06\instructions \factory.go文件，在其中添加这些指令的case语句。具体改动也比较简单，这里就不给出代码了。
#### 6.7 测试本章代码 
打开ch06\main.go文件，修改import语句，代码如下：
```go
package main 
import "fmt" 
import "strings" 
import "jvmgo/ch06/classpath" 
import "jvmgo/ch06/rtda/heap" 
```
main()函数不变，删掉其他函数，然后修改startJVM()函数，代码如下：
```go
func startJVM(cmd *Cmd) { 
    cp := classpath.Parse(cmd.XjreOption, cmd.cpOption) 
    classLoader := heap.NewClassLoader(cp) 
    className := strings.Replace(cmd.class, ".", "/", -1) 
    mainClass := classLoader.LoadClass(className) 
    mainMethod := mainClass.GetMainMethod() 
    if mainMethod != nil { 
        interpret(mainMethod) 
    } else { 
        fmt.Printf("Main method not found in class %s\n", cmd.class) 
    } 
} 
```
先创建ClassLoader实例，然后用它来加载主类，最后执行主类的main()方法。Class结构体的GetMainMethod()方法如下(在ch06\rtda\heap\class.go文件中)： 
```go
func (self *Class) GetMainMethod() *Method { 
    return self.getStaticMethod("main", "([Ljava/lang/String;)V")
}
``` 
它只是调用了getStaticMethod()方法而已，代码如下：
```go
func (self *Class) getStaticMethod(name, descriptor string) *Method { 
    for _, method := range self.methods { 
        if method.IsStatic() && method.name == name && method.descriptor == descriptor { 
            return method 
        } 
    }
    return nil
} 
```
接下来编辑ch06\interpreter.go文件，修改import语句，代码如下：
```go
package main 
import "fmt" 
import "jvmgo/ch06/instructions" 
import "jvmgo/ch06/instructions/base" 
import "jvmgo/ch06/rtda" 
import "jvmgo/ch06/rtda/heap" 
```
其他函数不变，只修改interpret()函数，代码如下：
```go
func interpret(method *heap.Method) { 
    thread := rtda.NewThread() 
    frame := thread.NewFrame(method) 
    thread.PushFrame(frame) 
    defer catchErr(frame) 
    loop(thread, method.Code()) 
}
```
Thread结构体的NewFrame()方法如下(在ch06\rtda\thread.go文件中)： 
```go
func (self *Thread) NewFrame(method *heap.Method) *Frame { 
    return newFrame(self, method) 
} 
```
在编译本章代码之前，还需要添加两个hack。因为对象是需要初始化的，所以每个类都至少有一个构造函数。即使用户自己不定义，编译器也会自动生成一个默认构造函数。在创建类实例时，编译器会在new指令的后面加入invokespecial指令来调用构造函数初始化对象。要到第7章才会实现invokespecial指令，为了测试putfield和getfield等指令，这里先给它一个空的实现。在ch06\instructions\references目录下创建nvokespecial.go文件，把下面的代码复制进去：
```go
package references 
import "jvmgo/ch06/instructions/base" 
import "jvmgo/ch06/rtda" 
type INVOKE_SPECIAL struct{ base.Index16Instruction } 
// hack! 
func (self *INVOKE_SPECIAL) Execute(frame *rtda.Frame) { 
    frame.OperandStack().PopRef() 
}
```
第5章通过打印局部变量表和操作数栈的方式观察计算结果，这样很不方便。这里用另外一个hack来解决这个问题。

在ch06\instructions\references目录下创建invokevirtual.go文件，把下面的代码复制进去： 
```go
package references 
import "fmt" 
import "jvmgo/ch06/instructions/base" 
import "jvmgo/ch06/rtda" 
import "jvmgo/ch06/rtda/heap" 
// Invoke instance method; dispatch based on class 
type INVOKE_VIRTUAL struct{ base.Index16Instruction } 
// hack! 
func (self *INVOKE_VIRTUAL) Execute(frame *rtda.Frame) { 
    cp := frame.Method().Class().ConstantPool() 
    methodRef := cp.GetConstant(self.Index).(*heap.MethodRef) 
    if methodRef.Name() == "println" { 
        stack := frame.OperandStack() 
        switch methodRef.Descriptor() { 
            case "(Z)V": fmt.Printf("%v\n", stack.PopInt() != 0) 
            case "(C)V": fmt.Printf("%c\n", stack.PopInt()) 
            case "(B)V": fmt.Printf("%v\n", stack.PopInt()) 
            case "(S)V": fmt.Printf("%v\n", stack.PopInt()) 
            case "(I)V": fmt.Printf("%v\n", stack.PopInt()) 
            case "(J)V": fmt.Printf("%v\n", stack.PopLong()) 
            case "(F)V": fmt.Printf("%v\n", stack.PopFloat()) 
            case "(D)V": fmt.Printf("%v\n", stack.PopDouble()) 
            default: panic("println: " + methodRef.Descriptor()) 
        }
        stack.PopRef() 
    } 
}
```
至于这两个hack为什么可以起作用，请阅读第7章，在那里会讨论方法调用和返回。有了上面的hack，可以修改6.6小节开头给出的Java例子，添加输出语句，代码如下：
```go
package jvmgo.book.ch06; 
public class MyObject {
    public static int staticVar; 
    public int instanceVar; 
    public static void main(String[] args) { 
        int x = 32768; // ldc 
        MyObject myObj = new MyObject(); // new 
        MyObject.staticVar = x; // putstatic 
        x = MyObject.staticVar; // getstatic 
        myObj.instanceVar = x; // putfield 
        x = myObj.instanceVar; // getfield 
        Object obj = myObj; 
        if (obj instanceof MyObject) { // instanceof 
            myObj = (MyObject) obj; // checkcast 
            System.out.println(myObj.instanceVar); 
        } 
    } 
} 
```
打开命令行窗口，执行下面的命令编译本章代码： 
```shell script
go install jvmgo\ch06 
```

命令执行完毕后，D：\go\workspace\bin目录会出现ch06.exe文件。用javac编译MyObject类，然后用ch06.exe执行MyObject程序，结果如图6-3和图6-4所示。 
![6-3](./img/6-3.png)  
图6-3 MyObject程序执行结果(1)
![6-4](./img/6-4.png)  
图6-4 MyObject程序执行结果(2)
#### 6.8 本章小结 
本章实现了方法区、运行时常量池、类和对象结构体、一个简单的类加载器，以及ldc和部分引用类指令。下一章将讨论方法调用和返回，到时就可以执行更加复杂的方法了。

