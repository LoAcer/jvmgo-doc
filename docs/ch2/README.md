第2章 搜索class文件 
====
第1章介绍了java命令的用法以及它如何启动Java应用程序：首先启动Java虚拟机，然后加载主类，最后调用主类的main（）方法。但是我们知道，即使是最简单的“Hello，World”程序，也是无法独自运行的，该程序的代码如下：

```java
public class HelloWorld { 
public static void main(String[] args) { 
System.out.println("Hello, world!"); 
} 
} 
```
::: TIP
加载HelloWorld类之前，首先要加载它的超类，也就是java.lang.Object。在调用main（）方法之前，因为虚拟机需要准备好参数数组，所以需要加载java.lang.String和java.lang.String[]类。把字符串打印到控制台还需要加载java.lang.System类，等等。那么，Java虚拟机从哪里寻找这些类呢？本章将详细讨论这个问题。
:::

####2.1 类路径
Java虚拟机规范并没有规定虚拟机应该从哪里寻找类，因此不同的虚拟机实现可以采用不同的方法。Oracle的Java虚拟机实现根据类路径（class path）来搜索类。按照搜索的先后顺序，类路径可以分为以下3个部分： 
- 启动类路径（bootstrap classpath） 
- 扩展类路径（extension classpath） 
- 用户类路径（user classpath） 

启动类路径默认对应jre\lib目录，Java标准库（大部分在rt.jar里）位于该路径。扩展类路径默认对应jre\lib\ext目录，使用Java扩展机制的类位于这个路径。我们自己实现的类，以及第三方类库则位于用户类路径。可以通过-Xbootclasspath选项修改启动类路径，不过通常并不需要这样做，所以这里就不详细介绍了。 

用户类路径的默认值是当前目录，也就是“.”。可以设置CLASSPATH环境变量来修改用户类路径，但是这样做不够灵活，所以不推荐使用。更好的办法是给java命令传递-classpath（或简写为-cp）选项。-classpath/-cp选项的优先级更高，可以覆盖CLASSPATH环境变量设置。第1章简单介绍过这个选项，这里再详细解释一下。
 
-classpath/-cp选项既可以指定目录，也可以指定JAR文件或者ZIP文件，如下： 
```shell script
java -cp path\to\classes ... 
java -cp path\to\lib1.jar ... 
java -cp path\to\lib2.zip ... 
```

还可以同时指定多个目录或文件，用分隔符分开即可。分隔符因操作系统而异。在Windows系统下是分号，在类UNIX（包括Linux、Mac OS X等）系统下是冒号。例如在Windows下： 
```shell script
java -cp path\to\classes;lib\a.jar;lib\b.jar;lib\c.zip ...
```
从Java 6开始，还可以使用通配符（*）指定某个目录下的所有JAR文件，格式如下： 
```shell script
java -cp classes;lib\* ...
```

####2.2 准备工作 

从第2章开始，每章的代码都是建立在前一章的基础之上。把ch01目录复制一份，然后改名为ch02。因为本章要创建的源文件都在classpath包中，所以在ch02目录中创建一个classpath子目录。现在目录结构看起来应该是这样：
```text
D:\go\workspace\src 
  |-jvmgo 
    |-ch01 
    |-ch02 
      |-classpath 
      |-cmd.go 
      |-main.go 
```


我们的Java虚拟机将使用JDK的启动类路径来寻找和加载Java标准库中的类，因此需要某种方式指定jre目录的位置。命令行选项是个不错的选择，所以增加一个非标准选项-Xjre。打开ch02\cmd.go，修改Cmd结构体，添加XjreOption字段，代码如下： 
```go
type Cmd struct { 
    helpFlag bool 
    versionFlag bool 
    cpOption string 
    XjreOption string 
    class string 
    args []string 
}
```
parseCmd（）函数也要相应修改，代码如下：
```go
func parseCmd() *Cmd { 
    cmd := &Cmd{} 
    flag.Usage = printUsage 
    flag.BoolVar(&cmd.helpFlag, "help", false, "print help message") 
    flag.BoolVar(&cmd.helpFlag, "?", false, "print help message") 
    flag.BoolVar(&cmd.versionFlag, "version", false, "print version and exit") 
    flag.StringVar(&cmd.cpOption, "classpath", "", "classpath") 
    flag.StringVar(&cmd.cpOption, "cp", "", "classpath") 
    flag.StringVar(&cmd.XjreOption, "Xjre", "", "path to jre") 
    flag.Parse() 
    ... //其他代码不变 
}
```
#### 2.3 实现类路径 
可以把类路径想象成一个大的整体，它由启动类路径、扩展类路径和用户类路径三个小路径构成。三个小路径又分别由更小的路径构成。是不是很像组合模式（composite pattern）？没错，本节就套用组合模式来设计和实现类路径。
##### 2.3.1 Entry接口 
先定义一个接口来表示类路径项。在ch02\classpath目录下创建entry.go文件，在其中定义Entry接口，代码如下： 
```go
package classpath 
import "os" 
import "strings" 
const pathListSeparator = string(os.PathListSeparator) 
type Entry interface { 
    readClass(className string) ([]byte, Entry, error) 
    String() string 
}
func newEntry(path string) Entry {...} 
```
常量pathListSeparator是string类型，存放路径分隔符，后面会用到。Entry接口中有两个方法。readClass（）方法负责寻找和加载class文件；String（）方法的作用相当于Java中的toString（），用于返回变量的字符串表示。 

readClass（）方法的参数是class文件的相对路径，路径之间用斜线（/）分隔，文件名有.class后缀。比如要读取java.lang.Object类，传入的参数应该是java/lang/Object.class。返回值是读取到的字节数据、最终定位到class文件的Entry，以及错误信息。Go的函数或方法允许返回多个值，按照惯例，可以使用最后一个返回值作为错误信息。

newEntry（）函数根据参数创建不同类型的Entry实例，代码如下：
```go
func newEntry(path string) Entry { 
    if strings.Contains(path, pathListSeparator) { 
        return newCompositeEntry(path) 
    }
    if strings.HasSuffix(path, "*") { 
        return newWildcardEntry(path) 
    }
    if strings.HasSuffix(path, ".jar") || strings.HasSuffix(path, ".JAR") || 
        strings.HasSuffix(path, ".zip") || strings.HasSuffix(path, ".ZIP") { 
        return newZipEntry(path) 
    }
    return newDirEntry(path) 
}
```
Entry接口有4个实现，分别是DirEntry、ZipEntry、CompositeEntry和WildcardEntry。下面分别介绍每一种实现。
##### 2.3.2 DirEntry 
在4种实现中，DirEntry相对简单一些，表示目录形式的类路径。在ch02\classpath目录下创建entry_dir.go文件，在其中定义DirEntry结构体，代码如下：
```go
package classpath 
import "io/ioutil" 
import "path/filepath" 
type DirEntry struct { 
    absDir string 
}
func newDirEntry(path string) *DirEntry {...} 
func (self *DirEntry) readClass(className string) ([]byte , Entry, error) {...} 
func (self *DirEntry) String() string {...}
``` 
DirEntry只有一个字段，用于存放目录的绝对路径。和Java语言不同，Go结构体不需要显示实现接口，只要方法匹配即可。Go没有专门的构造函数，本书统一使用new开头的函数来创建结构体实例，并把这类函数称为构造函数。newDirEntry（）函数的代码如下： 
```go
func newDirEntry(path string) *DirEntry { 
    absDir, err := filepath.Abs(path) 
    if err != nil { 
        panic(err) 
    }
    return &DirEntry{absDir} 
}
```
newDirEntry（）先把参数转换成绝对路径，如果转换过程出现错误，则调用panic（）函数终止程序执行，否则创建DirEntry实例并返回。

下面介绍readClass（）方法：
```go
func (self *DirEntry) readClass(className string) ([]byte, Entry, error) { 
    fileName := filepath.Join(self.absDir, className) 
    data, err := ioutil.ReadFile(fileName) 
    return data, self, err 
}
``` 
readClass（）先把目录和class文件名拼成一个完整的路径，然后调用ioutil包提供的ReadFile（）函数读取class文件内容，最后返回。String（）方法很简单，直接返回目录，代码如下： 
```go
func (self *DirEntry) String() string { 
    return self.absDir 
}
```
##### 2.3.3 ZipEntry 
ZipEntry表示ZIP或JAR文件形式的类路径。在ch02\classpath目录下创建entry_zip.go文件，在其中定义ZipEntry结构体，代码如下：
```go
package classpath 
import "archive/zip" 
import "errors" 
import "io/ioutil" 
import "path/filepath" 
type ZipEntry struct { 
absPath string 
}
func newZipEntry(path string) *ZipEntry {...} 
func (self *ZipEntry) readClass(className string) ([]byte, Entry, error) {...} 
func (self *ZipEntry) String() string {...} 
```
absPath字段存放ZIP或JAR文件的绝对路径。构造函数和String（）与DirEntry大同小异，就不多解释了，代码如下：
```go
func newZipEntry(path string) *ZipEntry { 
absPath, err := filepath.Abs(path) 
if err != nil { 
panic(err) 
}
return &ZipEntry{absPath} 
}
func (self *ZipEntry) String() string { 
return self.absPath 
}
```

下面重点介绍如何从ZIP文件中提取class文件，代码如下：
```go
func (self *ZipEntry) readClass(className string) ([]byte, Entry, error) {
r, err := zip.OpenReader(self.absPath) 
if err != nil { 
return nil, nil, err 
}
defer r.Close() 
for _, f := range r.File { 
if f.Name == className {
rc, err := f.Open() 
if err != nil { 
return nil, nil, err 
}
defer rc.Close() 
data, err := ioutil.ReadAll(rc) 
if err != nil { 
return nil, nil, err 
}
return data, self, nil 
} 
}
return nil, nil, errors.New("class not found: " + className) 
}
```
首先打开ZIP文件，如果这一步出错的话，直接返回。然后遍历ZIP压缩包里的文件，看能否找到class文件。如果能找到，则打开class文件，把内容读取出来，并返回。如果找不到，或者出现其他错误，则返回错误信息。有两处使用了defer语句来确保打开的文件得以关闭。readClass（）方法每次都要打开和关闭ZIP文件，因此效率不是很高。笔者进行了优化，但鉴于篇幅有限，就不展示具体代码了。感兴趣的读者可以阅读ch02\classpath\entry_zip2.go文件。
##### 2.3.4 CompositeEntry 
在ch02\classpath目录下创建entry_composite.go文件，在其中定义CompositeEntry结构体，代码如下：
```go
package classpath 
import "errors" 
import "strings" 
type CompositeEntry []Entry 
func newCompositeEntry(pathList string) CompositeEntry {...} 
func (self CompositeEntry) readClass(className string) ([]byte, Entry, error) {...} 
func (self CompositeEntry) String() string {...} 
```
如前所述，CompositeEntry由更小的Entry组成，正好可以表示成[]Entry。在Go语言中，数组属于比较低层的数据结构，很少直接使用。大部分情况下，使用更便利的slice类型。构造函数把参数（路径列表）按分隔符分成小路径，然后把每个小路径都转换成具体的Entry实例，代码如下：
```go
func newCompositeEntry(pathList string) CompositeEntry { 
compositeEntry := []Entry{} 
for _, path := range strings.Split(pathList, pathListSeparator) { 
entry := newEntry(path) 
compositeEntry = append(compositeEntry, entry) 
}
return compositeEntry 
}
```
相信读者已经想到readClass（）方法的代码了：依次调用每一个子路径的readClass（）方法，如果成功读取到class数据，返回数据即可；如果收到错误信息，则继续；如果遍历完所有的子路径还没有找到class文件，则返回错误。readClass（）方法的代码如下：
```go
func (self CompositeEntry) readClass(className string) ([]byte, Entry, error) { 
for _, entry := range self { 
data, from, err := entry.readClass(className) 
if err == nil { 
return data, from, nil 
} 
}
return nil, nil, errors.New("class not found: " + className) 
}
```
String（）方法也不复杂。调用每一个子路径的String（）方法，然后把得到的字符串用路径分隔符拼接起来即可，代码如下：
```
func (self CompositeEntry) String() string { 
strs := make([]string, len(self)) 
for i, entry := range self { 
strs[i] = entry.String() 
}
return strings.Join(strs, pathListSeparator) 
}
```
##### 2.3.5 WildcardEntry 
WildcardEntry实际上也是CompositeEntry，所以就不再定义新的类型了。在ch02\classpath目录下创建entry_wildcard.go文件，在其中定义newWildcardEntry（）函数，代码如下：
```go
package classpath 
import "os" 
import "path/filepath" 
import "strings" 
func newWildcardEntry(path string) CompositeEntry { 
baseDir := path[:len(path)-1] // remove * 
compositeEntry := []Entry{} 
walkFn := func(path string, info os.FileInfo, err error) error {...} 
filepath.Walk(baseDir, walkFn) 
return compositeEntry 
}
```
首先把路径末尾的星号去掉，得到baseDir，然后调用filepath包的Walk（）函数遍历baseDir创建ZipEntry。Walk（）函数的第二个参数也是一个函数，了解函数式编程的读者应该一眼就可以认出这种用法（即函数可作为参数）。walkFn变量的定义如下：
```go
walkFn := func(path string, info os.FileInfo, err error) error { 
if err != nil { 
return err 
}
if info.IsDir() && path != baseDir { 
return filepath.SkipDir 
}
if strings.HasSuffix(path, ".jar") || strings.HasSuffix(path, ".JAR") { 
jarEntry := newZipEntry(path) 
compositeEntry = append(compositeEntry, jarEntry)
}
return nil 
}
```
在walkFn中，根据后缀名选出JAR文件，并且返回SkipDir跳过子目录（通配符类路径不能递归匹配子目录下的JAR文件）。
##### 2.3.6 Classpath 
Entry接口和4个实现介绍完了，接下来实现Classpath结构体。还是在ch02\classpath目录下创建classpath.go文件，把下面的代码输入进去。
```go 
package classpath 
import "os" 
import "path/filepath" 
type Classpath struct { 
bootClasspath Entry 
extClasspath Entry 
userClasspath Entry 
}
func Parse(jreOption, cpOption string) *Classpath {...} 
func (self *Classpath) ReadClass(className string) ([]byte, Entry, error) {...} 
func (self *Classpath) String() string {...} 
```
Classpath结构体有三个字段，分别存放三种类路径。Parse（）函数使用-Xjre选项解析启动类路径和扩展类路径，使用-classpath/-cp选项解析用户类路径，代码如下：
```go
func Parse(jreOption, cpOption string) *Classpath { 
cp := &Classpath{} 
cp.parseBootAndExtClasspath(jreOption) 
cp.parseUserClasspath(cpOption) 
return cp 
}
``` 
parseBootAndExtClasspath（）方法的代码如下：
```go
func (self *Classpath) parseBootAndExtClasspath(jreOption string) { 
jreDir := getJreDir(jreOption) 
// jre/lib/* 
jreLibPath := filepath.Join(jreDir, "lib", "*") 
self.bootClasspath = newWildcardEntry(jreLibPath) 
// jre/lib/ext/* 
jreExtPath := filepath.Join(jreDir, "lib", "ext", "*") 
self.extClasspath = newWildcardEntry(jreExtPath) 
} 
```
优先使用用户输入的-Xjre选项作为jre目录。如果没有输入该选项，则在当前目录下寻找jre目录。如果找不到，尝试使用JAVA_HOME环境变量。getJreDir（）函数的代码如下：
```go
func getJreDir(jreOption string) string { 
if jreOption != "" && exists(jreOption) { 
return jreOption 
}
if exists("./jre") { 
return "./jre" 
}
if jh := os.Getenv("JAVA_HOME"); jh != "" { 
return filepath.Join(jh, "jre") 
}
panic("Can not find jre folder!") 
}
```
exists（）函数用于判断目录是否存在，代码如下：
```go 
func exists(path string) bool { 
if _, err := os.Stat(path); err != nil { 
if os.IsNotExist(err) { 
return false 
} 
}
return true 
}
```
parseUserClasspath（）方法的代码相对简单一些，如下：
```go 
func (self *Classpath) parseUserClasspath(cpOption string) { 
if cpOption == "" { 
cpOption = "." 
}
self.userClasspath = newEntry(cpOption) 
} 
```
如果用户没有提供-classpath/-cp选项，则使用当前目录作为用户类路径。ReadClass（）方法依次从启动类路径、扩展类路径和用户类路径中搜索class文件，代码如下：
```go
func (self *Classpath) ReadClass(className string) ([]byte, Entry, error) { 
className = className + ".class" 
if data, entry, err := self.bootClasspath.readClass(className); err == nil { 
return data, entry, err 
}
if data, entry, err := self.extClasspath.readClass(className); err == nil { 
return data, entry, err 
}
return self.userClasspath.readClass(className) 
} 
```
注意，传递给ReadClass（）方法的类名不包含“.class”后缀。最后，String（）方法返回用户类路径的字符串表示，代码如下：
```go
func (self *Classpath) String() string { 
return self.userClasspath.String() 
}
```
至此，整个类路径都实现了，下面我们来测试一下。
##### 2.4 测试本章代码 
打开ch02/main.go文件，添加两条import语句，代码如下：
```go
package main 
import "fmt" 
import "strings" 
import "jvmgo/ch02/classpath" 
func main() {...} 
func startJVM(cmd *Cmd) {...}
``` 
main（）函数不用变，重写startJVM（）函数，代码如下：
```go
func startJVM(cmd *Cmd) { 
cp := classpath.Parse(cmd.XjreOption, cmd.cpOption) 
fmt.Printf("classpath:%v class:%v args:%v\n", cp, cmd.class, cmd.args) 
className := strings.Replace(cmd.class, ".", "/", -1) 
classData, _, err := cp.ReadClass(className) 
if err != nil { 
fmt.Printf("Could not find or load main class %s\n", cmd.class) 
return 
}
fmt.Printf("class data:%v\n", classData) 
}
``` 
startJVM（）先打印出命令行参数，然后读取主类数据，并打印到控制台。虽然还是无法真正启动Java虚拟机，不过相比第1章，已经有了很大的进步。打开命令行窗口，执行下面的命令编译本章代码。
```shell script
go install jvmgo\ch02
```
编译成功后，在D：\go\workspace\bin目录下出现ch02.exe文件。执行ch02.exe，指定好-Xjre选项和类名，就可以把class文件的内容打印出来。虽然只是一堆看似杂乱无章的数字，但成就感还是会油然而生。笔者的测试结果如图2-1所示。 
![2-1](/img/2-1.png)
图2-1 ch02.exe的测试结果
#### 2.5 本章小结 
本章讨论了Java虚拟机从哪里寻找class文件，对类路径和-classpath命令行选项有了较为深入的了解，并且把抽象的类路径概念转变成了具体的代码。下一章将研究class文件格式，实现class文件解析。
