第3章 解析class文件
=====

::: tip 
第2章介绍了Java虚拟机从哪里搜索class文件，并且实现了类路径功能，已经可以把class文件读取到内存中。本章将详细讨论class文件格式，编写代码解析class文件，为下一步真正实现Java虚拟机做好准备。 
:::

在开始阅读本章之前，先把目录结构准备好。复制ch02目录，并改名为ch03，然后编辑ch03\main.go等文件，把import语句中的ch02都改成ch03 [1] ，最后在ch03目录中创建classfile子目录。现在的目录结构看起来应该如下所示： 
```text
D:\go\workspace\src 
  |-jvmgo 
    |-ch01 
    |-ch02 
    |-ch03 
      |-classfile 
      |-classpath 
      |-cmd.go 
      |-main.go 
```

[1]: 这个过程比较无趣，也容易出错。可以使用编辑器提供的“搜索和替换”功能来完成这项工作。

#### 3.1 class文件 
作为类（或者接口） [1] 信息的载体，每个class文件都完整地定义了一个类。为了使Java程序可以“编写一次，处处运行”，Java虚拟机规范对class文件格式进行了严格的规定。但是另一方面，对于从哪里加载class文件，给了足够多的自由。由第2章可知，Java虚拟机实现可以从文件系统读取和从JAR（或ZIP）压缩包中提取class文件。除此之外，也可以通过网络下载、从数据库加载，甚至是在运行中直接生成class文件。Java虚拟机规范（和本书）中所指的class文件，并非特指位于磁盘中的.class文件，而是泛指任何格式符合规范的class数据。 

构成class文件的基本数据单位是字节，可以把整个class文件当成一个字节流来处理。稍大一些的数据由连续多个字节构成，这些数据在class文件中以大端（big-endian）方式存储。为了描述class文件格式，Java虚拟机规范定义了u1、u2和u4三种数据类型来表示1、2和4字节无符号整数，分别对应Go语言的uint8、uint16和uint32类型。相同类型的多条数据一般按表（table）的形式存储在class文件中。表由表头和表项（item）构成，表头是u2或u4整数。假设表头是n，后面就紧跟着n个表项数据。 

Java虚拟机规范使用一种类似C语言的结构体语法来描述class文件格式。整个class文件被描述为一个ClassFile结构，代码如下：
```java
ClassFile { 
    u4 				magic; 
    u2 				minor_version; 
    u2 				major_version; 
    u2 				constant_pool_count; 
    cp_info 			constant_pool[constant_pool_count-1]; 
    u2 				access_flags; 
    u2 				this_class; 
    u2 				super_class; 
    u2 				interfaces_count; 
    u2 				interfaces[interfaces_count]; 
    u2 				fields_count; 
    field_info 		        fields[fields_count]; 
    u2 				methods_count; 
    method_info 		methods[methods_count]; 
    u2 				attributes_count; 
    attribute_info 		attributes[attributes_count]; 
} 
``` 

JDK提供了一个功能强大的命令行工具javap，可以用它反编译class文件。不过从控制台观察javap的输出并不是很直观，因此笔者用JavaFX编写了一个图形化的工具，叫作classpy。有兴趣的读者可以去GitHub网站 [2] 下载classpy的源代码或者打包好的JAR可执行文件。后面的小节中将以ClassFileTest类为例，使用classpy程序分析class文件格式。

ClassFileTest的代码如下：
```java
package jvmgo.book.ch03;
public class ClassFileTest { 
    public static final boolean FLAG = true; 
    public static final byte BYTE = 123; 
    public static final char X = 'X'; 
    public static final short SHORT = 12345; 
    public static final int INT = 123456789; 
    public static final long LONG = 12345678901L; 
    public static final float PI = 3.14f; 
    public static final double E = 2.71828; 
    public static void main(String[] args) throws RuntimeException { 
        System.out.println("Hello, World!"); 
    } 
} 
```
[1]: 本章后面提到的类，如无特别说明，均泛指类或者接口。 
[2]: https://github.com/zxh0/classpy。

#### 3.2 解析class文件 
本节将一边讨论class文件格式，一边编写代码实现class文件解析。Go语言内置了丰富的数据类型，非常适合处理class文件。为了便于读者参考，表3-1给出了Go和Java语言基本数据类型对照关系。在第4章中还会继续讨论Java数据类型。

表3-1 Go和Java语言基本数据类型对照关系

##### 3.2.1 读取数据 
解析class文件的第一步是从里面读取数据。虽然可以把class文件当成字节流来处理，但是直接操作字节很不方便，所以先定义一个结构体来帮助读取数据。

在ch03\classfile目录下创建class_reader.go文件，在其中定义ClassReader结构体和数据读取方法，代码如下：
```go
package classfile 
import "encoding/binary" 
type ClassReader struct { 
    data []byte 
}
func (self *ClassReader) readUint8() uint8 {...} // u1 
func (self *ClassReader) readUint16() uint16 {...} // u2 
func (self *ClassReader) readUint32() uint32 {...} // u4 
func (self *ClassReader) readUint64() uint64 {...} 
func (self *ClassReader) readUint16s() []uint16 {...} 
func (self *ClassReader) readBytes(length uint32) []byte {...}
 ```
ClassReader只是[]byte类型的包装而已。readUint8（）读取u1类型数据，代码如下： 
```go
func (self *ClassReader) readUint8() uint8 { 
    val := self.data[0] 
    self.data = self.data[1:] 
    return val 
}
```
注意，ClassReader并没有使用索引记录数据位置，而是使用Go语言的reslice语法跳过已经读取的数据。readUint16（）读取u2类型数据，代码如下： 
```go
func (self *ClassReader) readUint16() uint16 { 
    val := binary.BigEndian.Uint16(self.data) 
    self.data = self.data[2:] 
    return val 
}
```
Go标准库encoding/binary包中定义了一个变量BigEndian，正好可以从[]byte中解码多字节数据。readUint32（）读取u4类型数据，代码如下： 
```go
func (self *ClassReader) readUint32() uint32 { 
    val := binary.BigEndian.Uint32(self.data) 
    self.data = self.data[4:] 
    return val 
}
``` 
readUint64（）读取uint64（Java虚拟机规范并没有定义u8）类型数据，代码如下：
```go 
func (self *ClassReader) readUint64() uint64 { 
    val := binary.BigEndian.Uint64(self.data) 
    self.data = self.data[8:] 
    return val 
}
```
readUint16s（）读取uint16表，表的大小由开头的uint16数据指出，代码如下： 
```go
func (self *ClassReader) readUint16s() []uint16 { 
    n := self.readUint16() 
    s := make([]uint16, n) 
    for i := range s { 
        s[i] = self.readUint16() 
    }
    return s 
}
```
最后一个方法是readBytes（），用于读取指定数量的字节，代码如下： 
```go
func (self *ClassReader) readBytes(n uint32) []byte { 
    bytes := self.data[:n] 
    self.data = self.data[n:] 
    return bytes 
}
```
##### 3.2.2 整体结构 
有了ClassReader，可以开始解析class文件了。在ch03\classfile目录下创建class_file.go文件，在其中定义ClassFile结构体，代码如下：
```go
package classfile 
import "fmt" 
type ClassFile struct { 
    //magic		  uint32 
    minorVersion  uint16 
    majorVersion  uint16 
    constantPool  ConstantPool 
    AccessFlags	  uint16 
    ThisClass	  uint16 
    SuperClass	  uint16 
    Interfaces	  []uint16 
    fields        []*MemberInfo 
    methods       []*MemberInfo 
    attributes    []AttributeInfo 
}
```
ClassFile结构体如实反映了Java虚拟机规范定义的class文件格式。还会在class_file.go文件中实现一系列函数和方法，列举如下： 
```go
func Parse(classData []byte) (cf *ClassFile, err error) {...} 
func (self *ClassFile) read(reader *ClassReader) {...} 
func (self *ClassFile) readAndCheckMagic(reader *ClassReader) {...} 
func (self *ClassFile) readAndCheckVersion(reader *ClassReader) {...} 
func (self *ClassFile) MinorVersion() uint16 {...} // getter 
func (self *ClassFile) MajorVersion() uint16 {...} // getter 
func (self *ClassFile) ConstantPool() ConstantPool {...} // getter 
func (self *ClassFile) AccessFlags() uint16 {...} // getter 
func (self *ClassFile) Fields() []*MemberInfo {...} // getter 
func (self *ClassFile) Methods() []*MemberInfo {...} // getter 
func (self *ClassFile) ClassName() string {...} 
func (self *ClassFile) SuperClassName() string {...}func (self *ClassFile) InterfaceNames() []string {...} 
```
相比Java语言，Go的访问控制非常简单：只有公开和私有两种。所有首字母大写的类型、结构体、字段、变量、函数、方法等都是公开的，可供其他包使用。首字母小写则是私有的，只能在包内部使用。在本书的代码中，尽量只公开必要的变量、字段、函数和方法等。但是为了提高代码可读性，所有的结构体都是公开的，也就是首字母是大写的。 

Parse（）函数把[]byte解析成ClassFile结构体，代码如下： 
```go
func Parse(classData []byte) (cf *ClassFile, err error) { 
    defer func() { 
        if r := recover(); r != nil { 
            var ok bool 
            err, ok = r.(error) 
            if !ok { 
                err = fmt.Errorf("%v", r) 
            } 
        } 
    }() 
    cr := &ClassReader{classData} 
    cf = &ClassFile{} 
    cf.read(cr) 
    return 
} 
```
Go语言没有异常处理机制，只有一个panic-recover机制。read（）方法依次调用其他方法解析class文件，代码如下：
```go
func (self *ClassFile) read(reader *ClassReader) { 
    self.readAndCheckMagic(reader) // 见3.2.3
    self.readAndCheckVersion(reader) // 见3.2.4
    self.constantPool = readConstantPool(reader) // 见3.3
    self.accessFlags = reader.readUint16() 
    self.thisClass = reader.readUint16() 
    self.superClass = reader.readUint16() 
    self.interfaces = reader.readUint16s() 
    self.fields = readMembers(reader, self.constantPool) //见3.2.8
    self.methods = readMembers(reader, self.constantPool) 
    self.attributes = readAttributes(reader, self.constantPool) //见3.4 
} 
```
MajorVersion（）等6个方法是Getter方法，把结构体的字段暴露给其他包使用。MajorVersion（）的代码如下：
```go
func (self *ClassFile) MajorVersion() uint16 { 
    return self.majorVersion 
}
```
和Java有所不同，Go的Getter方法不以“get”开头。由于Getter方法非常简单，只是返回字段而已，为了节约篇幅，后文中不再给出Getter方法的代码。ClassName（）从常量池查找类名，代码如下：
```go
func (self *ClassFile) ClassName() string { 
    return self.constantPool.getClassName(self.thisClass) 
}
```
SuperClassName（）从常量池查找超类名，代码如下：
```go
func (self *ClassFile) SuperClassName() string { 
    if self.superClass > 0 { 
        return self.constantPool.getClassName(self.superClass) 
    }
    return "" // 只有 java.lang.Object没有超类 
}
```
 
InterfaceNames（）从常量池查找接口名，代码如下：
```go
func (self *ClassFile) InterfaceNames() []string { 
    interfaceNames := make([]string, len(self.interfaces)) 
    for i, cpIndex := range self.interfaces { 
        interfaceNames[i] = self.constantPool.getClassName(cpIndex) 
    }
    return interfaceNames 
}
``` 
下面详细介绍class文件的各个部分（常量池和属性表比较复杂，放到3.3和3.4节单独讨论）。
##### 3.2.3 魔数 
很多文件格式都会规定满足该格式的文件必须以某几个固定字节开头，这几个字节主要起标识作用，叫作魔数（magic number）。 例如PDF文件以4字节“%PDF”（0x25、0x50、0x44、0x46）开头，ZIP文件以2字节“PK”（0x50、0x4B）开头。class文件的魔数是“0xCAFEBABE”。readAndCheckMagic（）方法的代码如下： 
```go
func (self *ClassFile) readAndCheckMagic(reader *ClassReader) { 
    magic := reader.readUint32() 
    if magic != 0xCAFEBABE { 
        panic("java.lang.ClassFormatError: magic!") 
    } 
}
```
Java虚拟机规范规定，如果加载的class文件不符合要求的格式，Java虚拟机实现就抛出java.lang.ClassFormatError异常。但是因为我们才刚刚开始编写虚拟机，还无法抛出异常，所以暂时先调用panic（）方法终止程序执行。用classpy打开ClassFileTest.class文件，可以看到，开头4字节确实是0xCAFEBABE，如图3-1所示。
![3-1](/img/3-1.png)
图3-1 用classpy观察魔数

##### 3.2.4 版本号 
魔数之后是class文件的次版本号和主版本号，都是u2类型。假设某class文件的主版本号是M，次版本号是m，那么完整的版本号可以表示成“M.m”的形式。次版本号只在J2SE 1.2之前用过，从1.2 开始基本上就没什么用了（都是0）。主版本号在J2SE 1.2之前是45，从1.2开始，每次有大的Java版本发布，都会加1。表3-2列出了到本书写作为止，使用过的class文件版本号。

表3-2 class文件版本号
 
特定的Java虚拟机实现只能支持版本号在某个范围内的class文件。Oracle的实现是完全向后兼容的，比如Java SE 8支持版本号为45.0~52.0的class文件。如果版本号不在支持的范围内，Java虚拟机实现就抛出java.lang.UnsupportedClassVersionError异常。我们参考Java 8，支持版本号为45.0~52.0的class文件。如果遇到其他版本号，暂时先调用panic（）方法终止程序执行。下面是readAndCheckVersion（）方法的代码。
```go
func (self *ClassFile) readAndCheckVersion(reader *ClassReader) { 
    self.minorVersion = reader.readUint16() 
    self.majorVersion = reader.readUint16() 
    switch self.majorVersion { 
        case 45: 
        return 
        case 46, 47, 48, 49, 50, 51, 52: 
        if self.minorVersion == 0 { 
            return 
        } 
    }
    panic("java.lang.UnsupportedClassVersionError!") 
} 
```
因为笔者使用JDK8编译ClassFileTest类，所以主版本号是52（0x34），次版本号是0，如图3-2所示。 
![3-2](/img/3-2.png)
图3-2 用classpy观察版本号

##### 3.2.5 类访问标志 
版本号之后是常量池，但是由于常量池比较复杂，所以放到3.3节介绍。常量池之后是类访问标志，这是一个16位的“bitmask”，指出class文件定义的是类还是接口，访问级别是public还是private，等等。本章只对class文件进行初步解析，并不做完整验证，所以只是读取类访问标志以备后用。第6章会详细讨论访问标志。 

ClassFileTest的类访问标志是0x21，如图3-3所示。
![3-3](/img/3-3.png) 
图3-3 用classpy观察类访问标志
##### 3.2.6 类和超类索引 
类访问标志之后是两个u2类型的常量池索引，分别给出类名和超类名。class文件存储的类名类似完全限定名，但是把点换成了斜线，Java语言规范把这种名字叫作二进制名（binary names）。因为每个类都有名字，所以thisClass必须是有效的常量池索引。除 

java.lang.Object之外，其他类都有超类，所以superClass只在Object.class中是0，在其他class文件中必须是有效的常量池索引。如图3-4所示，ClassFileTest的类索引是5，超类索引是6。
![3-4](/img/3-4.png)
图3-4 用classpy观察类和超类索引
##### 3.2.7 接口索引表 
类和超类索引后面是接口索引表，表中存放的也是常量池索引，给出该类实现的所有接口的名字。ClassFileTest没有实现接口，所以接口表是空的，如图3-5所示。 
![3-5](/img/3-5.png)
图3-5 用classpy观察接口索引表

##### 3.2.8 字段和方法表 
接口索引表之后是字段表和方法表，分别存储字段和方法信息。字段和方法的基本结构大致相同，差别仅在于属性表。下面是Java虚拟机规范给出的字段结构定义。 
```go
field_info { 
    u2 access_flags; 
    u2 name_index; 
    u2 descriptor_index; 
    u2 attributes_count; 
    attribute_info attributes[attributes_count]; 
}
``` 
和类一样，字段和方法也有自己的访问标志。访问标志之后是一个常量池索引，给出字段名或方法名，然后又是一个常量池索引，给出字段或方法的描述符，最后是属性表。为了避免重复代码，用一个结构体统一表示字段和方法。在ch03\classfile目录中创建member_info.go文件，在其中定义MemberInfo结构体，代码如下： 
```go
package classfile 
type MemberInfo struct { 
    cp ConstantPool 
    accessFlags uint16 
    nameIndex uint16 
    descriptorIndex uint16 
    attributes []AttributeInfo 
}
func readMembers(reader *ClassReader, cp ConstantPool) []*MemberInfo {...} 
func readMember(reader *ClassReader, cp ConstantPool) *MemberInfo {...} 
func (self *MemberInfo) AccessFlags() uint16 {...} // getter
func (self *MemberInfo) Name() string {...} 
func (self *MemberInfo) Descriptor() string {...} 
```
cp字段保存常量池指针，后面会用到它。readMembers（）读取字段表或方法表，代码如下： 
```go
func readMembers(reader *ClassReader, cp ConstantPool) []*MemberInfo { 
    memberCount := reader.readUint16() 
    members := make([]*MemberInfo, memberCount) 
    for i := range members { 
        members[i] = readMember(reader, cp) 
    }
    return members 
}
```
readMember（）函数读取字段或方法数据，代码如下： 
```go
func readMember(reader *ClassReader, cp ConstantPool) *MemberInfo { 
    return &MemberInfo{ 
        cp: cp, 
        accessFlags: reader.readUint16(), 
        nameIndex: reader.readUint16(), 
        descriptorIndex: reader.readUint16(), 
        attributes: readAttributes(reader, cp), // 见 3.4
    }
}
```
属性表和readAttributes（）函数将在3.4节介绍。Name（）从常量池查找字段或方法名，Descriptor（）从常量池查找字段或方法描述符，代码如下：
```go
func (self *MemberInfo) Name() string {
    return 	self.cp.getUtf8(self.nameIndex) 
}
func (self *MemberInfo) Descriptor() string { 
    return self.cp.getUtf8(self.descriptorIndex) 
}
```

第6章会进一步讨论字段和方法。ClassFileTest有8个字段和两个方法（其中`<init>`是编译器生成的默认构造函数），如图3-6和图3-7所示。
![3-6](/img/3-6.png)
图3-6 用classpy观察字段表
![3-7](/img/3-7.png)
图3-7 用classpy观察方法表
#### 3.3 解析常量池 
常量池占据了class文件很大一部分数据，里面存放着各式各样的常量信息，包括数字和字符串常量、类和接口名、字段和方法名，等等。本节将详细介绍常量池和各种常量。
##### 3.3.1 ConstantPool结构体 
在ch03\classfile目录下创建constant_pool.go文件，在里面定义ConstantPool类型，代码如下所示：
```go
package classfile 
type ConstantPool []ConstantInfo 
func readConstantPool(reader *ClassReader) ConstantPool {...} 
func (self ConstantPool) getConstantInfo(index uint16) ConstantInfo {...} 
func (self ConstantPool) getNameAndType(index uint16) (string, string) {...} 
func (self ConstantPool) getClassName(index uint16) string {...} 
func (self ConstantPool) getUtf8(index uint16) string {...} 
```
常量池实际上也是一个表，但是有三点需要特别注意。第一，表头给出的常量池大小比实际大1。假设表头给出的值是n，那么常量池的实际大小是n–1。第二，有效的常量池索引是1~n–1。0是无效索引，表示不指向任何常量。第三，CONSTANT_Long_info和CONSTANT_Double_info各占两个位置。也就是说，如果常量池中存在这两种常量，实际的常量数量比n–1还要少，而且1~n–1的某些数也会变成无效索引。常量池由readConstantPool（）函数读取，代码如下： 
```go
func readConstantPool(reader *ClassReader) ConstantPool { 
    cpCount := int(reader.readUint16()) 
    cp := make([]ConstantInfo, cpCount) 
    for i := 1; i < cpCount; i++ { // 注意索引从1开始 
        cp[i] = readConstantInfo(reader, cp) 
        switch cp[i].(type) { 
            case *ConstantLongInfo, *ConstantDoubleInfo: 
            i++ // 占两个位置 
        } 
    }
    return cp 
}
```
getConstantInfo（）方法按索引查找常量，代码如下： 
```go
func (self ConstantPool) getConstantInfo(index uint16) ConstantInfo { 
    if cpInfo := self[index]; cpInfo != nil { 
        return cpInfo 
    }
    panic("Invalid constant pool index!") 
}
```
getNameAndType（）方法从常量池查找字段或方法的名字和描述符，代码如下：
```go
func (self ConstantPool) getNameAndType(index uint16) (string, string) { 
    ntInfo := self.getConstantInfo(index).(*ConstantNameAndTypeInfo) 
    name := self.getUtf8(ntInfo.nameIndex) 
    _type := self.getUtf8(ntInfo.descriptorIndex) 
    return name, _type 
}
```
getClassName（）方法从常量池查找类名，代码如下： 
```go
func (self ConstantPool) getClassName(index uint16) string { 
    classInfo := self.getConstantInfo(index).(*ConstantClassInfo)return self.getUtf8(classInfo.nameIndex) 
}
``` 
getUtf8（）方法从常量池查找UTF-8字符串，代码如下：
```go
func (self ConstantPool) getUtf8(index uint16) string { 
    utf8Info :=self.getConstantInfo(index).(*ConstantUtf8Info) 
    return utf8Info.str 
} 
```
ClassFileTest的常量池大小是61如图3-8所示。 
![3-8](/img/3-8.png)
图3-8 用classpy观察常量池大小
##### 3.3.2 ConstantInfo接口 
由于常量池中存放的信息各不相同，所以每种常量的格式也不同。常量数据的第一字节是tag，用来区分常量类型。下面是Java虚拟机规范给出的常量结构。 
```go
cp_info { 
    u1 tag; 
    u1 info[]; 
}
```
Java虚拟机规范一共定义了14种常量。在ch03\classfile目录下创建constant_info.go文件，在其中定义tag常量值，代码如下： 
```go
package classfile 
// tag常量值定义 
const ( 
    CONSTANT_Class = 7 
    CONSTANT_Fieldref = 9 
    CONSTANT_Methodref = 10 
    CONSTANT_InterfaceMethodref = 11 
    CONSTANT_String = 8 
    CONSTANT_Integer = 3 
    CONSTANT_Float = 4 
    CONSTANT_Long = 5 
    CONSTANT_Double = 6 
    CONSTANT_NameAndType = 12 
    CONSTANT_Utf8 = 1 
    CONSTANT_MethodHandle = 15 
    CONSTANT_MethodType = 16 
    CONSTANT_InvokeDynamic = 18 
)
```
继续编辑constant_pool.go，定义ConstantInfo接口来表示常量信息，代码如下：
```go 
type ConstantInfo interface { 
    readInfo(reader *ClassReader) 
}
func readConstantInfo(reader *ClassReader, cp ConstantPool) ConstantInfo {...} 
func newConstantInfo(tag uint8, cp ConstantPool) ConstantInfo {...}
``` 
readInfo（）方法读取常量信息，需要由具体的常量结构体实现。readConstantInfo（）函数先读出tag值，然后调用newConstantInfo（）函数创建具体的常量，最后调用常量的readInfo（）方法读取常量信息，代码如下：
```go 
func readConstantInfo(reader *ClassReader, cp ConstantPool) ConstantInfo { 
    tag := reader.readUint8() 
    c := newConstantInfo(tag, cp) 
    c.readInfo(reader) 
    return c 
}
```
 newConstantInfo（）根据tag值创建具体的常量，代码如下： 
```go
func newConstantInfo(tag uint8, cp ConstantPool) ConstantInfo { 
    switch tag { 
        case CONSTANT_Integer: return &ConstantIntegerInfo{} 
        case CONSTANT_Float: return &ConstantFloatInfo{} 
        case CONSTANT_Long: return &ConstantLongInfo{} 
        case CONSTANT_Double: return &ConstantDoubleInfo{} 
        case CONSTANT_Utf8: return &ConstantUtf8Info{} 
        case CONSTANT_String: return &ConstantStringInfo{cp: cp} 
        case CONSTANT_Class: return &ConstantClassInfo{cp: cp} 
        case CONSTANT_Fieldref:return &ConstantFieldrefInfo{ConstantMemberrefInfo{cp: cp}} 
        case CONSTANT_Methodref: 
        return &ConstantMethodrefInfo{ConstantMemberrefInfo{cp: cp}} 
        case CONSTANT_InterfaceMethodref: 
        return &ConstantInterfaceMethodrefInfo{ConstantMemberrefInfo{cp: cp}} 
        case CONSTANT_NameAndType: return &ConstantNameAndTypeInfo{} 
        case CONSTANT_MethodType: return &ConstantMethodTypeInfo{} 
        case CONSTANT_MethodHandle: return &ConstantMethodHandleInfo{} 
        case CONSTANT_InvokeDynamic: return &ConstantInvokeDynamicInfo{} 
        default: panic("java.lang.ClassFormatError: constant pool tag!") 
    } 
} 
```
下面的小节详细介绍各种常量。
##### 3.3.3 CONSTANT_Integer_info
CONSTANT_Integer_info使用4字节存储整数常量，其结构定义如下： 
```go
CONSTANT_Integer_info { 
    u1 tag; 
    u4 bytes; 
} 
```

CONSTANT_Integer_info和后面将要介绍的其他三种数字常量无论是结构，还是实现，都非常相似，所以把它们定义在同一个文件中。在ch03\classfile目录下创建cp_numeric.go文件，在其中定义ConstantIntegerInfo结构体，代码如下：
```go
package classfile 
import "math" 
type ConstantIntegerInfo struct { 
    val int32 
}
func (self *ConstantIntegerInfo) readInfo(reader *ClassReader) {...}
``` 
readInfo（）先读取一个uint32数据，然后把它转型成int32类型， 代码如下： 
```go
func (self *ConstantIntegerInfo) readInfo(reader *ClassReader) { 
    bytes := reader.readUint32() 
    self.val = int32(bytes)
} 
```
CONSTANT_Integer_info正好可以容纳一个Java的int型常量，但实际上比int更小的boolean、byte、short和char类型的常量也放在CONSTANT_Integer_info中。编译器给ClassFileTest类的INT字段生成了一个CONSTANT_Integer_info常量，如图3-9所示。
![3-9](/img/3-9.png) 
图3-9 用classpy观察CONSTANT_Integer_info常量
##### 3.3.4 CONSTANT_Float_info 
CONSTANT_Float_info使用4字节存储IEEE754单精度浮点数常量，结构如下： 
```go
CONSTANT_Float_info { 
    u1 tag; 
    u4 bytes; 
}
```
在cp_numeric.go文件中定义ConstantFloatInfo结构体，代码如下：
```go
type ConstantFloatInfo struct { 
    val float32 
}
func (self *ConstantFloatInfo) readInfo(reader *ClassReader) { 
    bytes := reader.readUint32() 
    self.val = math.Float32frombits(bytes) 
}
```
readInfo（）先读取一个uint32数据，然后调用math包的Float32frombits（）函数把它转换成float32类型。编译器给ClassFileTest类的PI字段生成了一个CONSTANT_Float_info常量，如图3-10所示。
![3-10](/img/3-10.png)
图3-10 用classpy观察CONSTANT_Float_info常量
##### 3.3.5 CONSTANT_Long_info 
CONSTANT_Long_info使用8字节存储整数常量，结构如下： 
```go
CONSTANT_Long_info { 
    u1 tag; 
    u4 high_bytes; 
    u4 low_bytes; 
}
```
 
在cp_numeric.go文件中定义ConstantLongInfo结构体，代码如下：
```go
type ConstantLongInfo struct {
    val int64 
}
func (self *ConstantLongInfo) readInfo(reader *ClassReader) { 
    bytes := reader.readUint64() 
    self.val = int64(bytes) 
}
```
readInfo（）先读取一个uint64数据，然后把它转型成int64类型。编译器给ClassFileTest类的LONG字段生成了一个CONSTANT_Long_info常量，如图3-11所示。
![3-11](/img/3-11.png)
图3-11 用classpy观察CONSTANT_Long_info常量
##### 3.3.6 CONSTANT_Double_info 
最后一个数字常量是CONSTANT_Double_info，使用8字节存储IEEE754双精度浮点数，结构如下： 
```
CONSTANT_Double_info { 
    u1 tag; 
    u4 high_bytes; 
    u4 low_bytes; 
}
```
在cp_numeric.go文件中定义ConstantDoubleInfo结构体，代码如下：
```go
type ConstantDoubleInfo struct { 
    val float64 
}
func (self *ConstantDoubleInfo) readInfo(reader *ClassReader) { 
    bytes := reader.readUint64() 
    self.val = math.Float64frombits(bytes) 
}
```
readInfo（）先读取一个uint64数据，然后调用math包的Float64frombits（）函数把它转换成float64类型。编译器给ClassFileTest类的E字段生成了一个CONSTANT_Double_info常量，如图3-12所示。
![3-12](/img/3-12.png)
图3-12 用classpy观察CONSTANT_Double_info常量
##### 3.3.7 CONSTANT_Utf8_info 
CONSTANT_Utf8_info常量里放的是MUTF-8编码的字符串，结构如下： 
```go
CONSTANT_Utf8_info { 
    u1 tag; 
    u2 length; 
    u1 bytes[length]; 
}
```
注意，字符串在class文件中是以MUTF-8（Modified UTF-8）方式编码的。但为什么没有用标准的UTF-8编码方式，笔者没有找到明确的原因 [1] 。MUTF-8编码方式和UTF-8大致相同，但并不兼容。差别有两点：一是null字符（代码点U+0000）会被编码成2字节： 
0xC0、0x80；二是补充字符（Supplementary Characters，代码点大于U+FFFF的Unicode字符）是按UTF-16拆分为代理对（Surrogate Pair）分别编码的。具体细节超出了本章的讨论范围，有兴趣的读者可以阅读Java虚拟机规范和Unicode规范的相关章节 [2] 。 
在ch03\classfile目录下创建cp_utf8.go文件，在其中定义ConstantUtf8Info结构体，代码如下：
```go
package classfile 
import "fmt" 
import "unicode/utf16" 
type ConstantUtf8Info struct {
    str string 
}
func (self *ConstantUtf8Info) readInfo(reader *ClassReader) {} 
```

readInfo（）方法先读取出[]byte，然后调用decodeMUTF8（）函数把它解码成Go字符串，代码如下： 
```go
func (self *ConstantUtf8Info) readInfo(reader *ClassReader) { 
    length := uint32(reader.readUint16()) 
    bytes := reader.readBytes(length) 
    self.str = decodeMUTF8(bytes) 
}
```
Java序列化机制也使用了MUTF-8编码。java.io.DataInput和java.io.DataOutput接口分别定义了readUTF（）和writeUTF（）方法，可以读写MUTF-8编码的字符串。decodeMUTF8（）函数的代码就是笔者根据java.io.DataInputStream.readUTF（）方法改写的。代码很长，解释起来也很乏味，所以这里就不详细解释了。因为Go语言字符串使用UTF-8编码，所以如果字符串中不包含null字符或补充字符，下面这个简化版的readMUTF8（）也是可以工作的。
```go
// 简化版，完整版请阅读本章源代码 
func decodeMUTF8(bytes []byte) string { 
    return string(bytes) 
}
```

相信细心的读者在前面的截图中已经看到了，字段名、字段描述符等就是以字符串的形式存储在class文件中的，如字段PI对应的CONSTANT_Utf8_info常量，如图3-13所示。 
![3-13](/img/3-13.png)
图3-13 用classpy观察CONSTANT_Utf8_info常量 
[1]:这个链接中有一些线索：http://stackoverflow.com/questions/15440584/why-does-java-use-modified-utf-8-instead-of-utf-8。
[2]:或者这篇文章：http://www.oracle.com/technetwork/articles/javase/supplementary-142654.html。
##### 3.3.8 CONSTANT_String_info 
CONSTANT_String_info常量表示java.lang.String字面量，结构如下： 
```go
CONSTANT_String_info { 
    u1 tag; 
    u2 string_index; 
}
```

可以看到，CONSTANT_String_info本身并不存放字符串数据，只存了常量池索引，这个索引指向一个CONSTANT_Utf8_info常量。在ch03\classfile目录下创建cp_string.go文件，在其中定义ConstantStringInfo结构体，代码如下： 
```go
package classfile 
type ConstantStringInfo struct { 
    cp ConstantPool 
    stringIndex uint16 
}
func (self *ConstantStringInfo) readInfo(reader *ClassReader) {...} 
func (self *ConstantStringInfo) String() string {...} 
```
readInfo（）方法读取常量池索引，代码如下： 
```go
func (self *ConstantStringInfo) readInfo(reader *ClassReader) { 
    self.stringIndex = reader.readUint16() 
}
```
String（）方法按索引从常量池中查找字符串，代码如下： 
```go
func (self *ConstantStringInfo) String() string { 
    return self.cp.getUtf8(self.stringIndex) 
}
```
ClassFileTest的main（）方法使用了字符串字面量“Hello，World！”，对应的CONSTANT_String_info常量如图3-14所示。
![3-14](/img/3-14.png)
图3-14 用classpy观察CONSTANT_String_info常量 

可以看到，string_index是52（0x34）。我们按图索骥，从常量池中找出第52个常量，确实是个CONSTANT_Utf8_info，如图3-15所示。
![3-15](/img/3-15.png) 
图3-15 用classpy观察CONSTANT_String_info常量（2）
##### 3.3.9 CONSTANT_Class_info 
CONSTANT_Class_info常量表示类或者接口的符号引用，结构如下：
```go
CONSTANT_Class_info { 
    u1 tag; 
    u2 name_index; 
}
``` 
和CONSTANT_String_info类似，name_index是常量池索引，指向CONSTANT_Utf8_info常量。在ch03\classfile目录下创建cp_class.go文件，在其中定义ConstantClassInfo结构体，代码如下：
```go
package classfile 
type ConstantClassInfo struct { 
    cp ConstantPool 
    nameIndex uint16 
}
func (self *ConstantClassInfo) readInfo(reader *ClassReader) { 
    self.nameIndex = reader.readUint16() 
}
func (self *ConstantClassInfo) Name() string { 
    return self.cp.getUtf8(self.nameIndex) 
}
```
代码和前一节大同小异，就不多解释了。类和超类索引，以及接口表中的接口索引指向的都是CONSTANT_Class_info常量。由图3-3可知，ClassFileTest的this_class索引是5。我们找到第5个常量，可以看到，的确是CONSTANT_Class_info。它的name_index是55（0x37），如图3-16所示。 

再看第55个常量，也的确是CONSTANT_Utf_info，如图3-17所示。
![3-16](/img/3-16.png)
图3-16 用classpy观察CONSTANT_Class_info常量 
![3-17](/img/3-17.png)
图3-17 用classpy观察CONSTANT_Class_info常量（2）
##### 3.3.10 CONSTANT_NameAndType_info 
CONSTANT_NameAndType_info给出字段或方法的名称和描述符。CONSTANT_Class_info和CONSTANT_NameAndType_info加在一起可以唯一确定一个字段或者方法。其结构如下： 
```go
CONSTANT_NameAndType_info { 
    u1 tag; 
    u2 name_index; 
    u2 descriptor_index; 
}
``` 
字段或方法名由name_index给出，字段或方法的描述符由descriptor_index给出。name_index和descriptor_index都是常量池索引，指向CONSTANT_Utf8_info常量。字段和方法名就是代码中出现的（或者编译器生成的）字段或方法的名字。Java虚拟机规范定义了一种简单的语法来描述字段和方法，可以根据下面的规则生成描述符。
- 1）类型描述符。 
    * ①基本类型byte、short、char、int、long、float和double的描述符是单个字母，分别对应B、S、C、I、J、F和D。注意，long的描述符是J而不是L。
    * ②引用类型的描述符是L＋类的完全限定名＋分号。 
    * ③数组类型的描述符是[＋数组元素类型描述符。 
- 2）字段描述符就是字段类型的描述符。 
- 3）方法描述符是（分号分隔的参数类型描述符）+返回值类型描述符，其中void返回值由单个字母V表示。

更详细的介绍可以参考Java虚拟机规范4.3节。表3-3给出了一些具体的例子。 

表3-3 字段和方法描述符示例 

我们都知道，Java语言支持方法重载（override），不同的方法可以有相同的名字，只要参数列表不同即可。这就是为什么CONSTANT_NameAndType_info结构要同时包含名称和描述符的原因。那么字段呢？Java是不能定义多个同名字段的，哪怕它们的类型各不相同。这只是Java语法的限制而已，从class文件的层面来看，是完全可以支持这点的。 
在ch03\classfile目录下创建cp_name_and_type.go文件，在其中定义ConstantName-AndTypeInfo结构体，代码如下：
```go
package classfile 
type ConstantNameAndTypeInfo struct { 
    nameIndex uint16 
    descriptorIndex uint16 
}
func (self *ConstantNameAndTypeInfo) readInfo(reader *ClassReader) { 
    self.nameIndex = reader.readUint16() 
    self.descriptorIndex = reader.readUint16() 
}
```
代码比较简单，就不多解释了。
##### 3.3.11 CONSTANT_Fieldref_info、 CONSTANT_Methodref_info和 CONSTANT_InterfaceMethodref_info
CONSTANT_Fieldref_info表示字段符号引用，CONSTANT_Methodref_info表示普通（非接口）方法符号引用，CONSTANT_InterfaceMethodref_info表示接口方法符号引用。这三种常量结构一模一样，为了节约篇幅，下面只给出CONSTANT_Fieldref_info的结构。 
```go
CONSTANT_Fieldref_info { 
    u1 tag; 
    u2 class_index; 
    u2 name_and_type_index; 
} 
```
class_index和name_and_type_index都是常量池索引，分别指向CONSTANT_Class_info和CONSTANT_NameAndType_info常量。先定义一个统一的结构体ConstantMemberrefInfo来表示这3种常量。
在ch03\classfile目录下创建cp_member_ref.go文件，把下面的代码输入进去。
```go
package classfile 
type ConstantMemberrefInfo struct { 
    cp ConstantPool 
    classIndex uint16 
    nameAndTypeIndex uint16 
}
func (self *ConstantMemberrefInfo) readInfo(reader *ClassReader) { 
    self.classIndex = reader.readUint16() 
    self.nameAndTypeIndex = reader.readUint16() 
}
func (self *ConstantMemberrefInfo) ClassName() string { 
    return self.cp.getClassName(self.classIndex) 
}
func (self *ConstantMemberrefInfo) NameAndDescriptor() (string, string) { 
    return self.cp.getNameAndType(self.nameAndTypeIndex) 
}
```
然后定义三个结构体“继承”ConstantMemberrefInfo。Go语言并没有“继承”这个概念，但是可以通过结构体嵌套来模拟，代码如下：
```go
type ConstantFieldrefInfo struct{ ConstantMemberrefInfo } 
type ConstantMethodrefInfo struct{ ConstantMemberrefInfo } 
type ConstantInterfaceMethodrefInfo struct{ ConstantMemberrefInfo }
```
ClassFileTest类的main（）方法使用了java.lang.System类的out字段，该字段由常量池第2项指出，如图3-18所示。
可以看到，class_index是50（0x32），name_and_type_index是51（0x33）。我们找到第50和第51个常量，可以看到，确实是CONSTANT_Class_info和CONSTANT_Name-AndType_info，如图3-19所示。
![3-18](/img/3-18.png)
图3-18 用classpy观察CONSTANT_Fieldref_info常量
![3-19](/img/3-19.png)
图3-19 用classpy观察CONSTANT_Fieldref_info常量（2）
##### 3.3.12 常量池小结 
还有三个常量没有介绍：CONSTANT_MethodType_info、CONSTANT_MethodHandle_info和CONSTANT_InvokeDynamic_info。它们是Java SE 7才添加到class文件中的，目的是支持新增的invokedynamic指令。本书不讨论invokedynamic指令，所以解析这三个常量的代码就不在这里介绍了。代码也非常简单，有兴趣的读者可以阅读随书源代码中的ch03\cp_invoke_dynamic.go文件。 

可以把常量池中的常量分为两类：字面量（literal）和符号引用（symbolic reference）。字面量包括数字常量和字符串常量，符号引用包括类和接口名、字段和方法信息等。除了字面量，其他常量都是通过索引直接或间接指向CONSTANT_Utf8_info常量，以CONSTANT_Fieldref_info为例，如图3-20所示。
![3-20](/img/3-20.png)
图3-20 常量引用关系 
本节只是简单介绍常量池和各种常量的结构，在第6章讨论运行时常量池，第7章讨论方法调用时，会进一步讨论它们的用途。
#### 3.4 解析属性表 
3.2节大致勾勒出了class文件的结构，3.3节介绍了常量池。细心的读者一定会发现，还有一些重要的信息没有出现，如方法的字节码等。那么这些信息存在哪里呢？答案是属性表。属性表可谓是个大杂烩，里面存储了各式各样的信息。本节将详细讨论属性表。
##### 3.4.1 AttributeInfo接口 
和常量池类似，各种属性表达的信息也各不相同，因此无法用统一的结构来定义。不同之处在于，常量是由Java虚拟机规范严格定义的，共有14种。但属性是可以扩展的，不同的虚拟机实现可以定义自己的属性类型。由于这个原因，Java虚拟机规范没有使用tag，而是使用属性名来区别不同的属性。属性数据放在属性名之后的u1表中，这样Java虚拟机实现就可以跳过自己无法识别的属性。属性的结构定义如下： 
```go
attribute_info { 
    u2 attribute_name_index; 
    u4 attribute_length; 
    u1 info[attribute_length]; 
}
```
注意，属性表中存放的属性名实际上并不是编码后的字符串，而是常量池索引，指向常量池中的CONSTANT_Utf8_info常量。在ch03\classfile目录下创建attribute_info.go文件，在其中定义AttributeInfo接口，代码如下： 
```go
package classfile 
type AttributeInfo interface { 
    readInfo(reader *ClassReader) 
}
func readAttributes(reader *ClassReader, cp ConstantPool) []AttributeInfo {...} 
func readAttribute(reader *ClassReader, cp ConstantPool) AttributeInfo {...} 
func newAttributeInfo(attrName string, attrLen uint32, cp ConstantPool) AttributeInfo {...} 
```
和ConstantInfo接口一样，AttributeInfo接口也只定义了一个readInfo（）方法，需要由具体的属性实现。readAttributes（）函数读取属性表，代码如下： 
```go
func readAttributes(reader *ClassReader, cp ConstantPool) []AttributeInfo { 
    attributesCount := reader.readUint16() 
    attributes := make([]AttributeInfo, attributesCount) 
    for i := range attributes { 
        attributes[i] = readAttribute(reader, cp) 
    }
    return attributes 
}
``` 
函数readAttribute（）读取单个属性，代码如下：
```go
func readAttribute(reader *ClassReader, cp ConstantPool) AttributeInfo { 
    attrNameIndex := reader.readUint16() 
    attrName := cp.getUtf8(attrNameIndex) 
    attrLen := reader.readUint32() 
    attrInfo := newAttributeInfo(attrName, attrLen, cp) 
    attrInfo.readInfo(reader) 
    return attrInfo 
} 
```
readAttribute（）先读取属性名索引，根据它从常量池中找到属性名，然后读取属性长度，接着调用newAttributeInfo（）函数创建具体的属性实例。Java虚拟机规范预定义了23种属性，先解析其中的8种。newAttributeInfo（）函数的代码如下：
```go 
func newAttributeInfo(attrName string, attrLen uint32, cp ConstantPool) AttributeInfo { 
    switch attrName { 
        case "Code": return &CodeAttribute{cp: cp} 
        case "ConstantValue": return &ConstantValueAttribute{} 
        case "Deprecated": return &DeprecatedAttribute{} 
        case "Exceptions": return &ExceptionsAttribute{} 
        case "LineNumberTable": return &LineNumberTableAttribute{} 
        case "LocalVariableTable": return &LocalVariableTableAttribute{} 
        case "SourceFile": return &SourceFileAttribute{cp: cp} 
        case "Synthetic": return &SyntheticAttribute{} 
        default: return &UnparsedAttribute{attrName, attrLen, nil} 
    } 
}
```
UnparsedAttribute结构体在ch03\classfile\attr_unparsed.go文件中，代码如下： 
```go
package classfile 
type UnparsedAttribute struct { 
    name string 
    length uint32 
    info []byte 
}
func (self *UnparsedAttribute) readInfo(reader *ClassReader) { 
    self.info = reader.readBytes(self.length) 
}
```
按照用途，23种预定义属性可以分为三组。第一组属性是实现Java虚拟机所必需的，共有5种；第二组属性是Java类库所必需的，共有12种；第三组属性主要提供给工具使用，共有6种。第三组属性是可选的，也就是说可以不出现在class文件中。如果class文件中存在第三组属性，Java虚拟机实现或者Java类库也是可以利用它们的，比如使用LineNumberTable属性在异常堆栈中显示行号。 
从class文件演进的角度来讲，JDK1.0时只有6种预定义属性，JDK1.1增加了3种。J2SE 5.0增加了9种属性，主要用于支持泛型和注解。Java SE 6增加了StackMapTable属性，用于优化字节码验证。Java SE 7增加了BootstrapMethods属性，用于支持新增的invokedynamic指令。Java SE 8又增加了三种属性。表3-5给出了这23种属性出现的Java版本、分组以及它们在class文件中的位置。 
表3-4 预定义属性


由于篇幅的限制，下面只介绍其中的8种属性。
##### 3.4.2 Deprecated和Synthetic属性 
Deprecated和Synthetic是最简单的两种属性，仅起标记作用，不包含任何数据。这两种属性都是JDK1.1引入的，可以出现在ClassFile、field_info和method_info结构中，它们的结构定义如下：
```go
Deprecated_attribute { 
    u2 attribute_name_index; 
    u4 attribute_length; 
}
Synthetic_attribute { 
    u2 attribute_name_index; 
    u4 attribute_length; 
}
```
由于不包含任何数据，所以attribute_length的值必须是0。Deprecated属性用于指出类、接口、字段或方法已经不建议使用，编译器等工具可以根据Deprecated属性输出警告信息。J2SE 5.0之前可以使用Javadoc提供的@deprecated标签指示编译器给类、接口、字段或方法添加Deprecated属性，语法格式如下：
```java
/** @deprecated */ 
public void oldMethod() {...} 
```
从J2SE 5.0开始，也可以使用@Deprecated注解，语法格式如下：
```java
@Deprecated 
public void oldMethod() {} 
```
Synthetic属性用来标记源文件中不存在、由编译器生成的类成员，引入Synthetic属性主要是为了支持嵌套类和嵌套接口。具体细节就不介绍了，感兴趣的读者可以参考Java虚拟机规范相关章节。在ch03\classfile目录下创建attr_markers.go文件，在其中定义 
DeprecatedAttribute和SyntheticAttribute结构体，代码如下：
```go
package classfile 
type DeprecatedAttribute struct { MarkerAttribute } 
type SyntheticAttribute struct { MarkerAttribute } 
type MarkerAttribute struct{} 
func (self *MarkerAttribute) readInfo(reader *ClassReader) { 
    // read nothing 
}
```
由于这两个属性都没有数据，所以readInfo（）方法是空的。
##### 3.4.3 SourceFile属性 
SourceFile是可选定长属性，只会出现在ClassFile结构中，用于指出源文件名。其结构定义如下：
```go
SourceFile_attribute { 
    u2 attribute_name_index; 
    u4 attribute_length; 
    u2 sourcefile_index; 
}
```
attribute_length的值必须是2。sourcefile_index是常量池索引，指向CONSTANT_Utf8_info常量。在ch03\classfile目录下创建attr_source_file.go文件，在其中定义SourceFileAttribute结构体，代码如下：
```go
package classfile 
type SourceFileAttribute struct { 
    cp ConstantPool 
    sourceFileIndex uint16 
}
func (self *SourceFileAttribute) readInfo(reader *ClassReader) { 
    self.sourceFileIndex = reader.readUint16() 
}
func (self *SourceFileAttribute) FileName() string { 
    return self.cp.getUtf8(self.sourceFileIndex) 
}
```
笔者的编译器给ClassFileTest生成了SourceFile属性，如图3-21所示。
![3-21](/img/3-21.png)
图3-21 用classpy观察SourceFile属性 
第47和第48个常量，确实都是CONSTANT_Utf8_info常量，如图3-22所示。
![3-22](/img/3-22.png) 
图3-22 用classpy观察SourceFile属性（2）
##### 3.4.4 ConstantValue属性 
ConstantValue是定长属性，只会出现在field_info结构中，用于表示常量表达式的值（详见Java语言规范的15.28节）。其结构定义如下：
```go
ConstantValue_attribute { 
    u2 attribute_name_index; 
    u4 attribute_length; 
    u2 constantvalue_index; 
}
``` 
attribute_length的值必须是2。constantvalue_index是常量池索引，但具体指向哪种常量因字段类型而异。表3-6给出了字段类型和常量类型的对应关系。 

表3-5 字段类型和常量类型对应关系

在ch03\classfile目录下创建attr_constant_value.go文件，在其中定义ConstantValueAttribute结构体，代码如下：
```go
package classfile 
type ConstantValueAttribute struct { 
    constantValueIndex uint16 
}
func (self *ConstantValueAttribute) readInfo(reader *ClassReader) { 
    self.constantValueIndex = reader.readUint16() 
}
func (self *ConstantValueAttribute) ConstantValueIndex() uint16 { 
    return self.constantValueIndex 
}
```
在第6章讨论类和对象时，会介绍如何使用ConstantValue属性。下面用classpy观察ClassFileTest类的FLAG字段，如图3-23所示。 
![3-23](/img/3-23.png)
图3-23 用classpy观察ConstantValue属性 

可以看到，属性表里确实有一个ConstantValue属性，constantvalue_index是10（0x0A），指向CONSTANT_Integer_info，如图3-24所示。
![3-24](/img/3-24.png)
图3-24 用classpy观察ConstantValue属性（2）
##### 3.4.5 Code属性 
Code是变长属性，只存在于method_info结构中。Code属性中存放字节码等方法相关信息。相比前面介绍的几种属性，Code属性比较复杂，其结构定义如下： 
```go
Code_attribute { 
    u2 attribute_name_index; 
    u4 attribute_length; 
    u2 max_stack; 
    u2 max_locals; 
    u4 code_length; 
    u1 code[code_length]; 
    u2 exception_table_length; 
    { 
        u2 start_pc; 
        u2 end_pc; 
        u2 handler_pc; 
        u2 catch_type; 
    } exception_table[exception_table_length]; 
    u2 attributes_count; 
    attribute_info attributes[attributes_count]; 
} 
```
max_stack给出操作数栈的最大深度，max_locals给出局部变量表大小。接着是字节码，存在u1表中。最后是异常处理表和属性表。在第4章讨论运行时数据区，并且实现操作数栈和局部变量表时，max_stack和max_locals就会派上用场。在第5章讨论指令集和解释器时，会用到字节码。在第10章讨论异常处理时，会使用异常处理表。

把Code属性结构翻译成Go结构体，定义在ch03\classfile\attr_code.go文件中，代码如下：
```go
package classfile 
type CodeAttribute struct { 
    cp ConstantPool 
    maxStack uint16 
    maxLocals uint16 
    code []byte 
    exceptionTable []*ExceptionTableEntry 
    attributes []AttributeInfo 
}
type ExceptionTableEntry struct { 
    startPc uint16 
    endPc uint16 
    handlerPc uint16 
    catchType uint16 
}
func (self *CodeAttribute) readInfo(reader *ClassReader) {...} 
```
readInfo（）方法的代码如下：
```go
func (self *CodeAttribute) readInfo(reader *ClassReader) { 
    self.maxStack = reader.readUint16() 
    self.maxLocals = reader.readUint16() 
    codeLength := reader.readUint32() 
    self.code = reader.readBytes(codeLength) 
    self.exceptionTable = readExceptionTable(reader) 
    self.attributes = readAttributes(reader, self.cp) 
}
```
readExceptionTable（）函数的代码如下： 
```go
func readExceptionTable(reader *ClassReader) []*ExceptionTableEntry { 
    exceptionTableLength := reader.readUint16() 
    exceptionTable := make([]*ExceptionTableEntry, exceptionTableLength) 
    for i := range exceptionTable { 
        exceptionTable[i] = &ExceptionTableEntry{ 
            startPc: reader.readUint16(),endPc: reader.readUint16(), 
            handlerPc: reader.readUint16(), 
            catchType: reader.readUint16(), 
        } 
    }
    return exceptionTable 
}
```
ClassFileTest.main（）方法的Code属性如图3-25所示。
![3-25](/img/3-25.png)
图3-25 用classpy观察Code属性
##### 3.4.6 Exceptions属性
Exceptions是变长属性，记录方法抛出的异常表，其结构定义如下：
```go
Exceptions_attribute { 
    u2 attribute_name_index; 
    u4 attribute_length; 
    u2 number_of_exceptions; 
    u2 exception_index_table[number_of_exceptions]; 
}
```
在ch03\classfile目录下创建attr_exceptions.go文件，在其中定义ExceptionsAttribute结构体，代码如下：
```go
package classfile 
type ExceptionsAttribute struct { 
    exceptionIndexTable []uint16 
}
func (self *ExceptionsAttribute) readInfo(reader *ClassReader) { 
    self.exceptionIndexTable = reader.readUint16s() 
}
func (self *ExceptionsAttribute) ExceptionIndexTable() []uint16 { 
    return self.exceptionIndexTable 
}
```
代码比较简单，就不多解释了。ClassFileTest.main（）方法的Exceptions属性如图3-26所示。
![3-26](/img/3-26.png)
图3-26 用classpy观察Code属性
##### 3.4.7 LineNumberTable和LocalVariableTable属性 
LineNumberTable属性表存放方法的行号信息，LocalVariableTable属性表中存放方法的局部变量信息。这两种属性和前面介绍的SourceFile属性都属于调试信息，都不是运行时必需的。在使用javac编译器编译Java程序时，默认会在class文件中生成这些信息。可以使用javac提供的-g：none选项来关闭这些信息的生成，这里就不多介绍了，具体请参考javac用法。 

LineNumberTable和LocalVariableTable属性表在结构上很像，下面以LineNumberTable为例进行讨论，它的结构定义如下：
```go
LineNumberTable_attribute { 
    u2 attribute_name_index; 
    u4 attribute_length; 
    u2 line_number_table_length; 
    { 
        u2 start_pc; 
        u2 line_number; 
    } line_number_table[line_number_table_length]; 
}
```
把上面的结构定义翻译成Go结构体，定义在 ch03\classfile\attr_line_number_table.go文件中，代码如下： 
```go
package classfile 
type LineNumberTableAttribute struct { 
    lineNumberTable []*LineNumberTableEntry
}
type LineNumberTableEntry struct { 
    startPc uint16 
    lineNumber uint16 
}
func (self *LineNumberTableAttribute) readInfo(reader *ClassReader) {...} 
```
readInfo（）方法读取属性表数据，代码如下： 
```go
func (self *LineNumberTableAttribute) readInfo(reader *ClassReader) { 
    lineNumberTableLength := reader.readUint16() 
    self.lineNumberTable = make([]*LineNumberTableEntry, lineNumberTableLength) 
    for i := range self.lineNumberTable { 
        self.lineNumberTable[i] = &LineNumberTableEntry{ 
            startPc: reader.readUint16(), 
            lineNumber: reader.readUint16(), 
        } 
    } 
}
```
在第10章讨论异常处理时会详细讨论LineNumberTable属性。
#### 3.5 测试本章代码 
在第2章的测试中，把class文件加载到了内存中，并且把一堆看似杂乱无章的数字打印到了控制台。相信读者一定不会满足于此。本节就来修改测试代码，把命令行工具临时打造成一个简化版的javap。 
打开ch03\main.go文件，修改import语句和startJVM（）函数，代码如下：
```go 
package main 
import "fmt" 
import "strings" 
import "jvmgo/ch03/classfile" 
import "jvmgo/ch03/classpath" 
func main() {...} 
func startJVM(options *cmdline.Options, class string, args []string) {...} 
```
main（）函数不用变，修改startJVM（）函数，代码如下： 
```go
func startJVM(cmd *Cmd) { 
    cp := classpath.Parse(cmd.XjreOption, cmd.cpOption) 
    className := strings.Replace(cmd.class, ".", "/", -1) 
    cf := loadClass(className, cp) 
    fmt.Println(cmd.class) 
    printClassInfo(cf) 
}
``` 
loadClass（）函数读取并解析class文件，代码如下：
```go
func loadClass(className string, cp *classpath.Classpath) *classfile.ClassFile { 
    classData, _, err := cp.ReadClass(className) 
    if err != nil { 
        panic(err) 
    }
    cf, err := classfile.Parse(classData) 
    if err != nil { 
        panic(err) 
    }
    return cf 
}
```
printClassInfo（）函数把class文件的一些重要信息打印出来，代码如下：
```go 
func printClassInfo(cf *classfile.ClassFile) { 
    fmt.Printf("version: %v.%v\n", cf.MajorVersion(), cf.MinorVersion()) 
    fmt.Printf("constants count: %v\n", len(cf.ConstantPool())) 
    fmt.Printf("access flags: 0x%x\n", cf.AccessFlags()) 
    fmt.Printf("this class: %v\n", cf.ClassName()) 
    fmt.Printf("super class: %v\n", cf.SuperClassName()) 
    fmt.Printf("interfaces: %v\n", cf.InterfaceNames()) 
    fmt.Printf("fields count: %v\n", len(cf.Fields())) 
    for _, f := range cf.Fields() { 
        fmt.Printf(" %s\n", f.Name()) 
    }
    fmt.Printf("methods count: %v\n", len(cf.Methods())) 
    for _, m := range cf.Methods() { 
        fmt.Printf(" %s\n", m.Name()) 
    } 
}
```
打开命令行窗口，执行下面的命令编译本章代码。
```shell script
go install jvmgo\ch03 
```


编译成功后，在D：\go\workspace\bin目录下会出现ch03.exe文件。执行ch03.exe，指定-Xjre选项和类名，就可以打印出class文件的信息。笔者把java.lang.String.class文件（位于rt.jar中）的信息打印了出来，如图3-27所示。如果读者想测试自己编写的类，记得要指定-classpath选项。与第2章相比，我们显然取得了很大的进步。 
![3-27](/img/3-27.png)
图3-27 ch03.exe的测试结果
#### 3.6 本章小结 
计算机科学家David Wheeler有一句名言：“计算机科学中的任何难题都可以通过增加一个中间层来解决” [1] 。ClassFile结构体就是为了实现类加载功能而增加的中间层。在第6章，我们会进一步处理ClassFile结构体，把它转变Class结构体，并放入方法区。不过在此之前，要先在第4章实现运行时数据区，在第5章实现字节码解释器。 

[1]: 原文为All problems in computer science can be solved by another level of indirection。

