[TOC]
# 引言
RUST无疑是编程语言史中最难学的高级语言，在仅依靠静态编译的前提下实现一门安全的编程语言，这是必须付出的代价。无论如何，成为一门编程语言的老手的最佳办法就是深入分析，学习，理解优秀的代码，那RUST标准库的代码必然是不可绕过的最佳教材。另外，掌握RUST也必然意味对标准库的熟练掌握，深入了解标准库接口后面的内核无疑能帮助我们更好的掌握如何使用标准库。

## 本书目的
本书主要目的：
1. 给出一个分析标准库代码的脉络，使得读者能够快速理清标准库代码的彼此依赖关系。
2. 通过对标准库代码的学习，分析，让读者对RUST代码的编写技巧，规则有更好的理解，为读者成为RUST老手奠定基础。

## 目标读者
本书不适合初级程序员，本书针对的最佳对象是资深的C/C++程序员, 转学RUST。本书也适合已经采用RUST编写了一段时间程序，但希望对RUST有更深的了解，尤其是希望进行操作系统内核编程或通用编程框架编程的程序员。对于Java/python/go等语言的资深程序员，本书可以作为RUST与其他语言相比较的一个参考。阅读本书之前，读者最好已经学习过官方教程，中文翻译版链接如下[RUST程序设计语言](https://rustwiki.org/zh-CN/book/)。本书不是标准库参考手册，如需要参考手册，中文翻译版链接如下[RUST标准库参考手册](https://rustwiki.org/zh-CN/std/)。本书难度应该属于死灵书级别，中文翻译版死灵书链接如下[RUST秘典](https://nomicon.purewhite.io/)

## 本书代码分析的原则
因为不可能对标准库的所有代码进行分析。所以本书仅对作者认为有必要分析的源代码进行分析，遵循以下原则：
1. 对于RUST的所有权，借用，生命周期概念理解至关重要的代码
2. 重要的语法知识点的代码
3. 逻辑上较复杂和难以理解的代码
4. 体现RUST的编码技巧的代码

本文对各语言通用的算法技巧不做更多的关注和分析，因为本书的重点是帮助读者理解RUST语言本身独有的特点。

对于逻辑上很简单的代码，本书倾向于不分析。对于从函数名容易理解其功能，本书倾向于不分析。
本书倾向于对函数尽量给出比标准库参考手册更深层的一些理解。

## 本书约定
对于代码的解析，以代码中文注释的方式放在本书的代码中。

## IDE
作者使用的IDE是Visual Code，并安装了rust语言插件，足以满足代码阅读需求

# RUST标准库体系概述
RUST语言的设计目标是操作系统内核级的现代语法的系统编程语言，使用静态编译，并且不采用GC机制，保证开发出的应用即具备极高性能， 又能够在编译阶段就保证内存安全，并发安全，分支安全等安全性。
现代高级语言的标准库是语言的一个紧密的组成部分，语言的很多特性实际是标准库实现。RUST的库也是如此，但与其他采用GC方案的语言不同，其他语言编程目标是在操作系统之上运行的用户态程序。RUST的编程目标要加上操作系统内核，需要考虑内核与用户态两种模型。C语言解决这个问题的方法是只提供用户态的标准库，操作系统内核的库由各操作系统内核源代码自行实现。
RUST的现代语言特性决定了标准库无法象C语言那样把操作系统内核及用户态程序区分成完全独立的两个部分，所以只能更细致的设计，做模块化的处理。RUST标准库体系分为三个模块：语言核心库--core; alloc库；用户态 std库。

## 标准库路径
安装rust后，rust标准库的代码路径在作者机器上的路径如下：
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\core\src
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\alloc\src， 
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\std\src
本书主要分析上述三个目录下的代码

## core库
RUST语言核心库，适用于操作系统内核及用户态，包括RUST的基础类型及方法和关联函数，基本Trait及基础Trait在基础类型上的实现，其他函数等内容。core库是硬件架构和操作系统无关的可移植库。主要内容：

### 编译器内置intrinsics函数
包括内存操作函数，算数函数，位操作函数等， 这些函数通常与CPU硬件架构紧密相关，且一般需要汇编来提供最佳性能。 intrinsic函数实际上也是对CPU指令的屏蔽层。

### 基本Trait
[推荐这个链接](https://rustmagazine.github.io/rust_magazine_2021/chapter_7/rusts-standard-library-traits.html)
本节给出core库中的Trait一览

#### 运算符（ops）Trait
主要是各种用于表达式的RUST符号重载，包括算数计算符号，逻辑运算符号，位操作符号，解引用(*)符号, [index]数组下标符号， start..=end /start..end/start../..end/..=end/.. 等Range范围符号， ?号，||{..}闭包符号等，RUST原则是所有的运算符号都要能重载, 所以所有运算操作都定义了重载Trait。

#### 编译器内部实现的派生宏 Trait
如果类型结构中的每一个变量都实现了该Trait, 则此结构的该Trait可通过派生宏实现
Clone, Copy: Copy浅复制，Clone提供深复制
Debug: 类型的格式化输出
Default: 类型的default值，
Eq, Ord，PartialEQ, PartialOrd: 实现后可以对类型的变量做大,小,相等比较
Sync, Send: 实现此Trait的类型变量的引用可以安全在线程间共享
Hash: 实现结构的整体Hash值，这个Trait Hash是因为复杂才被加入，意义没有前面的大

#### Iterator
迭代器，RUST基础构架之一，也是RUST所有学习资料的重点。也是每个类型结构代码分析的重点。

#### 类型转换Trait
AsRef， AsMut, From，Into，TryFrom，TryInto, FloatToInt, FromStr。

#### 异步编程Trait
此处略，需要单独做分析

#### 内存相关Trait
此处略，需要单独做分析

### 基本数据类型
包括整数类型，浮点类型，布尔类型，字符类型，单元类型，内容主要是实现运算符Trait, 类型转换Trait, 派生宏Trait等，字符类型包括对unicode，ascii的不同编码的处理。整数类型有大小端变换的处理。

### 数组、切片及Range
这些类型对Iterator Trait, 运算符Trait, 类型转换Trait, 派生宏Trait及其他一些方法函数。

### Option/Result/Marker等关键的语言级别Enum类型
RUST安全特性的重点，也是各种学习资料的重点，不赘述。后继章节将对代码给以说明

### RUST内存相关类型及内容
alloc, mem, ptr等模块，RUST的内存操作。是本书代码分析的第一个重点。

### RUST字符串相关库
字符串str，string，fmt, panic, debug, log等

### RUST时间库
Duration等 

## alloc库
alloc库主要实现需要进行动态堆内存申请的智能指针类型，集合类型及他们的方法，函数，Trait等内容，这些仅需要建立在core库模块之上，std会对alloc模块库的内容做重新的封装。alloc库适用于操作系统内核及用户态程序。
包括：
1.基本内存申请；Allocator Trait; Allocator的实现结构Global
2.基础智能指针：Box<T>, Rc<T>, 
3.动态数组内存类型: RawVec<T>, Vec<T>
4.字符串类型：&str, String
5.并发编程指针类型: Arc<T>
6.指针内访问类型: Cell<T>, RefCell<T>
还有些其他类型，一般仅在标准库内部使用，后文在需要的时候再介绍及分析。

## std库
std是在操作系统支撑下运行的只适用于用户态程序的库，core库实现的内容基本在std库也有对应的实现。其他内容主要是将操作系统系统调用封装为适合rust特征的结构和Trait,包括：
1.进程，线程库
2.网络库
3.文件操作库
4.环境变量及参数
5.互斥与同步库，读写锁
6.定时器
7.输入输出的数据结构，
8.系统事件，对epoll,kevent等的封装
可以将std库看做基本常用的容器类型及操作系统封装库。

## 小结
RUST的目标和现代编程语言的特点决定了它的库需要仔细的模块化设计。RUST的alloc库及std库都是基于core库。RUST的库设计非常巧妙和仔细，使得RUST完美的实现了对各种硬件架构平台的兼容，对各种操作系统平台的兼容。


# RUST泛型小议
RUST是一门生存在泛型的基础之上的语言。其他语言不使用泛型也不影响编程，泛型只是一个语法中的强大工具。与之相对，RUST是离开了泛型就无法完成程序编写，泛型与语法共生。

## 直接针对泛型的方法和trait实现
其他语言的泛型，是作为类型结构体成员，或是函数的输入/返回参数出现在代码中，是配角。RUST的泛型则可以作为主角，可以直接对泛型实现方法和trait。如：
```rust
//T:?Sized基本上就是所有的类型，直接impl <T> Borrow<T>实际上隐含了 T:Sized。所以 T:?Sized比T范围更广阔
impl<T: ?Sized> Borrow<T> for T {
    fn borrow(&self) -> &T {
        self
    }
}

impl<T: ?Sized> BorrowMut<T> for T {
    fn borrow_mut(&mut self) -> &mut T {
        self
    }
}
```
以上代码基本上就是对所有的类型都实现了Borrow<T>的trait。  
直接针对泛型做方法和trait的实现是强大的工具，它的作用：  
- 针对泛型的代码会更内聚，方法总比函数具备更明显的模块性
- 逻辑更清晰及系统化更好
- 具备更好的可扩展性
- 更好的支持函数式编程

## 泛型的层次关系
RUST的泛型从一般到特殊会形成一种层次结构，有些类似于面对对象的基类和子类关系： 
最基层： T  没有任何约束的T是泛型的基类   
一级子层： 裸指针类型`* const T/* mut T`; 切片类型`[T]`; 数组类型`[T;N]`; 引用类型`&T/&mut T`; trait约束类型`T:trait`; 泛型元组`(T, U...)`; 泛型复合类型`struct <T>; enum <T>; union<T>` 及具体类型 `u8/u16/i8/bool/f32/&str/String...`     
二级子层： 对一级子层的T赋以具体类型 如：`* const u8; [i32]`，或者将一级子层中的T再次做一级子层的具化，例如：`* const [T]; [*const T]; &(*const T); * const T where T:trait; struct <T:trait>` 

可以一直递归下去，但没有太多的意义。
显然，针对基层类型实现的方法和trait可以应用到层级高的泛型类型中。
例如：
```rust
impl <T> Option<T> {...}
impl<T, U> Option<(T, U)> {...}
impl<T: Copy> Option<&T> {...}
impl<T: Default> Option<T> {...}
```
以上是标准库对Option<T> 的不同泛型进行的方法实现定义。一般先针对基层泛型实现方法及trait，然后再针对高层次的泛型做方法及trait实现。  

类似的实现再试举如下几例：  
```rust
impl <T:?Sized> *const T {...}
impl <T:?Sized> *const [T] {...}
impl <T:?Sized> *mut T{ ...}
impl <T:?Sized> *mut [T] {...}
impl <T> [T] { ...}
impl <T, const N:usize> [T;N]{...}
impl AsRef<[u8]> for str {...}
impl AsRef<str> for str {...}
```

RUST中，可以定义新的trait, 并根据需要在已定义的类型上实现新的trait。这就显然比其他的面对对象的语言具备更好的可扩展性。

# RUST标准库内存模块代码分析
内存模块的代码路径举例如下(以作者电脑上的路径):
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\core\src\alloc\*.*
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\core\src\ptr\*.*
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\core\src\mem\*.*
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\core\src\intrinsic.rs
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\alloc\src\alloc.rs

RUST之所以被认为难学，是因为RUST与C相同，需要对内存做彻底的控制，即程序可以在代码中编写专属内存管理系统，并将内存管理系统与语言类型相关联，将内存块与语言类型做自如的转换。对于当前现代语法的高级语言如Java/Python/JS/Go，内存管理实际上是编译器的任务，这就导致大部分程序员对于内存管理缺乏经验，所以对RUST内存安全相关的所有权/生命周期等缺乏实践认知。相对于C，RUST的现代语法特性及内存安全导致RUST的内存块与类型系统的转换的细节不容易被透彻理解。本节将从标准库的内存模块的代码分析中给出RUST内存的本质。理解了RUST内存及内存安全，RUST语言的最难关便过了。

从内存角度考察一个变量，则每个变量具备统一的内存参数，这些参数是：
1. 变量的首地址，是一个usize的数值
2. 变量类型占用的内存块大小
3. 变量类型内存字节对齐的基数
4. 变量类型中成员内存顺序  

如果变量成员是复合类型，可递归上面的四个参数。 
RUST认为变量类型成员顺序与编译优化不可分割，因此，变量成员内存顺序完全由编译器控制，这与C不同，C中变量类型成员的顺序是不能被编译器改动的。这使得C变量的内存布局对程序员是透明的。这种透明性导致了C语言在设计类型内存布局的操作中会出现很多坏代码。如，直接用头指针+偏移数值来获得类型内部变量的指针，直接导致变量类型可修改性极差。  
与C相同，RUST具备将一块内存块直接转换成某一类型变量的能力。这一能力是RUST操作系统内核编程及高效的一个基石。但因为这个转换使得代码可以绕过编译器的类型系统检查，造成了BUG也绕过了编译器的某些错误检查，而这些错误很可能在系统运行很久之后才真正的出错，造成排错的极高成本。
GC类语言去掉了这一能力，但也牺牲了性能，且无法作为系统级别语言。RUST没有因噎废食，在保留能力的同时给出这一能力明确的危险标识unsafe, 加上整体的内存安全框架设计，使得此类错误更易被发现，更易被定位，极大的降低了错误的数目及排错的成本。
unsafe容易让初学RUST语言的程序员产生排斥感，但unsafe实际上是RUST不可分割的部分，一个好的RUST程序员绝不是不使用unsafe，而是能够准确的把握好unsafe使用的合适场合及合适范围，必要的时候必须使用，但不滥用。

掌握RUST的内存，主要有如下几个部分：
1. 编译器提供的固有内存操作函数
2. 内存块与类型系统的结合点：裸指针 `*const T/*mut T`
3. 裸指针的包装结构: `NonNull<T>/Unique<T>`
4. 未初始化内存块的处理：`MaybeUninit<T>/ManuallyDrop<T>`
5. 堆内存申请及释放
   
## 裸指针标准库代码分析
裸指针`*const T/* mut T`将内存和类型系统相连接，*const T代表了一个内存块，指示了内存块首地址，大小，对齐等属性，以及后文提到的元数据，但不保证这个内存块的有效性和安全性。
与`*const T/* mu T`不同，`&T/&mut T`则保证内存块是安全和有效的，这表示`&T/&mut T`满足内存块首地址对齐，内存块已经完成了初始化。在RUST中，`&T/&mut T`是被绑定在某一内存块上，只能用于读写这一内存块。  
对于内存块更复杂的操作，由`*const T/*mut T` 负责，主要有：
1. 将usize类型数值强制转换成裸指针类型，以此数值为首地址的内存块被转换为相应的类型。这一转换是不安全的。
2. 在不同的裸指针类型之间进行强制转换，实质上完成了裸指针指向的内存块的类型强转，这一转换是不安全的。
3. `*const u8`作为堆内存申请的内存块绑定变量
4. 内存块置值操作，如清零或置一个魔术值
5. 显示的内存块拷贝操作，某些情况下，内存块拷贝是必须的高性能方式。
6. 利用指针偏移计算获取新的内存块， 在数组及切片访问，字符串，协议字节填写，文件缓存等都需要指针偏移计算。
7. 从外部的C函数接口对接的指针参数
8... 

RUST的裸指针类型不象C语言的指针类型那样仅仅是一个地址值，为满足实现内存安全的类型系统需求，并兼顾内存使用效率和方便性，RUST的裸指针实质是一个较复杂的类型结构体。

### 裸指针具体实现
`*const T/*mut T`实质是个数据结构体，由两个部分组成，第一个部分是一个内存地址，第二个部分对这个内存地址的约束性描述-元数据
```rust
//从下面结构定义可以看到，裸指针本质就是PtrComponents<T>
pub(crate) union PtrRepr<T: ?Sized> {
    pub(crate) const_ptr: *const T,
    pub(crate) mut_ptr: *mut T,
    pub(crate) components: PtrComponents<T>,
}

pub(crate) struct PtrComponents<T: ?Sized> {
    //*const ()保证元数据部分是空 
    pub(crate) data_address: *const (),
    //不同类型指针的元数据
    pub(crate) metadata: <T as Pointee>::Metadata,
}

//从下面Pointee的定义可以看到一个RUST的编程技巧，即Trait可以只用来实现对关联类型的指定，Pointee这一Trait即只用来指定Metadata的类型。
pub trait Pointee {
    /// The type for metadata in pointers and references to `Self`.
    type Metadata: Copy + Send + Sync + Ord + Hash + Unpin;
}
//廋指针元数据是单元类型，即是空
pub trait Thin = Pointee<Metadata = ()>;
```
元数据的规则:
* 对于固定大小类型的指针（实现了 `Sized` Trait）, RUST定义为廋指针(thin pointer)，元数据大小为0，类型为(),这里要注意，RUST中数组也是固定大小的类型，运行中对数组下标合法性的检测，就是比较是否已经越过了数组的内存大小。
* 对于动态大小类型的指针(DST 类型)，RUST定义为胖指针(fat pointer 或 wide pointer), 元数据为：  
    * 对于结构类型，如果最后一个成员是动态大小类型(结构的其他成员不允许为动态大小类型)，则元数据为此动态大小类型  
    的元数据  
    * 对于`str`类型, 元数据是按字节计算的长度值，元数据类型是usize  
    * 对于切片类型，例如`[T]`类型，元数据是数组元素的数目值，元数据类型是usize   
    * 对于trait对象，例如 dyn SomeTrait， 元数据是 [DynMetadata<Self>][DynMetadata]（后面代码解释）
    （例如：DynMetadata<dyn SomeTrait>)
随着RUST的发展，有可能会根据需要引入新的元数据种类。

在标准库代码当中没有指针类型如何实现Pointee Trait的代码，编译器针对每个类型自动的实现了Pointee。
如下为rust编译器代码的一个摘录
```rust
    pub fn ptr_metadata_ty(&'tcx self, tcx: TyCtxt<'tcx>) -> Ty<'tcx> {
        // FIXME: should this normalize?
        let tail = tcx.struct_tail_without_normalization(self);
        match tail.kind() {
            // Sized types
            ty::Infer(ty::IntVar(_) | ty::FloatVar(_))
            | ty::Uint(_)
            | ty::Int(_)
            | ty::Bool
            | ty::Float(_)
            | ty::FnDef(..)
            | ty::FnPtr(_)
            | ty::RawPtr(..)
            | ty::Char
            | ty::Ref(..)
            | ty::Generator(..)
            | ty::GeneratorWitness(..)
            | ty::Array(..)
            | ty::Closure(..)
            | ty::Never
            | ty::Error(_)
            | ty::Foreign(..)
            | ty::Adt(..)
            // 如果是固定类型，元数据是单元类型 tcx.types.unit，即为空
            | ty::Tuple(..) => tcx.types.unit,

            //对于字符串和切片类型，元数据为长度tcx.types.usize，是元素长度
            ty::Str | ty::Slice(_) => tcx.types.usize,

            //对于dyn Trait类型， 元数据从具体的DynMetadata获取*
            ty::Dynamic(..) => {
                let dyn_metadata = tcx.lang_items().dyn_metadata().unwrap();
                tcx.type_of(dyn_metadata).subst(tcx, &[tail.into()])
            },
            
            //以下类型不应有元数据
            ty::Projection(_)
            | ty::Param(_)
            | ty::Opaque(..)
            | ty::Infer(ty::TyVar(_))
            | ty::Bound(..)
            | ty::Placeholder(..)
            | ty::Infer(ty::FreshTy(_) | ty::FreshIntTy(_) | ty::FreshFloatTy(_)) => {
                bug!("`ptr_metadata_ty` applied to unexpected type: {:?}", tail)
            }
        }
    }
```
以上代码中的中文注释比较清晰的说明了编译器对每一个类型（或类型指针）都实现了Pointee中元数据类型的获取。
对于Trait对象的元数据的具体结构定义见如下代码：
```rust
//dyn Trait裸指针的元数据结构
pub struct DynMetadata<Dyn: ?Sized> {
    //堆中的函数VTTable变量的指针
    vtable_ptr: &'static VTable,
    //标示结构对Dyn的所有权关系，
    //其中PhantomData与具体变量的联系在初始化时由编译器自行推断完成, 这里PhantomData主要对编译器做出
    //提示做Drop check是需要注意Dyn由此结构drop。
    phantom: crate::marker::PhantomData<Dyn>,
}

struct VTable {
    //指向实现Trait的类型结构体的drop_in_place函数的指针
    drop_in_place: fn(*mut ()),
    //指向实现Trait的类型结构体的大小
    size_of: usize,
    //指向实现Trait的类型结构体字节对齐
    align_of: usize,
    //后继是结构体实现Trait的所有方法的指针数组
}
```

元数据类型相同的裸指针可以任意的转换，例如：可以有 * const [usize; 3] as * const[usize; 5] 这种语句
元数据类型不同的裸指针之间不能转换，例如；* const [usize;3] as *const[usize] 这种语句无法通过编译器 

### 裸指针的操作函数——intrinsic模块内存相关固有函数
intrinsics模块中的函数由编译器内置实现，并提供给其他模块使用。固有函数不提供代码，所以对其主要是了解功能和如何使用，intrinsics模块的内存函数一般不由库以外的代码直接调用，而是由mem模块和ptr模块封装后再提供给其他模块。 

内存申请及释放函数：
`intrinsics::drop_in_place<T:Sized?>(to_drop: * mut T)` 在某些情况下，我们会主动的将变量设置成不允许编译器自动调用变量的drop函数， 此时如果仍然需要对变量调用drop，则在代码中显示调用此函数以出发对T类型的drop调用。  
`intrinsics::forget<T:Sized?> (_:T)`, 代码中调用这个函数后，编译器不对forget的变量自动调用变量的drop函数。 
`intrinsics::needs_drop<T>()->bool`, 判断T类型是否需要做drop操作，实现了Copy Trait的类型会返回false  

类型转换：
`intrinsics::transmute<T,U>(e:T)->U`, 对于内存布局相同的类型 T和U, 完成将类型T变量转换为类型U变量，此时T的所有权将转换为U的所有权   

指针偏移函数:
`intrinsics::offset<T>(dst: *const T, offset: usize)->* const T`, 相当于C的类型指针加计算   
`intrinsics::ptr_offset_from<T>(ptr: *const T, base: *const T) -> isize` 基于类型T内存布局的两个裸指针之间的偏移量  

内存块内容修改函数:
`intrinsics::copy<T>(src:*const T, dst: *mut T, count:usize)`, 内存拷贝， src和dst内存可重叠， 类似c语言中的memmove, 此时dst原有内存如果已经初始化，则会出现内存泄漏。src的所有权实际会被复制，从而也造成重复drop问题。    
`intrinsics::copy_no_overlapping<T>(src:*const T, dst: * mut T, count:usize)`, 内存拷贝， src和dst内存不重叠   
`intrinsics::write_bytes(dst: *mut T, val:u8, count:usize)` , C语言的memset的RUST实现, 此时，原内存如果已经初始化，则原内存的变量可能造成内存泄漏，且因为编译器会继续对dst的内存块做drop调用，有可能会UB。  

类型内存参数函数：
`intrinsics::size_of<T>()->usize` 类型内存空间字节大小  
`intrinsics::min_align_of<T>()->usize` 返回类型对齐字节大小  
`intrinsics::size_of_val<T>(_:*const T)->usize`返回指针指向的变量内存空间字节大小  
`intrinsics::min_align_of_val<T>(_: * const T)->usize` 返回指针指向的变量对齐字节大小  

禁止优化的内存函数：
形如`volatile_xxxx` 的函数是通知编译器不做内存优化的操作函数,一般硬件相关操作需要禁止优化。  
`intrinsics::volatile_copy_nonoverlapping_memory<T>(dst: *mut T, src: *const T, count: usize)` 内存拷贝  
`intrinsics::volatile_copy_memory<T>(dst: *mut T, src: *const T, count: usize)` 功能类似C语言memmove  
`intrinsics::volatile_set_memory<T>(dst: *mut T, val: u8, count: usize)`  功能类似C语言memset  
`intrinsics::volatile_load<T>(src: *const T) -> T`读取内存或寄存器，字节对齐  
`intrinsics::volatile_store<T>(dst: *mut T, val: T)`内存或寄存器写入，字节对齐  
`intrinsics::unaligned_volatile_load<T>(src: *const T) -> T` 字节非对齐  
`intrinsics::unaligned_volatile_store<T>(dst: *mut T, val: T)`字节非对齐  

内存比较函数：
`intrinsics::raw_eq<T>(a: &T, b: &T) -> bool` 内存比较，类似C语言memcmp  
`pub fn ptr_guaranteed_eq<T>(ptr: *const T, other: *const T) -> bool` 判断两个指针是否判断, 相等返回ture, 不等返回false  
`pub fn ptr_guaranteed_ne<T>(ptr: *const T, other: *const T) -> bool` 判断两个指针是否不等，不等返回true  

### 裸指针方法
RUST针对`*const T/*mut T`的类型实现了若干方法，能够对语言的原生类型实现方法，并能够扩展，这个是在其他语言中少见的：
```rust
impl <T:?Sized> * const T {
    ...
}
impl <T:?Sized> *mut T{
    ...
}
impl <T> *const [T] {
    ...
}
impl <T> *mut [T] {
    ...
}
```
对于裸指针，RUST 将之分为最基础的 `* const T/* mut T`， 以及在`* const T/*mut T` 基础上特化的切片类型[T]的裸指针`* const [T]/*mut [T]`, 可以认为 `* const[T]`是`* const T`的子类，当然，RUST不使用这个定义。
标准库针对这两种类型实现了一些关联函数及方法。这里一定注意，所有针对 `* const T`的方法在`* const [T]`上都是适用的。

以上有几点值得注意：
1. 可以针对原生类型实现方法(实现Trait)，这体现了RUST类型系统的强大扩展性，也是对函数式编程的强大支持
2. 针对泛型约束实现方法，我们可以大致认为`*const T/* mut T`实质是一种泛型约束，`*const [T]/*mut [T]`是更进一步的约束，这使得RUST可以具备更好的数据抽象能力，简化代码，复用模块。

#### 裸指针的创建
直接从已经初始化的变量创建裸指针：
```rust
    &T as *const T;
    &mut T as * mut T;
```
直接用usize的数值创建裸指针：
```rust
    {
        let  a: usize = 0xf000000000000000;
        unsafe {a as * const T}
    }
```
操作系统内核经常需要直接将一个地址数值转换为某一类型的裸指针

RUST也提供了一些其他的裸指针创建关联函数：
`ptr::null<T>() -> *const T` 创建一个0值的`*const T`，实际上就是 `0 as *const T`，用null()函数明显更符合程序员的习惯  
`ptr::null_mut<T>()->*mut T` 除了类型以外，其他同上  
`ptr::from_raw_parts<T: ?Sized>(data_address: *const (), metadata: <T as Pointee>::Metadata) -> *const T` 从内存地址和元数据创建裸指针  
`ptr::from_raw_parts_mut<T: ?Sized>(data_address: *mut (), metadata: <T as Pointee>::Metadata) -> *mut T` 功能同上，创建可变裸指针   
RUST裸指针类型转换时，经常使用以上两个函数获得需要的指针类型。

切片类型的裸指针创建函数如下：
`ptr::slice_from_raw_parts<T>(data: *const T, len: usize) -> *const [T] `  
`ptr::slice_from_raw_parts_mut<T>(data: *mut T, len: usize) -> *mut [T]` 由裸指针类型及切片长度获得切片类型裸指针，调用代码应保证data事实上就是切片的裸指针地址。由类型裸指针转换为切片类型裸指针最突出的应用之一是内存申请，申请的内存返回 * const u8的指针，这个裸指针是没有包含内存大小的，只有头地址，因此需要将这个指针转换为 * const [u8]，将申请的内存大小包含入裸指针结构体中。 
slice_from_raw_parts代码如下： 
```rust
pub const fn slice_from_raw_parts<T>(data: *const T, len: usize) -> *const [T] {
    //data.cast()将*const T转换为 *const()
    from_raw_parts(data.cast(), len)
}

pub const fn from_raw_parts<T: ?Sized>(
    data_address: *const (),
    metadata: <T as Pointee>::Metadata,
) -> *const T {
    //由以下代码可以确认 * const T实质就是PtrRepr类型结构体。
    unsafe { PtrRepr { components: PtrComponents { data_address, metadata } }.const_ptr }
}
```

#### 不属于方法的裸指针函数
`ptr::drop_in_place<T: ?Sized>(to_drop: *mut T)` 此函数是编译器实现的，用于不需要RUST自动drop时，由程序代码调用以释放内存  
`ptr::metadata<T: ?Sized>(ptr: *const T) -> <T as Pointee>::Metadata `用来返回裸指针的元数据  
`ptr::eq<T>(a: *const T, b: *const T)->bool` 比较指针，此处需要注意，地址比较不但是地址，也比较元数据
ptr模块的函数大部分逻辑都比较简单。很多就是对intrinsic 函数做调用。

#### 裸指针类型转换方法
裸指针类型之间的转换：
`*const T::cast<U>(self) -> *const U ` ，本质上就是一个`*const T as *const U`。利用RUST的类型推断，此函数可以简化代码并支持链式调用。 
`*mut T::cast<U>(self)->*mut U` 同上。
调用以上的函数要注意，如果后继要把返回的指针转换成引用，那必须保证T类型与U类型内存布局完全一致。如果仅仅是将返回值做数值应用，则此约束可以不遵守，cast函数转换后的类型通常由编译器自行推断，有时需要仔细分析。  

裸指针与引用之间的类型转换：
```*const T::as_ref<`a>(self) -> Option<&`a T>``` 将裸指针转换为引用，因为*const T可能为零，所有需要转换为``Option<& `a T>``类型，转换的安全性由程序员保证，尤其注意满足RUST对引用的安全要求。这里要注意，转换后的生命周期实际上与原指针指向变量的生命周期相独立。因此，生命周期的正确性将由程序员保证。  
```*mut T::as_ref<`a>(self)->Option<&`a T>```   同上
```*mut T::as_mut<`a>(self)->Option<&`a mut T>```同上，但转化类型为 &mut T。  

切片类型裸指针类型转换：
`ptr::*const [T]::as_ptr(self) -> *const T` 将切片类型的裸指针转换为切片成员类型的裸指针， 这个转换会导致指针的元数据丢失  
`ptr::*mut [T]::as_mut_ptr(self) -> *mut T` 同上

#### 裸指针结构体属性相关方法：
`ptr::*const T::to_raw_parts(self) -> (*const (), <T as super::Pointee>::Metadata)`   
`ptr::*mut T::to_raw_parts(self)->(* const (), <T as super::Pointee>::Metadata)`  由裸指针获得地址及元数据  
`ptr::*const T::is_null(self)->bool`  
`ptr::*mut T::is_null(self)->bool此`函数判断裸指针的地址值是否为0  

切片类型裸指针：
`ptr::*const [T]:: len(self) -> usize` 获取切片长度，直接从裸指针的元数据获取长度  
`ptr:: *mut [T]:: len(self) -> usize` 同上

#### 裸指针偏移计算相关方法
`ptr::*const T::offset(self, count:isize)->* const T` 得到偏移后的裸指针  
`ptr::*const T::wrapping_offset(self, count: isize) -> *const T` 考虑溢出绕回的offset  
`ptr::*const T::offset_from(self, origin: *const T) -> isize` 计算两个裸指针的offset值  
`ptr::*mut T::offset(self, count:isize)->* mut T` 偏移后的裸指针  
`ptr::*const T::wrapping_offset(self, count: isize) -> *const T` 考虑溢出绕回的offset  
`ptr::*const T::offset_from(self, origin: *const T) -> isize` 计算两个裸指针的offset值  
以上两个方法基本上通过intrinsic的函数实现

`ptr::*const T::add(self, count: usize) -> Self`   
`ptr::*const T::wraping_add(self, count: usize)->Self`  
`ptr::*const T::sub(self, count:usize) -> Self`  
`ptr::*const T::wrapping_sub(self, count:usize) -> Self`   
`ptr::*mut T::add(self, count: usize) -> Self`   
`ptr::*mut T::wraping_add(self, count: usize)->Self`  
`ptr::*mut T::sub(self, count:usize) -> Self`  
`ptr::*mut T::wrapping_sub(self, count:usize) -> Self`   
以上是对offset函数的包装，使之更符合语义习惯，并便于理解  

#### 裸指针直接赋值方法
```rust
    //该方法用于仅给指针结构体的 address部分赋值  
    pub fn set_ptr_value(mut self, val: *const u8) -> Self {
        // 以下代码因为只修改PtrComponent.address，所以不能直接用相等
        // 代码采取的方案是取self的可变引用，将此引用转换为裸指针的裸指针，
        let thin = &mut self as *mut *const T as *mut *const u8;
        // 这个赋值仅仅做了address的赋值，对于瘦指针，这个相当于赋值操作，
        // 对于胖指针，则没有改变胖指针的元数据。这种操作方式仅仅在极少数的情况下
        // 可以使用，极度危险。
        unsafe { *thin = val };
        self
    }
```
本节还有一部分裸指针方法没有介绍，留到mem模块分析完以后再介绍会更易于理解。 

### 裸指针小结
裸指针相关的代码多数比较简单，重要的是理解裸指针的概念，理解intrinsic 相关函数，这样才能够准确的理解代码。

#### RUST引用`&T`的安全要求
1. 引用的内存地址必须满足类型T的内存对齐要求
2. 引用的内存内容必须是初始化过的
举例：
 ```rust
    #[repr(packed)]
    struct RefTest {a:u8, b:u16, c:u32}
    fn main() {
        let test = RefTest{a:1, b:2, c:3};
        //下面代码无法通过编译，因为test.b 内存字节位于奇数，无法用于借用
        let ref1 = &test.b
    }
 ```

## MaybeUninit<T>标准库代码分析
RUST对于变量的要求是必须初始化后才能使用，否则就会编译告警。但在程序中，总有内存还未初始化，但需要使用的情况：
1. 从堆申请的内存块，这些内存块都是没有初始化的
2. 需要定义一个新的泛型变量时，并且不合适用转移所有权进行赋值时
3. 需要定义一个新的变量，但希望不初始化便能使用其引用时
4. 定义一个数组，但必须在后继代码对数组成员初始化时
5. ...

为了处理这种需要在代码中使用未初始化内存的情况，RUST标准库定义了MaybeUninit<T>

### MaybeUninit<T>结构定义
源代码如下：
```rust
    #[repr(transparent)] 
    pub union MaybeUninit<T> {
        uninit: (),
        value: ManuallyDrop<T>,
    }
```
属性`repr(transparent)`实际上表示外部的封装结构在内存中等价于内部的变量,
MaybeUninit的内存布局就是`ManuallyDrop<T>`的内存布局，从后文可以看到，`ManuallyDrop<T>`实际就是T的内存布局。所以MaybeUninit在内存中实质也就是T类型。
RUST的引用使用的内存块必须保证内存对齐及赋以初始值，未初始化的内存块和清零的内存块都不能满足引用的条件。但堆内存申请后都是未初始化的，且在程序中某些情况下也需要先将内存设置为未初始化，尤其在处理泛型时。因此，RUST提供了MaybeUninit<T>容器来实现对未初始化变量的封装，以便在不引发编译错误完成对T类型未初始化变量的相关操作.
如果T类型的变量未初始化，那需要显式的提醒编译器不做T类型的drop操作，因为drop操作可能会对T类型内部的变量做处理，从而引用未初始化的内容，造成UB。实际上，未初始化的内存不必做drop。
RUST用ManuallyDrop<T>封装结构完成了对编译器的显示提示，对于用ManuallyDrop<T>封装的变量，生命周期终止的时候编译器不会调用drop操作。

### ManuallyDrop<T> 结构及方法
源代码如下：
```rust
#[repr(transparent)]
pub struct ManuallyDrop<T: ?Sized> {
    value: T,
}
```
一个变量被ManuallyDrop获取所有权后，RUST编译器将不再对其自动调用drop操作。因此如果封装入ManuallyDrop的变量实际上需要drop，那必须将ManuallyDrop的变量的所有权在后继转移出去。因为对于模块外的代码，value是私有的，所以必须调用方法才能将value的所有权转移出去。

重点关注的一些方法： 
`ManuallyDrop<T>::new（val:T) -> ManuallyDrop<T>`, 此函数返回ManuallyDrop变量拥有传入的T类型变量所有权，并将此块内存直接用ManuallyDrop封装, 对于val，编译器不再主动做drop操作。 
```rust
    pub const fn new(value: T) -> ManuallyDrop<T> {
        //所有权转移到结构体内部，value生命周期结束时不会引发drop
        ManuallyDrop { value }
    }
```  
`ManuallyDrop<T>::into_inner(slot: ManuallyDrop<T>)->T`, 将封装的T类型变量所有权转移出来，转移出来的变量生命周期终止时，编译器会自动调用类型的drop。  
```rust
    pub const fn into_inner(slot: ManuallyDrop<T>) -> T {
        //将value解封装，所有权转移到返回值中，编译器重新对所有权做处理
        slot.value
    }
```

`ManuallyDrop<T>::drop(slot: &mut ManuallyDrop<T>)`，drop掉内部变量，封装入ManuallyDrop<T>的变量一定是在程序运行的某一时期不需要编译器drop，所以调用这个函数的时候一定要注意正确性。 
`ManuallyDrop<T>::deref(&self)-> & T`, 返回内部包装的变量的引用 
```rust
    fn deref(&self) -> &T {
        //返回后，代码可以用&T对self.value做读操作,但不改变drop的规则
        &self.value
    }
```
`ManuallyDrop<T>::deref_mut(&mut self)-> & mut T`返回内部包装的变量的可变引用，调用代码可以利用可变引用对内部变量赋值，但不改变drop机制  

ManuallyDrop代码举例：
```rust
    use std::mem::ManuallyDrop;
    let mut x = ManuallyDrop::new(String::from("Hello World!"));
    x.truncate(5); // 此时会调用deref
    assert_eq!(*x, "Hello");
    // 但对x的drop不会再发生
```

#### MaybeUninit<T> 创建方法
`MaybeUninit<T>::uninit()->MaybeUninit<T>`, 可视为在栈空间上申请内存的方法，申请的内存大小是T类型的内存大小，该内存没有初始化。利用泛型和Union内存布局，RUST巧妙的利用此函数在栈上申请一块未初始化内存。此函数非常非常非常值得关注，在需要在栈空间定义一个未初始化泛型时，应第一时间想到MaybeUninit::<T>::uninit()。
```rust
    pub const fn uninit() -> MaybeUninit<T> {
        //变量内存布局与T类型完全一致
        MaybeUninit { uninit: () }
    }
```

`MaybeUninit<T>::new(val:T)->MaybeUninit<T>`, 内部用ManuallyDrop封装了val, 然后用MaybeUninit封装ManuallyDrop。因为如果T没有初始化过，调用这个函数会编译失败，所以此时内存实际上已经初始化过了。调用此函数要额外注意val的drop必须在后继有交代。
```rust
    pub const fn new(val: T) -> MaybeUninit<T> {
        //val这个时候是初始化过的。
        MaybeUninit { value: ManuallyDrop::new(val) }
    }
```
`MaybeUninit<T>::zeroed()->MaybeUninit<T>`, 申请了T类型内存并清零。 
```rust
    pub fn zeroed() -> MaybeUninit<T> {
        let mut u = MaybeUninit::<T>::uninit();
        unsafe {
            //因为没有初始化，所以不存在所有权问题，
            //必须使用ptr::write_bytes，否则无法给内存清0
            //ptr::write_bytes直接调用了intrinsics::write_bytes
            u.as_mut_ptr().write_bytes(0u8, 1);
        }
        u
    }
```
#### 对未初始化的变量赋值的方法
将值写入MaybeUninit<T>
`MaybeUninit<T>::write(val)->&mut T`, 这个函数是在未初始化时使用，如果已经调用过write，且不希望解封，那后继的赋值使用返回的&mut T。代码如下： 
```rust
    pub const fn write(&mut self, val: T) -> &mut T {
        //下面这个赋值，会导致原*self的MaybeUninit<T>的变量生命周期截止，会调用drop。但不会对内部的T类型变量做drop调用。所以如果*self内部的T类型变量已经被初始化且需要做drop，那会造成内存泄漏。所以下面这个等式实际上隐含了self内部的T类型变量必须是未初始化的或者T类型变量不需要drop。
        *self = MaybeUninit::new(val);
        // 函数调用后的赋值用返回的&mut T来做。
        unsafe { self.assume_init_mut() }
    }
```
#### 初始化后解封装的方法
用assume_init返回初始化后的变量并消费掉MaybeUninit<T>变量，这是最标准的做法：
`MaybeUninit<T>::assume_init()->T`,代码如下：
```rust
    pub const unsafe fn assume_init(self) -> T {
        // 调用者必须保证self已经初始化了
        unsafe {
            intrinsics::assert_inhabited::<T>();
            //把T的所有权返回，编译器会主动对T调用drop
            ManuallyDrop::into_inner(self.value)
        }
    }
```
assume_init_read是不消费self的情况下获得内部T变量，内部T变量的所有权已经转移到返回变量，后继要注意不能再次调用其他解封装函数，否则会出现双份所有权，导致UB
```rust
    pub const unsafe fn assume_init_read(&self) -> T {
        
        unsafe {
            intrinsics::assert_inhabited::<T>();
            //会调用ptr::read
            self.as_ptr().read()
        }
    }
    //此函即ptr::read, 会复制一个变量，此时注意，实际上src指向的变量的所有权已经转移给了返回变量，
    //所以调用此函数的前提是src后继一定不能调用T类型的drop函数，例如src本身处于ManallyDrop，或后继对src调用forget，或给src绑定新变量。
    //在RUST中，不支持 let xxx = *(&T) 这种转移所有权的方式，因此对于只有指针输入，又要转移所有权的，智能利用浅拷贝进行粗暴转移。
    pub const unsafe fn read<T>(src: *const T) -> T {` 
        //利用MaybeUninit::uninit申请未初始化的T类型内存
        let mut tmp = MaybeUninit::<T>::uninit();
        unsafe {
            //完成内存拷贝
            copy_nonoverlapping(src, tmp.as_mut_ptr(), 1);
            //初始化后的内存解封装并返回
            tmp.assume_init()
        }
    }
```
与上个函数比较类似的ManuallyDrop<T>::take方法，用take函数将变量复制并获得变量的所有权。此时原变量仍然保留在ManuallyDrop中，后继不能再调用其他解封装函数，否则可能会出现UB。这里要特别注意理解take已经把变量的所有权转移到返回变量中。  
```rust
    pub unsafe fn take(slot: &mut ManuallyDrop<T>) -> T {
        // 拷贝内部变量，并返回内部变量的所有权
        // 返回后，原有的变量所有权已经消失，不能再用into_inner来返回
        // 否则会UB
        unsafe { ptr::read(&slot.value) }
    }

```
`MaybeUninit<T>::assume_init_drop(&self)` 对于已经初始化过的MaybeUninit<T>， 如果不做他用，必须调用此函数以触发T类型的drop函数。  
`MaybeUninit<T>::assume_init_ref(&self)->&T` 返回内部T类型变量的借用，调用者应保证内部T类型变量已经初始化，返回值按照一个普通的引用使用。应注意返回值的生命周期应该小于self的生命周期
`MaybeUninit<T>::assume_init_mut(&mut self)->&mut T`返回内部T类型变量的可变借用，调用者应保证内部T类型变量已经初始化，返回值按照一个普通的可变引用使用。应注意返回值的生命周期应该小于self的生命周期  

#### MaybeUninit<[T]>的方法
创建一个MaybeUninit的未初始化数组：
`MaybeUninit<T>::uninit_array<const LEN:usize>()->[Self; LEN]` 此处对LEN的使用方式需要注意，这是不常见的一个泛型写法,这个函数同样的申请了一块内存。代码： 
```rust
    pub const fn uninit_array<const LEN: usize>() -> [Self; LEN] {
        unsafe { MaybeUninit::<[MaybeUninit<T>; LEN]>::uninit().assume_init() }
    }
```
这里要注意区别数组类型和数组元素的初始化。对于数组[MaybeUninit<T>;LEN]这一类型本身来说，初始化就是确定整体的内存大小，所以数组类型的初始化在声明后就已经完成了。这时assume_init()是正确的。这是一个理解上的盲点。 

`MaybeUninit<T>::array_assume_init<const N:usize>(array: [Self; N]) -> [T; N]` 这个函数没有把所有权转移出来，代码分析如下：
```rust
    pub unsafe fn array_assume_init<const N: usize>(array: [Self; N]) -> [T; N] {
        unsafe {
            //最后调用是*const T::read()，此处 as *const _的写法可以简化代码,read后，所有权已经转移到返回值
            //返回后，此数组内所有的MaybeUninit变量成员不能再解封装
            (&array as *const _ as *const [T; N]).read()
        }
    }
```

#### MaybeUnint<T>典型案列
对T类型变量申请内存及赋值：
```rust
    use std::mem::MaybeUninit;

    // 获得一个未初始化的i32引用类型内存
    let mut x = MaybeUninit::<&i32>::uninit();
    // 将&0写入变量，完成初始化
    x.write(&0);
    // 将初始化后的变量解封装供后继的代码使用。
    let x = unsafe { x.assume_init() };
```
以上代码，编译器不会对x.write进行报警，这是MaybeUninit<T>的最重要的应用，这个例子展示了RUST如何给未初始化内存赋值的处理方式。调用assume_init前，必须保证变量已经被正确初始化。

更复杂的初始化例子：
```rust
    use std::mem::{self, MaybeUninit};
    
    let data = {
    // data在声明后实际上就已经初始化完毕。
    let mut data: [MaybeUninit<Vec<u32>>; 1000] = unsafe {
        //这里注意实际调用是MaybeUninit::<[MaybeUninit<Vec<u32>>;1000]>::uninit(), RUST的类型推断机制完成了泛型实例化
        MaybeUninit::uninit().assume_init()
    };
    
    for elem in &mut data[..] {
    elem.write(vec![42]);
    }
    
    // 直接用transmute完成整个数组类型的转换
    // 仔细思考一下，这里除了用transmute，似乎没有其他办法了，
    unsafe { mem::transmute::<_, [Vec<u32>; 1000]>(data) }
    };
    
    assert_eq!(&data[0], &[42]);
```

下面例子说明一块内存被 MaybeUnint<T>封装后，编译器将不再对其做释放，必须在代码中显式释放：
```rust
    use std::mem::MaybeUninit;
    use std::ptr;
   
    let mut data: [MaybeUninit<String>; 1000] = unsafe { MaybeUninit::uninit().assume_init() };
    // 初始化了500个String变量
    let mut data_len: usize = 0;
    for elem in &mut data[0..500] {
        //write没有将所有权转移出ManuallyDrop
        elem.write(String::from("hello"));
        data_len += 1;
    }
    //编译器无法自动调用drop释放String变量, 必须显示用drop_in_place释放
    for elem in &mut data[0..data_len] {
        //实际上也可以调用assume_init_drop来完成此工作
        unsafe { ptr::drop_in_place(elem.as_mut_ptr()); }
    }
```
上例中，在没有assume_init()调用的情况下，必须手工调用drop_in_place释放内存。
MaybeUninit<T>是一个非常重要的类型结构，未初始化内存是编程中不可避免要遇到的情况，MaybeUninit<T>也就是RUST编程中必须熟练使用的一个类型。

## 裸指针模块再分析
有了MaybeUnint<T>做基础后，可以对裸指针其他至关重要的标准库函数做出分析

`ptr::read<T>(src: *const T) -> T` 此函数在MaybeUninit<T>节中已经给出了代码，ptr::read是对所有类型通用的一种复制方法，需要指出，此函数完成浅拷贝，复制后，src指向的变量的所有权会转移至返回值。所以，调用此函数的代码必须保证src指向的变量生命周期结束后不会被编译器自动调用drop，否则可能导致重复drop，出现UB问题。  

`ptr::read_unaligned<T>(src: *const T) -> T`当数据结构中有未内存对齐的成员变量时，需要用此函数读取内容并转化为内存对齐的变量。否则会引发UB(undefined behaiver) 如下例： 

/// 从字节数组中读一个usize的值:
 ```rust
    use std::mem;
   
    fn read_usize(x: &[u8]) -> usize {
        assert!(x.len() >= mem::size_of::<usize>());
       
        let ptr = x.as_ptr() as *const usize;
        //此处必须用ptr::read_unaligned，因为不确定字节是否对齐
        unsafe { ptr.read_unaligned() }
    }
```
例子中，为了从byte串中读取一个usize，需要用read_unaligned来获取值，不能象C语言那样通过指针类型转换直接获取值。

`ptr::write<T>(dst: *mut T, src: T)` 代码如下：
```rust
pub const unsafe fn write<T>(dst: *mut T, src: T) {
    unsafe {
        //浅拷贝
        copy_nonoverlapping(&src as *const T, dst, 1);
        //必须调用forget，这里所有权已经转移。不允许再对src做drop操作
        intrinsics::forget(src);
    }
}
```
write函数本质上就是一个所有权转移的操作。完成src到dst的浅拷贝，然后调用了forget(src), 这使得src的Drop不再被调用。从而将所有权转移到dst。此函数是mem::replace， mem::transmute_copy的基础。底层由intrisic:: copy_no_overlapping支持。 
这个函数中，如果dst已经初始化过，那原dst变量的所有权将被丢失掉，有可能引发内存泄漏。 

`ptr::write_unaligned<T>(dst: *mut T, src: T)` 与read_unaligned相对应。举例如下：
```rust
    #[repr(packed, C)]
    struct Packed {
        _padding: u8,
        unaligned: u32,
    }
    
    let mut packed: Packed = unsafe { std::mem::zeroed() };
    
    // Take the address of a 32-bit integer which is not aligned.
    // In contrast to `&packed.unaligned as *mut _`, this has no undefined behavior.
    // 对于结构中字节没有按照2幂次对齐的成员，要用addr_of_mut!宏来获得地址，无法用取引用的方式。
    let unaligned = std::ptr::addr_of_mut!(packed.unaligned);
    
    unsafe { std::ptr::write_unaligned(unaligned, 42) };
    
     assert_eq!({packed.unaligned}, 42); // `{...}` forces copying the field instead of creating a reference.
```
`ptr::read_volatile<T>(src: *const T) -> T`  是intrinsics::volatile_load的封装  
`ptr::write_volatile<T>(dst: *mut T, src:T)` 是intrinsics::volatiel_store的封装  

`ptr::macro addr_of($place:expr)` 因为用&获得引用必须是字节按照2的幂次对齐的地址，所以用这个宏获取非地址对齐的变量地址  
```rust
pub macro addr_of($place:expr) {
    //关键字是&raw const，这个是RUST的原始引用语义，但目前还没有在官方做公开。
    //区别与&, &要求地址必须满足字节对齐和初始化，&raw 则没有这个问题
    &raw const $place
}
```
`ptr::macro addr_of_mut($place:expr)` 作用同上。  
```rust
pub macro addr_of_mut($place:expr) {
    &raw mut $place
}
```
指针的通用函数请参考[Rust库函数参考](https://doc.rust-lang.org/core/ptr/index.html#functions)

## NonNull<T> 代码分析
结构体定义如下：
```rust
#[repr(transparent)]
pub struct NonNull<T: ?Sized> {
    pointer: *const T,
}
```
属性`repr(transparent)`实际上表示外部的封装结构在内存中等价于内部的变量。`NonNull<T>`在内存中与`*const T`完全一致。可以直接转化为`* const T`。
裸指针的值因为可以为0，如果敞开来用，会有很多无法控制的代码隐患。按照RUST的习惯，标准库定义了非0的指针封装结构NonNull<T>，从而可以用Option<NonNull<T>>来对值可能为0的裸指针做出强制安全代码逻辑。不需要Option的则认为裸指针不会取值为0。
NonNull<T>本身是协变(covarient)类型.
*RUST中的协变*，在RUST中，不同的生命周期被视为不同的类型，对于带有生命周期的类型变量做赋值操作时，仅允许子类型赋给基类型(长周期赋给短周期), 为了从基本类型生成复合类型的子类型和基类型的关系，RUST引入了协变性。从基本类型到复合类型的协变性有 协变(covarient)/逆变(contracovarient)/不变(invarient)三种
程序员分析代码时，可以从基本类型之间的生命周期关系及协变性确定复合类型变量之间的生命周期关系，从而做合适的赋值操作。

因为NonNull<T>实际上是封装`* mut T`类型，但`* mut T` 与NonNull<T>的协变性不同，所以程序员如果不能确定需要协变类型，就不要使用NonNull<T>

### NonNull<T>创建关联方法
创建一个悬垂(dangling)指针, 保证指针满足类型内存对齐要求。该指针可能指向一个正常的变量，所以不能认为指向的内存是未初始化的。
```rust
    pub const fn dangling() -> Self {
        unsafe {
            //取内存对齐地址作为裸指针的地址。调用者应保证不对此内存地址进行读写
            let ptr = mem::align_of::<T>() as *mut T;
            NonNull::new_unchecked(ptr)
        }
    }
```
new函数，由输入的`*mut T`裸指针创建NonNull<T>。代码如下：
```rust
    pub fn new(ptr: *mut T) -> Option<Self> {
        if !ptr.is_null() {
            //ptr的安全性已经检查完毕
            Some(unsafe { Self::new_unchecked(ptr) })
        } else {
            None
        }
    }
```
```NonNull::<T>::new_unchecked(* mut T)->Self``` 用`* mut T`生成NonNull<T>，不检查`* mut T`是否为0，调用者应保证`* mut T`不为0。  
from_raw_parts函数，类似裸指针的from_raw_parts。  
```rust
    pub const fn from_raw_parts(
        data_address: NonNull<()>,
        metadata: <T as super::Pointee>::Metadata,
    ) -> NonNull<T> {
        unsafe {
            //需要先用from_raw_parts_mut形成* mut T指针
            NonNull::new_unchecked(super::from_raw_parts_mut(data_address.as_ptr(), metadata))
        }
    }
```
由From Trait创建NonNull<T>
```rust
impl<T: ?Sized> const From<&mut T> for NonNull<T> {
    fn from(reference: &mut T) -> Self {
        unsafe { NonNull { pointer: reference as *mut T } }
    }
}

impl<T: ?Sized> const From<&T> for NonNull<T> {
    fn from(reference: &T) -> Self {
        //此处说明NonNull也可以接收不可变引用，不能后继将这个变量转换为可变引用
        unsafe { NonNull { pointer: reference as *const T } }
    }
}
```

### NonNull<T>类型转换方法
NonNull的方法基本与`*const T/* mut T`相同，也容易理解，下文仅做罗列和简单说明
`NonNull::<T>::as_ptr(self)->* mut T` 返回内部的pointer 裸指针  
```NonNull::<T>::as_ref<`a>(&self)->&`a T ``` 返回的引用的生命周期与引用指向的变量生命周期无关，调用者应保证返回的引用的生命周期符合安全性要求
```NonNull::<T>::as_mut<`a>(&mut self)->&`a mut T``` 与 as_ref类似，但返回可变引用。  
```NonNull::<T>::cast<U>(self)->NonNull<U>``` 指针类型转换，程序员应该保证T和U的内存布局相同 

### NonNull<[T]> 方法
```NonNull::<[T]>::slice_from_raw_parts(data: NonNull<T>, len: usize) -> Self``` 将类型指针转化为类型的切片类型指针，实质是`ptr::slice_from_raw_parts`的一种包装。  
`NonNull::<[T]>::as_non_null_ptr(self) -> NonNull<T>` * const [T]::as_ptr的NonNull版本

### NonNull<T>的使用实例
以下的实例展示了 NonNull在动态申请堆内存的使用：
```rust
    impl Global {
        fn alloc_impl(&self, layout: Layout, zeroed: bool) -> Result<NonNull<[u8]>, AllocError> {
            match layout.size() {
                0 => Ok(NonNull::slice_from_raw_parts(layout.dangling(), 0)),
                // SAFETY: `layout` is non-zero in size,
                size => unsafe {
                    //raw_ptr是 *const u8类型
                    let raw_ptr = if zeroed { alloc_zeroed(layout) } else { alloc(layout) };
                    //NonNull::new处理了raw_ptr为零的情况,返回NonNull<u8>,此时内存长度还与T不匹配
                    let ptr = NonNull::new(raw_ptr).ok_or(AllocError)?;
                    //将NonNull<u8>转换为NonNull<[u8]>, NonNull<[u8]>已经是类型T的内存长度。后继可以直接转换为T类型的指针了。这个转换极为重要。
                    Ok(NonNull::slice_from_raw_parts(ptr, size))
                },
            }
        }
        ....
    }
```
基本上，如果`* const T/*mut T`要跨越函数使用，或作为数据结构体的成员时，应将之转化成NonNull<T> 或Unique T。*const T应该仅仅保持在单一函数内。

#### NonNull<T> 与MaybeUninit<T>相关函数
```NonNull<T>::as_uninit_ref<`a>(&self) -> &`a MaybeUninit<T>``` NonNull与MaybeUninit的引用基本就是直接转换的关系，一体双面 
```rust
    pub unsafe fn as_uninit_ref<'a>(&self) -> &'a MaybeUninit<T> {
        // self.cast将NonNull<T>转换为NonNull<MaybeUninit<T>>
        //self.cast.as_ptr将NonNull<MaybeUninit<T>>转换为 *mut MaybeUninit<T>
        unsafe { &*self.cast().as_ptr() }
    }
``` 
```NonNull<T>::as_uninit_mut<`a>(&self) -> &`a mut MaybeUninit<T>```  
```NonNull<[T]>::as_uninit_slice<'a>(&self) -> &'a [MaybeUninit<T>]```  
```rust
    pub unsafe fn as_uninit_slice<'a>(&self) -> &'a [MaybeUninit<T>] {
        // 下面的函数调用ptr::slice_from_raw_parts
        unsafe { slice::from_raw_parts(self.cast().as_ptr(), self.len()) }
    }
```
```NonNull<[T]>::as_uninit_slice_mut<'a>(&self) -> &'a mut [MaybeUninit<T>]```   

## Unique<T> 代码分析
Unique<T>类型结构定义如下:
```rust
    #[repr(transparent)]
    pub struct Unique<T: ?Sized> {
        pointer: *const T,
        _marker: PhantomData<T>,
    }
```
和NonNull对比，Unique多了PhantomData<T>类型变量。这个定义使得编译器知晓，Unique<T>拥有了pointer指向的内存的所有权，NonNull<T>没有这个特性。具备所有权后，Unique<T>可以实现Send, Sync等Trait。因为获得了所有权，此块内存无法用于他处，这也是Unique的名字由来原因.
指针在被Unique封装前，必须保证是NonNull的。
对于RUST从堆内存申请的内存块，其指针都是用Unique<T>封装后来作为智能指针结构体内部成员变量，保证智能指针结构体拥有申请出来的内存块的所有权。

Unique模块的函数及代码与NonNull函数代码相类似，此处不分析。
`Unique::cast<U>(self)->Unique<U>` 类型转换，程序员应该保证T和U的内存布局相同  
`Unique::<T>::new(* mut T)->Option<Self>` 此函数内部判断* mut T是否为0值  
`Unique::<T>::new_unchecked(* mut T)->Self` 封装* mut T, 调用代码应该保证* mut T的安全性  
`Unique::as_ptr(self)->* mut T`  
`Unique::as_ref(&self)->& T` 因为Unique具备所有权，此处&T的生命周期与self相同，不必特别声明声明周期  
`Unique::as_mut(&mut self)->& mut T` 同上  

## mem模块函数  

### 泛型类型创建
`mem::zeroed<T>() -> T` 返回一个内存块清零的泛型变量，内存块在栈空间，代码如下：
```rust
pub unsafe fn zeroed<T>() -> T {
    // 调用代码必须确认T类型的变量可以取全零值
    unsafe {
        intrinsics::assert_zero_valid::<T>();
        MaybeUninit::zeroed().assume_init()
    }
}
```
`mem::uninitialized<T>() -> T` 返回一个未初始化过的泛型变量，内存块在栈空间。
```rust
pub unsafe fn uninitialized<T>() -> T {
    // 调用者必须确认T类型的变量允许未初始化的任意值
    unsafe {
        intrinsics::assert_uninit_valid::<T>();
        MaybeUninit::uninit().assume_init()
    }
}
```

### 泛型类型拷贝与替换 
`mem::take<T: Default>(dest: &mut T) -> T` 将dest设置为默认内容(不改变所有权)，用一个新变量返回dest的内容。
```rust
pub fn take<T: Default>(dest: &mut T) -> T {
    //即mem::replace，见下文
    //此处，对于引用类型，编译器禁止用*dest来转移所有权，所以不能用let xxx = *dest; xxx这种形式返回T
    //其他语言简单的事情在RUST中必须用一个较难理解的方式来进行解决。replace()对所有权有仔细的处理
    replace(dest, T::default())
}
```
`mem::replace<T>(dest: &mut T, src: T) -> T` 用src的内容赋值dest(不改变所有权)，用一个新变量返回dest的内容。replace函数的难点在于了解所有权的转移。
```rust
pub const fn replace<T>(dest: &mut T, src: T) -> T {
    unsafe {
        //因为要替换dest, 所以必须对dest原有变量的所有权做处理，因此先用read将*dest的所有权转移到T，交由调用者进行处理, RUST不支持对引用类型做解引用的相等来转移所有权。将一个引用的所有权进行转移的方式只有粗暴的内存浅拷贝这种方法。
        //使用这个函数，调用代码必须了解T类型的情况，T类型有可能需要显式的调用drop函数。ptr::read前文已经分析过。
        let result = ptr::read(dest);
        //ptr::write本身会导致src的所有权转移到dest，后继不允许在src生命周期终止时做drop。ptr::write会用forget(src)做到这一点。
        ptr::write(dest, src);
        result
    }
}
```

`mem::transmute_copy<T, U>(src: &T) -> U` 新建类型U的变量，并把src的内容拷贝到U。调用者应保证T类型的内容与U一致，src后继的所有权问题需要做处理。
```rust
pub const unsafe fn transmute_copy<T, U>(src: &T) -> U {
    if align_of::<U>() > align_of::<T>() {
        // 如果两个类型字节对齐U 大于 T. 使用read_unaligned
        unsafe { ptr::read_unaligned(src as *const T as *const U) }
    } else {
        //用read即可完成
        unsafe { ptr::read(src as *const T as *const U) }
    }
}
```
#### 所有权转移的底层实现
所有权的转移实际上是两步：1.栈上内存的浅拷贝；2：原先的变量置标志表示所有权已转移。置标志的变量如果没有重新绑定其他变量，则在生命周期结束的时候被drop。 引用及指针自身也是一个isize的值变量，也有所有权，也具备生命周期。 
##### 变量调用drop的时机
如下例子：
```rust
struct TestPtr {a: i32, b:i32}
impl Drop for TestPtr {
    fn drop(&mut self) {
        println!("{} {}", self.a, self.b);
    }
}
fn main() {
   let test = Box::new(TestPtr{a:1,b:2});
   let test1 = *test;
   let mut test2 = TestPtr{a:2, b:3};
   //此行代码会导致先释放test2拥有所有权的变量，然后再给test2赋值。代码后的输出会给出证据
   //将test1的所有权转移给test2，无疑代表着test2现有的所有权会在后继无法访问，因此drop被立即调用。
   test2 = test1;
   println!("{:?}", test2);
}
```
输出：
2 3
TestPtr { a: 1, b: 2 }
1 2

### 其他函数
`mem::forget<T>(t:T)` 通知RUST不做变量的drop操作  
```rust
pub const fn forget<T>(t: T) {
    //没有使用intrinsic::forget, 实际上效果一致，这里应该是尽量规避用intrinsic函数
    let _ = ManuallyDrop::new(t);
}
```
`mem::forget_unsized<T:Sized?>` 对intrinsics::forget的封装 
`mem::size_of<T>()->usize`/`mem::min_align_of<T>()->usize`/`mem::size_of_val<T>(val:& T)->usize`/`mem::min_align_of_val<T>(val: &T)->usize`/`mem::needs_drop<T>()->bool` 基本就是直接调用intrinsic模块的同名函数  
`mem::drop<T>(_x:T)` 释放内存  

## RUST堆内存申请及释放

### RUST类型系统的内存布局 
RUST提供了`Layout`内存布局类型, 此布局类型结构主要用于做堆内存申请。
`Layout`的数据结构如下：
```rust
pub struct Layout {
    // 类型需占用的内存大小，用字节数目表示
    size_: usize,
    //  按照此字节数目进行类型内存对齐， NonZeroUsize见代码后面文字分析
    align_: NonZeroUsize,
}
```
*`NonZeroUsize`是一种非0值的usize, 这种类型主要应用于不可取0的值，本结构中， 字节对齐属性变量不能被置0，所以用`NonZeroUsize`来确保安全性。如果用usize类型，那代码中就可能会把0置给align_，导致bug产生。这是RUST的一个设计规则，所有的约束要在类型定义即显性化，从而使bug在编译中就被发现。*

每一个RUST的类型都有自身独特的内存布局Layout。一种类型的Layout可以用`intrinsic::<T>::size_of()`及`intrinsic::<T>::min_align_of()`获得的类型内存大小和对齐来获得。
RUST的内存布局更详细原理阐述请参考[RUST内存布局] (https://doc.rust-lang.org/nomicon/data.html)，
Layout比较有典型意义的函数：
```rust
impl Layout {
    ...
    ...

    //array函数是计算n个T类型变量形成的数组所需的Layout，是从代码了解Rust Layout概念的一个好的实例
    //这里主要注意的是T类型的对齐会导致内存申请不是T类型的内存大小*n
    //而且对齐也是数组的算法
    pub fn array<T>(n: usize) -> Result<Self, LayoutError> {
        //获得n个T类型的内存Layout
        let (layout, offset) = Layout::new::<T>().repeat(n)?;
        debug_assert_eq!(offset, mem::size_of::<T>());
        //以完全对齐的大小  ，得出数组的Layout
        Ok(layout.pad_to_align())
    }

    //计算n个T类型需要的内存Layout, 以及成员之间的空间
    pub fn repeat(&self, n: usize) -> Result<(Self, usize), LayoutError> {
        // 所有的成员必须以成员的对齐大小来做内存对齐,首先计算对齐需要的padding空间
        let padded_size = self.size() + self.padding_needed_for(self.align());
        // 计算共需要多少内存空间，如果溢出，返回error
        let alloc_size = padded_size.checked_mul(n).ok_or(LayoutError)?;

        //由已经验证过得原始数据生成Layout，并返回单成员占用的空间
        unsafe { Ok((Layout::from_size_align_unchecked(alloc_size, self.align()), padded_size)) }
    }

    //填充以得到一个与T类型完全对齐的，最小的内存大小的Layout
    pub fn pad_to_align(&self) -> Layout {
        //得到T类型与对齐之间的空间大小
        let pad = self.padding_needed_for(self.align());
        // 完全对齐的大小
        let new_size = self.size() + pad;
        
        //以完全对齐的大小生成新的Layout
        Layout::from_size_align(new_size, self.align()).unwrap()
    }

    //计算T类型长度与完全对齐的差
    pub const fn padding_needed_for(&self, align: usize) -> usize {
        let len = self.size();

        // 实际上相当与C语言的表达式
        //   len_rounded_up = (len + align - 1) & !(align - 1);
        // 就是对对齐大小做除，如果有余数，商加1，是一种常用的方式.
        // 但注意，在rust中C语言的"+"等同于wrapping_add, C语言的“-”等同于
        // wrapping_sub
        let len_rounded_up = len.wrapping_add(align).wrapping_sub(1) & !align.wrapping_sub(1);
        //减去len，得到差值
        len_rounded_up.wrapping_sub(len)
    }

    //不检查输入参数，根据输入参数表示的原始数据生成Layout变量,调用代码应保证安全性
    pub const unsafe fn from_size_align_unchecked(size: usize, align: usize) -> Self {
        // 必须保证align满足不为0.
        Layout { size_: size, align_: unsafe { NonZeroUsize::new_unchecked(align) } }
    }

    //对参数进行检查，生成一个类型的Layout
    pub const fn from_size_align(size: usize, align: usize) -> Result<Self, LayoutError> {
        //必须保证对齐是2的幂次
        if !align.is_power_of_two() {
            return Err(LayoutError);
        }

        //满足下面的表达式，则size将不可能对齐 
        if size > usize::MAX - (align - 1) {
            return Err(LayoutError);
        }

        // 参数已经检查完毕.
        unsafe { Ok(Layout::from_size_align_unchecked(size, align)) }
    }
    ...
    ...
}
```

### `#[repr(transparent)]`内存布局模式
repr(transparent)用于仅包含一个成员变量的类型，该类型的内存布局与成员变量类型的内存布局完全一致。类型仅仅具备编译阶段的意义，在运行时，类型变量与其成员变量可以认为是一个相同变量，可以相互无障碍类型转换。使用repr(transparent)布局的类型基本是一种封装结构。

#### `#[repr(packed)]`内存布局模式
强制类型成员变量以1字节对齐，此种结构在协议分析和结构化二进制数据文件中经常使用

#### `#[repr(align(n))]` 内存布局模式
强制类型以2的幂次对齐

#### `#[repr(RUST)]`内存布局模式
默认的布局方式，采用此种布局，RUST编译器会根据情况来自行优化内存

#### `#[repr(C)]`内存布局模式
采用C语言布局方式， 所有结构变量按照声明的顺序在内存排列。默认4字节对齐。

### RUST堆内存申请与释放接口
资深的C/C++程序员都了解，在大型系统开发时，往往需要自行实现内存管理模块，以根据系统的特点优化内存使用及性能，并作出内存跟踪。
对于操作系统，内存管理模块更是核心功能。
对于C/C++小型系统，没有内存管理，仅仅是调用操作系统的内存系统调用，内存管理交给操作系统负责。操作系统内存管理模块接口是内存申请及内存释放的系统调用
对于GC语言，内存管理由虚拟机或语言运行时负责，利用语言提供的new来完成类型结构内存获取。
RUST的内存管理分成了三个界面：
1. 由智能指针类型提供的类型创建函数，一般有new, 与其他的GC类语言相同，同时增加了一些更直观的函数。
2. 智能指针使用实现Allocator Trait的类型做内存申请及释放。Allocator使用编译器提供的函数名申请及释放内存。
3. 实现了GlobalAlloc Trait的类型来完成独立的内存管理模块，并用#[global_allocator]注册入编译器，替代编译器默认的内存申请及释放函数。
这样，RUST达到了：
1. 对于小规模的程序，拥有与GC语言相类似的内存获取机制
2. 对于大型程序和操作系统内核，从语言层面提供了独立的内存管理模块接口，达成了将现代语法与内存管理模块共同存在，相互配合的目的。
但因为所有权概念的存在，从内存申请到转换为类型系统仍然还存在复杂的工作。
堆内存申请和释放的Trait GlobalAlloc定义如下:
```rust
pub unsafe trait GlobalAlloc {
    //申请内存，因为Layout中内存大小不为0，所以，alloc不会申请大小为0的内存
    unsafe fn alloc(&self, layout: Layout) -> *mut u8;
    //释放内存
    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout);
    
    //申请后的内存应初始化为0
    unsafe fn alloc_zeroed(&self, layout: Layout) -> *mut u8 {
        let size = layout.size();
        let ptr = unsafe { self.alloc(layout) };
        if !ptr.is_null() {
            // 此处必须使用write_bytes，确保每个字节都清零
            unsafe { ptr::write_bytes(ptr, 0, size) };
        }
        ptr
    }

    //其他方法
    ...
    ...
}
```
在内核编程或大的框架系统编程中，开发人员通常开发自定义的堆内存管理模块，模块实现GlobalAlloc Trait并添加#[global_allocator]标识。对于用户态，RUST标准库有默认的GlobalAlloc实现。
```rust
extern "Rust" {
    // 编译器会将实现了GlobalAlloc Trait，并标记 #[global_allocator]的四个方法自动转化为以下的函数
    #[rustc_allocator]
    #[rustc_allocator_nounwind]
    fn __rust_alloc(size: usize, align: usize) -> *mut u8;
    #[rustc_allocator_nounwind]
    fn __rust_dealloc(ptr: *mut u8, size: usize, align: usize);
    #[rustc_allocator_nounwind]
    fn __rust_realloc(ptr: *mut u8, old_size: usize, align: usize, new_size: usize) -> *mut u8;
    #[rustc_allocator_nounwind]
    fn __rust_alloc_zeroed(size: usize, align: usize) -> *mut u8;
}

//对__rust_xxxxx_再次封装
pub unsafe fn alloc(layout: Layout) -> *mut u8 {
    unsafe { __rust_alloc(layout.size(), layout.align()) }
}

pub unsafe fn dealloc(ptr: *mut u8, layout: Layout) {
    unsafe { __rust_dealloc(ptr, layout.size(), layout.align()) }
}

pub unsafe fn realloc(ptr: *mut u8, layout: Layout, new_size: usize) -> *mut u8 {
    unsafe { __rust_realloc(ptr, layout.size(), layout.align(), new_size) }
}

pub unsafe fn alloc_zeroed(layout: Layout) -> *mut u8 {
    unsafe { __rust_alloc_zeroed(layout.size(), layout.align()) }
}

```
再实现Allocator Trait，对以上四个函数做封装处理。作为RUST其他模块对堆内存的申请和释放接口。

```rust
pub unsafe trait Allocator {
    fn allocate(&self, layout: Layout) -> Result<NonNull<[u8]>, AllocError>;

    fn allocate_zeroed(&self, layout: Layout) -> Result<NonNull<[u8]>, AllocError> {
        let ptr = self.allocate(layout)?;
        // SAFETY: `alloc` returns a valid memory block
        // 复杂的类型转换，实际是调用 *const u8::write_bytes(0, layout.size_)
        unsafe { ptr.as_non_null_ptr().as_ptr().write_bytes(0, ptr.len()) }
        Ok(ptr)
    }

    unsafe fn deallocate(&self, ptr: NonNull<u8>, layout: Layout);

    ...
}
```
Global 实现了 Allocator Trait。Rust大部分alloc库数据结构的实现使用Global作为Allocator。
```rust
unsafe impl Allocator for Global {
    fn allocate(&self, layout: Layout) -> Result<NonNull<[u8]>, AllocError> {
        //上文已经给出alloc_impl的说明
        self.alloc_impl(layout, false)
    }

    fn allocate_zeroed(&self, layout: Layout) -> Result<NonNull<[u8]>, AllocError> {
        self.alloc_impl(layout, true)
    }

    unsafe fn deallocate(&self, ptr: NonNull<u8>, layout: Layout) {
        if layout.size() != 0 {
            // SAFETY: `layout` is non-zero in size,
            // other conditions must be upheld by the caller
            unsafe { dealloc(ptr.as_ptr(), layout) }
        }
    }
    ...
    ...
}
```

Allocator使用GlobalAlloc接口获取内存，然后将GlobalAlloc申请到的* mut u8转换为确定大小的单一指针NonNull<[u8]>， 并处理申请内存可能出现的不成功。NonNull<[u8]>此时内存布局与 T的内存布局已经相同，后继可以转换为真正需要的T的指针并进一步转化为相关类型的引用，从而符合RUST类型系统安全并进行后继的处理。
以上是堆内存的申请和释放。 基于泛型，RUST也巧妙实现了栈内存的申请和释放机制 `mem::MaybeUninit<T>`

用Box的内存申请做综合举例：
```rust
    //此处A是一个A:Allocator类型
    pub fn try_new_uninit_in(alloc: A) -> Result<Box<mem::MaybeUninit<T>, A>, AllocError> {
        //实质是T类型的内存Layout
        let layout = Layout::new::<mem::MaybeUninit<T>>();
        //allocate(layout)?返回NonNull<[u8]>, NonNull<[u8]>::<MaybeUninit<T>>::cast()返回NonNull<MaybeUninit<T>>
        let ptr = alloc.allocate(layout)?.cast();
        //as_ptr 成为 *mut MaybeUninit<T>类型裸指针
        unsafe { Ok(Box::from_raw_in(ptr.as_ptr(), alloc)) }
    }
    
    pub unsafe fn from_raw_in(raw: *mut T, alloc: A) -> Self {
        //使用Unique封装* mut T，并拥有了*mut T指向的变量的所有权
        Box(unsafe { Unique::new_unchecked(raw) }, alloc)
    }
```
以上代码可以看到，NonNull<[u8]>可以直接通过cast 转换为NonNull<MaybeUninit<T>>, 这是另一种MaybeUninit<T>的生成方法，直接通过指针类型转换将未初始化的内存转换为MaybeUninit<T>。

## 小结
本章主要分析了RUST标准库内存相关模块， 内存相关模块代码多数不复杂，主要是要对内存块与类型系统之间的转换要有比较深刻的理解，并能领会在实际编码过程中在那些场景会使用内存相关的代码和API。RUST的内存安全给编码加了非常多的限制，有些时候这些限制只有通过内存API来有效的突破。如将引用指向的变量所有权转移出来的take函数。后继我们会看到几乎每个标准库的模块都大量的使用了ptr, mem模块中的方法和函数。只要是大型系统，不熟悉内存模块的代码，基本上无法做出良好的程序。

# RUST的固有（intrinsic）函数库
代码路径：
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\core\src\intrinsic.rs

intrinsic库函数是指由编译器内置实现的函数，一般如下特点的函数用固有函数：
1. 与CPU架构相关性很大，必须利用汇编实现或者利用汇编才能具备最高性能的函数，
2. 和编译器密切相关的函数，由编译器来实现最为合适。
上面内存库章节中已经介绍了内存部分的intrinsic库函数，本节对其他部分做简略介绍
## intrinsic 原子操作函数
原子操作函数主要用于多核CPU，多线程CPU时对数据的原子操作。intrinsic库中atomic_xxx及atomic_xxx_xxx类型的函数即为原子操作函数。原子操作函数主要用于并发编程中做临界保护，并且是其他临界保护机制的基础，如Mutex，RWlock等。
## 数学函数及位操作函数
各种整数及浮点的数学函数实现。这一部分放在intrinsic主要是因为现代CPU对浮点计算由很多支持，这些数学函数由汇编语言来实现更具备效率，那就有必要由编译器来内置实现。
## intrinsic 指令优化及调试函数
断言类: assert_xxxx 类型的函数
函数栈：caller_location
## 小结
intrinsic函数库是从编译器层面完成跨CPU架构的一个手段，intrinsic通常被上层的库所封装。但在操作系统编程和框架编程时，仍然会不可避免的需要接触。

# RUST基本类型代码分析(一)
原生数据类型，Option类型，Result类型的某些代码是分析其他模块的基础，因此先对这些类型的部分代码做个基础分析。

## 整形数据类型
代码目录如下：
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\core\src\num

整形数据类型标准函数库主要包括：
1. 整形位操作函数：左移，右移，为1的位数目，为0的位数目，头部为0的位数目，尾部为0的位数目，头部为1的位数目，尾部为1的位数目，循环左移，循环右移
2. 整形字节序函数：字节序反转，位序反转，大小端变换
3. 数学函数：针对溢出做各种不同处理的加减乘除，传统的整形数学库函数如对数，幂，绝对值，取两者大值及两者小值
   
整形有有符号整形，无符号整形，大整形(大于计算机字长的整形)，但基本内容都是实现以上方法
### 无符号整形类型相关库代码分析
标准库用宏简化的对不同位长的无符号整形的方法实现。本文着重介绍若干不易注意的方法，如大小端转换，对数学方法仅给出加法做为代表。代码如下：
```rust
macro_rules! uint_impl {
    ($SelfT:ty, $ActualT:ident, $SignedT:ident, $BITS:expr, $MaxV:expr,
        //以下主要是rust doc文档需要
        $rot:expr, $rot_op:expr, $rot_result:expr, $swap_op:expr, $swapped:expr,
        $reversed:expr, $le_bytes:expr, $be_bytes:expr,
        $to_xe_bytes_doc:expr, $from_xe_bytes_doc:expr) => {
```
这个宏实现所有无符号整形的方法：
```rust        
        pub const MIN: Self = 0;
        //按位非
        pub const MAX: Self = !0;

        pub const BITS: u32 = $BITS;
```
以上是无符号整形的常量
```rust        
        //利用intrinsics的位函数完成整数的位操作相关函数，这里仅分析一个，其他请参考标准库手册
        pub const fn count_ones(self) -> u32 {
            intrinsics::ctpop(self as $ActualT) as u32
        }

        //其他位操作函数
        ...
        ...
```
字节序变换是网络编程与结构化数据文件的必须功能，RUST将之在整形的方法里实现：
```rust
        //内部字节交换，后继大小端变换使用
        pub const fn swap_bytes(self) -> Self {
            intrinsics::bswap(self as $ActualT) as Self
        }
        
        //big endian 到硬件架构字节序
        pub const fn from_be(x: Self) -> Self {
            #[cfg(target_endian = "big")]
            {
                x
            }
            #[cfg(not(target_endian = "big"))]
            {
                x.swap_bytes()
            }
        }

        //little endian 转换为硬件架构字节序
        pub const fn from_le(x: Self) -> Self {
            #[cfg(target_endian = "little")]
            {
                x
            }
            #[cfg(not(target_endian = "little"))]
            {
                x.swap_bytes()
            }
        }

        //硬件架构字节序到big endian
        pub const fn to_be(self) -> Self { // or not to be?
            #[cfg(target_endian = "big")]
            {
                self
            }
            #[cfg(not(target_endian = "big"))]
            {
                self.swap_bytes()
            }
        }

        //硬件架构字节序到little endian
        pub const fn to_le(self) -> Self {
            #[cfg(target_endian = "little")]
            {
                self
            }
            #[cfg(not(target_endian = "little"))]
            {
                self.swap_bytes()
            }
        }

        //获得大端字节序字节数组
        pub const fn to_be_bytes(self) -> [u8; mem::size_of::<Self>()] {
            self.to_be().to_ne_bytes()
        }
        
        //获得小端
        pub const fn to_le_bytes(self) -> [u8; mem::size_of::<Self>()] {
            self.to_le().to_ne_bytes()
        }
        
        //硬件平台字节序
        pub const fn to_ne_bytes(self) -> [u8; mem::size_of::<Self>()] {
            unsafe { mem::transmute(self) }
        }
        
        //从big endian 字节数组获得类型值
        pub const fn from_be_bytes(bytes: [u8; mem::size_of::<Self>()]) -> Self {
            Self::from_be(Self::from_ne_bytes(bytes))
        }

        //从little endian 字节数组获得类型值
        pub const fn from_le_bytes(bytes: [u8; mem::size_of::<Self>()]) -> Self {
            Self::from_le(Self::from_ne_bytes(bytes))
        }

        //从硬件架构字节序字节数组获得类型值
        pub const fn from_ne_bytes(bytes: [u8; mem::size_of::<Self>()]) -> Self {
            unsafe { mem::transmute(bytes) }
        }
```
RUST的整数类形各种算术方法突出的展示了RUST对安全的极致关注。算术方法也更好的支持了链式调用的函数式编程风格。对于算术溢出，RUST给出了各种情况下的处理方案：
```rust
        //对溢出做检查的加法运算,溢出情况下会返回wrapping_add的值，即溢出后值回绕
        //这里每种类型运算都以加法为例，其他诸如减、乘、除、幂次请参考官方标准库手册
        pub const fn overflowing_add(self, rhs: Self) -> (Self, bool) {
            let (a, b) = intrinsics::add_with_overflow(self as $ActualT, rhs as $ActualT);
            (a as Self, b)
        }
        //其他的对溢出做检查的数学运算，略
        ...
        ...
        
        //溢出后对最大值取余，即回绕
        pub const fn wrapping_add(self, rhs: Self) -> Self {
            intrinsics::wrapping_add(self, rhs)
        }
        //以边界值取余的其他数学运算方法，略
        ...
        ...

        //饱和加法，超过边界值结果为边界值
        pub const fn saturating_add(self, rhs: Self) -> Self {
            intrinsics::saturating_add(self, rhs)
        }
        //其他饱和型的数学运算，略
        ...
        ...

        //对加法有效性检查的加法运算，如发生溢出，则返回异常
        pub const fn checked_add(self, rhs: Self) -> Option<Self> {
            let (a, b) = self.overflowing_add(rhs);
            if unlikely!(b) {None} else {Some(a)}
        }

        //无检查add,  是 + 符号的默认调用函数。
        pub const unsafe fn unchecked_add(self, rhs: Self) -> Self {
            // 调用者要保证不发生错误
            unsafe { intrinsics::unchecked_add(self, rhs) }
        }
        //其他对有效性检查的数学运算， 略
        ...
        ...
        
        pub const fn min_value() -> Self { Self::MIN }

        pub const fn max_value() -> Self { Self::MAX }
}
```
算术算法基本上是使用了intrinsics提供的函数。
下面用u8给出一个具体的实例
```rust
impl u8 {
    //利用宏实现 u8类型的方法
    uint_impl! { u8, u8, i8, 8, 255, 2, "0x82", "0xa", "0x12", "0x12", "0x48", "[0x12]",
    "[0x12]", "", "" }
    
    pub const fn is_ascii(&self) -> bool {
        *self & 128 == 0
    }
    
    //其他ASCII相关函数，请参考标准库手册，略
    ...
    ...
}

//u16 数学函数方法 实现
impl u16 {
    uint_impl! { u16, u16, i16, 16, 65535, 4, "0xa003", "0x3a", "0x1234", "0x3412", "0x2c48",
    "[0x34, 0x12]", "[0x12, 0x34]", "", "" }
    widening_impl! { u16, u32, 16, unsigned }
}

//其他无符号整形的实现，略
...
...
```

RUST整形库代码逻辑并不复杂，宏也很简单。但因为RUST将其他语言的独立的数学库函数，单独的大小端变换等集成入整形(浮点类型)，有可能造成出于习惯而无法找到相应的函数。

## RUST Option类型标准库代码分析
代码路径：
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\core\src\option.rs

Option<T> 主要用来在编程中，类型T的变量可以不存在，代表一种异常。以往会选择T类型的一个值代表不存在的异常情况，从而导致异常情况处理只能依赖于程序员，采用Option<T>后，对异常情况的处理会由编译器负责。
在初始化时无法确定T类型的值时，除了MaybeUninit<T>外，还可以用Option<T>来声明变量并初始化为None。

从Option<T>的标准库代码中可以发现，Option<T>实际上是不属于RUST语言最基础的语法的，它是在RUST语言最基础的enum语法的基础上派生出来的一种库类型。这展示了RUST语言的一个设计思维，编译器仅仅做最基础的部分，其他的交由库来解决。实际上，在学习RUST时，很多重要的常用类型都是标准库提供并解决了非常多的需求。如RefCell<T>, Arc<T>等。需要细心地体会和学习RUST如何构建这些基础设施并养成习惯。

Option<T>主要是解封装方法及Try trait。但Option<T>更酷的打开方式应该是用以map为代表的方法来完成函数链式调用。

Option<T>的若干重点方法源代码如下：
```rust
impl<T> Option<T> {
    //当希望获取Option封装的变量的借用的时候，使用这个函数
    //&Option<T> 转换为Option<&T>以取得能够真正操作的引用
    //因为Option可能为None，所以返回值只能是Option<&T>
    pub const fn as_ref(&self) -> Option<&T> {
        match *self {
            Some(ref x) => Some(x),
            None => None,
        }
    }

    //类似于as_ref，但返回的是可变引用
    pub const fn as_mut(&mut self) -> Option<&mut T> {
        //略
    }
```
以下解封装函数，看过源码后功能即一目了然，因为解封装用过于常用，因此将全部的解封装代码都给出
```rust
    //解封装函数，None时输出指定的错误信息并退出程序
    pub fn expect(self, msg: &str) -> T {
        match self {
            Some(val) => val,
            None => expect_failed(msg),
        }
    }

    //解封装函数，None时输出固定的错误信息并退出程序
    pub const fn unwrap(self) -> T {
        match self {
            Some(val) => val,
            None => panic!("called `Option::unwrap()` on a `None` value"),
        }
    }

    //解封装，None时返回给定的默认值
    pub fn unwrap_or(self, default: T) -> T {
        match self {
            Some(x) => x,
            None => default,
        }
    }
    
    //解封装，None时调用给定的闭包函数
    pub fn unwrap_or_else<F: FnOnce() -> T>(self, f: F) -> T {
        match self {
            Some(x) => x,
            None => f(),
        }
    }

    //确认不会为None时的解封装
    pub unsafe fn unwrap_unchecked(self) -> T {
        debug_assert!(self.is_some());
        match self {
            Some(val) => val,
            // SAFETY: the safety contract must be upheld by the caller.
            None => unsafe { hint::unreachable_unchecked() },
        }
    }
```
针对函数式编程的链式调用设计的方法：
```rust
    //主要用于函数式编程，在不解封装的情况下对Option的值进行处理，并按需要返回适合的类型
    pub fn map<U, F: FnOnce(T) -> U>(self, f: F) -> Option<U> {
        match self {
            Some(x) => Some(f(x)),
            None => None,
        }
    }

    //同上，None时返回默认值
    pub fn map_or<U, F: FnOnce(T) -> U>(self, default: U, f: F) -> U {
        match self {
            Some(t) => f(t),
            None => default,
        }
    }

    //同上，None时调用默认闭包函数
    pub fn map_or_else<U, D: FnOnce() -> U, F: FnOnce(T) -> U>(self, default: D, f: F) -> U {
        match self {
            Some(t) => f(t),
            None => default(),
        }
    }

    //将Option转换为Result，也是为支持函数式编程
    pub fn ok_or<E>(self, err: E) -> Result<T, E> {
        match self {
            Some(v) => Ok(v),
            None => Err(err),
        }
    }

    //同上，None时调用默认函数处理
    pub fn ok_or_else<E, F: FnOnce() -> E>(self, err: F) -> Result<T, E> {
        match self {
            Some(v) => Ok(v),
            None => Err(err()),
        }
    }
    
    //主要还是应用于函数式编程，值是Some(x)的时候返回预设值
    pub fn and<U>(self, optb: Option<U>) -> Option<U> {
        match self {
            Some(_) => optb,
            None => None,
        }
    }

    //主要用于函数式编程，与and 形成系列，值为Some(x)调用函数并返回函数值
    pub fn and_then<U, F: FnOnce(T) -> Option<U>>(self, f: F) -> Option<U> {
        match self {
            Some(x) => f(x),
            None => None,
        }
    }

    //如果是Some(x), 判断是否满足预设条件
    pub fn filter<P: FnOnce(&T) -> bool>(self, predicate: P) -> Self {
        if let Some(x) = self {
            if predicate(&x) {
                return Some(x);
            }
        }
        None
    }

    //如果是Some(x)返回本身，如果是None，返回预设值
    pub fn or(self, optb: Option<T>) -> Option<T> {
        match self {
            Some(_) => self,
            None => optb,
        }
    }

    //如果是Some(x)返回本身，否则返回预设函数
    pub fn or_else<F: FnOnce() -> Option<T>>(self, f: F) -> Option<T> {
        match self {
            Some(_) => self,
            None => f(),
        }
    }

    //类似xor操作
    pub fn xor(self, optb: Option<T>) -> Option<T> {
        match (self, optb) {
            //一方为Some,一方为None，返回Some值
            (Some(a), None) => Some(a),
            (None, Some(b)) => Some(b),
            //两者都为Some，或两者都为None, 返回None
            _ => None,
        }
    }
```
其他方法
```rust
    //不解封装的重新设置内部的值，并返回值的可变引用
    //例子：let a = None; a.insert(1);
    //上例也是一种常用方法，利用None可以实现不知道初始值但需要有一个变量的情况。
    pub fn insert(&mut self, value: T) -> &mut T {
        //原有*self会被drop
        *self = Some(value);
        //确认不会为None
        unsafe { self.as_mut().unwrap_unchecked() }
    }

    //使用一个闭包生成变量
    pub fn get_or_insert_with<F: FnOnce() -> T>(&mut self, f: F) -> &mut T {
        if let None = *self {
            *self = Some(f());
        }

        match self {
            //此处模式匹配有些特殊，详细请见后面的对结构体引用类型`&T/&mutT`的match语法研究
            Some(v) => v,
            None => unsafe { hint::unreachable_unchecked() },
        }
    }

    //mem::replace分析请参考前文，用None替换原来的变量，并用新变量返回self，同时也完成了所有权的转移
    pub const fn take(&mut self) -> Option<T> {
        mem::replace(self, None)
    }

    //用新value替换原变量，并把原变量返回
    pub const fn replace(&mut self, value: T) -> Option<T> {
        mem::replace(self, Some(value))
    }

    //针对Option的zip操作
    pub fn zip<U>(self, other: Option<U>) -> Option<(T, U)> {
        match (self, other) {
            (Some(a), Some(b)) => Some((a, b)),
            _ => None,
        }
    }

    //执行一个函数
    pub fn zip_with<U, F, R>(self, other: Option<U>, f: F) -> Option<R>
    where
        F: FnOnce(T, U) -> R,
    {
        //此处，顺序应该是先执行self? other？，然后再调用函数
        Some(f(self?, other?))
    }
}
```
### 对结构体引用类型“&T/&mut T"的match语法研究
如下代码：
```rust
#[derive(Debug)]
struct TestStructA {a:i32, b:i32}
fn main() {
    let c = TestStructA{a:1, b:2};
    let d = [1,2,3];

    match ((&c, &d)) {
        (&e, &f) => println!("{:?} {:?}", e, f),
        _  => println!("match nothing"),
    }
}
```
以上代码编译时，会发生如下错误：
```rust
error[E0507]: cannot move out of a shared reference
  --> src/main.rs:9:7
   |
9  | match (&c) {
   |       ^^^^
10 |     (&e, &d) => println!("{:?}", e),
   |     --
   |     ||
   |     |data moved here
   |     |move occurs because `e` has type `TestStructA`, which does not implement the `Copy` trait
   |     help: consider removing the `&`: `e`

```
可见，如果match 引用，那对后继的绑定是有讲究的。对引用做match，本意便是不想要转移所有权。因此，在match的分支中就不能有引发所有权移动的绑定出现。

再请参考如下代码：
```rust
struct TestStructA {a:i32,b:i32}
fn main() {
    let c = TestStructA{a:1, b:2};
    let d = [1,2,3];

    match ((&c, &d)) {
        (&TestStructA{a:ref u, b:ref w}, &[ref x, ..]) => println!("{} {} {}", *u, *w, *x),
        _  => println!("match nothing"),
    }
}
```
如果不想转移所有权，那上面代码的match就应该是一个标准的写法，对结构内部的变量也需要用引用来绑定，尤其是结构内部变量如果没有实现Copy Trait，那就必须用引用，否则也会引发编译告警。

为了编码上的方便，RUST针对以上的代码，支持如下简化形式：
```rust

struct TestStructA {a:i32,b:i32}
fn main() {
    let c = TestStructA{a:1, b:2};
    let d = [1, 2, 3];

    match ((&c, &d)) {
        //对比上述代码，头部少了&，模式绑定内部少了 ref，但代码功能完全一致
        (TestStructA{a: u, b: w}, [x,..]) => println!("{} {} {}", *u, *w, *x),
        _  => println!("match nothing"),
    }
}
```
如果不知道RUST的这个语义，很可能会对这里的类型绑定感到疑惑。
从实际的使用场景分析，对结构体引用做match，其目的就是对结构体内部的成员的引用做pattern绑定。而且如果结构体内部的成员不支持Copy，那也不可能对结构体成员做pattern绑定。所以，此语法也是在RUST的所有权定义下的一个必然的简化选择。

## RUST Result类型标准库代码分析
代码路径：
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\core\src\result.rs
Result<T,E>采用了与Option<T>类似的思路来处理函数或方法返回值存在异常的情况。
Result<T,E>的Try trait十分重要，另外，以map为代表的函数同样打开函数链式调用的通道。
Result<T,E>值得关注方法的源代码如下：
```rust

impl<T, E> Result<T, E> {

    //应用于函数式编程，如果是Ok, 利用闭包直接处理Result值，返回需要的新Result类型
    pub fn map<U, F: FnOnce(T) -> U>(self, op: F) -> Result<U, E> {
        match self {
            Ok(t) => Ok(op(t)),
            Err(e) => Err(e),
        }
    }

    //如果是Ok, 利用闭包处理Result值，返回需要的类型，如果是Err返回默认值
    pub fn map_or<U, F: FnOnce(T) -> U>(self, default: U, f: F) -> U {
        match self {
            Ok(t) => f(t),
            Err(_) => default,
        }
    }

    //如果是Ok, 调用闭包处理Result，返回需要的类型, 如果是Err，调用错误闭包函数处理错误
    pub fn map_or_else<U, D: FnOnce(E) -> U, F: FnOnce(T) -> U>(self, default: D, f: F) -> U {
        match self {
            Ok(t) => f(t),
            Err(e) => default(e),
        }
    }

    //如果是Err, 调用闭包函数处理错误，返回需要的类型， Ok则返回原值
    pub fn map_err<F, O: FnOnce(E) -> F>(self, op: O) -> Result<T, F> {
        match self {
            Ok(t) => Ok(t),
            Err(e) => Err(op(e)),
        }
    }

    //Result传递，Ok则返回给定的Result类型值，否则返回原值
    pub fn and<U>(self, res: Result<U, E>) -> Result<U, E> {
        match self {
            Ok(_) => res,
            Err(e) => Err(e),
        }
    }

    //Ok 则调用闭包处理,返回需要的Result类型值， 否则返回原值
    pub fn and_then<U, F: FnOnce(T) -> Result<U, E>>(self, op: F) -> Result<U, E> {
        match self {
            Ok(t) => op(t),
            Err(e) => Err(e),
        }
    }

    //Ok返回原值，Err返回传入的默认Result类型值
    pub fn or<F>(self, res: Result<T, F>) -> Result<T, F> {
        match self {
            Ok(v) => Ok(v),
            Err(_) => res,
        }
    }

    //Ok返回原值，Err调用函数进行处理，返回需要的Result类型值
    pub fn or_else<F, O: FnOnce(E) -> Result<T, F>>(self, op: O) -> Result<T, F> {
        match self {
            Ok(t) => Ok(t),
            Err(e) => op(e),
        }
    }

    //解封装，Ok返回封装内的值，Err返回默认值
    pub fn unwrap_or(self, default: T) -> T {
        match self {
            Ok(t) => t,
            Err(_) => default,
        }
    }

    //解封装， Ok返回封装内的值， Err调用处理函数处理
    pub fn unwrap_or_else<F: FnOnce(E) -> T>(self, op: F) -> T {
        match self {
            Ok(t) => t,
            Err(e) => op(e),
        }
    }

    //确认返回一定是Ok时的解封装函数
    pub unsafe fn unwrap_unchecked(self) -> T {
        debug_assert!(self.is_ok());
        match self {
            Ok(t) => t,
            // SAFETY: the safety contract must be upheld by the caller.
            Err(_) => unsafe { hint::unreachable_unchecked() },
        }
    }

    //确认返回一定是Err时调用的解封装函数
    pub unsafe fn unwrap_err_unchecked(self) -> E {
        debug_assert!(self.is_err());
        match self {
            // SAFETY: the safety contract must be upheld by the caller.
            Ok(_) => unsafe { hint::unreachable_unchecked() },
            Err(e) => e,
        }
    }
}
```
Result<T,E>的解封装函数如下：
```rust
impl<T, E: fmt::Debug> Result<T, E> {
    //解封装，Ok解封装，Err输出参数信息并退出
    pub fn expect(self, msg: &str) -> T {
        match self {
            Ok(t) => t,
            Err(e) => unwrap_failed(msg, &e),
        }
    }

    //解封装，Ok解封装，Err输出固定信息并退出
    pub fn unwrap(self) -> T {
        match self {
            Ok(t) => t,
            Err(e) => unwrap_failed("called `Result::unwrap()` on an `Err` value", &e),
        }
    }
}

impl<T: fmt::Debug, E> Result<T, E> {
    //解封装，对于Ok输出参数指定的信息并退出，Err解封装
    pub fn expect_err(self, msg: &str) -> E {
        match self {
            Ok(t) => unwrap_failed(msg, &t),
            Err(e) => e,
        }
    }
    
    //解封装，对于Ok输出固定的信息并退出，Err解封装
    pub fn unwrap_err(self) -> E {
        match self {
            Ok(t) => unwrap_failed("called `Result::unwrap_err()` on an `Ok` value", &t),
            Err(e) => e,
        }
    }
}


impl<T: Default, E> Result<T, E> {
    //解封装，Ok解封装， Err返回T的Default值
    pub fn unwrap_or_default(self) -> T {
        match self {
            Ok(x) => x,
            Err(_) => Default::default(),
        }
    }
}

impl<T, E: Into<!>> Result<T, E> {
    //解封装，Ok解封装，Err返回Never类型
    pub fn into_ok(self) -> T {
        match self {
            Ok(x) => x,
            Err(e) => e.into(),
        }
    }
}

impl<T: Into<!>, E> Result<T, E> {
    //解封装，Err解封装， Ok返回Never类型
    pub fn into_err(self) -> E {
        match self {
            Ok(x) => x.into(),
            Err(e) => e,
        }
    }
}

impl<T, E> Result<Option<T>, E> {
    //将Result<>转换为Option
    pub const fn transpose(self) -> Option<Result<T, E>> {
        match self {
            Ok(Some(x)) => Some(Ok(x)),
            Ok(None) => None,
            Err(e) => Some(Err(e)),
        }

    }
}

```

# RUST标准库的基础Trait

## 编译器内置Trait代码分析
代码路径：
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\core\src\marker.rs

marker trait是没有实现体，是一种特殊的类型性质，这类性质无法用类型成员来表达，因此用trait来实现是最合适的。

```rust
//线程之间可移动
pub unsafe auto trait Send {
    // empty.
}
//裸指针不支持Send Trait实现，利用"!"表示明确的对Trait的不支持
impl<T: ?Sized> !Send for *const T {}
impl<T: ?Sized> !Send for *mut T {}

mod impls {
    // 实现Sync的类型的不可变引用类型支持Send
    unsafe impl<T: Sync + ?Sized> Send for &T {}
    // 实现Send的类型的可变引用类型支持 Send
    unsafe impl<T: Send + ?Sized> Send for &mut T {}
}

// 线程间可共享
pub unsafe auto trait Sync {
    // Empty
}
//裸指针不支持Sync Trait
impl<T: ?Sized> !Sync for *const T {}
impl<T: ?Sized> !Sync for *mut T {}

//类型内存大小固定
pub trait Sized {
    // Empty.
}

//如果一个Sized的类型要强制转换为动态大小类型，那必须实现Unsize Trait
//例如 [T;N] 实现了 Unsize<[T]>
pub trait Unsize<T: ?Sized> {
    // Empty.
}

//模式匹配表达式匹配时编译器需要使用的Trait，如果一个结构实现了PartialEq，该Trait会自动被实现。
pub trait StructuralPartialEq {
    // Empty.
}

//主要用于模式匹配，如果一个结构实现了Eq, 该Trait会自动被实现。
pub trait StructuralEq {
    // Empty.
}
```
以下给出了一个针对所有的原生类型都实现Copy Trait的实现代码, 实现了Copy Trait的类型编译器不必调用drop来对类型进行内存释放。
可以看到，RUST针对原生类型可以直接扩展Trait实现，者极大的提高了RUST的语法一致性及函数式编程的能力：

```rust
//Copy，略
pub trait Copy: Clone {
    // Empty.
}

//统一实现原生类型对Copy Trait的支持
mod copy_impls {

    use super::Copy;

    macro_rules! impl_copy {
        ($($t:ty)*) => {
            $(
                impl Copy for $t {}
            )*
        }
    }

    impl_copy! {
        usize u8 u16 u32 u64 u128
        isize i8 i16 i32 i64 i128
        f32 f64
        bool char
    }

    impl Copy for ! {}

    impl<T: ?Sized> Copy for *const T {}

    impl<T: ?Sized> Copy for *mut T {}

    impl<T: ?Sized> Copy for &T {}

    //& mut T不支持Copy，以保证RUST的借用规则
}
```

PhantomData<T>类型可以在其他类型结构体中定义一个变量，标记此结构体逻辑上拥有，但不需要或不方便在结构体成员变量体现的某个属性。实质上，智能指针一般都需要利用Unique<T>，以PhantomData来实现对堆内存的逻辑拥有权.
PhantomData最常用来标记生命周期及所有权。主要给编译器提示检验类型变量的生命周期和类型构造时输入的生命周期关系。也用来提示拥有PhantomData<T>的结构体会负责对T做drop操作。需要编译器做drop检查的时候更准确的判断出内存安全错误。
PhantomData<T>属性与所有权或生命周期的关系由编译器自行推断。具体实例可参考官方标准库文档及后继相关章节。
PhantomData是个单元结构体，单元结构体的变量名就是单元结构体的类型名。
所以使用的时候直接使用PhantomData即可，编译器会将泛型的类型实例化信息自动带入PhantomData中
```rust
pub struct PhantomData<T: ?Sized>;
```
## ops 运算符 Trait 代码分析
代码路径如下：
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\core\src\ops\*.rs

RUST中，所有的运算符号都可以重载。对于Ops运算符，RUST都可以提供*两个不同类型*之间的运算。
### 一个小规则
在重载函数中，如果重载的符号出現，编译器用规定的默认操作来实现。例如：
```rust
        impl const BitAnd for u8 {
            type Output = u8;
            //下面函数内部的 & 符号不再引发重载，是编译器的默认按位与操作。
            fn bitand(self, rhs: u8) -> u8 { self & u8 }
        }
```
### 数学运算符 Trait
易理解，略

### 位运算符 Trait
易理解，略

### 关系运算符Trait
代码路径如下：
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\core\src\cmp.rs

关系运算符的代码稍微复杂，这里给出较完整的代码。
```rust
//"==" "!="的实现Trait，对于在整个类型定义域内存在值无法满足相等条件的，就只实现该Trait的类型
//例如浮点类型 “NaN != NaN" , 实质上，在代码上，对于相等/不等判断只需要实现这个Trait即可
//可以为一个类型实现不同与此类型的PartialEq
pub trait PartialEq<Rhs: ?Sized = Self> {
    /// “==” 重载方法
    fn eq(&self, other: &Rhs) -> bool;

    ///`!=` 重载方法
    fn ne(&self, other: &Rhs) -> bool {
        !self.eq(other)
    }
}

//对于全作用域所有值都可相等的类型。实现这个Trait，PartialEq和Eq区别实现，也是Rust安全性的体现之一
//代码就用PartialEq即可
pub trait Eq: PartialEq<Self> {
    fn assert_receiver_is_total_eq(&self) {}
}
```
对于"<,>,<=,>="等四种运算，同上，对于全域如果有可能出现无法比较的情况，仅实现PartialOrd<Rhs>，如下：
```rust
// "<" ">" ">=" "<=" 运算符重载结构, 事实上关系运算只需要重载这个Trait
// Ord Trait 不用编码,
// 可以为一个类型实现不同于此类型的PartialEq
pub trait PartialOrd<Rhs: ?Sized = Self>: PartialEq<Rhs> {
    // 显然，只能有一个比较函数, 对于全域都满足比较的，此函数内部一般用Ord
    // Trait的cmp，对于无法比较的，需要实现独立的代码，如浮点,因为存在不可比较
    //的值，所以需要用Option
    fn partial_cmp(&self, other: &Rhs) -> Option<Ordering>;

    // "<" 运算符重载
    fn lt(&self, other: &Rhs) -> bool {
        matches!(self.partial_cmp(other), Some(Less))
    }
    
    //"<="运算符重载
    fn le(&self, other: &Rhs) -> bool {
        // Pattern `Some(Less | Eq)` optimizes worse than negating `None | Some(Greater)`.
        !matches!(self.partial_cmp(other), None | Some(Greater))
    }

    //">"运算符重载, 代码略
    fn gt(&self, other: &Rhs) -> bool;
    
    //">="运算符重载，代码略
    fn ge(&self, other: &Rhs) -> bool;

    //eq已经在PatialEq中包含
}

//Ord是全域值都可比较的Trait，其与PartialOrd结果应该一致
pub trait Ord: Eq + PartialOrd<Self> {
    //通常partial_cmp() == Some(cmp()),因为全域值
    //都可以比较，不会出现Ordering之外的情况
    fn cmp(&self, other: &Self) -> Ordering;

    fn max(self, other: Self) -> Self
    where
        Self: Sized,
    {
        //见下面代码分析
        max_by(self, other, Ord::cmp)
    }

    fn min(self, other: Self) -> Self
    where
        Self: Sized,
    {
        //见下面代码分析
        min_by(self, other, Ord::cmp)
    }

    fn clamp(self, min: Self, max: Self) -> Self
    where
        Self: Sized,
    {
        assert!(min <= max);
        if self < min {
            min
        } else if self > max {
            max
        } else {
            self
        }
    }
}
//用于表示关系结果的结构体
#[derive(Clone, Copy, PartialEq, Debug, Hash)]
#[repr(i8)]
pub enum Ordering {
    /// 小于.
    Less = -1,
    /// 等于.
    Equal = 0,
    /// 大于.
    Greater = 1,
}

impl Ordering {
    //对Ordering做逆操作, 代码略
    pub const fn reverse(self) -> Ordering ;

    //用来简化代码及更好的支持函数式编程
    //举例：
    // let x: (i64, i64, i64) = (1, 2, 7);
    // let y: (i64, i64, i64) = (1, 5, 3);
    // let result = x.0.cmp(&y.0).then(x.1.cmp(&y.1)).then(x.2.cmp(&y.2));
    pub const fn then(self, other: Ordering) -> Ordering {
        match self {
            Equal => other,
            _ => self,
        }
    }

    //用来简化代码实及支持函数式编程
    pub fn then_with<F: FnOnce() -> Ordering>(self, f: F) -> Ordering {
        match self {
            Equal => f(),
            _ => self,
        }
    }
}

//用输入的闭包比较函数获取两个值中大的一个
pub fn max_by<T, F: FnOnce(&T, &T) -> Ordering>(v1: T, v2: T, compare: F) -> T {
    match compare(&v1, &v2) {
        Ordering::Less | Ordering::Equal => v2,
        Ordering::Greater => v1,
    }
}

//用输入的闭包比较函数获取两个值中小的一个
pub fn min_by<T, F: FnOnce(&T, &T) -> Ordering>(v1: T, v2: T, compare: F) -> T {
    match compare(&v1, &v2) {
        Ordering::Less | Ordering::Equal => v1,
        Ordering::Greater => v2,
    }
}

pub fn min<T: Ord>(v1: T, v2: T) -> T {
    v1.min(v2)
}

pub fn min_by_key<T, F: FnMut(&T) -> K, K: Ord>(v1: T, v2: T, mut f: F) -> T {
    min_by(v1, v2, |v1, v2| f(v1).cmp(&f(v2)))
}

pub fn max<T: Ord>(v1: T, v2: T) -> T {
    v1.max(v2)
}

pub fn max_by_key<T, F: FnMut(&T) -> K, K: Ord>(v1: T, v2: T, mut f: F) -> T {
    max_by(v1, v2, |v1, v2| f(v1).cmp(&f(v2)))
}
```
以下是利用泛型和Adapter模式的典型的解决一类问题的RUST解决方案，下面是对有序的类型实现逆序的方案
```rust
//对于实现了PartialOrd的类型实现一个Ord的反转，这个设计是典型的RUST的思考方式，
//利用一个Adpater设计模式+泛型，很轻松的解决了一类需求
//adapter的设计模式例子
pub struct Reverse<T>(pub T);

impl<T: PartialOrd> PartialOrd for Reverse<T> {
    fn partial_cmp(&self, other: &Reverse<T>) -> Option<Ordering> {
        other.0.partial_cmp(&self.0)
    }

    fn lt(&self, other: &Self) -> bool {
        other.0 < self.0
    }

    //其他方法，略
    ...
    ...
}
```
以下是关系运算的原生类型的实现，可以参考
```rust
// 具体的实现宏 
mod impls {
    use crate::cmp::Ordering::{self, Equal, Greater, Less};
    use crate::hint::unreachable_unchecked;
    
    //PartialEq在原生类型上的实现,利用宏减少重复代码
    macro_rules! partial_eq_impl {
        ($($t:ty)*) => ($(
            //Rhs是默认的
            impl PartialEq for $t {
                //编译器默认的符号
                fn eq(&self, other: &$t) -> bool { (*self) == (*other) }
                fn ne(&self, other: &$t) -> bool { (*self) != (*other) }
            }
        )*)
    }
    //单元类型，一定相等
    impl PartialEq for () {
        fn eq(&self, _other: &()) -> bool {
            true
        }
        fn ne(&self, _other: &()) -> bool {
            false
        }
    }
    //所有类型都实现PartialEq
    partial_eq_impl! {
        bool char usize u8 u16 u32 u64 u128 isize i8 i16 i32 i64 i128 f32 f64
    }
    
    macro_rules! eq_impl {
        ($($t:ty)*) => ($(
            #[stable(feature = "rust1", since = "1.0.0")]
            impl Eq for $t {}
        )*)
    }

    //浮点不实现Eq
    eq_impl! { () bool char usize u8 u16 u32 u64 u128 isize i8 i16 i32 i64 i128 }

    //关系运算，利用宏减少代码, 这个宏仅仅针对浮点数
    macro_rules! partial_ord_impl {
        ($($t:ty)*) => ($(
            #[stable(feature = "rust1", since = "1.0.0")]
            impl PartialOrd for $t {
                fn partial_cmp(&self, other: &$t) -> Option<Ordering> {
                    //RUST的典型的代码思路，需要学习及仔细体会,
                    //专门为浮点做的比较
                    match (self <= other, self >= other) {
                        (false, false) => None,
                        (false, true) => Some(Greater),
                        (true, false) => Some(Less),
                        (true, true) => Some(Equal),
                    }
                }
                fn lt(&self, other: &$t) -> bool { (*self) < (*other) }
                fn le(&self, other: &$t) -> bool { (*self) <= (*other) }
                fn ge(&self, other: &$t) -> bool { (*self) >= (*other) }
                fn gt(&self, other: &$t) -> bool { (*self) > (*other) }
            }
        )*)
    }

    //仅在浮点数实现
    partial_ord_impl! { f32 f64 }
    
    //为支持全域值可比较的类型实现的宏
    macro_rules! ord_impl {
        ($($t:ty)*) => ($(
            impl PartialOrd for $t {
                //复用Ord的cmp函数
                fn partial_cmp(&self, other: &$t) -> Option<Ordering> {
                    Some(self.cmp(other))
                }
                fn lt(&self, other: &$t) -> bool { (*self) < (*other) }
                fn le(&self, other: &$t) -> bool { (*self) <= (*other) }
                fn ge(&self, other: &$t) -> bool { (*self) >= (*other) }
                fn gt(&self, other: &$t) -> bool { (*self) > (*other) }
            }

            impl Ord for $t {
                fn cmp(&self, other: &$t) -> Ordering {
                    if *self < *other { Less }
                    else if *self == *other { Equal }
                    else { Greater }
                }
            }
        )*)
    }
    //浮点数不支持Ord
    ord_impl! { char usize u8 u16 u32 u64 u128 isize i8 i16 i32 i64 i128 }

    //A实现了PartialEq<B>, PartialOrd<B>后，对&A实现PartialEq<&B>， PartialOrd<&B>
    impl<A: ?Sized, B: ?Sized> PartialEq<&B> for &A
    where
        A: PartialEq<B>,
    {
        fn eq(&self, other: &&B) -> bool {
            //注意这个调用方式，此时不能用self.eq调用。
            //eq方法参数为引用
            PartialEq::eq(*self, *other)
        }
        fn ne(&self, other: &&B) -> bool {
            PartialEq::ne(*self, *other)
        }
    }
    
}

```
以上较完整的给出了关系运算Trait的代码，可以看到，RUST标准库除了对原生类型做了Trait的实现，也针对受约束的泛型尽可能的做了关系运算符 Trait的实现，以便最大的减少后继的开发量。程序员需要精通RUST的标准库已经针对那些泛型类型做好了实现，避免再重复的造轮子。

### ？运算符 Trait代码分析
代码路径：try_trait.rs

?操作是RUST支持函数式编程的一个重要支柱。不过即使不用于函数式编程，也可大幅简化代码。
当一个类型实现了Try Trait时。可对这个类型做？操作简化代码。
Try Trait也是try..catch在RUST中的一种实现方式，但从代码的表现形式上更加简化。另外，因为能够返回具体类型，这种实现方式就不仅局限于处理异常，可以扩展到其他类似的场景。
可以定义返回类型的方法，支持链式函数调用。
Try Trait定义如下：
```rust
pub trait Try: FromResidual {
    /// ?操作符正常结束的返回值类型
    type Output;

    /// ?操作符提前返回的值类型，后继会用实例来说明
    type Residual;

    /// 从Self::Output返回值类型中获得实现Try Trait的类型的值
    /// 函数必须符合下面代码的原则，
    /// `Try::from_output(x).branch() --> ControlFlow::Continue(x)`.
    /// 例子：
    /// ```
    /// assert_eq!(<Result<_, String> as Try>::from_output(3), Ok(3));
    /// assert_eq!(<Option<_> as Try>::from_output(4), Some(4));
    /// assert_eq!(
    ///     <std::ops::ControlFlow<String, _> as Try>::from_output(5),
    ///     std::ops::ControlFlow::Continue(5),
    /// );
    fn from_output(output: Self::Output) -> Self;

    /// branch函数会返回ControlFlow，据此决定流程继续还是提前返回
    /// 例子：
    ///
    /// assert_eq!(Ok::<_, String>(3).branch(), ControlFlow::Continue(3));
    /// assert_eq!(Err::<String, _>(3).branch(), ControlFlow::Break(Err(3)));
    ///
    /// assert_eq!(Some(3).branch(), ControlFlow::Continue(3));
    /// assert_eq!(None::<String>.branch(), ControlFlow::Break(None));
    ///
    /// assert_eq!(ControlFlow::<String, _>::Continue(3).branch(), ControlFlow::Continue(3));
    /// assert_eq!(
    ///     ControlFlow::<_, String>::Break(3).branch(),
    ///     ControlFlow::Break(ControlFlow::Break(3)),
    /// );
    fn branch(self) -> ControlFlow<Self::Residual, Self::Output>;
}

pub trait FromResidual<R = <Self as Try>::Residual> {
    /// 该函数从提前返回的值中获取实现Try Trait的类型的值
    ///
    /// 此函数必须符合下面代码的原则
    /// `FromResidual::from_residual(r).branch() --> ControlFlow::Break(r)`.
    /// 例子：
    /// assert_eq!(Result::<String, i64>::from_residual(Err(3_u8)), Err(3));
    /// assert_eq!(Option::<String>::from_residual(None), None);
    /// assert_eq!(
    ///     ControlFlow::<_, String>::from_residual(ControlFlow::Break(5)),
    ///     ControlFlow::Break(5),
    /// );
    fn from_residual(residual: R) -> Self;
}
```
Try Trait对? 操作支持的举例如下：
```rust
//不用? 操作的代码
 pub fn simple_try_fold_3<A, T, R: Try<Output = A>>(
     iter: impl Iterator<Item = T>,
     mut accum: A,
     mut f: impl FnMut(A, T) -> R,
 ) -> R {
     for x in iter {
         let cf = f(accum, x).branch();
         match cf {
             ControlFlow::Continue(a) => accum = a,
             ControlFlow::Break(r) => return R::from_residual(r),
         }
     }
     R::from_output(accum)
 }
// 使用? 操作的代码:
 fn simple_try_fold<A, T, R: Try<Output = A>>(
     iter: impl Iterator<Item = T>,
     mut accum: A,
     mut f: impl FnMut(A, T) -> R,
 ) -> R {
     for x in iter {
         accum = f(accum, x)?;
     }
     R::from_output(accum)
 }
```
由上，可推断出T?表示如下代码
```rust
   match((T as Try).branch()) {
       ControlFlow::Continue(a) => a,
       ControlFlow::Break(r) => return (T as Try)::from_residual(r),
   }
```
ControlFlow类型代码如下, 主要用于指示代码控制流程指示， 逻辑上可类比于continue, break 关键字 代码如下：
```rust
pub enum ControlFlow<B, C = ()> {
    //代码过程继续执行，可以从C中得到代码过程的中间结果
    Continue(C),
    /// 代码过程应退出，可以从B中的到代码退出时的中间结果
    Break(B),
}
```

#### Option<T>的Try Trait实现
实现代码如下：
```rust
impl<T> ops::Try for Option<T> {
    type Output = T;
    // Infallible是一种错误类型，但该错误永远也不会发生，这里需要返回None，
    // 所以需要用Option类型，但因为只用None。所以Some使用Infallible来表示不会被使用，这也表现了RUST的安全理念，一定在类型定义的时候保证代码安全。
    type Residual = Option<convert::Infallible>;

    fn from_output(output: Self::Output) -> Self {
        Some(output)
    }

    fn branch(self) -> ControlFlow<Self::Residual, Self::Output> {
        match self {
            Some(v) => ControlFlow::Continue(v),
            None => ControlFlow::Break(None),
        }
    }
}

impl<T> const ops::FromResidual for Option<T> {
    fn from_residual(residual: Option<convert::Infallible>) -> Self {
        match residual {
            None => None,
        }
    }
}
```
所以，一个Option<T>？等同于如下代码：
```rust
   match(Option<T>.branch()) {
       ControlFlow::Continue(a) => a,
       //下面代码实际就是return None
       ControlFlow::Break(None) => return (Option<T>::from_residual(None)),
   }
```
Result<T,E>类型的Try Trait请自行分析

#### 小结
利用Try Trait，程序员可以实现自定义类型的?，提供函数式编程的有力手段并简化代码，提升代码的理解度。

### Range 运算符代码分析
代码路径：  
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\core\src\ops\range.rs
Range是符号 .. , start..end , start.. , ..end , ..=end，start..=end 形式
#### Range相关的边界结构Bound
源代码：
```rust
pub enum Bound<T> {
    /// 边界包括在Range内
    Included(T),
    /// 边界不包括在Range内
    Excluded(T),
    /// 边界是无限的，边界不存在
    Unbounded,
}
```
Include边界的值包含，Exclued边界的值不包含，Unbounded边界值不存在
#### RangeFull
` ..  `的数据结构。
```rust
struct RangeFull;
```
#### Range<Idx>
`start.. end`的数据结构
```rust
pub struct Range<Idx> {
    pub start: Idx,
    pub end: Idx,
}
```
#### RangeFrom<Idx>
`start..`的数据结构, 略
#### RangeTo<Idx>
`.. end`的数据结构, 略
#### RangeInclusive<Idx>
`start..=end`的数据结构,略
#### RangeToInclusive<Idx>
`..=end`的数据结构,略

以上的Idx需要满足Idx:PartialOrd<Idx>

#### RangeBounds<T: ?Sized>
所有Range统一实现的Trait。
```rust
pub trait RangeBounds<T: ?Sized> {
    /// 获取范围的起始值
    ///
    /// 例子
    /// assert_eq!((..10).start_bound(), Unbounded);
    /// assert_eq!((3..10).start_bound(), Included(&3));
    fn start_bound(&self) -> Bound<&T>;

    /// 获取范围的终止值.
    /// 例子
    /// assert_eq!((3..).end_bound(), Unbounded);
    /// assert_eq!((3..10).end_bound(), Excluded(&10));
    fn end_bound(&self) -> Bound<&T>;

    /// 范围是否包括某个值.
    /// 例子
    /// assert!( (3..5).contains(&4));
    /// assert!(!(3..5).contains(&2));
    ///
    /// assert!( (0.0..1.0).contains(&0.5));
    /// assert!(!(0.0..1.0).contains(&f32::NAN));
    /// assert!(!(0.0..f32::NAN).contains(&0.5));
    /// assert!(!(f32::NAN..1.0).contains(&0.5));
    fn contains<U>(&self, item: &U) -> bool
    where
        T: PartialOrd<U>,
        U: ?Sized + PartialOrd<T>,
    {
        //比较有意思的典型的RUST代码
        (match self.start_bound() {
            Included(start) => start <= item,
            Excluded(start) => start < item,
            Unbounded => true,
        }) && (match self.end_bound() {
            Included(end) => item <= end,
            Excluded(end) => item < end,
            Unbounded => true,
        })
    }
}

```
RangeBounds针对RangeFull，RangeTo, RangeInclusive, RangeToInclusive, RangeFrom, Range结构都进行了实现。同时针对(Bound<T>, Bound<T>)的元组做了实现。

#### Range的灵活性与独立性
完全可以定义 ((0,0)..(100,100))； ("1st".."30th")这种极有表现力的Range。
Range使用的时候，需要先定义一个取值集合，定义类型表示这个集合，针对类型实现PartialOrd。就可以对这个集合的类型用Range符号了。
值得注意的是，对于Range<Idx>, 如果一个变量类型为U, 则如果实现了PartialOrd<U> for Idx， 那U就有可能属于Range, 即U可以与Idx不同。
Range操作符多用于与Index运算符结合或与Iterator Trait结合使用，独立的Range在。在后继的Index运算符和Iterator中会研究Range是如何与他们结合的。

#### 小结
基于泛型的Range类型提供了非常好的语法手段，只要某类型支持排序，那就可以定义一个在此类型基础上实现的Range类型。再结合Index和Iterator, 将高效的实现极具冲击力的代码。

### RUST的Index 运算符代码分析
代码路径：  
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\core\src\ops\index.rs

数组下标符号[]由Index, IndexMut两个Trait完成重载。数组下标符号重载使得程序更有可读性。两个Trait如下定义：
```rust
// [T][Idx] 形式重载
pub trait Index<Idx: ?Sized> {
    /// The returned type after indexing.
    type Output: ?Sized;

    /// 如果传入的参数超过内存界限将马上引发panic
    fn index(&self, index: Idx) -> &Self::Output;
}
//mut [T][Idx]形式重载
pub trait IndexMut<Idx: ?Sized>: Index<Idx> {
    fn index_mut(&mut self, index: Idx) -> &mut Self::Output;
}
```
由以上可以看出["Hary"], ["Bold"]之类的表达形式都是可以存在的。

#### 切片数据结构[T]的Index实现
```rust
impl<T, I> ops::Index<I> for [T]
where
    I: SliceIndex<[T]>,
{
    type Output = I::Output;

    fn index(&self, index: I) -> &I::Output {
        index.index(self)
    }
}

impl<T, I> ops::IndexMut<I> for [T]
where
    I: SliceIndex<[T]>,
{
    fn index_mut(&mut self, index: I) -> &mut I::Output {
        index.index_mut(self)
    }
}
```
SliceIndex Trait 被设计同时满足Index及切片类型的一些方法的需求。因为这些需求在逻辑上是同领域的。集中在SliceIndex Trait模块性更好。如：
`[T]::get<I:SliceIndex>(&self, I)->Option<&I::Output>` 就是直接调用SliceIndex中的方法来实现成员获取。

```rust
mod private_slice_index {
    use super::ops;
    //在私有模块中定义一个Sealed Trait，后继的SliceIndex继承Sealed。
    //带来的结果是只有在本模块实现了Sealed Trait的类型才能实现SliceIndex
    //即使SliceIndex是公有定义，其他类型仍然不能够实现SliceIndex
    pub trait Sealed {}

    impl Sealed for usize {}
    impl Sealed for ops::Range<usize> {}
    impl Sealed for ops::RangeTo<usize> {}
    impl Sealed for ops::RangeFrom<usize> {}
    impl Sealed for ops::RangeFull {}
    impl Sealed for ops::RangeInclusive<usize> {}
    impl Sealed for ops::RangeToInclusive<usize> {}
    impl Sealed for (ops::Bound<usize>, ops::Bound<usize>) {}
}

pub unsafe trait SliceIndex<T: ?Sized>: private_slice_index::Sealed {
    /// 此类型通常为T或者T的引用，切片，裸指针类型
    type Output: ?Sized;

    // 从slice变量中用self获取Option<Output>变量
    fn get(self, slice: &T) -> Option<&Self::Output>;

    fn get_mut(self, slice: &mut T) -> Option<&mut Self::Output>;

    //slice是序列的头指针，后面的具体实现会看到为什么用 *const
    unsafe fn get_unchecked(self, slice: *const T) -> *const Self::Output;

    unsafe fn get_unchecked_mut(self, slice: *mut T) -> *mut Self::Output;
    
    //如果self超出slice的安全范围，会panic
    fn index(self, slice: &T) -> &Self::Output;

    fn index_mut(self, slice: &mut T) -> &mut Self::Output;
}

unsafe impl<T> SliceIndex<[T]> for usize {
    type Output = T;

    //此函数主要用在不适合使用下标时，例如不确定切片长度，又不希望panic
    fn get(self, slice: &[T]) -> Option<&T> {
        // 这里slice 被强制转化成了* const [T]
        if self < slice.len() { unsafe { Some(&*self.get_unchecked(slice)) } } else { None }
    }

    fn get_mut(self, slice: &mut [T]) -> Option<&mut T> {
        //这里slice 被强制转化成了*mut [T]
        if self < slice.len() { unsafe { Some(&mut *self.get_unchecked_mut(slice)) } } else { None }
    }
    
    //此函数主要用在不适合下标的情况下
    unsafe fn get_unchecked(self, slice: *const [T]) -> *const T {
        //slice.as_ptr()获得* const T，利用* const T的add方法来取得slice成员的地址
        unsafe { slice.as_ptr().add(self) }
    }

    unsafe fn get_unchecked_mut(self, slice: *mut [T]) -> *mut T {
        //as_mut_ptr返回* mut T指针，在用add方法获得成员地址。
        unsafe { slice.as_mut_ptr().add(self) }
    }

    fn index(self, slice: &[T]) -> &T {
        //使用编译器内置支持，为了效率直接使用了内置的数组下标表示。此操作可能引发panic
        &(*slice)[self]
    }

    fn index_mut(self, slice: &mut [T]) -> &mut T {
        // 使用编译器内置下标运算符，可能引发panic
        &mut (*slice)[self]
    }
}
```
以上就是针对[T]的以无符号数作为下标取出单一元素的ops::Index 及 ops::IndexMut的底层实现。
针对Range做下标的代码实现
```rust

unsafe impl<T> SliceIndex<[T]> for ops::Range<usize> {
    type Output = [T];
    
    //不会引发panic的方法
    //此处注意，self是Rang<usize>
    fn get(self, slice: &[T]) -> Option<&[T]> {
        //提前做判断
        if self.start > self.end || self.end > slice.len() {
            None
        } else {
            unsafe { Some(&*self.get_unchecked(slice)) }
        }
    }
    
    //可变引用获取
    fn get_mut(self, slice: &mut [T]) -> Option<&mut [T]> {
        if self.start > self.end || self.end > slice.len() {
            None
        } else {
            unsafe { Some(&mut *self.get_unchecked_mut(slice)) }
        }
    }

    //不对输出参数做判断, 调用者要保证输入参数没有问题
    unsafe fn get_unchecked(self, slice: *const [T]) -> *const [T] {
        // 先将*const [T] 转换为 * const T，完成指针运算，然后再转换成* const [T]
        unsafe { ptr::slice_from_raw_parts(slice.as_ptr().add(self.start), self.end - self.start) }
    }
    
    //与上面函数类似，略
    unsafe fn get_unchecked_mut(self, slice: *mut [T]) -> *mut [T] {
        unsafe {
            ptr::slice_from_raw_parts_mut(slice.as_mut_ptr().add(self.start), self.end - self.start)
        }
    }

    fn index(self, slice: &[T]) -> &[T] {
        //超出范围会直接panic
        if self.start > self.end {
            slice_index_order_fail(self.start, self.end);
        } else if self.end > slice.len() {
            slice_end_index_len_fail(self.end, slice.len());
        }
        //将* const [T]转化为切片引用
        unsafe { &*self.get_unchecked(slice) }
    }

    fn index_mut(self, slice: &mut [T]) -> &mut [T] {
        //超出范围会直接panic
        if self.start > self.end {
            slice_index_order_fail(self.start, self.end);
        } else if self.end > slice.len() {
            slice_end_index_len_fail(self.end, slice.len());
        }
        unsafe { &mut *self.get_unchecked_mut(slice) }
    }
}
```
以上是实现用Range从slice中取出子slice的实现。同样是使用裸指针来最高效的实现逻辑。实际上，不用裸指针就没法实现。

```rust
unsafe impl<T> SliceIndex<[T]> for ops::RangeTo<usize> {
    type Output = [T];

    fn get(self, slice: &[T]) -> Option<&[T]> {
        //将RangeTo转换成Range, 然后对ops::Range<usize>的方法直接调用
        (0..self.end).get(slice)
    }

    fn get_mut(self, slice: &mut [T]) -> Option<&mut [T]> {
        //对ops::Range<usize>的方法直接调用
        (0..self.end).get_mut(slice)
    }
    
    //其他方法也是直接对Range<usize>做一个调用略
}
```
RangeFrom, RangeInclusive, RangeToInclusive, RangeFull等与RangeTo的实现类似，略。

##### 小结
RUST切片的下标计算展示了裸指针的使用技巧，在数组类的成员操作中，基本无法脱离裸指针。在这里，只要不越界，裸指针操作是安全的。

#### 数组数据结构[T;N]的ops::Index实现

```rust
//注意这里的Trait约束的写法
impl<T, I, const N: usize> Index<I> for [T; N]
where
    [T]: Index<I>,
{
    type Output = <[T] as Index<I>>::Output;

    fn index(&self, index: I) -> &Self::Output {
        Index::index(self as &[T], index)
    }
}

impl<T, I, const N: usize> IndexMut<I> for [T; N]
where
    [T]: IndexMut<I>,
{
    fn index_mut(&mut self, index: I) -> &mut Self::Output {
        IndexMut::index_mut(self as &mut [T], index)
    }
}
```
以上， `self as &[T]` 即把[T;N]转化为了切片[T], 所以数组的Index就是[T]的Index实现

# RUST的Iterator实现代码分析 
代码路径：  
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\core\src\iter\*.*  

Iterator在函数式编程中是居于最核心的地位。在函数式编程中，最关键的就是把问题的解决方式设计成能够使用Iterator方案来解决。RUST基本上可以说是原生的Iterator语言，几乎所有的核心关键类型的方法库都依托于Iterator。

## RUST的Iterator与其他语言Iterator比较
RUST定义了三种迭代器Trait:
1. Iterator遍历的是变量本身
```rust
pub trait IntoIterator {
    type Item;
    type IntoIter: Iterator<Item = Self::Item>;
    fn into_iter(self) -> Self::IntoIter;
}
```
into_iter返回的迭代器迭代时，会消费变量及容器，完全迭代后容器将不再存在。

2. Iterator遍历的是变量不可变引用：
这个一般不做Trait，而是容器类型实现一个方法：
```pub fn iter(&self) -> I:Iterator```
此方法返回一个迭代器，这种迭代器适用的一个例子是对网络接口做遍历以获得统计值

3. Iterator遍历的是变量的可变引用：
同2 容器类型实现方法：
```pub fn iter_mut(&self) -> I:Iterator ```
这种迭代器适用的一个例子是定时器遍历长连接，更新连接活动时间。
其他语言中没有变量迭代器，这是RUST独有的所有权和drop机制带来的一种迭代器。在适合的场景下会缩减代码量及提高效率。
一般的，RUST对于实现上面三个Trait，要求额外实现下面的两种机制
T::iter() 等同于 &T::into_iter()
T::iter_mut() 等同于 &mut T::into_iter()

## Iterator Trait 定义
```rust
pub trait Iterator {
    /// 每次迭代时返回的变量类型.
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    
    //size_hint返回值是此迭代器最少产生多少个有效迭代输出，最多产生多少有有效迭代输出。
    //所以，诸如(0..10).int_iter(), 最少是10个，最多也是10个，
    //而（0..10).filter(|x| x%2 == 0), 因为编译器不会提前计算，所以符合条件的最少可能是0个，最多是10个
    fn size_hint(&self) -> (usize, Option<usize>) {
        (0, None)
    }

    //其他方法
    ...
    ...
}

impl<I: Iterator + ?Sized> Iterator for &mut I {
    type Item = I::Item;
    fn next(&mut self) -> Option<I::Item> {
        (**self).next()
    }
    fn size_hint(&self) -> (usize, Option<usize>) {
        (**self).size_hint()
    }
    fn advance_by(&mut self, n: usize) -> Result<(), usize> {
        (**self).advance_by(n)
    }
    fn nth(&mut self, n: usize) -> Option<Self::Item> {
        (**self).nth(n)
    }
}
```
上面代码：如果一个类型I已经实现了 Iterator, 那针对这个结构的可变引用类型 &mut I, 标准库已经做了统一的 Iterator Trait实现。

## ops::Range类型的Iterator实现
代码路径：  

%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\core\src\iter\range.rs

Range被直接实现Iterator trait，没有iter()这样生成迭代器的调用。
定义如下：
```rust 
impl<A: Step> Iterator for ops::Range<A> {
    type Item = A;
    
    fn next(&mut self) -> Option<A> {
        self.spec_next()
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        if self.start < self.end {
            let hint = Step::steps_between(&self.start, &self.end);
            (hint.unwrap_or(usize::MAX), hint)
        } else {
            (0, Some(0))
        }
    }

    fn nth(&mut self, n: usize) -> Option<A> {
        self.spec_nth(n)
    }
    ...
    ...
    
}
```
Range Iterator的具体实现RangeIteratorImpl trait
```rust
impl<A: Step> RangeIteratorImpl for ops::Range<A> {
    type Item = A;

    default fn spec_next(&mut self) -> Option<A> {
        if self.start < self.end {
            //self.start.clone()是为了不转移self.start的所有权
            let n =
                Step::forward_checked(self.start.clone(), 1).expect("`Step` invariants not upheld");
            //mem::replace将self.start赋值为n，返回self.start的值，这个方式适用于任何类型，且处理了所有权问题
            //mem::replace是效率最高的代码方式
            Some(mem::replace(&mut self.start, n))
        } else {
            None
        }
    }

    ...
}
```
由上面的代码可以看出，每一次next实际都对Range本身做出了修改。

只有基于实现`Step Trait`的类型的Range才支持了Iterator, 而代码关键是Step Trait的方法，
 `Step Trait`的定义如下：
```rust
pub trait Step: Clone + PartialOrd + Sized {
    /// 从start 到end一共多少step
    fn steps_between(start: &Self, end: &Self) -> Option<usize>;

    /// 向前count步返回值
    fn forward_checked(start: Self, count: usize) -> Option<Self>;

    /// 向前count步 返回值，出错退出
    fn forward(start: Self, count: usize) -> Self {
        Step::forward_checked(start, count).expect("overflow in `Step::forward`")
    }

    /// 向前不检查 count步
    unsafe fn forward_unchecked(start: Self, count: usize) -> Self {
        Step::forward(start, count)
    }

    /// 向后count步
    fn backward_checked(start: Self, count: usize) -> Option<Self>;

    /// 向后count步，出错退出
    fn backward(start: Self, count: usize) -> Self {
        Step::backward_checked(start, count).expect("overflow in `Step::backward`")
    }
    
    /// 向后count步，出错退出
    unsafe fn backward_unchecked(start: Self, count: usize) -> Self {
        Step::backward(start, count)
    }
}
```
照此，可以实现一个自定义类型的类型, 并支持Step Trait，如此，即可使用Range符号的Iterator。例如，一个二维的点的range,例如Range<(i32, i32)>的变量((0,0)..(10,10)), 三维的点的range，数列等。

一下是为所有证书类型实现Step的宏：
```rust

macro_rules! step_identical_methods {
    () => {
        unsafe fn forward_unchecked(start: Self, n: usize) -> Self {
            // 调用代码需要保证加法不会越界.
            unsafe { start.unchecked_add(n as Self) }
        }

        unsafe fn backward_unchecked(start: Self, n: usize) -> Self {
            // 调用代码需要保证减法不会越界.
            unsafe { start.unchecked_sub(n as Self) }
        }

        fn forward(start: Self, n: usize) -> Self {
            // debug 情况下 以下代码会panic，release以下代码会被优化掉
            if Self::forward_checked(start, n).is_none() {
                let _ = Self::MAX + 1;
            }
            // release中的加法 
            start.wrapping_add(n as Self)
        }

        fn backward(start: Self, n: usize) -> Self {
            // debug情况，以下代码会panic，release挥别优化掉.
            if Self::backward_checked(start, n).is_none() {
                let _ = Self::MIN - 1;
            }
            // release下的用法
            start.wrapping_sub(n as Self)
        }
    };
}

macro_rules! step_integer_impls {
    {
        //比CPU字长小的无符号整数类型及有符号整数类型
        narrower than or same width as usize:
            $( [ $u_narrower:ident $i_narrower:ident ] ),+;
        //比CPU字长大的无符号整数类型及有符号整数类型
        wider than usize:
            $( [ $u_wider:ident $i_wider:ident ] ),+;
    } => {
        $(
            //为所有比CPU字长小的无符号整数类型的Step实现
            impl Step for $u_narrower {
                //通用实现
                step_identical_methods!();

                fn steps_between(start: &Self, end: &Self) -> Option<usize> {
                    if *start <= *end {
                        // u_nrrower类型字长必须小于usize字长
                        Some((*end - *start) as usize)
                    } else {
                        None
                    }
                }

                fn forward_checked(start: Self, n: usize) -> Option<Self> {
                    //将类型转换可能不成功显化，这是需要养成的RUST的特有思维
                    match Self::try_from(n) {
                        //checked_add完成溢出检查
                        Ok(n) => start.checked_add(n),
                        Err(_) => None, 
                    }
                }

                fn backward_checked(start: Self, n: usize) -> Option<Self> {
                    match Self::try_from(n) {
                        Ok(n) => start.checked_sub(n),
                        Err(_) => None, // if n is out of range, `unsigned_start - n` is too
                    }
                }
            }
            
            //略
            ...
    }
}
```
Range实现Iterator的代码不复杂，但是从类型转换及加减法的处理上深刻的体现了RUST的安全理念。

## slice的Iterator实现
代码路径：  
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\core\src\slice\iter.rs  

首先定义了适合&[T]的Iter结构：
```rust
pub struct Iter<'a, T: 'a> {
    //当前元素的指针，与end用不同的类型表示
    ptr: NonNull<T>,
    //尾元素指针，用ptr == end以快速检测iterator是否为空
    end: *const T, 
    //这里PhantomData 主要用来做生命周期标识，用来做Iter结构体与切片之间的生命周期关系检测
    _marker: PhantomData<&'a T>, 
}

pub struct IterMut<'a, T: 'a> {
    ptr: NonNull<T>,
    end: *mut T, 
    _marker: PhantomData<&'a mut T>,
}
```
这里，一个疑惑就是为什么不用下标及切片长度来作为Iter结构。这里主要是因为可变的Iterator实现无法支持。
例如，给出如下结构：
```rust
pub struct IterMut <'a, T:'a> {
    current: usize,
    len: usize,
    slice: 'a mut &[T]
}
```
显然，当IterMut结构是可变借用时，无法再返回一个内部成员的借用用作迭代器的迭代返回值。
```rust
impl<'a, T> IterMut<'a, T> {
    pub(super) fn new(slice: &'a mut [T]) -> Self {
        let ptr = slice.as_mut_ptr();
        unsafe {
            assume(!ptr.is_null());

            let end = if mem::size_of::<T>() == 0 {
                (ptr as *mut u8).wrapping_add(slice.len()) as *mut T
            } else {
                ptr.add(slice.len())
            };

            Self { ptr: NonNull::new_unchecked(ptr), end, _marker: PhantomData }
        }
    }
    
    ...
    ...
}
//用宏来实现切片的Iterator trait
iterator! {struct IterMut -> *mut T, &'a mut T, mut, {mut}, {}}

//上面的宏定义
macro_rules! iterator {
    (
        struct $name:ident -> $ptr:ty,
        $elem:ty,
        $raw_mut:tt,
        {$( $mut_:tt )?},
        {$($extra:tt)*}
    ) => {
        // 正向next函数辅助宏，实际的逻辑见post_inc_start函数
        macro_rules! next_unchecked {
            ($self: ident) => {& $( $mut_ )? *$self.post_inc_start(1)}
        }

        // 反向的next函数
        macro_rules! next_back_unchecked {
            ($self: ident) => {& $( $mut_ )? *$self.pre_dec_end(1)}
        }

        // 0长度元素next的移动
        macro_rules! zst_shrink {
            ($self: ident, $n: ident) => {
                //0元素数组因为不能移动指针，所以移动尾指针
                $self.end = ($self.end as * $raw_mut u8).wrapping_offset(-$n) as * $raw_mut T;
            }
        }
        
        //具体的方法实现 
        // $name 即 IterMut
        impl<'a, T> $name<'a, T> {
            // 从Iterator获得切片.
            fn make_slice(&self) -> &'a [T] {
                // Iter::ptr::as_ptr，由内存首地址和切片长度创建切片指针，然后转换为引用
                unsafe { from_raw_parts(self.ptr.as_ptr(), len!(self)) }
            }

            //实质的next
            unsafe fn post_inc_start(&mut self, offset: isize) -> * $raw_mut T {
                if mem::size_of::<T>() == 0 {
                    //0字节元素偏移实现，调整end的值，ptr不变
                    zst_shrink!(self, offset);
                    self.ptr.as_ptr()
                } else {
                    //非0字节元素，返回首地址，然后后移正确的字节
                    let old = self.ptr.as_ptr();
                    self.ptr = unsafe { NonNull::new_unchecked(self.ptr.as_ptr().offset(offset)) };
                    old
                }
            }

            // 从尾部做Iterator的实际实现函数
            unsafe fn pre_dec_end(&mut self, offset: isize) -> * $raw_mut T {
                if mem::size_of::<T>() == 0 {
                    //对于0字节元素，从头部及从尾部逻辑相同
                    zst_shrink!(self, offset);
                    self.ptr.as_ptr()
                } else {
                    //尾部的end即偏移后的位置。
                    self.end = unsafe { self.end.offset(-offset) };
                    self.end
                }
            }
        }

        //Iterator的实现, 即
        //impl<'a, T> Iterator for IterMut<'a, T>
        impl<'a, T> Iterator for $name<'a, T> {
            // $elem即&'a T
            type Item = $elem;

            fn next(&mut self) -> Option<$elem> {
                unsafe {
                    //安全性确认
                    assume(!self.ptr.as_ptr().is_null());
                    if mem::size_of::<T>() != 0 {
                        assume(!self.end.is_null());
                    }
                    if is_empty!(self) {
                        //Iter为空的话，返回None
                        None
                    } else {
                        //实际调用post_inc_start(1)
                        Some(next_unchecked!(self))
                    }
                }
            }

            fn size_hint(&self) -> (usize, Option<usize>) {
                //用len!宏计算Iter的长度
                let exact = len!(self);
                (exact, Some(exact))
            }

            fn count(self) -> usize {
                len!(self)
            }

            fn nth(&mut self, n: usize) -> Option<$elem> {
                //如果n大于Iter的长度，清空
                if n >= len!(self) {
                    if mem::size_of::<T>() == 0 {
                        self.end = self.ptr.as_ptr();
                    } else {
                        unsafe {
                            self.ptr = NonNull::new_unchecked(self.end as *mut T);
                        }
                    }
                    return None;
                }
                // 否则，失效前n-1个元素，然后做neet
                unsafe {
                    self.post_inc_start(n as isize);
                    Some(next_unchecked!(self))
                }
            }

            fn advance_by(&mut self, n: usize) -> Result<(), usize> {
                //取长度与n中的小值
                let advance = cmp::min(len!(self), n);

                //失效advance-1个值
                unsafe { self.post_inc_start(advance as isize) };
                //返回
                if advance == n { Ok(()) } else { Err(advance) }
            }

            //从尾部Iterator
            fn last(mut self) -> Option<$elem> {
                //实质调用post_dec_end(1)
                self.next_back()
            }

            //其他，略
            ...
            ...
            
        }
    }
}

//判断Iterator是否为空的宏
macro_rules! is_empty {
    // 可以满足0字节元素的切片及非0字节元素的切片
    ($self: ident) => {
        //Iter::ptr == Iter::end
        $self.ptr.as_ptr() as *const T == $self.end
    };
}

//取Iterator长度的宏
macro_rules! len {
    ($self: ident) => {{
        let start = $self.ptr;
        let size = size_from_ptr(start.as_ptr());
        //判断元素是否为0字节
        if size == 0 {
            // 用end减start得到0字节元素的切片长度
            ($self.end as usize).wrapping_sub(start.as_ptr() as usize)
        } else {
            //非0字节，用内存字节数除以单元素长度
            let diff = unsafe { unchecked_sub($self.end as usize, start.as_ptr() as usize) };
            unsafe { exact_div(diff, size) }
        }
    }};
}


```
对于切片，RUST的所有权，借用等规定导致其迭代器实际上是一个非常好的编码训练工具，代码粗略看一遍后值得自己将其实现一遍，可以有效提高对RUST的认识和编码水平。

## <a id="str_iter">字符串Iterator代码分析</a>
题外话，&str.len()返回字符串切片字节占用数，&str.chars().count()返回字符数目。
字符串切片获取Iterator有如下3个函数
&str::chars() 获得以UTF-8编码的字符串的Iterator
&str::bytes() 获得一个[u8]的Iterator
&str::char_indices() 获得一个元组，第一个成员是字符字节数组的序号，第二个成员是字符本身
bytes()主要用于提高在程序员确定采用ASCII字符串下的运行效率。
我们以&str::chars()的Iterator来看一下具体的实现
```rust
pub struct Chars<'a> {
    //利用slice通用的iter做实例化,实际是一个adapter设计模式
    pub(super) iter: slice::Iter<'a, u8>,
}

pub fn chars(&self) -> Chars<'_> {
    //self.as_bytes()获得一个&[u8]
    Chars { iter: self.as_bytes().iter() }
}
impl<'a> Iterator for Chars<'a> {
    type Item = char;

    fn next(&mut self) -> Option<char> {
        //next_code_point见后面代码分析
        next_code_point(&mut self.iter).map(|ch| {
            unsafe { char::from_u32_unchecked(ch) }
        })
    }

    fn count(self) -> usize {
        // 利用切片iterator的filter来实现
        self.iter.filter(|&&byte| !utf8_is_cont_byte(byte)).count()
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        let len = self.iter.len();
        //最少按四个字节一个字符，最多按一个字节一个字符
        ((len + 3) / 4, Some(len))
    }

    fn last(mut self) -> Option<char> {
        self.next_back()
    }

}

pub fn next_code_point<'a, I: Iterator<Item = &'a u8>>(bytes: &mut I) -> Option<u32> {
    // iterator.next 
    let x = *bytes.next()?;
    if x < 128 {
        //ascii字符
        return Some(x as u32);
    }

    //因为是字符串，此时第二个字节一定会有
    let init = utf8_first_byte(x, 2); 
    //获取下一个字节，一定存在   
    let y = unwrap_or_0(bytes.next());
    let mut ch = utf8_acc_cont_byte(init, y);
    if x >= 0xE0 {
        // 三个字节的UTF-8
        let z = unwrap_or_0(bytes.next());
        let y_z = utf8_acc_cont_byte((y & CONT_MASK) as u32, z);
        ch = init << 12 | y_z;
        if x >= 0xF0 {
            //四个字节的UTF-8
            let w = unwrap_or_0(bytes.next());
            ch = (init & 7) << 18 | utf8_acc_cont_byte(y_z, w);
        }
    }

    Some(ch)
}
```
&str的Iterator实现是一个说明Iterator设计模式优越性的经典实例。如果直接使用循环，则&str与&[T]必然会有很多的重复代码，使用Iterator模式后，重复代码被抽象到了Iterator模块中。&str复用了&[T]的iter。

## array 的Iterator实现

### Unsize Trait
```rust
pub trait Unsize<T: ?Sized> {
    // Empty.
}
```
实现了Unsize Trait，可以把一个固定内存大小的变量强制转换为相关的可变大小类型， 如[T;N]实现了Unsize<[T]>, 因此[T;N]可以转换为[T]，一般是指针转换。

### Iter所用的结构
```rust
pub struct IntoIter<T, const N: usize> {
    /// data是迭代中的数组.
    /// 这个数组中，只有data[alive]是有效的，访问其他的部分，即data[..alive.start]
    /// 及data[end..]会发生UB
    /// [MaybeUninit<T>;N]的用法需要体会，
    data: [MaybeUninit<T>; N],

    /// 表明数组中有效的成员的下标范围.
    /// 必须满足:
    /// - `alive.start <= alive.end`
    /// - `alive.end <= N`
    alive: Range<usize>,
}
```
上面这个结构是因为需要对array内成员做消费设计的。因为数组成员不支持所有权转移，所以采用了这种设计方式。数组的Iterator实现是理解所有权的一个极佳例子。

### into_iter实现
```rust
impl<T, const N: usize> IntoIter<T, N> {
    pub fn new(array: [T; N]) -> Self {
        // 
        // 因为RUST特性目前还不支持数组的transmute，所以用了内存跨类型的transmute_copy，此函数将从栈中申请一块内存。
        // 拷贝完毕后，原数组的所有权已经转移到data，data内数据事实上已经初始化，但仍然还是MaybeUninit<T>的类型。此时，需要对原数组调用mem::forget反应所有权已经失去。
        // mem::forget不会导致内存泄漏。
        unsafe {
            let iter = Self { data: mem::transmute_copy(&array), alive: 0..N };
            mem::forget(array);
            iter
        }
    }

    pub fn as_slice(&self) -> &[T] {
        // 仅针对有效的部分返回切片引用。已经消费的不返回。
        unsafe {
            //此处调用SliceIndex::<Range>::get_unchecked
            //slice是&[MaybeUninit<T>]类型
            let slice = self.data.get_unchecked(self.alive.clone());
            MaybeUninit::slice_assume_init_ref(slice)
        }
    }

    pub fn as_mut_slice(&mut self) -> &mut [T] {
        unsafe {
            //此处调用SliceIndex::<Range>::get_unchecked_mut
            //slice 是 & mut [MaybeUninit<T>]类型
            let slice = self.data.get_unchecked_mut(self.alive.clone());
            MaybeUninit::slice_assume_init_mut(slice)
        }
    }
}

impl<T, const N: usize> Iterator for IntoIter<T, N> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        // 下面使用Range的Iterator特性实现next. alive的start会变化，从而导致start之前的数组元素无法再被访问。因为已经被消费掉。
        // Option::map完成下标值传递。
        self.alive.next().map(|idx| {
            // SliceIndex::<usize>::get_unchecked, MaybeUninit::<T>::assume_init_read()
            // 前面有过说明，assume_init_read()从堆栈中申请了T大小的内存，然后进行内存拷贝，然后返回变量
            // 此时array元素的所有权转移到返回值。
            unsafe { self.data.get_unchecked(idx).assume_init_read() }
        })
    }
    ...
    ...
}

impl<T, const N: usize> Drop for IntoIter<T, N> {
    // 这里没有被消费掉的成员必须显示释放掉。
    fn drop(&mut self) {
        // as_mut_slice()获得所有具有所有权的元素，这些元素需要调用drop来释放。这里，data变量中的元素始终封装在MaybeUninit<T>中
        unsafe { ptr::drop_in_place(self.as_mut_slice()) }
    }
}

```
数组的Iterator最关键的点就是如何将数组成员的所有权取出，这是RUST语法带来的额外的麻烦和复杂性。最终的解决办法显示了RUST编码的所有权转移的一些通用的底层技巧。

```rust
impl<T, const N: usize> IntoIterator for [T; N] {
    type Item = T;
    type IntoIter = IntoIter<T, N>;

    /// 创建消费型的iterator, 如果T不实现`Copy`, 则调用此函数后，数组不可再被访问。
    fn into_iter(self) -> Self::IntoIter {
        IntoIter::new(self)
    }
}
```
以上创建消费数组成员的Iterator。

### iter(), iter_mut()实现
下面的数组成员引用的Iterator实质上是将数组强制转换为切片类型，应用切片类型的迭代器。
```rust
impl<'a, T, const N: usize> IntoIterator for &'a [T; N] {
    type Item = &'a T;
    type IntoIter = Iter<'a, T>;
    
    fn into_iter(self) -> Iter<'a, T> {
        //点号导致self强制转换成[T], 然后调用切片类型的iter
        self.iter()
    }
}

impl<'a, T, const N: usize> IntoIterator for &'a mut [T; N] {
    type Item = &'a mut T;
    type IntoIter = IterMut<'a, T>;
    
    
    fn into_iter(self) -> IterMut<'a, T> {
        //self被强制转换为切片类型
        self.iter_mut()
    }
}
```

## Iterator的适配器代码分析

### Map 适配器代码分析

Map相关代码如下：
```rust

pub trait Iterator {
    //其他内容
    ...
    ...

    //创建map Iterator
    fn map<B, F>(self, f: F) -> Map<Self, F>
    where
        Self: Sized,
        F: FnMut(Self::Item) -> B,
    {
        Map::new(self, f)
    }
    ...
}

//此结构是一个adapter的结构
pub struct Map<I, F> {
    // Map的底层Iterator
    pub(crate) iter: I,
    // Map操作闭包函数
    f: F,
}

impl<I, F> Map<I, F> {
    //由Iterator::map 函数和这个函数可以理解Iterator的lazy特性，
    //Iterator的创建实际上仅仅建立了数据结构，直到next才有操作。
    pub(in crate::iter) fn new(iter: I, f: F) -> Map<I, F> {
        Map { iter, f }
    }
}
```
Map适配器结构相当直接而简单。
```rust
//针对Map实现Iterator
impl<B, I: Iterator, F> Iterator for Map<I, F>
where
    F: FnMut(I::Item) -> B,
{
    type Item = B;

    fn next(&mut self) -> Option<B> {
        //利用底层Iterator的next，Option::map实现next
        self.iter.next().map(&mut self.f)
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        self.iter.size_hint()
    }

    //其他函数，其实现技巧与next类似
    ...
    ...
}
```
### Chain 适配器代码分析
相关代码如下：
```rust
pub trait Iterator {
    ...
    ...
    //创建Chain Iterator
    fn chain<U>(self, other: U) -> Chain<Self, U::IntoIter>
    where
        Self: Sized,
        U: IntoIterator<Item = Self::Item>,
    {
        Chain::new(self, other.into_iter())
    }
    ...
    ...
}


pub struct Chain<A, B> {
    //迭代器A
    a: Option<A>,
    //迭代器B
    b: Option<B>,
}
impl<A, B> Chain<A, B> {
    pub(in super::super) fn new(a: A, b: B) -> Chain<A, B> {
        Chain { a: Some(a), b: Some(b) }
    }
}

macro_rules! fuse {
    ($self:ident . $iter:ident . $($call:tt)+) => {
        //$iter可能已经被置为None
        match $self.$iter {
            //若$iter不为None,则调用iter的系列函数
            Some(ref mut iter) => match iter.$($call)+ {
                //函数返回None
                None => {
                    //设置$iter为None,并返回None
                    $self.$iter = None;
                    None
                }
                //其他返回函数返回值
                item => item,
            },
            //a为None时返回None
            None => None,
        }
    };
}

//与fuse类似，略
macro_rules! maybe {
    ($self:ident . $iter:ident . $($call:tt)+) => {
        match $self.$iter {
            Some(ref mut iter) => iter.$($call)+,
            None => None,
        }
    };
}

impl<A, B> Iterator for Chain<A, B>
where
    A: Iterator,
    B: Iterator<Item = A::Item>,
{
    type Item = A::Item;

    fn next(&mut self) -> Option<A::Item> {
        //先执行self.a.next
        match fuse!(self.a.next()) {
            //若self.a.next返回None，则执行self.b.next
            None => maybe!(self.b.next()),
            //不为None，返回a的返回值
            item => item,
        }
    }
    ...
    ...
}

```
### 其他
Iterator的adapter还有很多，如StedBy, Filter, Zip, Intersperse等等。具体请参考标准库手册。基本上所有的adapter都是遵循Adapter的设计模式来实现的。且每一个适配器的结构及代码逻辑都是比较简单且易理解的。
### 小结
RUST的Iterater的adapter是突出的体现RUST的语法优越性的特性，借助Trait和强大的泛型机制，与c/c++/java相比较，RUST以很少的代码在标准库就实现了最丰富的adapter。而其他语言标准库往往不存在这些适配器，需要其他库来实现。
Iterator的adapter实现了强大的基于Iterator的函数式编程基础设施。函数式编程的基础框架之一便是基于Iterator和闭包实现丰富的adapter。这也凸显了RUST在语言级别对函数式编程的良好支持。

## Option的Iterator实现代码分析
Option实现Iterator是比较令人疑惑的，毕竟用Iterator肯定代码更多，逻辑也复杂。主要目的应该是为了重用Iterator构建的各种adapter，及为了函数式编程的需要。仅分析IntoIterator Trait所涉及的结构及方法
相关类型结构定义：
```rust
//into_iter的结构
pub struct IntoIter<A> {
    //实际的Iterator实现结构
    inner: Item<A>,
}

//Item同时满足into_iter(), iter(), iter_mut()
//标准库编码者的设计方式，当然也可以用其他设计
struct Item<A> {
    opt: Option<A>,
}

impl<T> IntoIterator for Option<T> {
    type Item = T;
    type IntoIter = IntoIter<T>;

    //创建Iterator的实现结构体，self所有权传入结构体
    fn into_iter(self) -> IntoIter<T> {
        IntoIter { inner: Item { opt: self } }
    }
}

//具体实现者
impl<A> Iterator for Item<A> {
    type Item = A;

    fn next(&mut self) -> Option<A> {
        //所有权传出，并用None替换原变量的值
        self.opt.take()
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        match self.opt {
            Some(_) => (1, Some(1)),
            None => (0, Some(0)),
        }
    }
}

//消费变量的Iterator实现
impl<A> Iterator for IntoIter<A> {
    type Item = A;

    fn next(&mut self) -> Option<A> {
        self.inner.next()
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        self.inner.size_hint()
    }
}
```   

Result<T,E>的 Iterator与Option<T>的Iterator非常相似，略

# RUST基本类型代码分析(二)
## 整形类型标准库代码分析
### NonZero数据类型
`NonZeroU8, NonZeroU16，NonZeroU32, NonZeroU64, NonZeroU128, NonZeroUsize`
`NonZeroI8, NonZeroI16, NonZeroI32, NonZeroI64, NonZeroI128, NonZeroIsize`
以上为NonZero的类型，内存结构与相应的整形数据完全相同，可以转换。上文提过，当需要0表示特殊含义时，使用NonZero类型以保证代码安全。
重要函数：
```rust
//利用宏简化定义代码
macro_rules! nonzero_integers {
    ( $($Ty: ident($Int: ty); )+ ) => {
        $(
            #[derive(Copy, Clone, Eq, PartialEq, Ord, PartialOrd, Hash)]
            #[repr(transparent)]
            pub struct $Ty($Int);

            impl $Ty {
                pub const unsafe fn new_unchecked(n: $Int) -> Self {
                    unsafe { Self(n) }
                }

                pub const fn new(n: $Int) -> Option<Self> {
                    if n != 0 {
                        Some(unsafe { Self(n) })
                    } else {
                        None
                    }
                }

                pub const fn get(self) -> $Int {
                    self.0
                }

            }

            impl const From<$Ty> for $Int {
                fn from(nonzero: $Ty) -> Self {
                    nonzero.0
                }
            }
            //本类型和本类型"|"运算符重载
            impl const BitOr for $Ty {
                type Output = Self;
                fn bitor(self, rhs: Self) -> Self::Output {
                    unsafe { $Ty::new_unchecked(self.get() | rhs.get()) }
                }
            }
            //本类型与基础类型的"|"运算符重载
            impl const BitOr<$Int> for $Ty {
                type Output = Self;
                fn bitor(self, rhs: $Int) -> Self::Output {
                    unsafe { $Ty::new_unchecked(self.get() | rhs) }
                }
            }
            //基础类型与本类型的"|"运算符重载
            impl const BitOr<$Ty> for $Int {
                type Output = $Ty;
                fn bitor(self, rhs: $Ty) -> Self::Output {
                    unsafe { $Ty::new_unchecked(self | rhs.get()) }
                }
            }
            //"|="运算符重载
            impl const BitOrAssign for $Ty {
                fn bitor_assign(&mut self, rhs: Self) {
                    *self = *self | rhs;
                }
            }

            impl const BitOrAssign<$Int> for $Ty {
                fn bitor_assign(&mut self, rhs: $Int) {
                    *self = *self | rhs;
                }
            }

            //其他运算符的重载,略
            ...
            ...
        )+
    }
}

nonzero_integers! {
    NonZeroU8(u8);
    NonZeroU16(u16);
    NonZeroU32(u32);
    NonZeroU64(u64);
    NonZeroU128(u128);
    NonZeroUsize(usize);
    NonZeroI8(i8);
    NonZeroI16(i16);
    NonZeroI32(i32);
    NonZeroI64(i64);
    NonZeroI128(i128);
    NonZeroIsize(isize);
}
```
NonZero 类型典型的体现了RUST程序设计的安全原则，所有的异常应该用类型系统表示出来，以强制获得处理。不要用临时性的措施。这样可以最大限度的避免bug的产生。

### 整形数据ops数学运算符，位运算符重载实现代码分析
以Add为例说明：
```rust
Add<Rhs = Self> {
    type Output;

    fn add(self, rhs: Rhs) -> Self::Output;
}

//利用宏简化操作
macro_rules! add_impl {
    ($($t:ty)*) => ($(
        impl const Add for $t {
            type Output = $t;
            //"+"号编译器默认实现是uncheckd_add
            fn add(self, other: $t) -> $t { self + other }
        }

        forward_ref_binop! { impl const Add, add for $t, $t }
    )*)
}

//利用宏实现所有整形和浮点型运算符的重载
add_impl! { usize u8 u16 u32 u64 u128 isize i8 i16 i32 i64 i128 f32 f64 }
```
其他数学运算符及位运算符与此接近，因为代码逻辑简单，请参考标准库手册，略


## bool类型方法代码分析
```rust
    pub fn then<T, F: FnOnce() -> T>(self, f: F) -> Option<T> {
        if self { Some(f()) } else { None }
    }

    pub fn then<T, F: FnOnce() -> T>(self, f: F) -> Option<T> {
        if self { Some(f()) } else { None }
    }
```
配合ops::Try，以上函数主要是在语言级别更好的支持函数式编程

## RUST字符(char)类型标准库代码分析
RUST的字符标准库主要是编程中常用到的字符相关操作,所有标准库的函数都可以作为RUST的训练，本节摘录一些显示RUST编码特点的内容。

由字符串转换为字符类型：
见如下代码：
```rust
impl FromStr for char {
    type Err = ParseCharError;
    
    //因为字符串用utf-8编码，而char是4字节变量，所以从字符串获取字符类型
    //不是简单的字符数组取值的关系，
    fn from_str(s: &str) -> Result<Self, Self::Err> {
        //s.chars()请参考前文
        let mut chars = s.chars();
        //下面对字符串做判断，字符串中应该只有一个字符存在，具体完成utf-8的字符到
        //char的转换在chars.next()中完成, 请参考前文
        match (chars.next(), chars.next()) {
            (None, _) => Err(ParseCharError { kind: CharErrorKind::EmptyString }),
            (Some(c), None) => Ok(c),
            _ => Err(ParseCharError { kind: CharErrorKind::TooManyChars }),
        }
    }
}
```
str::chars()函数请见前文<a href="#str_iter">字符串Iterator代码分析</a>
u32转换为char,代码如下：
```rust
impl TryFrom<u32> for char {
    type Error = CharTryFromError;

    fn try_from(i: u32) -> Result<Self, Self::Error> {
        if (i > MAX as u32) || (i >= 0xD800 && i <= 0xDFFF) {
            Err(CharTryFromError(()))
        } else {
            // 无法用as，只能用tranmute暴力转换
            Ok(unsafe { transmute(i) })
        }
    }
}
```
从任一进制的数值转换为char,代码如下：
```rust
pub fn from_digit(num: u32, radix: u32) -> Option<char> {
    //不支持大于36进制的数
    if radix > 36 {
        panic!("from_digit: radix is too high (maximum 36)");
    }
    if num < radix {
        //转换为u8，后面可以与Byte类型做加法, b'0'是Byte类型的字面量
        let num = num as u8;
        if num < 10 { Some((b'0' + num) as char) } else { Some((b'a' + num - 10) as char) }
    } else {
        None
    }
}
```
将字符转换为某一进制的数值,以下例子充分的说明了RUST的安全性，相对于只有一种加法的C，RUST显著的降低了程序Bug出现的可能性
```rust
    pub fn to_digit(self, radix: u32) -> Option<u32> {
        assert!(radix <= 36, "to_digit: radix is too high (maximum 36)");
        // 利用wrapping_sub同时处理大于及小于'0'的字符，并且规避溢出
        let mut digit = (self as u32).wrapping_sub('0' as u32);
        if radix > 10 {
            if digit < 10 {
                return Some(digit);
            }
            // 用saturating_add保证digit不会折返
            digit = (self as u32 | 0b10_0000).wrapping_sub('a' as u32).saturating_add(10);
        }
        //利用bool类型的方法简化了编程
        (digit < radix).then_some(digit)
    }
```
将字符转换为"\u{xxxx}"的形式：
```rust
//escape_unicode充分的展示了函数式编程的设计思想
//即以迭代器为中心来设计问题解决方案，
//对于任何一个问题，首先就看是否能设计一个实现Iterator Trait的结构来解决问题
pub fn escape_unicode(self) -> EscapeUnicode {
   let c = self as u32;

    // c|1避免有32个0出现
    let msb = 31 - (c | 1).leading_zeros();

    // 计算有多少个字符
    let ms_hex_digit = msb / 4;
    //生成结构，以便用Iterator解决问题
    EscapeUnicode {
        c: self,
        state: EscapeUnicodeState::Backslash,
        hex_digit_idx: ms_hex_digit as usize,
    }
}

//为了
pub struct EscapeUnicode {
    c: char,
    state: EscapeUnicodeState,

    // 当前还有多少个字符没有转换 
    hex_digit_idx: usize,
}

// 显示转换的当前状态 
#[derive(Clone, Debug)]
enum EscapeUnicodeState {
    //转换完成
    Done,
    //下一步应输出右括号
    RightBrace,
    //下一步应输出字母
    Value,
    //下一步应输出左括号
    LeftBrace,
    //输出Type的字符
    Type,
    //输出斜杠，第一个状态
    Backslash,
}

impl Iterator for EscapeUnicode {
    type Item = char;

    fn next(&mut self) -> Option<char> {
        match self.state {
            EscapeUnicodeState::Backslash => {
                self.state = EscapeUnicodeState::Type;
                Some('\\')
            }
            EscapeUnicodeState::Type => {
                self.state = EscapeUnicodeState::LeftBrace;
                Some('u')
            }
            EscapeUnicodeState::LeftBrace => {
                self.state = EscapeUnicodeState::Value;
                Some('{')
            }
            EscapeUnicodeState::Value => {
                let hex_digit = ((self.c as u32) >> (self.hex_digit_idx * 4)) & 0xf;
                let c = from_digit(hex_digit, 16).unwrap();
                if self.hex_digit_idx == 0 {
                    self.state = EscapeUnicodeState::RightBrace;
                } else {
                    self.hex_digit_idx -= 1;
                }
                Some(c)
            }
            EscapeUnicodeState::RightBrace => {
                self.state = EscapeUnicodeState::Done;
                Some('}')
            }
            EscapeUnicodeState::Done => None,
        }
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        let n = self.len();
        (n, Some(n))
    }

    fn count(self) -> usize {
        self.len()
    }

    fn last(self) -> Option<char> {
        match self.state {
            EscapeUnicodeState::Done => None,

            EscapeUnicodeState::RightBrace
            | EscapeUnicodeState::Value
            | EscapeUnicodeState::LeftBrace
            | EscapeUnicodeState::Type
            | EscapeUnicodeState::Backslash => Some('}'),
        }
    }
}

impl fmt::Display for EscapeUnicode {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        //利用Iterator输出转换字符串 
        for c in self.clone() {
            f.write_char(c)?;
        }
        Ok(())
    }
}
```
EscapeUnicode 实现了Display Trait。可以调用to_string来输出字符串
RUST的字符模块的其他转换函数与EscapeUnicode采用了类似的设计，下面列出这些转换函数，但代码分析省略
`pub fn escape_debug(self) -> EscapeDebug` char的Debug转换输出
`pub fn to_lowercase(self) -> ToLowercase` char转换为小写
`pub fn to_uppercase(self) -> ToUppercase` char转换为大写

编码为UTF-8的字符串
```rust
    //dst应该保证有足够的空间放置utf-8字符串，&mut str的地址就是dst
    pub fn encode_utf8(self, dst: &mut [u8]) -> &mut str {
        unsafe { from_utf8_unchecked_mut(encode_utf8_raw(self as u32, dst)) }
    }

    pub unsafe fn from_utf8_unchecked_mut(v: &mut [u8]) -> &mut str {
        //调用者保证v能被安全的转换
        unsafe { &mut *(v as *mut [u8] as *mut str) 
    }

    pub fn encode_utf8_raw(code: u32, dst: &mut [u8]) -> &mut [u8] {
        let len = len_utf8(code);
        match (len, &mut dst[..]) {
            //rust语法的强大展现，逻辑很简单，分析略
            (1, [a, ..]) => {
                *a = code as u8;
            }
            (2, [a, b, ..]) => {
                *a = (code >> 6 & 0x1F) as u8 | TAG_TWO_B;
                *b = (code & 0x3F) as u8 | TAG_CONT;
            }
            (3, [a, b, c, ..]) => {
                *a = (code >> 12 & 0x0F) as u8 | TAG_THREE_B;
                *b = (code >> 6 & 0x3F) as u8 | TAG_CONT;
                *c = (code & 0x3F) as u8 | TAG_CONT;
            }
            (4, [a, b, c, d, ..]) => {
                *a = (code >> 18 & 0x07) as u8 | TAG_FOUR_B;
                *b = (code >> 12 & 0x3F) as u8 | TAG_CONT;
                *c = (code >> 6 & 0x3F) as u8 | TAG_CONT;
                *d = (code & 0x3F) as u8 | TAG_CONT;
            }
            _ => panic!(
                "encode_utf8: need {} bytes to encode U+{:X}, but the buffer has {}",
                len,
                code,
                dst.len(),
            ),
        };
        &mut dst[..len]
    }
}
```

## 字符串标准库代码分析
字符串模块的一个核心是Iterator，已经在Iterator章节中有过说明。
除了Iterator，字符串其他的方法及函数库代码摘要分析如下：
```rust
    pub const fn len(&self) -> usize {
        //字符串的len是字符串字节数目
        self.as_bytes().len()
    }
    //是否是字符的边界 
    pub fn is_char_boundary(&self, index: usize) -> bool {
        // 0 位置总是边界
        if index == 0 {
            return true;
        }

        match self.as_bytes().get(index) {
            
            None => index == self.len(),

            // 巧妙的对字符边界的总结: b < 128 || b >= 192
            Some(&b) => (b as i8) >= -0x40,
        }
    }
    //目前I的类型仅支持：
    // usize, ..(RangeFull), start..(RangeFrom), start..end(Range)
    // start..=end(RangeInclusive), ..end(RangeTo), ..=end(RangeToInclusive)  
    // get函数不会panic,但更习惯用str[usize],或者str[Range]来完成 
    pub fn get<I: SliceIndex<str>>(&self, i: I) -> Option<&I::Output> {
        i.get(self)
    }
    //对i.get给出一个分析
    unsafe impl SliceIndex<str> for ops::Range<usize> {
        type Output = str;
        
        fn get(self, slice: &str) -> Option<&Self::Output> {
            //必须满足Range的两端都在字符边界处，否则返回None
            if self.start <= self.end
                && slice.is_char_boundary(self.start)
                && slice.is_char_boundary(self.end)
            {
                // 重新建立了一个&[str],具体见下面的函数
                Some(unsafe { &*self.get_unchecked(slice) })
            } else {
                None
            }
        }
        //最终离不开内存和裸指针
        unsafe fn get_unchecked(self, slice: *const str) -> *const Self::Output {
            let slice = slice as *const [u8];
            let ptr = unsafe { slice.as_ptr().add(self.start) };
            let len = self.end - self.start;
            ptr::slice_from_raw_parts(ptr, len) as *const str
        }
        ...
    }
    
    //其他可以用Index实现的get_xxx函数及split_at函数，略
    ...
```
下面通过字符串的查找函数给出RUST良好的程序结构设计的一个例子：
```rust
    //字符串查找函数，可以用模式匹配查找子串
    //支持如下例子中的查找    
    /// let s = "Löwe 老虎 Léopard Gepardi";
    /// 字符的查找
    /// assert_eq!(s.find('L'), Some(0));
    /// assert_eq!(s.find('é'), Some(14));
    /// 
    /// 子字符串的查找
    /// assert_eq!(s.find("pard"), Some(17));
    /// 
    /// 满足函数要求的字符或字符串的查找
    /// assert_eq!(s.find(char::is_lowercase), Some(1));
    /// assert_eq!(s.find(|c: char| c.is_whitespace() || c.is_lowercase()), Some(1));
    /// assert_eq!(s.find(|c: char| (c < 'o') && (c > 'a')), Some(4));
    /// 
    /// 字符数组的查找，注意RUST中字符数组与字符串是不同的两个类型
    /// assert_eq!(s.find(['老', 'G']))
```
由以上注释可以看到，rust的字符串查找函数功能强大，使用直观且易于理解。后继代码将展现RUST具备的：
1. 良好的扩展性，即使是原生类型，也可以直接在其上增加自定义Trait, 从而得到最直观的代码表现，而其他语言如C++/Java是无法在已经定义好的类型上做扩充的。只能创建新类型来实现对已有类型的功能扩展。不但在代码上不直观及冗余，也造成了额外的学习负担。
2. Trait语义的强大，即使对于闭包类型，也可以实现Trait。
   
```rust
    pub fn find<'a, P: Pattern<'a>>(&'a self, pat: P) -> Option<usize> {
        //利用Pattern Trait支持了众多类型的查找
        pat.into_searcher(self).next_match().map(|(i, _)| i)
    }
```
要设计这样一个find方法:
1. 显然，参数需要是一个泛型，但泛型应该支持同样的接口，即Pattern trait
2. 需要利用find的输入泛型参数，self来构造一个结构，并以这个结构为基础来实现方法完成查找。Pattern trait 的类型显然不可能作为这个结构(字符，字符切片，字符数组，闭包函数，字符串). 这个结构只能由Pattern trait的方法构造，事实上，Pattern trait最重要的工作就是构造这个结构。
3. 2构造的结构应该支持统一的接口，真正的实现查找

具体的实现定义如下：
```rust    
    //模式 Trait 定义及公共行为
    pub trait Pattern<'a>: Sized {
        /// 与具体类型相适配的搜索算法的实现类型，类型必须实现Searcher Trait
        type Searcher: Searcher<'a>;

        /// 创建Searcher，根据输入的str及类型自身属性
        fn into_searcher(self, haystack: &'a str) -> Self::Searcher;

        /// 检查str是否存在对模式匹配的内容
        fn is_contained_in(self, haystack: &'a str) -> bool {
            self.into_searcher(haystack).next_match().is_some()
        }

        //略
        ...
    }
```
以下为Searcher trait定义。
```rust    
    //Pattern匹配搜索算法的具体实现Trait
    pub unsafe trait Searcher<'a> {
        /// Searcher针对的字符串
        fn haystack(&self) -> &'a str;

        /// 执行下一次搜索， 返回搜索算法给出的:
        ///   [SearchStep::Match(a,b)] haystack[a..b]匹配了模式
        ///   [SearchStep::Reject(a,b)] haystack[a..b]不能匹配模式
        ///   [SearchStep::Done]
        /// next的返回结果应该上次放回的结果首尾相连。即如果上次返回Match(0,1), next的返回
        /// 应该是Reject(1,_)或Match(1,_)。第一个返回必须是Reject(0,_)或match(0,_), Done之前
        /// 的返回应该是Reject(_, haystack.len()-1)或Match(_, haystack.len())
        fn next(&mut self) -> SearchStep;

        /// 找到下一个匹配结果是Match的匹配结果
        fn next_match(&mut self) -> Option<(usize, usize)> {
            loop {
                match self.next() {
                    SearchStep::Match(a, b) => return Some((a, b)),
                    SearchStep::Done => return None,
                    _ => continue,
                }
            }
        }

        /// 找到下一个Reject
        fn next_reject(&mut self) -> Option<(usize, usize)> {
            loop {
                match self.next() {
                    SearchStep::Reject(a, b) => return Some((a, b)),
                    SearchStep::Done => return None,
                    _ => continue,
                }
            }
        }
   }

    pub enum SearchStep {
        /// 匹配时输出Match及子字符串的位置
        Match(usize, usize),
        /// 确定不匹配的子字符串的位置信息, 可以有多个不匹配的子字符串
        Reject(usize, usize),
        /// 字符串已经遍历完毕
        Done,
    }
```
下面为单字符的Pattern trait的系列实现，仅展示一下相应的逻辑关系。
```rust
    //针对char类型的Searcher Trait具现化类型
    pub struct CharSearcher<'a> { /*略*/ }
    //实现Searcher Trait
    unsafe impl<'a> Searcher<'a> for CharSearcher<'a> {
        //略
        ...
    }

    // 针对char 的Pattern实现, 支持如 "abc".find('a') 的形态
    impl<`a, `b> Pattern<`a> for char { 
        type Searcher = CharSearcher
        
        //略
        ...
    }
```
下面为多字符的Pattern trait的实现，因为是比较典型的设计，所以重点的进行分析：
首先，设计字符匹配的trait，并在闭包，字符数组及其引用，字符切片类型中实现
```rust
    //支持  "abc".find(&['a','b'])的形态
    //      "abc".find(&['a','b'][..]) 的形态 &['a','b'][..] 实质是&[char]类型，注意与&str类型的区别
    //      "abc".find(|ch| ch > 'a' && ch < 'c') 的形态
    
    //利用MultiCharEq trait 综合[char; N], &[char], FnMut(char)->bool 
    //字符匹配操作
    trait MultiCharEq {
        fn matches(&mut self, c: char) -> bool;
    }
    
    //为FnMut(char)->bool 实现MultiCharEq
    impl<F> MultiCharEq for F
    where
        F: FnMut(char) -> bool,
    {
        fn matches(&mut self, c: char) -> bool {
            (*self)(c)
        }
    }
    
    //为[char;N]实现 MultiCharEq
    impl<const N: usize> MultiCharEq for [char; N] {
        fn matches(&mut self, c: char) -> bool {
            self.iter().any(|&m| m == c)
        }
    }

    impl<const N: usize> MultiCharEq for &[char; N] {
        fn matches(&mut self, c: char) -> bool {
            self.iter().any(|&m| m == c)
        }
    }
    
    // 为&[char]实现MultiCharEq
    impl MultiCharEq for &[char] {
        #[inline]
        fn matches(&mut self, c: char) -> bool {
            self.iter().any(|&m| m == c)
        }
    }
 ```
然后是基于泛型的统一的Pattern trait和Searcher的实现
 ```rust   
    //利用输入类型构造一个泛型结构
    struct MultiCharEqPattern<C: MultiCharEq>(C);

    //与MultiCharEqPattern相匹配的Searcher Trait具现的结构体
    struct MultiCharEqSearcher<'a, C: MultiCharEq> {
        char_eq: C,
        haystack: &'a str,
        char_indices: super::CharIndices<'a>,
    }
    
    // 实现Pattern
    impl<'a, C: MultiCharEq> Pattern<'a> for MultiCharEqPattern<C> {
        type Searcher = MultiCharEqSearcher<'a, C>;
        
        //创建泛型Searcher结构
        fn into_searcher(self, haystack: &'a str) -> MultiCharEqSearcher<'a, C> {
            MultiCharEqSearcher { haystack, char_eq: self.0, char_indices: haystack.char_indices()}
        }
    }

    //针对泛型Searcher结构实现Searcher trait
    unsafe impl<'a, C: MultiCharEq> Searcher<'a> for MultiCharEqSearcher<'a, C> {
        fn haystack(&self) -> &'a str {
            self.haystack
        }

        fn next(&mut self) -> SearchStep {
            let s = &mut self.char_indices;
            //pre_len用来计算char在字符串中占用了几个字节
            let pre_len = s.iter.iter.len();
            if let Some((i, c)) = s.next() {
                let len = s.iter.iter.len();
                //计算当前字符占用的字节数
                let char_len = pre_len - len;
                if self.char_eq.matches(c) {
                    return SearchStep::Match(i, i + char_len);
                } else {
                    return SearchStep::Reject(i, i + char_len);
                }
            }
            SearchStep::Done
        }
    }
```
下面是如何将MultiCharEqPattern及MultiCharEqSearcher应用在各类型的Pattern实现中。
```rust

    /////////////////////////////////////////////////////////////////////////////
    //利用宏简化代码
    macro_rules! pattern_methods {
        ($t:ty, $pmap:expr, $smap:expr) => {
            type Searcher = $t;

            fn into_searcher(self, haystack: &'a str) -> $t {
                //这里实际上是用self创建了MultiCharEqPattern(self)
                //随后用MutiEqPattern(self)创建MultiCharEqSearcher
                //然后封装MultiCharEqSearcher，创建一个与self类型关联的Searcher类型的Searcher
                ($smap)(($pmap)(self).into_searcher(haystack))
            }

            fn is_contained_in(self, haystack: &'a str) -> bool {
                ($pmap)(self).is_contained_in(haystack)
            }

            fn is_prefix_of(self, haystack: &'a str) -> bool {
                ($pmap)(self).is_prefix_of(haystack)
            }

            fn strip_prefix_of(self, haystack: &'a str) -> Option<&'a str> {
                ($pmap)(self).strip_prefix_of(haystack)
            }

            fn is_suffix_of(self, haystack: &'a str) -> bool
            where
                $t: ReverseSearcher<'a>,
            {
                ($pmap)(self).is_suffix_of(haystack)
            }

            fn strip_suffix_of(self, haystack: &'a str) -> Option<&'a str>
            where
                $t: ReverseSearcher<'a>,
            {
                ($pmap)(self).strip_suffix_of(haystack)
            }
        };
    }

    // 利用宏简化代码
    macro_rules! searcher_methods {
        (forward) => {
            fn haystack(&self) -> &'a str {
                self.0.haystack()
            }
            fn next(&mut self) -> SearchStep {
                //实质是MultiCharEqSearcher<>::next
                self.0.next()
            }
            fn next_match(&mut self) -> Option<(usize, usize)> {
                self.0.next_match()
            }
            fn next_reject(&mut self) -> Option<(usize, usize)> {
                self.0.next_reject()
            }
        };
        (reverse) => {
            fn next_back(&mut self) -> SearchStep {
                self.0.next_back()
            }
            fn next_match_back(&mut self) -> Option<(usize, usize)> {
                self.0.next_match_back()
            }
            fn next_reject_back(&mut self) -> Option<(usize, usize)> {
                self.0.next_reject_back()
            }
        };
    }

    //下面这个结构比较清晰的说明了 Pattern, MultiCharEqPattern, MultiCharEqSearcher的关系
    //使得代码更清晰
    pub struct CharArraySearcher<'a, const N: usize>(
        <MultiCharEqPattern<[char; N]> as Pattern<'a>>::Searcher,
    );

    /// 针对&[char;N]的Pattern, MultiCharEqPattern, MultiCharEqSearcher的关系
    pub struct CharArrayRefSearcher<'a, 'b, const N: usize>(
        <MultiCharEqPattern<&'b [char; N]> as Pattern<'a>>::Searcher,
    );
    
    // 利用上面的宏对[char;N]类型的Pattern Trait实现
    impl<'a, const N: usize> Pattern<'a> for [char; N] {
        pattern_methods!(CharArraySearcher<'a, N>, MultiCharEqPattern, CharArraySearcher);
    }
    // 对[char;N]的searcher关联类型的Searcher Trait实现，
    unsafe impl<'a, const N: usize> Searcher<'a> for CharArraySearcher<'a, N> {
        searcher_methods!(forward);
    }

    unsafe impl<'a, const N: usize> ReverseSearcher<'a> for CharArraySearcher<'a, N> {
        searcher_methods!(reverse);
    }

    // 针对&[char;N]的Pattern Trait 实现
    impl<'a, 'b, const N: usize> Pattern<'a> for &'b [char; N] {
        pattern_methods!(CharArrayRefSearcher<'a, 'b, N>, MultiCharEqPattern, CharArrayRefSearcher);
    }

    // 对&[char;N]的searcher关联类型的Searcher Trait 实现
    unsafe impl<'a, 'b, const N: usize> Searcher<'a> for CharArrayRefSearcher<'a, 'b, N> {
        searcher_methods!(forward);
    }

    unsafe impl<'a, 'b, const N: usize> ReverseSearcher<'a> for CharArrayRefSearcher<'a, 'b, N> {
        searcher_methods!(reverse);
    }

    //针对&[char]的Searcher具现化结构体
    pub struct CharSliceSearcher<'a, 'b>(<MultiCharEqPattern<&'b [char]> as Pattern<'a>>::Searcher);

    //Searcher Trait 实现
    unsafe impl<'a, 'b> Searcher<'a> for CharSliceSearcher<'a, 'b> {
        searcher_methods!(forward);
    }

    unsafe impl<'a, 'b> ReverseSearcher<'a> for CharSliceSearcher<'a, 'b> {
        searcher_methods!(reverse);
    }

    impl<'a, 'b> DoubleEndedSearcher<'a> for CharSliceSearcher<'a, 'b> {}

    // 对&[char]的Pattern Trait的实现
    impl<'a, 'b> Pattern<'a> for &'b [char] {
        pattern_methods!(CharSliceSearcher<'a, 'b>, MultiCharEqPattern, CharSliceSearcher);
    }

    //针对FnMut(char)->bool的Searcher具现化结构体
    pub struct CharPredicateSearcher<'a, F>(<MultiCharEqPattern<F> as Pattern<'a>>::Searcher)
    where
        F: FnMut(char) -> bool;

    //Searcher Trait 实现
    unsafe impl<'a, F> Searcher<'a> for CharPredicateSearcher<'a, F>
    where
        F: FnMut(char) -> bool,
    {
        searcher_methods!(forward);
    }

    unsafe impl<'a, F> ReverseSearcher<'a> for CharPredicateSearcher<'a, F>
    where
        F: FnMut(char) -> bool,
    {
        searcher_methods!(reverse);
    }

    impl<'a, F> DoubleEndedSearcher<'a> for CharPredicateSearcher<'a, F> where F: FnMut(char) -> bool {}

    //针对FnMut(char)->bool的Pattern Trait 实现
    impl<'a, F> Pattern<'a> for F
    where
        F: FnMut(char) -> bool,
    {
        pattern_methods!(CharPredicateSearcher<'a, F>, MultiCharEqPattern, CharPredicateSearcher);
    }
```
多字符搜索代码不复杂，但结构设计则可圈可点。而且似乎是不得不这样做设计。RUST利用泛型及trait能够自然的得到比较好的设计结果。
我们针对泛型做一个方法时，自然会对泛型用一个共用的trait——Pattern来约束。因为方法实现需要不同于泛型但紧密关联的另一个结构体，那这个结构体类型便自然的形成trait里的一个关联类型Searcher。而这个关联类型也自然应该用另一个trait——Searcher来约束。
Searcher的变量应该在Pattern里面生成出来。Searcher trait应该提供查找的方法。
这就是RUST语法自然导致好的设计的一个例子。

以下对子字符串搜索给出一些详细的解释，主要说明TwoWay算法
```rust
 
    //针对str实现的pattern， 支持如"abc".find("ab")的形态
    impl<'a, 'b> Pattern<'a> for &'b str {
        //StrSeacher见下面该结构的代码注释
        type Searcher = StrSearcher<'a, 'b>;

        fn into_searcher(self, haystack: &'a str) -> StrSearcher<'a, 'b> {
            StrSearcher::new(haystack, self)
        }
        
        //略
    }

    pub struct StrSearcher<'a, 'b> {
        // 被查找目标字符串
        haystack: &'a str,
        // 查找的子字符串
        needle: &'b str,
        // 查找算法实现体
        searcher: StrSearcherImpl,
    }

    enum StrSearcherImpl {
        //两种搜索算法，后继还可以根据需要再扩充其他的算法
        Empty(EmptyNeedle),
        TwoWay(TwoWaySearcher),
    }

   
    impl<'a, 'b> StrSearcher<'a, 'b> {
        fn new(haystack: &'a str, needle: &'b str) -> StrSearcher<'a, 'b> {
            if needle.is_empty() {
                //略
                ...
                ...
            } else {
                StrSearcher {
                    haystack,
                    needle,
                    searcher: StrSearcherImpl::TwoWay(TwoWaySearcher::new(
                        needle.as_bytes(),
                        haystack.len(),
                    )),
                }
            }
        }
    }

    unsafe impl<'a, 'b> Searcher<'a> for StrSearcher<'a, 'b> {
        fn haystack(&self) -> &'a str {
            self.haystack
        }

        fn next(&mut self) -> SearchStep {
            //此处隐藏StrSearcher后继不会更换算法。如果更换搜索算法，应该将StrSearcher整体做替换
            //
            match self.searcher {
                StrSearcherImpl::Empty(ref mut searcher) => {
                    //略
                    ...
                    ...
                }
                StrSearcherImpl::TwoWay(ref mut searcher) => {
                    if searcher.position == self.haystack.len() {
                        return SearchStep::Done;
                    }
                    let is_long = searcher.memory == usize::MAX;
                    match searcher.next::<RejectAndMatch>(
                        self.haystack.as_bytes(),
                        self.needle.as_bytes(),
                        is_long,
                    ) {
                        SearchStep::Reject(a, mut b) => {
                            // 因为searcher使用&[u8]来搜索，返回可能不是字节边界
                            while !self.haystack.is_char_boundary(b) {
                                b += 1;
                            }
                            searcher.position = cmp::max(b, searcher.position);
                            SearchStep::Reject(a, b)
                        }
                        //这个表示语法注意一下
                        otherwise => otherwise,
                    }
                }
            }
        }

        fn next_match(&mut self) -> Option<(usize, usize)> {
            match self.searcher {
                StrSearcherImpl::Empty(..) => loop {
                    //略
                    ...
                },
                StrSearcherImpl::TwoWay(ref mut searcher) => {
                    let is_long = searcher.memory == usize::MAX;
                    // 如果匹配，那匹配点一定是字符边界
                    if is_long {
                        searcher.next::<MatchOnly>(
                            self.haystack.as_bytes(),
                            self.needle.as_bytes(),
                            true,
                        )
                    } else {
                        searcher.next::<MatchOnly>(
                            self.haystack.as_bytes(),
                            self.needle.as_bytes(),
                            false,
                        )
                    }
                }
            }
        }
    }
    /*  查找子字符串算法的关键问题如下：
        1. 每次比较的匹配的位置在哪里？
        2. 不匹配时应移动多少个位置开始新一次匹配？

        显然，如果子字符串不存在周期性的重复，那每次比较如果不同就只能后移一个字符然后开始新的匹配
        所以，算法主要就是在子字符串中存在周期性重复的字符的情况下来如何更好的提高效率
        TwoWay算法仅在子字符串整体有周期性时发生左右，仅是内部少量字符的周期性，TwoWay算法不考虑。
        设待比较字符串为H, 子字符串为S, 周期字符串为w, 周期为p，w的前缀为w-则S为 w(w|w-)+
        对于S，TwoWay算法找到一个crit_pos, 先从S的crit_pos的位置开始与H做比较到S的尾部，如果对应字符位置crit_pos+i
        的比较不成功，会在S上偏移i，清除记录，然后继续比较。
        如果直到尾部比较都成功，则会记录，然后开始比较头部，如果头部比较不成功，则偏移p，后继比较会考虑比较成功的记录。
        
        TwoWay不是最快的算法，但占用内存少，且也在一定程度上提高了效率。是比较适合的库方法
        */
    struct TwoWaySearcher {
        // constants
        /// 每次比较的开始位置，从此位置向尾部
        crit_pos: usize,
        /// 每次反向比较的开始位置，从此位置向前部
        crit_pos_back: usize,
        // 周期
        period: usize,
        /// 子字符串的位图，用来做一个快速甄别和判断
        byteset: u64,

        // 在待比较字符串的位置,从头部向后查找
        position: usize,
        // 待比较字符串的位置，从尾部向前查找
        end: usize,
        /// 在尾部比较成功后，记录已经比较过的字符串
        memory: usize,
        /// 同上，不过是反方向比较
        memory_back: usize,
    }

    impl TwoWaySearcher {
        fn new(needle: &[u8], end: usize) -> TwoWaySearcher {
            let (crit_pos_false, period_false) = TwoWaySearcher::maximal_suffix(needle, false);
            let (crit_pos_true, period_true) = TwoWaySearcher::maximal_suffix(needle, true);
            
            //找到更偏向尾部的位置
            let (crit_pos, period) = if crit_pos_false > crit_pos_true {
                (crit_pos_false, period_false)
            } else {
                (crit_pos_true, period_true)
            };

            //这里可以看出，只有从头部开始的周期字符串获得支持
            if needle[..crit_pos] == needle[period..period + crit_pos] {
                let crit_pos_back = needle.len()
                    - cmp::max(
                        TwoWaySearcher::reverse_maximal_suffix(needle, period, false),
                        TwoWaySearcher::reverse_maximal_suffix(needle, period, true),
                    );

                TwoWaySearcher {
                    crit_pos,
                    crit_pos_back,
                    period,
                    byteset: Self::byteset_create(&needle[..period]),

                    position: 0,
                    end,
                    memory: 0,
                    memory_back: needle.len(),
                }
            } else {
                // 字符串内没有周期性，及仅具备局部周期的字符串

                TwoWaySearcher {
                    crit_pos,
                    crit_pos_back: crit_pos,
                    period: cmp::max(crit_pos, needle.len() - crit_pos) + 1,
                    byteset: Self::byteset_create(needle),

                    position: 0,
                    end,
                    memory: usize::MAX, // Dummy value to signify that the period is long
                    memory_back: usize::MAX,
                }
            }
        }

        fn byteset_create(bytes: &[u8]) -> u64 {
            bytes.iter().fold(0, |a, &b| (1 << (b & 0x3f)) | a)
        }

        fn byteset_contains(&self, byte: u8) -> bool {
            (self.byteset >> ((byte & 0x3f) as usize)) & 1 != 0
        }

        fn next<S>(&mut self, haystack: &[u8], needle: &[u8], long_period: bool) -> S::Output
        where
            S: TwoWayStrategy,
        {
            // `next()` uses `self.position` as its cursor
            let old_pos = self.position;
            let needle_last = needle.len() - 1;
            'search: loop {
                // Check that we have room to search in
                // position + needle_last can not overflow if we assume slices
                // are bounded by isize's range.
                let tail_byte = match haystack.get(self.position + needle_last) {
                    Some(&b) => b,
                    None => {
                        self.position = haystack.len();
                        return S::rejecting(old_pos, self.position);
                    }
                };
                
                //及早返回不匹配的信息
                if S::use_early_reject() && old_pos != self.position {
                    return S::rejecting(old_pos, self.position);
                }

                // 用位图判断出tail_byte不在子字符串中，可以立刻偏移到下一个字节再比较
                if !self.byteset_contains(tail_byte) {
                    self.position += needle.len();
                    if !long_period {
                        self.memory = 0;
                    }
                    continue 'search;
                }

                // 如果memory有值且大于crip_pos, 那就从memory开始比较，memory前的已经匹配完毕
                // long_period 没有memory的逻辑，和暴力比较无差异
                let start =
                    if long_period { self.crit_pos } else { cmp::max(self.crit_pos, self.memory) };
                for i in start..needle.len() {
                    if needle[i] != haystack[self.position + i] {
                        self.position += i - self.crit_pos + 1;
                        if !long_period {
                            self.memory = 0;
                        }
                        continue 'search;
                    }
                }

                // See if the left part of the needle matches
                let start = if long_period { 0 } else { self.memory };
                for i in (start..self.crit_pos).rev() {
                    if needle[i] != haystack[self.position + i] {
                        //period后面的字符已经比较完毕，period一般大于crit_pos
                        self.position += self.period;
                        if !long_period {
                            self.memory = needle.len() - self.period;
                        }
                        continue 'search;
                    }
                }

                // 比较全部完成，
                let match_pos = self.position;

                // 为下一次比较做准备
                self.position += needle.len();
                if !long_period {
                    self.memory = 0; // set to needle.len() - self.period for overlapping matches
                }

                return S::matching(match_pos, match_pos + needle.len());
            }
        }

        // 略

    }
    // TwoWayStrategy allows the algorithm to either skip non-matches as quickly
    // as possible, or to work in a mode where it emits Rejects relatively quickly.
    trait TwoWayStrategy {
        type Output;
        fn use_early_reject() -> bool;
        fn rejecting(a: usize, b: usize) -> Self::Output;
        fn matching(a: usize, b: usize) -> Self::Output;
    }

    /// Skip to match intervals as quickly as possible
    enum MatchOnly {}

    impl TwoWayStrategy for MatchOnly {
        type Output = Option<(usize, usize)>;

        fn use_early_reject() -> bool {
            false
        }
        fn rejecting(_a: usize, _b: usize) -> Self::Output {
            None
        }
        fn matching(a: usize, b: usize) -> Self::Output {
            Some((a, b))
        }
    }

    /// Emit Rejects regularly
    enum RejectAndMatch {}

    impl TwoWayStrategy for RejectAndMatch {
        type Output = SearchStep;

        fn use_early_reject() -> bool {
            true
        }
        fn rejecting(a: usize, b: usize) -> Self::Output {
            SearchStep::Reject(a, b)
        }
        fn matching(a: usize, b: usize) -> Self::Output {
            SearchStep::Match(a, b)
        }
    }
```
以上对字符串查找的方法进行了分析，利用Pattern的还有以下的方法：
```rust    
    //生成一个支持Iterator的结构完成split
    pub fn split<'a, P: Pattern<'a>>(&'a self, pat: P) -> Split<'a, P> {
        Split(SplitInternal {
            start: 0,
            end: self.len(),
            matcher: pat.into_searcher(self),
            allow_trailing_empty: true,
            finished: false,
        })
    }

    //略
    ...
    ...
```
## 切片标准库代码分析

### 切片排序
#### 插入排序
```rust
/// 插入排序, 复杂度O(n^2).
fn insertion_sort<T, F>(v: &mut [T], is_less: &mut F)
where
    F: FnMut(&T, &T) -> bool,
{
    //排序场景下，基本不能使用iterator
    for i in 1..v.len() {
        //利用
        shift_tail(&mut v[..i + 1], is_less);
    }
}

/// 将最后的值左移到遇到更小的值.
fn shift_tail<T, F>(v: &mut [T], is_less: &mut F)
where
    F: FnMut(&T, &T) -> bool,
{
    let len = v.len();
    
    // 因为是对泛型排序，RUST的排序算法比较复杂， 需要指出，&mut [T] 保证了外界不会有对数组或数组元素的引用，而数组元素本身的内存
    // 浅拷贝等同于所有权转移，不会出现内存安全问题。
    unsafe {
        if len >= 2 && is_less(v.get_unchecked(len - 1), v.get_unchecked(len - 2)) {
            // ManuallyDrop把drop的权利从rust编译器接管
            let mut tmp = mem::ManuallyDrop::new(ptr::read(v.get_unchecked(len - 1)));
            // CopyOnDrop会在drop的时候做src到dest的拷贝
            let mut hole = CopyOnDrop { src: &mut *tmp, dest: v.get_unchecked_mut(len - 2) };
            ptr::copy_nonoverlapping(v.get_unchecked(len - 2), v.get_unchecked_mut(len - 1), 1);
            
            //正常的排序内存置换操作
            for i in (0..len - 2).rev() {
                if !is_less(&*tmp, v.get_unchecked(i)) {
                    break;
                }

                ptr::copy_nonoverlapping(v.get_unchecked(i), v.get_unchecked_mut(i + 1), 1);
                hole.dest = v.get_unchecked_mut(i);
            }
        }
    }
}
```
上面的排序算法最重要的是理解在元素转移的过程为什么没有影响所有权，为什么没有引发内存安全问题。同时，最基础的排序算法也要使用mem及ptr模块的库,可见不熟悉mem及ptr模块，就无法熟练使用RUST。

# 内部可变类型代码分析

## Borrow Trait 代码分析
代码路径如下：   
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\core\src\option.rs  
Borrow Trait代码定义如下：
```rust
//实现Borrow Trait的类型一般是封装结构，如智能指针Box<T>，Rc<T>, String，Cell, RefCell等，通过borrow将
//内部变量的引用提供给外部。通常的情况下，这些也都实现了Deref，AsRef等Trait把内部变量暴露出来，所以这些
//Trait之间有些重复。但Borrow Trait 更主要的场景是在RefCell等结构中提供内部可变性，这是Deref, AsRef等
//Trait无能为力的区域。后继分析相关类型时再给出进一步分析。
pub trait Borrow<Borrowed: ?Sized> {
    fn borrow(&self) -> &Borrowed;
}

pub trait BorrowMut<Borrowed: ?Sized>: Borrow<Borrowed> {
    fn borrow_mut(&mut self) -> &mut Borrowed;
}
//每一个类型都实现了针对自身的Borrow Trait
impl<T: ?Sized> Borrow<T> for T {
    fn borrow(&self) -> &T {
        self
    }
}

//每一个类型都实现了针对自身的BorrowMut Trait
impl<T: ?Sized> BorrowMut<T> for T {
    fn borrow_mut(&mut self) -> &mut T {
        self
    }
}

//每一个类型的引用都实现了对自身的Borrow Trait
impl<T: ?Sized> Borrow<T> for &T {
    fn borrow(&self) -> &T {
        &**self
    }
}
//每一个类型的可变引用都实现了针对自身的Borrow Trait
impl<T: ?Sized> Borrow<T> for &mut T {
    fn borrow(&self) -> &T {
        &**self
    }
}

//每一个类型的可变引用都实现了针对自身的BorrowMut
impl<T: ?Sized> BorrowMut<T> for &mut T {
    fn borrow_mut(&mut self) -> &mut T {
        &mut **self
    }
}
```

## Cell模块类型代码分析
代码路径：
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\core\src\cell.rs  

Cell模块提供了内部可变性的功能。对应于以下场景：
一个变量存在多个引用，希望通过这些引用都可以修改此变量。
RUST的可变引用与不可变引用不能同时共存，这导致了无法通过普通的引用语法完成上述场景。RUST提供的方案是Cell方案。思路很简单，提供一个封装结构，此封装结构以非可变引用存在于其他类型结构体成员变量中。然后在这个封装结构的基础上，利用unsafe 代码完成内部变量的改变。
Cell模块类型的层次如下：
1. UnsafeCell<T>主要提供了获取内部变量原生可变指针的方法。
2. Cell<T>基于UnsafeCell<T>的基础上实现了安全的内部可变性, Cell<T>的多个不可变引用都可以用set方法改变内部的T类型变量。Cell<T>本身没有可变引用及不可变引用的区别，与RUST的内存安全性理念有些不符合。程序员使用RefCell<T>完成内部可变性是更合适的方案。
3. RefCell<T>基于Cell<T>及UnsafeCell<T>，并实现Borrow Trait 及 BorrowMut Trait，可以实现在生命周期不重合的情况下的多个可变引用，且可变引用与不可变引用不可以同时存在。RefCell<T>是与RUST内存安全理念相契合的内部可变性实现方案。
   
UnsafeCell是RUST的内部可变结构的最底层基础设施，Cell结构和RefCell结构都是用UnsafeCell来实现内部可变性的。 
RUST不提供同时存在两个以上的可变引用的方案。
```rust

pub struct UnsafeCell<T: ?Sized> {
    value: T,
}
impl<T> UnsafeCell<T> {
    //创建包装结构
    pub const fn new(value: T) -> UnsafeCell<T> {
        UnsafeCell { value }
    }

    //解包装
    pub const fn into_inner(self) -> T {
        self.value
    }
}

impl<T: ?Sized> UnsafeCell<T> {
    pub const fn get(&self) -> *mut T {
        // 将裸指针导出，这是为什么起名是UnsafeCell的原因
        // 此裸指针的安全性由调用代码保证
        self as *const UnsafeCell<T> as *const T as *mut T
    }

    //给出一个正常的可变引用
    pub const fn get_mut(&mut self) -> &mut T {
        &mut self.value
    }
    
    //参数与get有区别，是关联函数
    pub const fn raw_get(this: *const Self) -> *mut T {
        this as *const T as *mut T
    }
}
//对任意T的类型，可以为T.into() 创建UnsafeCell类型变量
impl<T> const From<T> for UnsafeCell<T> {
    fn from(t: T) -> UnsafeCell<T> {
        UnsafeCell::new(t)
    }
}
```
可以看到，UnsafeCell的get函数返回了裸指针，UnsafeCell逃脱RUST对引用安全检查的方法实际上就是个通常的unsafe 的裸指针操作，没有任何神秘性可言。

Cell<T> 内部包装UnsafeCell<T>， 利用UnsafeCell<T>的方法获得裸指针后，用unsafe代码对内部变量进行赋值，从而绕开了RUST语言编译器对引用的约束。Cell<T>的赋值没有任何限制，实际上和直接使用裸指针赋值是等同的，但因为没有直接暴露裸指针，所以保证了不会出现悬垂指针。
```rust
#[repr(transparent)]
pub struct Cell<T: ?Sized> {
    value: UnsafeCell<T>,
}
```
Cell<T>创建方法：
```rust
impl<T> const From<T> for Cell<T> {
    fn from(t: T) -> Cell<T> {
        Cell::new(t)
    }
}

impl<T> Cell<T> {
    pub const fn new(value: T) -> Cell<T> {
        Cell { value: UnsafeCell::new(value) }
    }
```
Cell<T> 改变内部变量的方法:
```rust
    
    pub fn set(&self, val: T) {
        //实际调用mem::replace
        let old = self.replace(val);
        //这里不调用drop, old也应该因为生命周期终结被释放。
        //此处调用drop以确保万无一失
        drop(old);
    }

    pub fn swap(&self, other: &Self) {
        //此处注意，ptr::eq不仅仅比较地址，也比较元数据
        if ptr::eq(self, other) {
            return;
        }
        //此段不会出现在跨线程的场景下
        unsafe {
            ptr::swap(self.value.get(), other.value.get());
        }
    }

    //此函数也会将原有的值及所有权返回
    pub fn replace(&self, val: T) -> T {
        // 利用unsafe粗暴将指针转变为可变引用，然后赋值，此处必须用
        // replace，原有值的所有权需要有交代。
        mem::replace(unsafe { &mut *self.value.get() }, val)
    }
```
获取内部值的解封装方法:
```rust
    pub const fn into_inner(self) -> T {
        //解封装
        self.value.into_inner()
    }
}

impl<T: Default> Cell<T> {
    //take后，变量所有权已经转移出来
    pub fn take(&self) -> T {
        self.replace(Default::default())
    }
}

impl<T: Copy> Cell<T> {
    pub fn get(&self) -> T {
        //只适合于Copy Trait类型，否则会导致所有权转移，引发UB
        unsafe { *self.value.get() }
    }
```
对函数式编程支持的方法
```rust    
    //函数式编程，因为T支持Copy，所以没有所有权问题 
    pub fn update<F>(&self, f: F) -> T
    where
        F: FnOnce(T) -> T,
    {
        let old = self.get();
        let new = f(old);
        self.set(new);
        new
    }
}

```
获取内部变量指针的方法：
```rust
impl<T: ?Sized> Cell<T> {
    //通常应该不使用这个机制，安全隐患非常大
    pub const fn as_ptr(&self) -> *mut T {
        self.value.get()
    }
    //获取内部的可变引用，即使调用了这个函数，也不影响set做值更新
    pub fn get_mut(&mut self) -> &mut T {
        self.value.get_mut()
    }

    pub fn from_mut(t: &mut T) -> &Cell<T> {
        // 利用repr[transparent]直接做转换
        unsafe { &*(t as *mut T as *const Cell<T>) }
    }
}
```
切片类型相关方法
```rust
//Unsized Trait实现
impl<T: CoerceUnsized<U>, U> CoerceUnsized<Cell<U>> for Cell<T> {}

impl<T> Cell<[T]> {
    pub fn as_slice_of_cells(&self) -> &[Cell<T>] {
        // 粗暴的直接转换
        unsafe { &*(self as *const Cell<[T]> as *const [Cell<T>]) }
    }
}

impl<T, const N: usize> Cell<[T; N]> {
    pub fn as_array_of_cells(&self) -> &[Cell<T>; N] {
        // 粗暴的直接转换
        unsafe { &*(self as *const Cell<[T; N]> as *const [Cell<T>; N]) }
    }
}
```

以下为RefCell<T>类型相关的结构， 删除了一些和debug相关的内容，使代码简化及理解简单
```rust
/// 满足以下需求：

pub struct RefCell<T: ?Sized> {
    //用以标识对外是否有可变引用，是否有不可变引用，有多少个不可变引用
    //是引用计数的实现体
    borrow: Cell<BorrowFlag>,
    //包装内部的变量
    value: UnsafeCell<T>,
}
```
引用计数类型BorrowFlag的定义：
```rust
// 正整数表示RefCell执行borrow()调用生成的不可变引用"Ref"的数目
// 负整数表示RefCell执行borrow_mut()调用生成的可变引用"RefMut"的数目
// 多个RefMut存在的条件是在多个RefMut指向同一个"RefCell"的不同部分的情况，如多个RefMut指向
// 一个slice的不重合的部分。
type BorrowFlag = isize;
// 0表示没有执行过borrow()或borrow_mut()调用
const UNUSED: BorrowFlag = 0;

//有borrow_mut()被执行且生命周期没有终结
fn is_writing(x: BorrowFlag) -> bool {
    x < UNUSED
}

//有borrow()被执行且生命周期没有终结
fn is_reading(x: BorrowFlag) -> bool {
    x > UNUSED
}

//对RefCell<T>中成员变量borrow的引用封装类型，计数逻辑的负责类型
struct BorrowRef<'b> {
    borrow: &'b Cell<BorrowFlag>,
}

impl<'b> BorrowRef<'b> {
    //每次new，代表对RefCell<T>产生了一个新的引用，需增加引用计数
    fn new(borrow: &'b Cell<BorrowFlag>) -> Option<BorrowRef<'b>> {
        // 引用计数加1，
        let b = borrow.get().wrapping_add(1);
        if !is_reading(b) {
            // 1.如果有borrow_mut()调用且生命周期没有终结
            // 2.如果到达isize::MAX
            None
        } else {
            // 增加一个不可变借用计数:
            borrow.set(b);
            Some(BorrowRef { borrow })
        }
    }
}

// Drop，代表对RefCell<T>的引用生命周期结束，需减少引用计数
impl Drop for BorrowRef<'_> {
    fn drop(&mut self) {
        let borrow = self.borrow.get();
        //一定应该是正整数
        debug_assert!(is_reading(borrow));
        //不可变引用计数减一
        self.borrow.set(borrow - 1);
    }
}
impl Clone for BorrowRef<'_> {
    //每次clone实际上增加了对RefCell<T>的不可变引用，
    fn clone(&self) -> Self {
        //不可变引用计数加1
        let borrow = self.borrow.get();
        debug_assert!(is_reading(borrow));
        assert!(borrow != isize::MAX);
        self.borrow.set(borrow + 1);
        BorrowRef { borrow: self.borrow }
    }
}
```
下面是从RefCell<T>取得不可变引用的具体实现类型及逻辑。
```rust
//RefCell<T> borrow()调用获取的类型
pub struct Ref<'b, T: ?Sized + 'b> {
    //对RefCell<T>中value的引用
    value: &'b T,
    //对RefCell<T>中borrow引用的封装
    borrow: BorrowRef<'b>,
}
//Deref将获得内部value
impl<T: ?Sized> Deref for Ref<'_, T> {
    type Target = T;

    fn deref(&self) -> &T {
        self.value
    }
}

impl<'b, T: ?Sized> Ref<'b, T> {
    /// 与再执行RefCell<T>::borrow等价。但用clone可以在不必有RefCell<T>的情况下增加引用
    /// 不选择实现Clone Trait，是因为要用RefCell<T>.borrow().clone()来复制
    /// RefCell<T>
    pub fn clone(orig: &Ref<'b, T>) -> Ref<'b, T> {
        Ref { value: orig.value, borrow: orig.borrow.clone() }
    }

    //通常的情况下，F的返回引用与Ref中的引用是强相关的，即获得返回引用等同于获得Ref中value的引用
    pub fn map<U: ?Sized, F>(orig: Ref<'b, T>, f: F) -> Ref<'b, U>
    where
        F: FnOnce(&T) -> &U,
    {
        Ref { value: f(orig.value), borrow: orig.borrow }
    }

    //同上，例如value是一个切片引用，filter后获得切片的一部分
    pub fn filter_map<U: ?Sized, F>(orig: Ref<'b, T>, f: F) -> Result<Ref<'b, U>, Self>
    where
        F: FnOnce(&T) -> Option<&U>,
    {
        match f(orig.value) {
            Some(value) => Ok(Ref { value, borrow: orig.borrow }),
            None => Err(orig),
        }
    }

  
    /// leak调用后，此Ref不再调用drop，从而导致RefCell中的计数器无法恢复原状，也会导致可变引用无法再被创建 
    pub fn leak(orig: Ref<'b, T>) -> &'b T {
        mem::forget(orig.borrow);
        orig.value
    }
}

impl<'b, T: ?Sized + Unsize<U>, U: ?Sized> CoerceUnsized<Ref<'b, U>> for Ref<'b, T> {}
```
以下是borrow_mut()的相关结构及逻辑:
```rust
//作用与BorrowRef相同
struct BorrowRefMut<'b> {
    borrow: &'b Cell<BorrowFlag>,
}

//RefMut生命周期终止时调用
impl Drop for BorrowRefMut<'_> {
    fn drop(&mut self) {
        //可变引用计数减一(数学运算为加)
        let borrow = self.borrow.get();
        debug_assert!(is_writing(borrow));
        self.borrow.set(borrow + 1);
    }
}

impl<'b> BorrowRefMut<'b> {
    fn new(borrow: &'b Cell<BorrowFlag>) -> Option<BorrowRefMut<'b>> {
        //初始的borrow_mut，引用计数必须是0，不存在其他可变引用
        match borrow.get() {
            UNUSED => {
                borrow.set(UNUSED - 1);
                Some(BorrowRefMut { borrow })
            }
            _ => None,
        }
    }

    // 不通过RefCell获取新的RefMut的方法，对于新的RefMut，必须是一个整体的可变引用分为几个
    // 组成部分的可变引用，如结构体成员，或数组成员。且可变引用之间互相不重合，不允许两个可变引用能修改同一块内存
    fn clone(&self) -> BorrowRefMut<'b> {
        //不可变引用计数增加(算数减)
        let borrow = self.borrow.get();
        debug_assert!(is_writing(borrow));
        // Prevent the borrow counter from underflowing.
        assert!(borrow != isize::MIN);
        self.borrow.set(borrow - 1);
        BorrowRefMut { borrow: self.borrow }
    }
}

//从RefCell<T> borrow_mut返回的结构体
pub struct RefMut<'b, T: ?Sized + 'b> {
    //可变引用
    value: &'b mut T,
    //计数器
    borrow: BorrowRefMut<'b>,
}

//Deref后返回内部变量的引用
impl<T: ?Sized> Deref for RefMut<'_, T> {
    type Target = T;

    fn deref(&self) -> &T {
        self.value
    }
}
//DerefMut返回内部变量可变引用
impl<T: ?Sized> DerefMut for RefMut<'_, T> {
    fn deref_mut(&mut self) -> &mut T {
        self.value
    }
}
```
对RefCell<T>的方法实现：
变量创建方法：
```rust
impl<T> RefCell<T> {
    pub const fn new(value: T) -> RefCell<T> {
        RefCell {
            value: UnsafeCell::new(value),
            //初始化一定是UNUSED
            borrow: Cell::new(UNUSED),
        }
    }
```
解封装方法：
```rust

    //实际会消费RefCell，并将内部变量返回，此时，如果
    //执行过borrow()/borrow_mut() 并生命周期未终结，
    //有可能会在后继出现悬垂指针问题，此处应该做一下判断
    //尽量规避此方法的使用
    pub const fn into_inner(self) -> T {
        self.value.into_inner()
    }
```
改变内部值的方法：
```rust
    //将原有内部变量替换为新值，既然是RefCell, 通常应使用borrow_mut
    //获得可变引用，再对值做修改，下面函数实际也是用borrow_mut完成，
    //但更多应该是用在泛型中
    pub fn replace(&self, t: T) -> T {
        mem::replace(&mut *self.borrow_mut(), t)
    }

    //同上，只是用函数获取新值
    pub fn replace_with<F: FnOnce(&mut T) -> T>(&self, f: F) -> T {
        let mut_borrow = &mut *self.borrow_mut();
        let replacement = f(mut_borrow);
        mem::replace(mut_borrow, replacement)
    }

    //两个引用交换值，也交换了值的所有权
    pub fn swap(&self, other: &Self) {
        mem::swap(&mut *self.borrow_mut(), &mut *other.borrow_mut())
    }
}
```
下面实际上已经实现了Borrow trait,
```rust
impl<T: ?Sized> RefCell<T> {
    //Borrow Trait实现
    pub fn borrow(&self) -> Ref<'_, T> {
        //真正是try_borrow()
        self.try_borrow().expect("already mutably borrowed")
    }
    //不可变引用 borrow真正实现
    pub fn try_borrow(&self) -> Result<Ref<'_, T>, BorrowError> {
        match BorrowRef::new(&self.borrow) {
            Some(b) => {
                // 保证了self.borrow一定是is_reading()为真，直接从裸指针
                //转换，对RUST来讲，转换后的引用与原变量没有内存安全的联系。
                // 从这个函数看，RefCell<T>应该尽量使用RefCell的方法操作，除非绝对把握
                // 不要直接将内部变量的正常引用导出，否则安全隐患巨大。
                Ok(Ref { value: unsafe { &*self.value.get() }, borrow: b })
            }
            None => Err(BorrowError {
                
            }),
        }
    }

    //BorrowMut Trait实现
    pub fn borrow_mut(&self) -> RefMut<'_, T> {
        self.try_borrow_mut().expect("already borrowed")
    }

    pub fn try_borrow_mut(&self) -> Result<RefMut<'_, T>, BorrowMutError> {
        match BorrowRefMut::new(&self.borrow) {
            Some(b) => {
                // 一定不存在非可变引用，也仅有本次的可变引用，这个可变引用直接从
                // 裸指针转换，对RUST编译器，转换后的可变引用与原变量没有内存安全的联系。
                Ok(RefMut { value: unsafe { &mut *self.value.get() }, borrow: b })
            }
            None => Err(BorrowMutError {
                
            }),
        }
    }
```
直接获取内部变量指针：
```rust
    //此函数如果没有绝对的安全把握，不要用
    pub fn as_ptr(&self) -> *mut T {
        self.value.get()
    }

    //此函数如果没有绝对的安全把握，不要用
    pub fn get_mut(&mut self) -> &mut T {
        self.value.get_mut()
    }
```
其他方法：
```rust    
    //在leak操作后，做leak的逆操作，实际上对计数器进行了恢复，
    pub fn undo_leak(&mut self) -> &mut T {
        *self.borrow.get_mut() = UNUSED;
        self.get_mut()
    }

    //规避计数器计数的方法，与borrow操作近似
    pub unsafe fn try_borrow_unguarded(&self) -> Result<&T, BorrowError> {
        //如果没有borrow_mut(),则返回引用
        if !is_writing(self.borrow.get()) {
            Ok(unsafe { &*self.value.get() })
        } else {
            Err(BorrowError {
            })
        }
    }
}
```
内部值获取方法：
```rust
impl<T: Default> RefCell<T> {
    //对RefCell<T>应该不使用这个函数，尤其是在有borrow()/borrow_mut()
    //且生命周期没有终结时
    pub fn take(&self) -> T {
        self.replace(Default::default())
    }
}
```
系统编译器内嵌trait实现：
```rust
//支持线程间转移
unsafe impl<T: ?Sized> Send for RefCell<T> where T: Send {}
//不支持线程间共享
impl<T: ?Sized> !Sync for RefCell<T> {}

impl<T: Clone> Clone for RefCell<T> {
    //clone实际上仅仅是增加计数
    fn clone(&self) -> RefCell<T> {
        //self.borrow().clone 实质是((*self.borrow()).clone)
        //连续解引用后做clone的调用
        //Ref<T>不支持Clone，所以解引用的到&T        
        RefCell::new(self.borrow().clone())
    }

    fn clone_from(&mut self, other: &Self) {
        //self.get_mut().clone_from 实质是
        // (*self.get_mut()).clone_from()
        // &mut T不支持Clone，所以解引用到T
        self.get_mut().clone_from(&other.borrow())
    }
}

impl<T: Default> Default for RefCell<T> {
    fn default() -> RefCell<T> {
        RefCell::new(Default::default())
    }
}

impl<T: ?Sized + PartialEq> PartialEq for RefCell<T> {
    fn eq(&self, other: &RefCell<T>) -> bool {
        *self.borrow() == *other.borrow()
    }
}

impl<T> const From<T> for RefCell<T> {
    fn from(t: T) -> RefCell<T> {
        RefCell::new(t)
    }
}

impl<T: CoerceUnsized<U>, U> CoerceUnsized<RefCell<U>> for RefCell<T> {}
```
RefCell<T>的代码实现，是理解RUST解决问题的思维的好例子。 编程中，RefCell的计数器是针对RUST语法的一个精巧的设计，利用drop的自动调用，编程只需要关注new，这就节省了程序员极大的精力，也规避了错误的发生。borrow_mut()机制则解决了多个可修改借用。
利用RUST的非安全个性和自动drop的机制，可以自行设计出RefCell<T>这样的标准库解决方案，而不是借助于编译器。这是RUST的一个突出特点，也是其能与C一样成为系统级语言的原因。
## Pin及UnPin
Pin<T>主要解决需要程序员在编程时要时刻注意处理可能的变量地址改变的情况。利用Pin<T>，程序员只需要在初始的时候注意到这个场景并定义好。后继就可以不必再关心。
Pin是一个对指针&mut T的包装结构，包装后因为&mut T的独占性。封装结构外，不可能再存在变量的引用及不可变引用。所有的引用都只能使用Pin<T>来完成，导致RUST的需要引用的一些内存操作无法进行，如实质上是指针交换的调用mem::swap，从而保证了指针指向的变量在代码中会被固定在某个内存位置。当然，编译器也不会再做优化。

实现Unpin Trait的类型不受Pin的约束，RUST中实现Copy trait的类型基本上都实现了Unpin Trait。
结构定义
```rust
#[repr(transparent)]
#[derive(Copy, Clone)]
pub struct Pin<P> {
    pointer: P,
}
```
Pin变量创建：
```rust
impl<P: Deref<Target: Unpin>> Pin<P> {
    // 支持Unpin类型可以用new创建Pin<T>
    pub const fn new(pointer: P) -> Pin<P> {
        unsafe { Pin::new_unchecked(pointer) }
    }
    ...
}

impl<P: Deref> Pin<P> {
    //实现Deref的类型，用下面的行为创建Pin<T>, 调用者应该保证P可以被Pin，
    pub const unsafe fn new_unchecked(pointer: P) -> Pin<P> {
        Pin { pointer }
    }
    ...
}
```
Pin自身的new方法仅针对Pin实际上不起作用的Unpin类型。对于其他不支持Unpin的类型，通常使用智能指针提供的Pin创建方法，如Boxed::pin。
new_unchecked则提供给其他智能指针的安全的创建方法内部使用。

```rust
impl <P: Deref<Target: Unpin>> Pin<P> {
    ...
    /// 解封装，取消内存pin操作
    pub const fn into_inner(pin: Pin<P>) -> P {
        pin.pointer
    }
}

impl <P:Deref> Pin<P> {
    ...
    
    //对应于new_unchecked
    pub const unsafe fn into_inner_unchecked(pin: Pin<P>) -> P {
        pin.pointer
    }
}
```
指针转换
```rust
impl<P: Deref> Pin<P> {
    ...
    /// 需要返回一个Pin的引用，以为P自身就是指针，返回P是不合理及不安全的，所以此函数被用来
    /// 返回Pin住的解引用的指针类型
    pub fn as_ref(&self) -> Pin<&P::Target> {
        // SAFETY: see documentation on this function
        unsafe { Pin::new_unchecked(&*self.pointer) }
    }
}
impl <P:DerefMut> Pin<P> { 
    ...  
    /// 需要返回一个Pin的可变引用，以为P自身就是指针，所以此函数被用来返回Pin住的解引用的指针类型
    pub fn as_mut(&mut self) -> Pin<&mut P::Target> {
        unsafe { Pin::new_unchecked(&mut *self.pointer) }
    }
}
impl <'a, T:?Sized> Pin<&'a T> {
    //&T 不会导致在安全RUST领域的类如mem::replace之类的地址改变操作
    pub const fn get_ref(self) -> &'a T {
        self.pointer
    }
}

impl<'a, T: ?Sized> Pin<&'a mut T> {
    //不可变引用可以随意返回，不会影响Pin的语义
    pub const fn into_ref(self) -> Pin<&'a T> {
        Pin { pointer: self.pointer }
    }

    //Unpin的可变引用可以返回，Pin对Unpin类型无作用
    pub const fn get_mut(self) -> &'a mut T
    where
        T: Unpin,
    {
        self.pointer
    }

    //后门，要确定安全，会导致Pin失效
    pub const unsafe fn get_unchecked_mut(self) -> &'a mut T {
        self.pointer
    }
    ...
}

impl<T: ?Sized> Pin<&'static T> {
    pub const fn static_ref(r: &'static T) -> Pin<&'static T> {
        unsafe { Pin::new_unchecked(r) }
    }
}

impl<'a, P: DerefMut> Pin<&'a mut Pin<P>> {
    pub fn as_deref_mut(self) -> Pin<&'a mut P::Target> {
        unsafe { self.get_unchecked_mut() }.as_mut()
    }
}

impl<T: ?Sized> Pin<&'static mut T> {
    pub const fn static_mut(r: &'static mut T) -> Pin<&'static mut T> {
        // SAFETY: The 'static borrow guarantees the data will not be
        // moved/invalidated until it gets dropped (which is never).
        unsafe { Pin::new_unchecked(r) }
    }
}

impl<P: Deref> Deref for Pin<P> {
    type Target = P::Target;
    fn deref(&self) -> &P::Target {
        Pin::get_ref(Pin::as_ref(self))
    }
}
//只有Unpin才支持对mut的DerefMut trait，不支持Unpin的，不能用DerefMut，以保证Pin
impl<P: DerefMut<Target: Unpin>> DerefMut for Pin<P> {
    fn deref_mut(&mut self) -> &mut P::Target {
        Pin::get_mut(Pin::as_mut(self))
    }
}

```
内部可变性函数：
```rust
impl <P:DerefMut> Pin<P> {
    //修改值，实质也提供了内部可变性
    pub fn set(&mut self, value: P::Target)
    where
        P::Target: Sized,
    {
        *(self.pointer) = value;
    }
}

impl<'a, T: ?Sized> Pin<&'a T> {
    //函数式编程，func返回的pointer与self.pointer应该强相关，如结构中
    //某一变量的引用，或slice中某一元素的引用
    pub unsafe fn map_unchecked<U, F>(self, func: F) -> Pin<&'a U>
    where
        U: ?Sized,
        F: FnOnce(&T) -> &U,
    {
        let pointer = &*self.pointer;
        let new_pointer = func(pointer);

        // SAFETY: the safety contract for `new_unchecked` must be
        // upheld by the caller.
        unsafe { Pin::new_unchecked(new_pointer) }
    }

}

impl<'a, T: ?Sized> Pin<&'a mut T> {
    
    pub unsafe fn map_unchecked_mut<U, F>(self, func: F) -> Pin<&'a mut U>
    where
        U: ?Sized,
        F: FnOnce(&mut T) -> &mut U,
    {
        // 这个可能导致Pin住的内容移动，调用者要保证不出问题
        let pointer = unsafe { Pin::get_unchecked_mut(self) };
        let new_pointer = func(pointer);
        unsafe { Pin::new_unchecked(new_pointer) }
    }
}
```
利用Pin的封装及基于trait约束的方法实现，使得指针pin在内存中的需求得以实现。是RUST利用封装语义完成语言需求的又一经典案例
## Lazy<T>分析

OnceCell是一种内部可变的类型，其用于初始化没有初始值，仅支持赋值一次的类型。
```rust
pub struct OnceCell<T> {
    // Option<T>支持None作为初始化的值
    inner: UnsafeCell<Option<T>>,
}
```
OnceCell封装UnsafeCell以支持内部可变性。
创建方法:
``` rust
impl<T> const From<T> for OnceCell<T> {
    fn from(value: T) -> Self {
        OnceCell { inner: UnsafeCell::new(Some(value)) }
    }
}
impl<T> OnceCell<T> {
    /// 初始化为空，
    pub const fn new() -> OnceCell<T> {
        //注意，此时给UnsafeCell分配了一个T类型的地址空间
        OnceCell { inner: UnsafeCell::new(None) }
    }
```
获取内部引用
```rust
    pub fn get(&self) -> Option<&T> {
        // 生成一个内部变量的引用，
        unsafe { &*self.inner.get() }.as_ref()
    }

    /// 直接用返回结果取可以&mut T，然后再解封装后用可变引用即可改变内部封装变量的值，会突破只赋值一次
    /// 的既定语义，此函数最好不使用
    pub fn get_mut(&mut self) -> Option<&mut T> {
        unsafe { &mut *self.inner.get() }.as_mut()
    }
```
对内部值进行修改方法：
```rust
    /// 通过此函数仅能给OnceCell内部变量做一次赋值
    pub fn set(&self, value: T) -> Result<(), T> {
        // SAFETY: Safe because we cannot have overlapping mutable borrows
        let slot = unsafe { &*self.inner.get() };
        if slot.is_some() {
            return Err(value);
        }

        let slot = unsafe { &mut *self.inner.get() };
        *slot = Some(value);
        Ok(())
    }

    //见下面函数
    pub fn get_or_init<F>(&self, f: F) -> &T
    where
        F: FnOnce() -> T,
    {
        //Ok::<T,!>(f()) 即Result类型初始化，例如Ok::<i32,!>(3)
        match self.get_or_try_init(|| Ok::<T, !>(f())) {
            Ok(val) => val,
        }
    }

    //有值就返回值，没有值用f生成值
    pub fn get_or_try_init<F, E>(&self, f: F) -> Result<&T, E>
    where
        F: FnOnce() -> Result<T, E>,
    {
        if let Some(val) = self.get() {
            return Ok(val);
        }
        /// 下面代码关键是cold, 防止优化后的代码出现意外，因为此函数会被多次调用
        /// 这是一个较冷门的知识点
        #[cold]
        fn outlined_call<F, T, E>(f: F) -> Result<T, E>
        where
            F: FnOnce() -> Result<T, E>,
        {
            f()
        }
        let val = outlined_call(f)?;
        assert!(self.set(val).is_ok(), "reentrant init");
        Ok(self.get().unwrap())
    }
```
解封装方法：
```rust
    //消费了OnceCell,并且返回内部变量
    pub fn into_inner(self) -> Option<T> {
        self.inner.into_inner()
    }

    //替换OnceCell，并将替换的OnceCell消费掉，并且返回内部变量
    pub fn take(&mut self) -> Option<T> {
        mem::take(self).into_inner()
    }
}
```
OnceCell<T>对trait的实现：
```rust
impl<T> Default for OnceCell<T> {
    fn default() -> Self {
        Self::new()
    }
}

impl<T: Clone> Clone for OnceCell<T> {
    fn clone(&self) -> OnceCell<T> {
        let res = OnceCell::new();
        if let Some(value) = self.get() {
            match res.set(value.clone()) {
                Ok(()) => (),
                Err(_) => unreachable!(),
            }
        }
        res
    }
}

impl<T: PartialEq> PartialEq for OnceCell<T> {
    fn eq(&self, other: &Self) -> bool {
        self.get() == other.get()
    }
}
```
基于OnceCell<T>实现惰性结构Lazy<T>,惰性结构在第一次调用解引用的时候被赋值，随后使用这个值。
此结构强迫代码区分初始化必须有值及不必赋值的情况。
```rust
/// 惰性类型，在第一次使用时进行赋值和初始化
pub struct Lazy<T, F = fn() -> T> {
    //初始化可以为空
    cell: OnceCell<T>,
    //对cell做初始化赋值的函数
    init: Cell<Option<F>>,
}

impl<T, F> Lazy<T, F> {
    /// 函数作为变量被保存
    pub const fn new(init: F) -> Lazy<T, F> {
        Lazy { cell: OnceCell::new(), init: Cell::new(Some(init)) }
    }
}

impl<T, F: FnOnce() -> T> Lazy<T, F> {
    //完成赋值操作
    pub fn force(this: &Lazy<T, F>) -> &T {
        //如果cell为空，则用init作初始化赋值，注意这里init的take调用已经将init替换成None，
        this.cell.get_or_init(|| match this.init.take() {
            Some(f) => f(),
            None => panic!("`Lazy` instance has previously been poisoned"),
        })
    }
}

//在对Lazy解引用时才进行赋值操作
impl<T, F: FnOnce() -> T> Deref for Lazy<T, F> {
    type Target = T;
    fn deref(&self) -> &T {
        Lazy::force(self)
    }
}

impl<T: Default> Default for Lazy<T> {
    /// Creates a new lazy value using `Default` as the initializing function.
    fn default() -> Lazy<T> {
        Lazy::new(T::default)
    }
}
```

## 小结
从内部可变类型，以及前面的NonNull<T>, Unique<T>, NonZeroSize<T>,可以看到RUST灵活的利用封装结构，提供了一系列在语言原生类型增强的功能。

# 智能指针

## Box<T>代码分析
除了数组外的智能指针的堆内存申请，一般都先由Box<T>来完成，然后再将申请到的内存转移到智能指针自身的结构中。

以下为Box<T>结构定义及创建方法相关内容：
```rust
//Box结构
pub struct Box<
    T: ?Sized,
    //默认的堆内存申请为Global单元结构体，可修改为其他
    A: Allocator = Global,
  //用Unique<T>表示对申请的堆内存拥有所有权  
>(Unique<T>, A);
```
Box<T>的创建方法：
```rust
//以Global作为默认的堆内存分配器的实现
impl<T> Box<T> {
    pub fn new(x: T) -> Self {
        //box 是关键字，就是实现从堆内存申请内存，写入内容然后形成Box<T>
        //这个关键字的功能可以从后继的方法中分析出来, 此方法实际等同与new_in(x, Global);
        box x
    }
    ...
}

//不限定堆内存分配器的更加通用的方法实现
impl<T, A: Allocator> Box<T, A> {

    //Box::new(x) 实际上的逻辑等同与 Box::new_in(x, Global)
    pub fn new_in(x: T, alloc: A) -> Self {
        //new_uninit_in见后面代码分析
        let mut boxed = Self::new_uninit_in(alloc);
        unsafe {
            //实际是MaybeUninit<T>::as_mut_ptr()得到*mut T，::write将x写入申请的堆内存中
            boxed.as_mut_ptr().write(x);
            //从Box<MaybeUninit<T>,A>转换为Box<T,A>
            boxed.assume_init()
        }
    }

    //内存部分章节有过分析
    pub fn new_uninit_in(alloc: A) -> Box<mem::MaybeUninit<T>, A> {
        //获取Layout以便申请堆内存
        let layout = Layout::new::<mem::MaybeUninit<T>>();
        //见后面的代码分析
        match Box::try_new_uninit_in(alloc) {
            Ok(m) => m,
            Err(_) => handle_alloc_error(layout),
        }
    }

    //内存申请的真正执行函数
    pub fn try_new_uninit_in(alloc: A) -> Result<Box<mem::MaybeUninit<T>, A>, AllocError> {
        //申请内存需要的内存Layout
        let layout = Layout::new::<mem::MaybeUninit<T>>();
        //申请内存并完成错误处理，cast将NonNull<[u8]>转换为NonNull<MaybeUninit<T>>
        //NonNull<MaybeUninit<T>>.as_ptr为 *mut <MaybeUninit<T>>
        //后继Box的drop会释放此处的内存
        //from_raw_in即将ptr转换为Unique<T>并形成Box结构变量
        let ptr = alloc.allocate(layout)?.cast();
        unsafe { Ok(Box::from_raw_in(ptr.as_ptr(), alloc)) }
    }

    ...
}

impl<T, A: Allocator> Box<mem::MaybeUninit<T>, A> {
    //申请的未初始化内存，初始化后，应该调用这个函数将
    //Box<MaybeUninit<T>>转换为Box<T>，
    pub unsafe fn assume_init(self) -> Box<T, A> {
        //因为类型不匹配，且无法强制转换，所以先将self消费掉并获得
        //堆内存的裸指针，再用裸指针生成新的Box，完成类型转换V
        let (raw, alloc) = Box::into_raw_with_allocator(self);
        unsafe { Box::from_raw_in(raw as *mut T, alloc) }
    }
}
impl<T: ?Sized, A: Allocator> Box<T, A> {
    //从裸指针构建Box类型，裸指针应该是申请堆内存返回的指针
    //用这个方法生成Box，当Box被drop时，会引发对裸指针的释放操作
    pub unsafe fn from_raw_in(raw: *mut T, alloc: A) -> Self {
        //由裸指针生成Unique，再生成Box
        Box(unsafe { Unique::new_unchecked(raw) }, alloc)
    }
    
    //此函数会将传入的b:Box消费掉，并将内部的Unique也消费掉，
    //返回裸指针，此时裸指针指向的内存已经不会再被drop.
    pub fn into_raw_with_allocator(b: Self) -> (*mut T, A) {
        let (leaked, alloc) = Box::into_unique(b);
        (leaked.as_ptr(), alloc)
    }
    pub fn into_unique(b: Self) -> (Unique<T>, A) {
        //对b的alloc做了一份拷贝
        let alloc = unsafe { ptr::read(&b.1) };
        //Box::leak(b)返回&mut T可变引用，具体分析见下文
        //leak(b)生成的&mut T实质上已经不会有Drop调用释放
        (Unique::from(Box::leak(b)), alloc)
    }

    //将b消费掉，并将b内的变量取出来返回
    pub fn leak<'a>(b: Self) -> &'a mut T
    where
        A: 'a,
    {
        //生成ManuallyDrop<Box<T>>, 消费掉了b，此时不会再对b做Drop调用，导致了一个内存leak
        //ManuallyDrop<Box<T>>.0 是Box<T>，ManuallyDrop<T>没有.0的语法，因此会先做解引用，是&Box<T>
        //&Box<T>.0即Unique<T>，Unique<T>.as_ptr获得裸指针，然后利用unsafe代码生成可变引用
        unsafe { &mut *mem::ManuallyDrop::new(b).0.as_ptr() }
    }
    ...
}

unsafe impl< T: ?Sized, A: Allocator> Drop for Box<T, A> {
    fn drop(&mut self) {
        // FIXME: Do nothing, drop is currently performed by compiler.
    }
}
```
以上是Box的最常用的创建方法的代码。对于所有的堆申请，申请后的内存变量类型是MaybeUninit<T>，然后对MaybeUninit<T>用ptr::write完成初始化，随后再assume_init进入正常变量状态，这是rust的基本套路。

Box<T>的Pin方法：
```rust
impl<T> Box<T> {
    //如果T没有实现Unpin Trait, 则内存不会移动
    pub fn pin(x: T) -> Pin<Box<T>> {
        //任意的指针可以Into到Pin,因为Pin实现了任意类型的可变引用的From trait
        (box x).into()
    }
    ...
}
impl<T:?Sized> Box<T> {}
    pub fn into_pin(boxed: Self) -> Pin<Self>
    where
        A: 'static,
    {
        unsafe { Pin::new_unchecked(boxed) }
    }
    ...
}
//不限定堆内存分配器的更加通用的方法实现
impl<T, A: Allocator> Box<T, A> {
    //生成Box<T>后，在用Into<Pin> Trait生成Pin<Box>
    pub fn pin_in(x: T, alloc: A) -> Pin<Self>
    where
        A: 'static,
    {
        Self::new_in(x, alloc).into()
    }

    ...
}
```
Box<[T]>的方法：
```rust
impl<T,A:Allocator> Box<T, A> {
    //切片
    pub fn into_boxed_slice(boxed: Self) -> Box<[T], A> {
        //要转换指针类型，需要先得到裸指针
        let (raw, alloc) = Box::into_raw_with_allocator(boxed);
        //将裸指针转换为切片裸指针，再生成Box, 此处因为不知道长度，
        //只能转换成长度为1的切片指针
        unsafe { Box::from_raw_in(raw as *mut [T; 1], alloc) }
    }
    ...
}

impl<T, A: Allocator> Box<[T], A> {
    //使用RawVec作为底层堆内存管理结构，并转换为Box
    pub fn new_uninit_slice_in(len: usize, alloc: A) -> Box<[mem::MaybeUninit<T>], A> {
        unsafe { RawVec::with_capacity_in(len, alloc).into_box(len) }
    }

    //内存清零
    pub fn new_zeroed_slice_in(len: usize, alloc: A) -> Box<[mem::MaybeUninit<T>], A> {
        unsafe { RawVec::with_capacity_zeroed_in(len, alloc).into_box(len) }
    }
}
impl<T, A: Allocator> Box<[mem::MaybeUninit<T>], A> {
    //初始化完毕， 
    pub unsafe fn assume_init(self) -> Box<[T], A> {
        let (raw, alloc) = Box::into_raw_with_allocator(self);
        unsafe { Box::from_raw_in(raw as *mut [T], alloc) }
    }
}
```
其他方法及trait:
```rust
impl<T: Default> Default for Box<T> {
    /// Creates a `Box<T>`, with the `Default` value for T.
    fn default() -> Self {
        box T::default()
    }
}

impl<T,A:Allocator> Box<T, A> {
    //消费掉Box，获取内部变量
    pub fn into_inner(boxed: Self) -> T {
        //对Box的*操作就是完成Box接口从堆内存到栈内存拷贝
        //然后调用Box的drop, 返回栈内存。编译器内置的操作
        *boxed
    }
    ...
}
```
以上即为Box<T>创建及析构的所有相关代码，其中较难理解的是leak方法。在RUST中，惯例对内存申请一般会使用Box<T>来实现，如果需要将申请的内存以另外的智能指针结构做封装，则调用Box::leak将堆指针传递出来
## RawVec<T>代码分析

RawVec<T>用于指向一块从堆内存申请出来的某一类型数据的数组buffer，可以未初始化或初始化为零。与数组有关的智能指针底层的内存申请基本上都采用了RawVec<T>
RawVec<T>的结构体，创建及Drop相关方法：
```rust
enum AllocInit {
    /// 内存块没有初始化
    Uninitialized,
    /// 内存块被初始化为0
    Zeroed,
}
pub(crate) struct RawVec<T, A: Allocator = Global> {
    //指向内存地址
    ptr: Unique<T>,
    //内存块中含有T类型变量的数目
    cap: usize,
    //Allocator 变量
    alloc: A,
}

impl<T> RawVec<T, Global> {
    //语法上的要求，一些const fn 只能调用const fn，所以这里设定了一个const 变量
    pub const NEW: Self = Self::new();

    // 一些创建方法，但仅仅是对其他函数调用，代码略
    pub const fn new() -> Self;
    pub fn with_capacity(capacity: usize) -> Self;
    pub fn with_capacity_zeroed(capacity: usize) -> Self;
}

impl<T, A: Allocator> RawVec<T, A> {
    // 最少申请的容量大小
    const MIN_NON_ZERO_CAP: usize = if mem::size_of::<T>() == 1 {
        8
    } else if mem::size_of::<T>() <= 1024 {
        4
    } else {
        1
    };

    //设置一个内存块大小为0的变量
    pub const fn new_in(alloc: A) -> Self {
        // `cap: 0` means "unallocated". zero-sized types are ignored.
        Self { ptr: Unique::dangling(), cap: 0, alloc }
    }

    //申请给定容量的内存块，内存块未初始化
    pub fn with_capacity_in(capacity: usize, alloc: A) -> Self {
        //见后继说明
        Self::allocate_in(capacity, AllocInit::Uninitialized, alloc)
    }

    //申请给定容量的内存块，内存块初始化为全零
    pub fn with_capacity_zeroed_in(capacity: usize, alloc: A) -> Self {
        Self::allocate_in(capacity, AllocInit::Zeroed, alloc)
    }

    //堆内存申请函数
    fn allocate_in(capacity: usize, init: AllocInit, alloc: A) -> Self {
        //ZST的类型不用申请
        if mem::size_of::<T>() == 0 {
            Self::new_in(alloc)
        } else {
            //获取T类型的layout,注意是用array类型来获取整个size
            let layout = match Layout::array::<T>(capacity) {
                Ok(layout) => layout,
                Err(_) => capacity_overflow(),
            };
            //看堆内存是否有足够的空间
            match alloc_guard(layout.size()) {
                Ok(_) => {}
                Err(_) => capacity_overflow(),
            }
            //申请内存返回是NonNull<[u8]>，NonNull<[u8]>包含了长度信息
            let result = match init {
                AllocInit::Uninitialized => alloc.allocate(layout),
                AllocInit::Zeroed => alloc.allocate_zeroed(layout),
            };
            //处理可能的错误
            let ptr = match result {
                Ok(ptr) => ptr,
                Err(_) => handle_alloc_error(layout),
            };

            Self {
                //直接将NonNull<[u8]>转化为NonNull<T>,再转换为 *mut T
                //再生成Unique<T>, 注意*mut T此时没有长度信息
                ptr: unsafe { Unique::new_unchecked(ptr.cast().as_ptr()) },
                //用申请的字节数，ptr不要和上面一行的ptr搞混掉。
                //NonNull<[u8]>附带有长度信息
                cap: Self::capacity_from_bytes(ptr.len()),
                alloc,
            }
        }
    }

    //由元数据直接生成，调用代码需要保证输入参数是正确的
    pub unsafe fn from_raw_parts_in(ptr: *mut T, capacity: usize, alloc: A) -> Self {
        Self { ptr: unsafe { Unique::new_unchecked(ptr) }, cap: capacity, alloc }
    }

    //返回与allocator申请到的一致的内存变量
    fn current_memory(&self) -> Option<(NonNull<u8>, Layout)> {
        if mem::size_of::<T>() == 0 || self.cap == 0 {
            None
        } else {
            // We have an allocated chunk of memory, so we can bypass runtime
            // checks to get our current layout.
            unsafe {
                let align = mem::align_of::<T>();
                let size = mem::size_of::<T>() * self.cap;
                let layout = Layout::from_size_align_unchecked(size, align);
                Some((self.ptr.cast().into(), layout))
            }
        }
    }
    ...
}
//may_dangle指明T中在释放的时候有可能会出现悬垂指针，但保证不会对悬垂指针做访问，编译器可以放宽strictly alive的规则，
//PhantomData<T>会针对T类型取消掉may_dangle的作用
unsafe impl<#[may_dangle] T, A: Allocator> Drop for RawVec<T, A> {
    /// .
    fn drop(&mut self) {
        if let Some((ptr, layout)) = self.current_memory() {
            //释放内存
            unsafe { self.alloc.deallocate(ptr, layout) }
        }
    }
}
```
RawVec转换为Box<[T],A>:
```rust
impl<T, A: Allocator> RawVec<T, A> {
    //将内存块中0到len-1之间的内存块，转换为Box<[MaybeUninit<T>]>类型，len应该小于self.capacity,
    //由调用者保证
    pub unsafe fn into_box(self, len: usize) -> Box<[MaybeUninit<T>], A> {
        debug_assert!(
            len <= self.capacity(),
            "`len` must be smaller than or equal to `self.capacity()`"
        );
        
        //RUST不再对self做drop调用
        let me = ManuallyDrop::new(self);
        unsafe {
            //me作为解引用，获取ptr, 然后直接将裸指针强制转换为MaybeUninit<T>，
            //生成slice的可变引用
            let slice = slice::from_raw_parts_mut(me.ptr() as *mut MaybeUninit<T>, len);
            //用Box::from_raw_in生成Box<[MaybeUninit<T>]>, 注意这里需要对me.alloc做个拷贝
            //因为me已经被forget，所以不能再用原先的alloc.
            Box::from_raw_in(slice, ptr::read(&me.alloc))
        }
    }
```
RawVec<T>内部成员获取方法：
```rust
    pub fn ptr(&self) -> *mut T {
        self.ptr.as_ptr()
    }

    pub fn capacity(&self) -> usize {
        if mem::size_of::<T>() == 0 { usize::MAX } else { self.cap }
    }

    pub fn allocator(&self) -> &A {
        &self.alloc
    }
```
RawVec内存空间预留，扩充，收缩相关方法：
```rust

    //保留空间，确保申请的内存大小满足输入参数的规定，否则的话，扩充内存
    pub fn reserve(&mut self, len: usize, additional: usize) {
        #[cold]
        fn do_reserve_and_handle<T, A: Allocator>(
            slf: &mut RawVec<T, A>,
            len: usize,
            additional: usize,
        ) {
            handle_reserve(slf.grow_amortized(len, additional));
        }

        if self.needs_to_grow(len, additional) {
            do_reserve_and_handle(self, len, additional);
        }
    }

    /// The same as `reserve`, but returns on errors instead of panicking or aborting.
    pub fn try_reserve(&mut self, len: usize, additional: usize) -> Result<(), TryReserveError> {
        if self.needs_to_grow(len, additional) {
            self.grow_amortized(len, additional)
        } else {
            Ok(())
        }
    }

    pub fn reserve_exact(&mut self, len: usize, additional: usize) {
        handle_reserve(self.try_reserve_exact(len, additional));
    }

    pub fn try_reserve_exact(
        &mut self,
        len: usize,
        additional: usize,
    ) -> Result<(), TryReserveError> {
        if self.needs_to_grow(len, additional) { self.grow_exact(len, additional) } else { Ok(()) }
    }

    //收缩空间置给定大小
    pub fn shrink_to_fit(&mut self, amount: usize) {
        handle_reserve(self.shrink(amount));
    }
}

impl<T, A: Allocator> RawVec<T, A> {
    //判断内存块空间是否足够
    fn needs_to_grow(&self, len: usize, additional: usize) -> bool {
        //wrapping_sub防止溢出
        additional > self.capacity().wrapping_sub(len)
    }
    
    //从字节数得出内存块数目
    fn capacity_from_bytes(excess: usize) -> usize {
        debug_assert_ne!(mem::size_of::<T>(), 0);
        excess / mem::size_of::<T>()
    }
    
    //根据NonNull来设置结构体ptr及容量
    fn set_ptr(&mut self, ptr: NonNull<[u8]>) {
        //ptr.cast会转换NonNull<[u8]>到NonNull<T>
        self.ptr = unsafe { Unique::new_unchecked(ptr.cast().as_ptr()) };
        //由字节数获得内存块数目
        self.cap = Self::capacity_from_bytes(ptr.len());
    }

    // 增长到满足len+additional的空间，
    fn grow_amortized(&mut self, len: usize, additional: usize) -> Result<(), TryReserveError> {
        // This is ensured by the calling contexts.
        debug_assert!(additional > 0);

        if mem::size_of::<T>() == 0 {
            return Err(CapacityOverflow.into());
        }

        // 计算需要的容量值，不能超过usize::MAX.
        let required_cap = len.checked_add(additional).ok_or(CapacityOverflow)?;

        // 每次以2的指数递增，且不能小于最小内存容量
        //  `cap <= isize::MAX` and the type of `cap` is `usize`.
        let cap = cmp::max(self.cap * 2, required_cap);
        let cap = cmp::max(Self::MIN_NON_ZERO_CAP, cap);

        //重新计算内存大小
        let new_layout = Layout::array::<T>(cap);

        // 见后文.
        let ptr = finish_grow(new_layout, self.current_memory(), &mut self.alloc)?;
        //更新ptr及cap
        self.set_ptr(ptr);
        Ok(())
    }

    // 与`grow_amortized`基本一致。只是要正好是len+additional的大小
    fn grow_exact(&mut self, len: usize, additional: usize) -> Result<(), TryReserveError> {
        if mem::size_of::<T>() == 0 {
            // Since we return a capacity of `usize::MAX` when the type size is
            // 0, getting to here necessarily means the `RawVec` is overfull.
            return Err(CapacityOverflow.into());
        }

        let cap = len.checked_add(additional).ok_or(CapacityOverflow)?;
        let new_layout = Layout::array::<T>(cap);

        // `finish_grow` is non-generic over `T`.
        let ptr = finish_grow(new_layout, self.current_memory(), &mut self.alloc)?;
        self.set_ptr(ptr);
        Ok(())
    }
    
    //收缩内存到amount长度
    fn shrink(&mut self, amount: usize) -> Result<(), TryReserveError> {
        assert!(amount <= self.capacity(), "Tried to shrink to a larger capacity");

        let (ptr, layout) = if let Some(mem) = self.current_memory() { mem } else { return Ok(()) };
        let new_size = amount * mem::size_of::<T>();

        let ptr = unsafe {
            let new_layout = Layout::from_size_align_unchecked(new_size, layout.align());
            //利用Allcator的函数完成内存申请，拷贝原有内容，并释放原内存
            self.alloc
                .shrink(ptr, layout, new_layout)
                .map_err(|_| AllocError { layout: new_layout, non_exhaustive: () })?
        };
        //更换指针和容量，这里虽然更换了self的内容，但没有改变编译器对self的所有权的认识
        self.set_ptr(ptr);
        Ok(())
    }
}
//内存增长具体实现
fn finish_grow<A>(
    new_layout: Result<Layout, LayoutError>,
    current_memory: Option<(NonNull<u8>, Layout)>,
    alloc: &mut A,
) -> Result<NonNull<[u8]>, TryReserveError>
where
    A: Allocator,
{
    // 检查new_layout是否为错误
    let new_layout = new_layout.map_err(|_| CapacityOverflow)?;
    //确保新的Layout的大小不引发异常
    alloc_guard(new_layout.size())?;

    let memory = if let Some((ptr, old_layout)) = current_memory {
        //原先已经申请过内存
        debug_assert_eq!(old_layout.align(), new_layout.align());
        unsafe
            // The allocator checks for alignment equality
            intrinsics::assume(old_layout.align() == new_layout.align());
            //调用Allocator的grow函数增长内存
            alloc.grow(ptr, old_layout, new_layout)
        }
    } else {
        //原先未申请过内存
        alloc.allocate(new_layout)
    };

    memory.map_err(|_| AllocError { layout: new_layout, non_exhaustive: () }.into())
}

fn handle_reserve(result: Result<(), TryReserveError>) {
    match result.map_err(|e| e.kind()) {
        Err(CapacityOverflow) => capacity_overflow(),
        Err(AllocError { layout, .. }) => handle_alloc_error(layout),
        Ok(()) => { /* yay */ }
    }
}

fn alloc_guard(alloc_size: usize) -> Result<(), TryReserveError> {
    if usize::BITS < 64 && alloc_size > isize::MAX as usize {
        Err(CapacityOverflow.into())
    } else {
        Ok(())
    }
}

fn capacity_overflow() -> ! {
    panic!("capacity overflow");
}
```

## Cow写时复制结构解析

与Borrow Trait互为逆的ToOwned Trait。 一般满足 T.borrow() 返回 &U，  U.to_owned()返回T
```rust
pub trait ToOwned {
    // 必须实现Borrow<Self> Trait， Owned.borrow()->&Self
    type Owned: Borrow<Self>;

    // 从本类型生成Owned类型，一般由指针生成原始变量
    fn to_owned(&self) -> Self::Owned;
    
    //替换target的内容，原内容会被drop掉
    fn clone_into(&self, target: &mut Self::Owned) {
        *target = self.to_owned();
    }
}

impl<T> ToOwned for T
//实现了Clone的类型自然实现ToOwned
where
    T: Clone
{
    type Owned = T;
    fn to_owned(&self) -> T {
        //创建一个新的T类型的变量
        self.clone()
    }

    fn clone_into(&self, target: &mut T) {
        target.clone_from(self);
    }
}
```
Cow解决一类复制问题: 在与原有变量没有变化时使用原有变量的引用来访问变量，当发生变化时，完成对变量的复制生成新的变量。
```rust
pub enum Cow<'a, B: ?Sized + 'a>
where
    B: ToOwned,
{
    /// 初始类型变量borrow后的指针
    Borrowed( &'a B),

    ///当对初始变量修改后，会对初始变量做克隆，然后成为Owned变量
    Owned(<B as ToOwned>::Owned),
}
```
Cow的创建一般用`let a = Cow::Borrowed(&T)这种方式直接完成，因为是写时复制，所以需要用Borrowed()来得到初始值，否则不符合语义要求。

典型的trait实现：
```rust
//解引用，会返回&B
impl<B: ?Sized + ToOwned> const Deref for Cow<'_, B>
where
    B::Owned: ~const Borrow<B>,
{
    type Target = B;

    fn deref(&self) -> &B {
        match *self {
            //如果是原有的变量，则返回原有变量引用
            Borrowed(borrowed) => borrowed,
            //如果值已经被修改，则返回包装变量的引用
            Owned(ref owned) => owned.borrow(),
        }
    }
}

//实现Borrow Trait
impl<'a, B: ?Sized> Borrow<B> for Cow<'a, B>
where
    B: ToOwned,
    <B as ToOwned>::Owned: 'a,
{
    fn borrow(&self) -> &B {
        //利用deref来返回
        &**self
    }
}

// Clone的实现，赋值的同样是一个Cow结构，需要满足写时复制的要求。
impl<B: ?Sized + ToOwned> Clone for Cow<'_, B> {
    fn clone(&self) -> Self {
        match *self {
            //这里是引用，满足Copy，直接赋值
            Borrowed(b) => Borrowed(b),
            //o的类型无法确定支持Copy或Clone，因此需要先得到B，然后由B
            //调用一个to_owned获得O的拷贝，对于支持Clone的类型自然支持
            // ToOwned
            Owned(ref o) => {
                let b: &B = o.borrow();
                Owned(b.to_owned())
            }
        }
    }

    fn clone_from(&mut self, source: &Self) {
        match (self, source) {
            //仅在双方都为Owned的情况下要先borrow后再复制，注意，此时self的原dest生命周期终止
            (&mut Owned(ref mut dest), &Owned(ref o)) => o.borrow().clone_into(dest),
            (t, s) => *t = s.clone(),
        }
    }
}
```
Cow<'a, T>的一些方法
```rust
impl<B: ?Sized + ToOwned> Cow<'_, B> {
    pub const fn is_borrowed(&self) -> bool {
        match *self {
            Borrowed(_) => true,
            Owned(_) => false,
        }
    }

    pub const fn is_owned(&self) -> bool {
        !self.is_borrowed()
    }

    //这个函数说明要对变量进行改变，因此，如果还是原变量的引用，则需要做复制操作
    pub fn to_mut(&mut self) -> &mut <B as ToOwned>::Owned {
        match *self {
            Borrowed(borrowed) => {
                //复制操作，复制原变量后，然后用Owned包装
                *self = Owned(borrowed.to_owned());
                match *self {
                    Borrowed(..) => unreachable!(),
                    Owned(ref mut owned) => owned,
                }
            }
            Owned(ref mut owned) => owned,
        }
    }

    //此函数也说明后继会修改，但会消费掉Cow
    pub fn into_owned(self) -> <B as ToOwned>::Owned {
        match self {
            Borrowed(borrowed) => borrowed.to_owned(),
            Owned(owned) => owned,
        }
    }
}

//由slice生成Cow的代码例
impl<'a, T: Clone> From<&'a [T]> for Cow<'a, [T]> {
    fn from(s: &'a [T]) -> Cow<'a, [T]> {
        //一般是先生成
        Cow::Borrowed(s)
    }
}
```
从Cow<'a, T>可以看到RUST基础语法的强大能力，大家可以思考一下如何用其他语言来实现这一写时复制的类型，会发现很难实现。
## Vec 分析
动态数组，结构体及创建，析构方法相关：
```rust
pub struct Vec<T, A: Allocator = Global> {
    //RawVec作为内存基础，这个是整体的容量
    buf: RawVec<T, A>,
    //数组元素个数，这是已经使用的成员的数目
    len: usize,
}

macro_rules! vec {
    () => (
        $crate::vec::Vec::new()
    );
    ($elem:expr; $n:expr) => (
        $crate::vec::from_elem($elem, $n)
    );
    ($($x:expr),*) => (
        //首先生成Box<[T;N]>，然后利用slice的into_vec生成Vec<T>
        $crate::slice::into_vec(box [$($x),*])
    );
    //这里实际上就是完成($x,)=>$x。去掉了','号。 
    ($($x:expr,)*) => (vec![$($x),*])
}

impl<T, A: Allocator> ops::Deref for Vec<T, A> {
    type Target = [T];

    fn deref(&self) -> &[T] {
        unsafe { slice::from_raw_parts(self.as_ptr(), self.len) }
    }
}

impl<T, A: Allocator> ops::DerefMut for Vec<T, A> {
    fn deref_mut(&mut self) -> &mut [T] {
        unsafe { slice::from_raw_parts_mut(self.as_mut_ptr(), self.len) }
    }
}
//Vec<T>的Index下标实现,实际上就是切片Index实现
impl<T, I: SliceIndex<[T]>, A: Allocator> Index<I> for Vec<T, A> {
    type Output = I::Output;

    
    fn index(&self, index: I) -> &Self::Output {
        //&**self会将Vec转换为&[T]
        Index::index(&**self, index)
    }
}

impl<T, I: SliceIndex<[T]>, A: Allocator> IndexMut<I> for Vec<T, A> {
    fn index_mut(&mut self, index: I) -> &mut Self::Output {
        IndexMut::index_mut(&mut **self, index)
    }
}

impl<T> Vec<T> {
    pub const fn new() -> Self {
        //初始化buf为空
        Vec { buf: RawVec::NEW, len: 0 }
    }
    //其他创建函数，因为仅仅是对其他函数的封装调用，代码略
    pub fn with_capacity(capacity: usize) -> Self;
    pub unsafe fn from_raw_parts(ptr: *mut T, length: usize, capacity: usize) -> Self;
}

//由Box转换为Vec，这是RUST的最令人无语的地方，内存安全导致必须对类型做各种其他语言不需要的复杂的变换
pub fn into_vec<T, A: Allocator>(b: Box<[T], A>) -> Vec<T, A> {
    unsafe {
        let len = b.len();
        let (b, alloc) = Box::into_raw_with_allocator(b);
        Vec::from_raw_parts_in(b as *mut T, len, len, alloc)
    }
}

//所有支持SpecFromElem trait的类型可以直接转换生成n个初始值为elem的Vec动态数组
pub fn from_elem_in<T: Clone, A: Allocator>(elem: T, n: usize, alloc: A) -> Vec<T, A> {
    <T as SpecFromElem>::from_elem(elem, n, alloc)
}

pub(super) trait SpecFromElem: Sized {
    fn from_elem<A: Allocator>(elem: Self, n: usize, alloc: A) -> Vec<Self, A>;
}
//所有实现了Clone的类型自然支持SpecFromElem trait
impl<T: Clone> SpecFromElem for T {
    default fn from_elem<A: Allocator>(elem: Self, n: usize, alloc: A) -> Vec<Self, A> {
        //见下文分析
        let mut v = Vec::with_capacity_in(n, alloc);
        v.extend_with(n, ExtendElement(elem));
        v
    }
}
//drop方法
unsafe impl<#[may_dangle] T, A: Allocator> Drop for Vec<T, A> {
    fn drop(&mut self) {
        unsafe {
            //这里的drop_in_place调用会引发Vec<T>内部的成员变量自身的drop，所以只drop有意义的值
            //成员变量有些可能已经被释放过，会出席悬垂指针，所以用may_dangle来通知编译器
            ptr::drop_in_place(ptr::slice_from_raw_parts_mut(self.as_mut_ptr(), self.len))
        }
        //会自动调用RawVec的drop释放堆内存
    }
}

impl<T, A: Allocator> Vec<T, A> {
    //对RawVec做初始化，实际是空值
    pub const fn new_in(alloc: A) -> Self {
        Vec { buf: RawVec::new_in(alloc), len: 0 }
    }

    //具体见RawVec的函数说明，这里创建了一个容量为输入参数的RawVec,
    //但Vec本身的长度为0，标示成员都还没有初始化
    pub fn with_capacity_in(capacity: usize, alloc: A) -> Self {
        Vec { buf: RawVec::with_capacity_in(capacity, alloc), len: 0 }
    }

    //利用原始数据生成Vec，调用代码应该保证安全性：
    // *mut T应该是从堆申请的，符合RawVec<T>申请规则的大小和对齐，并且是用alloc来做的申请 
    pub unsafe fn from_raw_parts_in(ptr: *mut T, length: usize, capacity: usize, alloc: A) -> Self {
        unsafe { Vec { buf: RawVec::from_raw_parts_in(ptr, capacity, alloc), len: length } }
    }

    //生成原始数据，此处首先要使得self被禁止drop，
    //一般后继利用这些原始数据生成新的RawVec，重新启用新RawVec的Drop
    pub fn into_raw_parts(self) -> (*mut T, usize, usize) {
        let mut me = ManuallyDrop::new(self);
        //me自动解引用后得到Vec
        (me.as_mut_ptr(), me.len(), me.capacity())
    }

    //生成原始数据，包括alloc的变量
    pub fn into_raw_parts_with_alloc(self) -> (*mut T, usize, usize, A) {
        let mut me = ManuallyDrop::new(self);
        let len = me.len();
        let capacity = me.capacity();
        let ptr = me.as_mut_ptr();
        let alloc = unsafe { ptr::read(me.allocator()) };
        (ptr, len, capacity, alloc)
    }

    pub fn capacity(&self) -> usize {
        //RawVec的capacity
        self.buf.capacity()
    }
    pub fn allocator(&self) -> &A {
        self.buf.allocator()
    }
    pub fn len(&self) -> usize {
        self.len
    }
    //极度不安全，最好不要用这个函数改变len
    pub unsafe fn set_len(&mut self, new_len: usize) {
        debug_assert!(new_len <= self.capacity());

        self.len = new_len;
    }
    pub fn is_empty(&self) -> bool {
        self.len() == 0
    }
```
Vec容量相关方法：
```rust
    //在当前的len的基础上扩张输入的参数的内存容量
    //不一定会出发对内存的重新申请，因为RawVec的容量可能是够的
    //容量不能超出usize::MAX
    pub fn reserve(&mut self, additional: usize) {
        self.buf.reserve(self.len, additional);
    }

    //精确的扩张容量
    pub fn reserve_exact(&mut self, additional: usize) {
        self.buf.reserve_exact(self.len, additional);
    }

    //如果reserve不成功，返回错误类型
    pub fn try_reserve(&mut self, additional: usize) -> Result<(), TryReserveError> {
        self.buf.try_reserve(self.len, additional)
    }

    //精确的容量
    pub fn try_reserve_exact(&mut self, additional: usize) -> Result<(), TryReserveError> {
        self.buf.try_reserve_exact(self.len, additional)
    }

    //收缩内部buf容量到正好是Vec长度
    pub fn shrink_to_fit(&mut self) {
        if self.capacity() > self.len {
            self.buf.shrink_to_fit(self.len);
        }
    }

    //收缩容量
    pub fn shrink_to(&mut self, min_capacity: usize) {
        if self.capacity() > min_capacity {
            self.buf.shrink_to_fit(cmp::max(self.len, min_capacity));
        }
    }
    //在有变量存在的情况下做收缩
    pub fn truncate(&mut self, len: usize) {
        unsafe {
            if len > self.len {
                return;
            }
            //计算需要删除的容量
            let remaining_len = self.len - len;
            //形成需要删除的部分的切片类型
            let s = ptr::slice_from_raw_parts_mut(self.as_mut_ptr().add(len), remaining_len);
            //修改Vec的长度。
            self.len = len;
            //调用drop_in_place，使得切片能够对内部的成员调用drop以完成删除
            //注意，此时不涉及Vec内部的buf删除，仅仅是删除Vec的成员
            ptr::drop_in_place(s);
        }
    }
```
将Vec<T>转换成其他类型
```rust
    //转换为Box<[T], A>类型。
    pub fn into_boxed_slice(mut self) -> Box<[T], A> {
        unsafe {
            //此处重要，进入Box后，堆内存的容量必须是切片长度的内存，否则释放会引发问题
            self.shrink_to_fit();
            //本Vec的Drop需要禁止，由Box负责内存释放
            let me = ManuallyDrop::new(self);
            //这里将RawVec做了一个拷贝，实际上是将RawVec所有权转移出来，必须的
            //拷贝是效率高的做法
            let buf = ptr::read(&me.buf);
            let len = me.len();
            //用RawVec生成Box
            buf.into_box(len).assume_init()
        }
    }

    pub fn as_slice(&self) -> &[T] {
        //会自动解引用
        self
    }

    pub fn as_mut_slice(&mut self) -> &mut [T] {
        self
    }

    pub fn as_ptr(&self) -> *const T {
        let ptr = self.buf.ptr();
        unsafe {
            assume(!ptr.is_null());
        }
        ptr
    }

    pub fn as_mut_ptr(&mut self) -> *mut T {
        let ptr = self.buf.ptr();
        unsafe {
            assume(!ptr.is_null());
        }
        ptr
    }
```
插入与删除方法：
```rust
    //在index的位置插入一个变量
    pub fn insert(&mut self, index: usize, element: T) {
        #[cold]
        #[inline(never)]
        fn assert_failed(index: usize, len: usize) -> ! {
            panic!("insertion index (is {}) should be <= len (is {})", index, len);
        }
        //如果index大于len，出错
        let len = self.len();
        if index > len {
            assert_failed(index, len);
        }

        //如果预留的空间不够，则至少扩充1个成员空间
        if len == self.buf.capacity() {
            self.reserve(1);
        }

        unsafe {
            {
                //获得index的成员内存地址
                let p = self.as_mut_ptr().add(index);
                //此处将index之后所有成员内存向后偏移一个地址，最高的效率
                ptr::copy(p, p.offset(1), len - index);
                //将变量写入index的成员地址
                ptr::write(p, element);
            }
            //修改长度
            self.set_len(len + 1);
        }
    }

    pub fn remove(&mut self, index: usize) -> T {
        #[cold]
        #[inline(never)]
        #[track_caller]
        fn assert_failed(index: usize, len: usize) -> ! {
            panic!("removal index (is {}) should be < len (is {})", index, len);
        }

        //如果index大于Vec的长度，出错
        let len = self.len();
        if index >= len {
            assert_failed(index, len);
        }
        unsafe {
            let ret;
            {
                // 得到index的成员地址
                let ptr = self.as_mut_ptr().add(index);
                // 将成员变量拷贝出来，并转移了所有权
                ret = ptr::read(ptr);

                // 将index+1后的所有成员内存拷贝到前面一个地址
                ptr::copy(ptr.offset(1), ptr, len - index - 1);
            }
            //改变长度
            self.set_len(len - 1);
            //将删除的变量及所有权返回。
            ret
        }
    }

    //在尾部插入一个元素
    pub fn push(&mut self, value: T) {
        //如果预留空间不够，则扩充一个空间
        if self.len == self.buf.capacity() {
            self.reserve(1);
        }
        unsafe {
            //获取当前尾部成员后面的内存地址
            let end = self.as_mut_ptr().add(self.len);
            //将变量写入内存地址
            ptr::write(end, value);
            //长度加1
            self.len += 1;
        }
    }
     
    //取出尾部成员
    pub fn pop(&mut self) -> Option<T> {
        if self.len == 0 {
            None
        } else {
            unsafe {
                self.len -= 1;
                //将尾部成员变量读出并连同所有权共同返回，此处因为self.len已经减1，后继drop时不会再对
                //原尾部成员drop。所以尾部成员的所有权已经被处理掉了
                Some(ptr::read(self.as_ptr().add(self.len())))
            }
        }
    }

    //删除所有成员
    pub fn clear(&mut self) {
        //重用
        self.truncate(0)
    }
    
    //leak方法，此方法后，需要再次将返回值转换到RavVec，否则会内存泄漏
    pub fn leak<'a>(self) -> &'a mut [T]
    where
        A: 'a,
    {
        //本Vec变量不再被drop
        let mut me = ManuallyDrop::new(self);
        //生成可变切片引用，此引用没有后继处理的话会造成内存泄漏
        unsafe { slice::from_raw_parts_mut(me.as_mut_ptr(), me.len) }
    }
    ...
}
```
Vec<T>的所有难点实际上都在RUST的各种指针类型转换及内存读写的所有权处理上。这也是所有的RUST的数据结构基础类型实现上相对比其他语言需要额外理解的复杂之处。    


# RUST不采用类的理
面对对象语言如C++,Java以类为中心，并以类的继承为最基本的一个设计理念。但继承实际上是极易被错误使用，并导致整个软件结构混乱的根源之一。
类起源基于C的struct总结出来的良好的设计模式：
```rust
struct xxx {
    ...
    int (*fn1)(struct xxx *self, ...);
    void (*fn2)(struct xxx *self, ...);
}
```
这个模式中，利用struct内部的函数指针实现了不同实体可以对外提供若干相同的方法，更好的实现了抽象。
以类为基础的语言，强化了成员和方法组合成对象的设计理念。用继承的方式来实现函数指针的多态性。但如果认真分析，会发现函数指针的多态性实际上更匹配的实现是在类中定义一个接口类的变量，变化这个变量来实现多态性。二十三种设计模式就充分的展示了这一原则。也就是常说的"使用组合而不是继承"

为了去除继承的滥用，RUST，Go等语言都不再使用类作为语言核心。取代的是使用结构+接口的模式。继承只能在接口进行。

RUST接口类的类似实现:
1.单元结构体，程序中使用直接用单元结构体的名字，实例即其本身。
2.可以用实现接口（Trait）的单元结构体来实现象C++和Java一样纯函数集的接口类。再利用结构内部定义dyn Trait变量的形式实现组合模式

举个例子
```rust
pub trait SomeTrait {
    fn1(...)->i32;
    fn2(...);
}
pub struct xxx {
    ...
    behavior : dyn Trait;
}

pub struct RealizeSomeTrait1;
impl SomeTrait for RealizeSomeTrait1 { fn1(...)->i32 {}；fn2(...){}}
let A = xxx<RealizeSomeTrait1>{   };
let A = xxx {.., behavior : RealizeSomeTrait1}
```
如果用泛型，则struct xxx实质会变成多个类型，无法对调用它的代码完成抽象化；用dyn Trait则不存在这个问题。
