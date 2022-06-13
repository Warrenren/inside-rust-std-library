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
%USER%\.  rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\alloc\src， 
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
如果不想转移所有权，那上面代码的match就应该是一个从正常语法理解的写法，对结构内部的变量也需要用引用来绑定，尤其是结构内部变量如果没有实现Copy Trait，那就必须用引用，否则也会引发编译告警。

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
这个是rust的标准写法，rust专门做了语法上的设计，如果不清楚这个设计，很可能会对这里的类型绑定感到疑惑。
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
代码路径：
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\core\src\pin.rs  

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
代码路径：
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\core\src\lazy.rs  

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
代码路径：
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\alloc\src\boxed.rs  

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
代码路径：
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\alloc\src\raw_vec.rs  

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

## Cow<T>写时复制结构解析
代码路径：
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\alloc\src\borrow.rs  

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
## Vec<T> 分析
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

## `Rc<T>` 分析
`Rc<T>`主要解决堆内存多份借用的情况，相比于`&Box<T>`的解决方案，`Rc<T>`可以基本上不用考虑生命周期导致的编码负担。同时利用伴生的`Weak<T>`解决了变量相互之间的循环引用问题。
之所以申请堆内存，一个原因便是被系统中多个模块所共享，所以`Rc<T>`是更常用的堆内存智能指针类型。
   
`Rc<T>`解决了两个数据结构互指的情况，这在结构体组合，循环链表，树，图的数据结构中都有大量的应用。
`Rc<T>`的结构定义如下：
 ```rust
//在堆内存申请的结构体
//注意，这里使用了C语言的内存布局，在内存中的成员的顺序必须按照声明的顺序排列
#[repr(C)]
 struct RcBox<T: ?Sized> {
     //拥有所有权的智能指针Rc<T>的计数
     strong: Cell<usize>,
     //不拥有所有权的智能指针Weak<T>的计数
     weak: Cell<usize>,
     value: T,
 }
 
 //和Unique<T>类似
 pub struct Rc<T: ?Sized> {
     //堆内存块的指针
     ptr: NonNull<RcBox<T>>,
     //表示拥有内存块的所有权，内存块由本结构释放
     phantom: PhantomData<RcBox<T>>,
 }
//没有堆内存块的所有权
pub struct Weak<T: ?Sized> {
    ptr: NonNull<RcBox<T>>,
}
 ```
 在创建两个结构体变量互指的指针时，会遇到生命周期陷阱，无论先释放那个结构变量，都会导致另外那个结构变量出现悬垂指针。但如果在代码中时刻关注这种情况，那就太不RUST了。
 为此， `Rc<T>`提供了weak和strong两种堆内存指针的方式，`Rc<T>`申请的堆内存可以没有初始化，未初始化的堆内存可以生成`WeakT>`用于给其他结构访问堆内存。同时堆内存用strong的方式来保护`Rc<T>`在未初始化时不被读写。且weak和strong可以相互之间转换，这就以rust方式解决了生命周期陷阱问题。
利用`Weak<T>`做指针的结构体，在需要访问堆内存时，可以从`Weak<T>`另外创建`Rc<T>`, 完成访问后即可让创建的`Rc<T>`生命周期终结。实际上，各需要访问堆内存的结构体仅保留`Weak<T>`应该是一个非常好的做法。


 `Rc<T>`的创建方法及析构方法代码如下：
 ```rust
 //由结构体成员生成结构的辅助方法
 impl<T: ?Sized> Rc<T> {
     //获取内部的RcBox
     fn inner(&self) -> &RcBox<T> {
         unsafe { self.ptr.as_ref() }
     }
 
     //由成员创建结构体
     fn from_inner(ptr: NonNull<RcBox<T>>) -> Self {
         Self { ptr, phantom: PhantomData }
     }
     //由裸指针创建结构体
     unsafe fn from_ptr(ptr: *mut RcBox<T>) -> Self {
         Self::from_inner(unsafe { NonNull::new_unchecked(ptr) })
     }
 }
 
 impl<T> Rc<T> {
     //由已初始化变量创建Rc<T>
     pub fn new(value: T
     ) -> Rc<T> {
         //首先创建RcBox<T>，然后生成Box<RcBox<T>>, 随后用leak得到RcBox<T>的堆内存指针，
         //用堆内存指针创建Rc<T>，内存申请由Box<T>实际执行
         Self::from_inner(
             Box::leak(box RcBox { strong: Cell::new(1), weak: Cell::new(1), value }).into(),
         )
     }
 
     //用于创建一个互相引用场景的Rc<T>
     pub fn new_cyclic(data_fn: impl FnOnce(&Weak<T>) -> T) -> Rc<T> {
         // 下面与new函数代码类似，只是value没有初始化。
         // 因为value没有初始化，strong赋值为0，但可以支持Weak<T>的引用
         let uninit_ptr: NonNull<_> = Box::leak(box RcBox {
             strong: Cell::new(0),
             weak: Cell::new(1),
             value: mem::MaybeUninit::<T>::uninit(),
         })
         .into();
         
         //init_ptr后继会被初始化，但此时还没有
         let init_ptr: NonNull<RcBox<T>> = uninit_ptr.cast();
 
         //生成Weak
         let weak = Weak { ptr: init_ptr };
 
         // 利用回调闭包获得value的值，将weak传递出去是因为cyclic默认结构体初始化需要使用weak.
         // 用回调函数的处理可以让初始化一次完成，以免初始化以后还要修改结构体的指针。
         let data = data_fn(&weak);
 
         unsafe {
             let inner = init_ptr.as_ptr();
             //addr_of_mut!可以万无一失，写入值后，初始化已经完成
             ptr::write(ptr::addr_of_mut!((*inner).value), data);
             
             //可以更新strong的值为1了
             let prev_value = (*inner).strong.get();
             debug_assert_eq!(prev_value, 0, "No prior strong references should exist");
             (*inner).strong.set(1);
         }
 
         //strong登场
         let strong = Rc::from_inner(init_ptr);
 
         // 这里是因为strong整体拥有一个weak计数，所以此处不对wek做drop处理以维持weak计数。前面的回调函数中应该使用weak.clone增加weak的计数。
         mem::forget(weak);
         strong
     }
 
     //生成一个未初始化的Rc<T>，此时只能本身来处理内存申请的工作
     pub fn new_uninit() -> Rc<mem::MaybeUninit<T>> {
         unsafe {
             //Rc自身的内存申请函数感觉过于复杂
             Rc::from_ptr(Rc::allocate_for_layout(
                 Layout::new::<T>(),
                 |layout| Global.allocate(layout),
                 |mem| mem as *mut RcBox<mem::MaybeUninit<T>>,
             ))
         }
     }
 
     //防止内存不足的创建函数
     pub fn try_new(value: T) -> Result<Rc<T>, AllocError> {
         // 就是用Box::try_new来完成try的工作
         Ok(Self::from_inner(
             Box::leak(Box::try_new(RcBox { strong: Cell::new(1), weak: Cell::new(1), value })?)
                 .into(),
         ))
     }
 
     //对未初始化的Rc的try new
     pub fn try_new_uninit() -> Result<Rc<mem::MaybeUninit<T>>, AllocError> {
         unsafe {
             //内存申请函数需要考虑申请不到的情况
             Ok(Rc::from_ptr(Rc::try_allocate_for_layout(
                 Layout::new::<T>(),
                 //就用Global Allocator，没有考虑其他，可能会有些问题
                 |layout| Global.allocate(layout),
                 |mem| mem as *mut RcBox<mem::MaybeUninit<T>>,
             )?))
         }
     }
     ...
 }

//堆内存申请函数
impl<T: ?Sized> Rc<T> {
    unsafe fn allocate_for_layout(
        value_layout: Layout,
        allocate: impl FnOnce(Layout) -> Result<NonNull<[u8]>, AllocError>,
        mem_to_rcbox: impl FnOnce(*mut u8) -> *mut RcBox<T>,
    ) -> *mut RcBox<T> {
        // 根据T计算RcBox需要的内存块布局，注意用RcBox<()>获取仅包含strong及cell两个成员的RcBox的layout这个技巧
        let layout = Layout::new::<RcBox<()>>().extend(value_layout).unwrap().0.pad_to_align();
        unsafe {
            //要考虑不成功的可能性
            Rc::try_allocate_for_layout(value_layout, allocate, mem_to_rcbox)
                .unwrap_or_else(|_| handle_alloc_error(layout))
        }
    }

    unsafe fn try_allocate_for_layout(
        value_layout: Layout,
        allocate: impl FnOnce(Layout) -> Result<NonNull<[u8]>, AllocError>,
        mem_to_rcbox: impl FnOnce(*mut u8) -> *mut RcBox<T>,
    ) -> Result<*mut RcBox<T>, AllocError> {
        //计算需要的内存块布局layout
        let layout = Layout::new::<RcBox<()>>().extend(value_layout).unwrap().0.pad_to_align();

        // 申请内存，有可能不成功
        let ptr = allocate(layout)?;

        // 将裸指针类型内存类型转换成*mut RcBox<xxx>类型，xxx有可能是MaybeUninit<T>，但也可能是初始化完毕的类型。总之，调用代码会保证初始化，所以此处正常的设置strong及weak，
        let inner = mem_to_rcbox(ptr.as_non_null_ptr().as_ptr());
        unsafe {
            debug_assert_eq!(Layout::for_value(&*inner), layout);

            ptr::write(&mut (*inner).strong, Cell::new(1));
            ptr::write(&mut (*inner).weak, Cell::new(1));
        }

        Ok(inner)
    }
    
    //根据一个裸指针来创建RcBox<T>，返回裸指针，这个函数完成时堆内存没有初始化，后继需要写入值
    unsafe fn allocate_for_ptr(ptr: *const T) -> *mut RcBox<T> {
        unsafe {
            Self::allocate_for_layout(
                // 用*const T获取Layout
                Layout::for_value(&*ptr),
                |layout| Global.allocate(layout),
                //此处应该也可以用mem as *mut RcBox<T>，
                //目前的代码感觉比较复杂，
                |mem| (ptr as *mut RcBox<T>).set_ptr_value(mem),
            )
        }
    }

    //从Box<T>转换成RcBox<T>
    fn from_box(v: Box<T>) -> Rc<T> {
        unsafe {
            //解封装Box，获取堆内存指针
            let (box_unique, alloc) = Box::into_unique(v);
            let bptr = box_unique.as_ptr();

            let value_size = size_of_val(&*bptr);
            //获得* mut RcBox<T>
            let ptr = Self::allocate_for_ptr(bptr);

            // 将T的内容拷贝如RcBox的value
            ptr::copy_nonoverlapping(
                bptr as *const T as *const u8,
                &mut (*ptr).value as *mut _ as *mut u8,
                value_size,
            );

            // 重要，这里仅仅释放堆内存，但是如果堆内存中的T类型变量还有其他需要释放的内存，则没有处理，即没有调用drop(T)，drop(T)由新生成的RcBox<T>再释放的时候负责
            box_free(box_unique, alloc);
            
            // 生成Rc<T>
            Self::from_ptr(ptr)
        }
    }
}

//析构
unsafe impl<#[may_dangle] T: ?Sized> Drop for Rc<T> {
    //只要strong计数为零，就drop掉堆内存变量
    //Weak可以不依赖与value是否正确。
    fn drop(&mut self) {
        unsafe {
            //strong计数减1
            self.inner().dec_strong();
            if self.inner().strong() == 0 {
                // 触发堆内存变量的drop()操作
                ptr::drop_in_place(Self::get_mut_unchecked(self));

                // 对于strong整体会有一个weak计数，需要减掉
                // 这里实际上与C语言一样容易出错。
                self.inner().dec_weak();

                if self.inner().weak() == 0 {
                    //只有weak为0的时候才能够释放堆内存
                    Global.deallocate(self.ptr.cast(), Layout::for_value(self.ptr.as_ref()));
                }
            }
        }
    }
}

 ```
`Weak<T>`的结构体及创建，析构方法：
 在RC方法内部，Weak可以由`Weak{ptr:self_ptr}`直接创建，可见前面代码的例子，但要注意weak计数和Weak变量需要匹配
 ```rust

impl<T> Weak<T> {
    //创建一个空的Weak
    pub fn new() -> Weak<T> {
        Weak { ptr: NonNull::new(usize::MAX as *mut RcBox<T>).expect("MAX is not 0") }
    }
}
//判断Weak是否为空的关联函数
pub(crate) fn is_dangling<T: ?Sized>(ptr: *mut T) -> bool {
    let address = ptr as *mut () as usize;
    address == usize::MAX
}
impl <T:?Sized> Weak<T> {
    pub fn as_ptr(&self) -> *const T {
        let ptr: *mut RcBox<T> = NonNull::as_ptr(self.ptr);

        if is_dangling(ptr) {
            ptr as *const T
        } else {
            //返回T类型变量的指针
            unsafe { ptr::addr_of_mut!((*ptr).value) }
        }
    }
    //会消费掉Weak，获取T类型变量指针，此指针以后需要重新组建Weak<T>,否则
    //堆内存中的RcBox的weak会出现计数错误
    pub fn into_raw(self) -> *const T {
        let result = self.as_ptr();
        mem::forget(self);
        result
    }
    
    //ptr是从into_raw得到的返回值
    pub unsafe fn from_raw(ptr: *const T) -> Self {
        let ptr = if is_dangling(ptr as *mut T) {
            ptr as *mut RcBox<T>
        } else {
            //需要从T类型的指针恢复RcBox的指针
            let offset = unsafe { data_offset(ptr) };
            unsafe { (ptr as *mut RcBox<T>).set_ptr_value((ptr as *mut u8).offset(-offset)) }
        };
        //RcBox的weak的计数已经有了这个计数
        Weak { ptr: unsafe { NonNull::new_unchecked(ptr) } }
    }

    //创建WeakInner
    fn inner(&self) -> Option<WeakInner<'_>> {
        if is_dangling(self.ptr.as_ptr()) {
            None
        } else {
            // We are careful to *not* create a reference covering the "data" field, as
            // the field may be mutated concurrently (for example, if the last `Rc`
            // is dropped, the data field will be dropped in-place).
            Some(unsafe {
                let ptr = self.ptr.as_ptr();
                WeakInner { strong: &(*ptr).strong, weak: &(*ptr).weak }
            })
        }
    }

    //从Weak得到Rc, 如前所述，对Rc正确的打开方式应该是仅用Weak，然后适当的时候升级到Rc<T>，
    //并且在使用完毕后就将Rc<T>生命周期终止掉，即这个函数返回的Rc<T>生命周期最好仅在一个函数中。
    pub fn upgrade(&self) -> Option<Rc<T>> {
        //获取内部的RcBox
        let inner = self.inner()?;
        if inner.strong() == 0 {
            None
        } else {
            //对RcBox<T>的strong增加计数
            inner.inc_strong();
            //利用RcBox生成新的Rc<T>
            Some(Rc::from_inner(self.ptr))
        }
    }
}
impl <T:?Sized> Rc<T> {
    ...

    //生成新的Weak<T>
    pub fn downgrade(this: &Self) -> Weak<T> {
        //增加weak计数
        this.inner().inc_weak();
        // 确保不出错
        debug_assert!(!is_dangling(this.ptr.as_ptr()));
        // 生成Weak<T>
        Weak { ptr: this.ptr }
    }
}
 ```
`Rc<T>`的其他方法：
```rust
 impl<T: Clone> Rc<T> {
    //Rc<T> 实际上是需要配合RefCell<T>来完成对堆内存的修改需求
    //下面的函数用了类似写时复制的方式，仅能在某些场景下使用
    //下面这个函数作为学习RUST的技巧是有用的，但建议最好不使用它
    pub fn make_mut(this: &mut Self) -> &mut T {
        if Rc::strong_count(this) != 1 {
            // 如果Rc多于一个引用，则创建一个拷贝的变量
            // 申请一个未初始化的Rc
            let mut rc = Self::new_uninit();
            unsafe {
                //将self中的value值写入新创建的变量
                let data = Rc::get_mut_unchecked(&mut rc);
                (**this).write_clone_into_raw(data.as_mut_ptr());
                //这里把this代表的Rc释放掉，并赋以新值。
                //make_mut的本意是从this中生成一个mut，因此将this代表的Rc<T>释放掉是合乎
                //函数的意义的
                *this = rc.assume_init();
            }
        } else if Rc::weak_count(this) != 0 {
            // 如果Rc<T>仅有一个strong引用，但有其他的weak引用
            // 同样需要新建一个Rc<T>
            let mut rc = Self::new_uninit();
            unsafe {
                //下面用了与strong !=1 的情况的不同写法，但应该完成了同样的工作
                let data = Rc::get_mut_unchecked(&mut rc);
                data.as_mut_ptr().copy_from_nonoverlapping(&**this, 1);
                
                //将strong引用减去，堆内存不再存在strong引用
                this.inner().dec_strong();
                // strong已经为0，所以将strong的weak计数减掉
                this.inner().dec_weak();
                //不能用*this = 的表达，因为会导致对堆内存变量的释放，这不符合语义。
                ptr::write(this, rc.assume_init());
            }
        }
        //已经确保了只有一个Rc<T>，且没有Weak<T>，可以任意对堆变量做修改了
        unsafe { &mut this.ptr.as_mut().value }
    }
 }
```
 
 Clone trait实现：
 ```rust
impl<T: ?Sized> Clone for Rc<T> {
    //clone就是增加一个strong的计数
    fn clone(&self) -> Rc<T> {
        self.inner().inc_strong();
        Self::from_inner(self.ptr)
    }
}

impl<T: ?Sized> Clone for Weak<T> {
    fn clone(&self) -> Weak<T> {
        if let Some(inner) = self.inner() {
            inner.inc_weak()
        }
        Weak { ptr: self.ptr }
    }
}
 ``` 
对`Rc<MaybeUninit<T>>`初始化后assume_init实现方法：
```rust
impl<T> Rc<mem::MaybeUninit<T>> {
    pub unsafe fn assume_init(self) -> Rc<T> {
        //先用ManuallyDrop将self封装以便不对self做drop操作
        //然后取出内部的堆指针形成新的Rc<T>。
        Rc::from_inner(mem::ManuallyDrop::new(self).ptr.cast())
    }
}
```
`Rc<T>`其他方法：
```rust
impl<T: ?Sized> Rc<T> {
    //相当于Rc<T>的leak函数
    pub fn into_raw(this: Self) -> *const T {
        let ptr = Self::as_ptr(&this);
        //把堆内存指针取出后，由调用代码负责释放，
        //本结构体要规避后继的释放操作
        mem::forget(this);
        ptr
    }

    //获得堆内存变量的指针，不会涉及安全问题,注意，这里ptr不是堆内存块的首地址，而是向后有偏移
    //因为RcBox<T>采用C语言的内存布局，所以value在最后
    pub fn as_ptr(this: &Self) -> *const T {
        let ptr: *mut RcBox<T> = NonNull::as_ptr(this.ptr);

        unsafe { ptr::addr_of_mut!((*ptr).value) }
    }

    //从堆内存T类型变量的指针重建Rc<T>，注意，这里的ptr一般是调用Rc<T>::into_raw()获得的裸指针
    //ptr不是堆内存块首地址，需要减去strong和weak的内存大小
    pub unsafe fn from_raw(ptr: *const T) -> Self {
        let offset = unsafe { data_offset(ptr) };

        // 减去偏移量，得到正确的RcBox堆内存的首地址
        let rc_ptr =
            unsafe { (ptr as *mut RcBox<T>).set_ptr_value((ptr as *mut u8).offset(-offset)) };

        unsafe { Self::from_ptr(rc_ptr) }
    }
```
into_raw, from_raw要成对使用，否则就必须对这两个方法的内存由清晰的认知。否则极易出现问题。
`Rc<T>` 转换为`Weak<T>`
```rust
    //对于Weak引用，要生成一个Weak<T>，
    pub fn downgrade(this: &Self) -> Weak<T> {
        this.inner().inc_weak();
        debug_assert!(!is_dangling(this.ptr.as_ptr()));
        //直接生成Weak
        Weak { ptr: this.ptr }
    }

    pub fn get_mut(this: &mut Self) -> Option<&mut T> {
        if Rc::is_unique(this) { unsafe { Some(Rc::get_mut_unchecked(this)) } } else { None }
    }

    pub unsafe fn get_mut_unchecked(this: &mut Self) -> &mut T {
        unsafe { &mut (*this.ptr.as_ptr()).value }
    }

}
```

## `Arc<T>`的代码实现

`Arc<T>` 是`Rc<T>`的多线程版本，实际上，就连代码都基本类似，只是把计数值的类型换成了原子变量
`Arc<T>`类型结构定义如下：
```rust
//在堆内存分配的结构体
#[repr(C)]
struct ArcInner<T: ?Sized> {
    //用原子变量实现计数，使得计数修改不会因多线程竞争而出错
    //AtomicUsize 如下：
    // pub struct AtomicUsize { v: UnsafeCell<usize>}
    strong: atomic::AtomicUsize,
    weak: atomic::AtomicUsize,
    data: T,
}

//支持Send
unsafe impl<T: ?Sized + Sync + Send> Send for ArcInner<T> {}
//支持Sync
unsafe impl<T: ?Sized + Sync + Send> Sync for ArcInner<T> {}

//Arc<T>的结构
pub struct Arc<T: ?Sized> {
    ptr: NonNull<ArcInner<T>>,
    phantom: PhantomData<ArcInner<T>>,
}
//对Send支持
unsafe impl<T: ?Sized + Sync + Send> Send for Arc<T> {}
//对Sync支持
unsafe impl<T: ?Sized + Sync + Send> Sync for Arc<T> {}

//Weak<T>的结构
pub struct Weak<T: ?Sized> {
    ptr: NonNull<ArcInner<T>>,
}

unsafe impl<T: ?Sized + Sync + Send> Send for Weak<T> {}
unsafe impl<T: ?Sized + Sync + Send> Sync for Weak<T> {}
```
`ArcInner<T>`对应`RcBox<T>`, `Arc<T>`对应`Rc<T>`, `sync::Weak<T>`对应`rc::Weak<T>`。逻辑与Rc<T>模块的逻辑都基本相同。的。 `Arc<T>`除了`ArcInner<T>`与`RcBox<T>`有区别，的区别就是计数器用原子变量实现。使得计数器的加减操作不会受多线程数据竞争的影响，从而使得`Arc<T>`能够在多线程环境下使用。这里需要注意`Rc<T>`及`Arc<T>`实际上仅仅是不可变引用的多线程替代(多于两个以上)，因此`Arc<T>`的实现中仅仅关注`Arc<T>`类型本身的多线程共享的保护机制。至于内部的泛型类型变量data，仍然需要泛型类型自身对多线程共享的实现。

 与 `Rc<T>`相同`Arc<T>`也提供了weak和strong两种堆内存指针的方式，`Arc<T>`申请的堆内存可以没有初始化，未初始化的堆内存可以生成`WeakT>`用于给其他结构访问堆内存。同时堆内存用strong的方式来保护`Arc<T>`在未初始化时不被读写。且weak和strong可以相互之间转换，这就以rust方式解决了生命周期陷阱问题。
利用`Weak<T>`做指针的结构体，在需要访问堆内存时，可以从`Weak<T>`另外创建`Arc<T>`, 完成访问后即可让创建的`Arc<T>`生命周期终结。实际上，各需要访问堆内存的结构体仅保留`Weak<T>`应该是一个非常好的做法。

 `Arc<T>`的创建方法及析构方法代码如下：
 ```rust
 //已经存在堆内存，从堆内存来创建Arc<T>
 impl<T:?Sized> Arc<T> {
    fn from_inner(ptr: NonNull<ArcInner<T>>) -> Self {
        Self { ptr, phantom: PhantomData }
    }

    unsafe fn from_ptr(ptr: *mut ArcInner<T>) -> Self {
        unsafe { Self::from_inner(NonNull::new_unchecked(ptr)) }
    }
 }
 
 impl<T> Arc<T> {
     //由已初始化变量创建Arc<T>
     pub fn new(data: T) -> Arc<T> {
         //首先创建ArcInner<T>，然后生成Box<ArcInner<T>>, 随后用leak得到ArcInner<T>的堆内存指针，
         //用堆内存指针创建Rc<T>，内存申请由Box<T>实际执行
        let x: Box<_> = box ArcInner {
            strong: atomic::AtomicUsize::new(1),
            weak: atomic::AtomicUsize::new(1),
            data,
        };
        Self::from_inner(Box::leak(x).into())
    }
     //用于创建一个互相引用场景的Arc<T>
    pub fn new_cyclic(data_fn: impl FnOnce(&Weak<T>) -> T) -> Arc<T> {
         // 下面与new函数代码类似，只是value没有初始化。
         // 因为value没有初始化，strong赋值为0，但可以支持Weak<T>的引用
        let uninit_ptr: NonNull<_> = Box::leak(box ArcInner {
            strong: atomic::AtomicUsize::new(0),
            weak: atomic::AtomicUsize::new(1),
            data: mem::MaybeUninit::<T>::uninit(),
        })
        .into();

         //init_ptr后继会被初始化，但此时还没有
        let init_ptr: NonNull<ArcInner<T>> = uninit_ptr.cast();

         //生成Weak
        let weak = Weak { ptr: init_ptr };

         // 利用回调闭包获得value的值，将weak传递出去是因为cyclic默认结构体初始化需要使用weak.
         // 用回调函数的处理可以让初始化一次完成，以免初始化以后还要修改结构体的指针。
        let data = data_fn(&weak);

        // Now we can properly initialize the inner value and turn our weak
        // reference into a strong reference.
        unsafe {
            let inner = init_ptr.as_ptr();
             //addr_of_mut!可以万无一失，写入值后，初始化已经完成
            ptr::write(ptr::addr_of_mut!((*inner).data), data);

             //可以更新strong的值为1了,注意这里的原子函数，这个函数不会被其他线程打断导致更新失败
            let prev_value = (*inner).strong.fetch_add(1, Release);
            debug_assert_eq!(prev_value, 0, "No prior strong references should exist");
        }

         //strong登场
        let strong = Arc::from_inner(init_ptr);

         // 这里是因为strong整体拥有一个weak计数，所以此处不对weak做drop处理以维持weak计数。前面的回调函数中应该使用weak.clone增加weak的计数。
        mem::forget(weak);
        strong
    }
 
     //生成一个未初始化的Arc<T>，此时只能本身来处理内存申请的工作
    pub fn new_uninit() -> Arc<mem::MaybeUninit<T>> {
        unsafe {
             //Arc自身的内存申请函数感觉过于复杂
            Arc::from_ptr(Arc::allocate_for_layout(
                Layout::new::<T>(),
                |layout| Global.allocate(layout),
                |mem| mem as *mut ArcInner<mem::MaybeUninit<T>>,
            ))
        }
    }
     //防止内存不足的创建函数
    pub fn try_new(data: T) -> Result<Arc<T>, AllocError> {
         // 就是用Box::try_new来完成try的工作
        let x: Box<_> = Box::try_new(ArcInner {
            strong: atomic::AtomicUsize::new(1),
            weak: atomic::AtomicUsize::new(1),
            data,
        })?;
        Ok(Self::from_inner(Box::leak(x).into()))
    }
 
     //对未初始化的Rc的try new
    pub fn try_new_uninit() -> Result<Arc<mem::MaybeUninit<T>>, AllocError> {
        unsafe {
             //内存申请函数需要考虑申请不到的情况
            Ok(Arc::from_ptr(Arc::try_allocate_for_layout(
                Layout::new::<T>(),
                 //就用Global Allocator，没有考虑其他，可能会有些问题
                |layout| Global.allocate(layout),
                |mem| mem as *mut ArcInner<mem::MaybeUninit<T>>,
            )?))
        }
    }
     ...
 }

//堆内存申请函数
impl<T: ?Sized> Arc<T> {
    unsafe fn allocate_for_layout(
        value_layout: Layout,
        allocate: impl FnOnce(Layout) -> Result<NonNull<[u8]>, AllocError>,
        mem_to_arcinner: impl FnOnce(*mut u8) -> *mut ArcInner<T>,
    ) -> *mut ArcInner<T> {
        // 根据T计算RcBox需要的内存块布局，注意用ArcInner<()>获取仅包含strong及cell两个成员的RcBox的layout这个技巧
        let layout = Layout::new::<ArcInner<()>>().extend(value_layout).unwrap().0.pad_to_align();
        unsafe {
            //要考虑不成功的可能性
            Arc::try_allocate_for_layout(value_layout, allocate, mem_to_arcinner)
                .unwrap_or_else(|_| handle_alloc_error(layout))
        }
    }

    unsafe fn try_allocate_for_layout(
        value_layout: Layout,
        allocate: impl FnOnce(Layout) -> Result<NonNull<[u8]>, AllocError>,
        mem_to_arcinner: impl FnOnce(*mut u8) -> *mut ArcInner<T>,
    ) -> Result<*mut ArcInner<T>, AllocError> {
        //计算需要的内存块布局layout
        let layout = Layout::new::<ArcInner<()>>().extend(value_layout).unwrap().0.pad_to_align();

        // 申请内存，有可能不成功
        let ptr = allocate(layout)?;

        // 将裸指针类型内存类型转换成*mut ArcInner<xxx>类型，xxx有可能是MaybeUninit<T>，但也可能是初始化完毕的类型。总之，调用代码会保证初始化，所以此处正常的设置strong及weak，
        let inner = mem_to_arcinner(ptr.as_non_null_ptr().as_ptr());
        debug_assert_eq!(unsafe { Layout::for_value(&*inner) }, layout);

        unsafe {
            ptr::write(&mut (*inner).strong, atomic::AtomicUsize::new(1));
            ptr::write(&mut (*inner).weak, atomic::AtomicUsize::new(1));
        }

        Ok(inner)
    }

    //根据一个裸指针来创建RcBox<T>，返回裸指针，这个函数完成时堆内存没有初始化，后继需要写入值
    unsafe fn allocate_for_ptr(ptr: *const T) -> *mut ArcInner<T> {
        // Allocate for the `ArcInner<T>` using the given value.
        unsafe {
            Self::allocate_for_layout(
                // 用*const T获取Layout
                Layout::for_value(&*ptr),
                |layout| Global.allocate(layout),
                //此处应该也可以用mem as *mut ArcInner<T>，
                //目前的代码感觉比较复杂，
                |mem| (ptr as *mut ArcInner<T>).set_ptr_value(mem) as *mut ArcInner<T>,
            )
        }
    }

    //从Box<T>转换成Arc<T>
     fn from_box(v: Box<T>) -> Arc<T> {
        unsafe {
            //解封装Box，获取堆内存指针
            let (box_unique, alloc) = Box::into_unique(v);
            let bptr = box_unique.as_ptr();

            let value_size = size_of_val(&*bptr);
            //获得* mut ArcInner<T>
            let ptr = Self::allocate_for_ptr(bptr);

            // 将T的内容拷贝入ArcInner的value
            ptr::copy_nonoverlapping(
                bptr as *const T as *const u8,
                &mut (*ptr).data as *mut _ as *mut u8,
                value_size,
            );

            // 重要，这里仅仅释放堆内存，但是如果堆内存中的T类型变量还有其他需要释放的内存，则没有处理，即没有调用drop(T)，drop(T)由新生成的ArcInner<T>再释放的时候负责
            box_free(box_unique, alloc);

            // 生成Arc<T>
            Self::from_ptr(ptr)
        }
    }
}

//析构
unsafe impl<#[may_dangle] T: ?Sized> Drop for Arc<T> {
    fn drop(&mut self) {
        //如果当前的strong不是1,则返回，fetch_xxx函数返回之前的值
        if self.inner().strong.fetch_sub(1, Release) != 1 {
            return;
        }

        acquire!(self.inner().strong);

        unsafe {
            //见下面代码的分析
            self.drop_slow();
        }
    }
}
impl <T:?Sized> Arc<T> {
    ...
    unsafe fn drop_slow(&mut self) {
        // 对堆内存的变量做drop操作，注意，这里不释放堆内存，只是释放变量所有权
        unsafe { ptr::drop_in_place(Self::get_mut_unchecked(self)) };

        // 所有的strong会创建一个Weak，对这个Weak做drop操作
        drop(Weak { ptr: self.ptr });
    }
    ...
}
 ```
`Weak<T>`的结构体及创建，析构方法：
 在RC方法内部，Weak可以由`Weak{ptr:self_ptr}`直接创建，可见前面代码的例子，但要注意weak计数和Weak变量需要匹配
 ```rust

impl<T> Weak<T> {
    //创建一个空的Weak
    pub fn new() -> Weak<T> {
        Weak { ptr: NonNull::new(usize::MAX as *mut ArcInner<T>).expect("MAX is not 0") }
    }
}

struct WeakInner<'a> {
    weak: &'a atomic::AtomicUsize,
    strong: &'a atomic::AtomicUsize,
}
//判断Weak是否为空的关联函数
pub(crate) fn is_dangling<T: ?Sized>(ptr: *mut T) -> bool {
    let address = ptr as *mut () as usize;
    address == usize::MAX
}

impl <T:?Sized> Weak<T> {
    pub fn as_ptr(&self) -> *const T {
        let ptr: *mut ArcInner<T> = NonNull::as_ptr(self.ptr);

        if is_dangling(ptr) {
            ptr as *const T
        } else {
            //返回T类型变量的指针
            unsafe { ptr::addr_of_mut!((*ptr).data) }
        }
    }
    //会消费掉Weak，获取T类型变量指针，此指针以后需要重新组建Weak<T>,否则
    //堆内存中的ArcInner的weak会出现计数错误
    pub fn into_raw(self) -> *const T {
        let result = self.as_ptr();
        mem::forget(self);
        result
    }
    
    //ptr是从into_raw得到的返回值
    pub unsafe fn from_raw(ptr: *const T) -> Self {
        let ptr = if is_dangling(ptr as *mut T) {
            ptr as *mut ArcInner<T>
        } else {
            //需要从T类型的指针恢复RcBox的指针
            let offset = unsafe { data_offset(ptr) };
            unsafe { (ptr as *mut ArcInner<T>).set_ptr_value((ptr as *mut u8).offset(-offset)) }
        };
        //ArcInner的weak的计数已经有了这个计数
        Weak { ptr: unsafe { NonNull::new_unchecked(ptr) } }
    }
    
    //创建一个WeakInner
    fn inner(&self) -> Option<WeakInner<'_>> {
        if is_dangling(self.ptr.as_ptr()) {
            None
        } else {
            //获取ArcInner<T>中strong和weak的引用
            Some(unsafe {
                let ptr = self.ptr.as_ptr();
                WeakInner { strong: &(*ptr).strong, weak: &(*ptr).weak }
            })
        }
    }
    //从Weak得到Arc, 如前所述，对Arc正确的打开方式应该是仅用Weak，然后适当的时候升级到Arc<T>
    //并且在使用完毕后就将ARc<T>生命周期终止掉，即这个函数返回Arc<T>生命周期最好仅在一个函数中。
    pub fn upgrade(&self) -> Option<Arc<T>> {
        //获取内部的ArcInner
        let inner = self.inner()?;
        //原子操作获得strong的值
        let mut n = inner.strong.load(Relaxed);
        
        //因为是多线程操作，所以此时n已经可能被改写，所以用loop
        //来确保n在已经改写的情况下正确
        loop {
            //如果strong是0，那堆内存已经被释放掉，不能再使用
            if n == 0 {
                return None;
            }

            // 不能多于最大的引用数目
            if n > MAX_REFCOUNT {
                abort();
            }

            //以下确保在strong当前值是n的时候做加1操作
            match inner.strong.compare_exchange_weak(n, n + 1, Acquire, Relaxed) {
                //当前值为1且已经加1，生成Arc<T>
                Ok(_) => return Some(Arc::from_inner(self.ptr)), // null checked above
                //如果当前值不为n，则将n设置为当前值，进入下一轮循环。
                Err(old) => n = old,
            }
        }
    }
}

impl <T:?Sized> Arc<T> {
    ...

    //生成新的Weak<T>
    pub fn downgrade(this: &Self) -> Weak<T> {
        // 获取weak count.
        let mut cur = this.inner().weak.load(Relaxed);

        //要确定当前的weak count与上面取得一致
        loop {
            // 如果是usize::MAX，证明在创建过程中，等创建完毕后
            // 再获取一次
            if cur == usize::MAX {
                hint::spin_loop();
                cur = this.inner().weak.load(Relaxed);
                continue;
            }
            
            //确保在weak与当前值一致的情况下做原子操作，将weak加1
            match this.inner().weak.compare_exchange_weak(cur, cur + 1, Acquire, Relaxed) {
                Ok(_) => {
                    // 确保不创建对不存在的变量的Weak
                    debug_assert!(!is_dangling(this.ptr.as_ptr()));
                    //创建Weak
                    return Weak { ptr: this.ptr };
                }
                //如果当前的值与取值不一致，将取值更换为当前值，再做一次循环
                Err(old) => cur = old,
            }
        }
    }
}
 ```
 以上代码中，对于多线程的处理需要额外注意并理解。这是原子变量处理多线程的典型用法

`Arc<T>`的其他方法：
```rust
 impl<T: Clone> Arc<T> {
    //Rc<T> 实际上是需要配合RefCell<T>来完成对堆内存的修改需求
    //下面的函数用了类似写时复制的方式，仅能在某些场景下使用
    //下面这个函数作为学习RUST的技巧是有用的，但建议最好不使用它
    pub fn make_mut(this: &mut Self) -> &mut T {
        //判断strong的值是否为1，如果为1，则设置为0，以防止其他线程做修改 
        if this.inner().strong.compare_exchange(1, 0, Acquire, Relaxed).is_err() {
            // strong不为1，需要创建一个复制的Arc<T>变量
            let mut arc = Self::new_uninit();
            unsafe {
                let data = Arc::get_mut_unchecked(&mut arc);
                (**this).write_clone_into_raw(data.as_mut_ptr());
                *this = arc.assume_init();
            }
        } else if this.inner().weak.load(Relaxed) != 1 {
            //当前为原strong为1且已经strong已经做了减1操作为0
            //那此时weak如果为1，证明没有多余的Weak<T>被派生
            //如果weak不为1，则证明有其他的Weak<T>存在，需要创建一个复制的Arc<T>

            //这里因为strong已经被减1，所以本线程已经没有Arc<T>，所以创建一个
            //Weak并由此变量的drop完成对weak计数的处理
            let _weak = Weak { ptr: this.ptr };

            // 创建一个新的复制的Arc<T>
            let mut arc = Self::new_uninit();
            unsafe {
                let data = Arc::get_mut_unchecked(&mut arc);
                data.as_mut_ptr().copy_from_nonoverlapping(&**this, 1);
                ptr::write(this, arc.assume_init());
            }
        } else {
            // strong及weak都是1，则恢复strong为1，直接使用当前的Arc<T>
            this.inner().strong.store(1, Release);
        }

        //返回&mut T
        unsafe { Self::get_mut_unchecked(this) }
       
    }
 }
```
 
 Clone trait实现：
 ```rust
impl<T: ?Sized> Clone for Arc<T> {
    fn clone(&self) -> Arc<T> {
        //增加一个strong计数
        let old_size = self.inner().strong.fetch_add(1, Relaxed);

        if old_size > MAX_REFCOUNT {
            abort();
        }
        //从内部创建一个新的ARC<T>
        Self::from_inner(self.ptr)
    }
}

impl<T: ?Sized> Clone for Weak<T> {
    fn clone(&self) -> Weak<T> {
        if let Some(inner) = self.inner() {
            inner.inc_weak()
        }
        Weak { ptr: self.ptr }
    }
    fn clone(&self) -> Weak<T> {
        let inner = if let Some(inner) = self.inner() {
            inner
        } else {
            //inner不存在，直接创建一个Weak<T>
            return Weak { ptr: self.ptr };
        };
        //对weak计数加1
        let old_size = inner.weak.fetch_add(1, Relaxed);

        if old_size > MAX_REFCOUNT {
            abort();
        }
        //创建Weak<T>
        Weak { ptr: self.ptr }
    }
}
 ``` 
对`Arc<MaybeUninit<T>>`初始化后assume_init实现方法：
```rust
impl<T> Arc<mem::MaybeUninit<T>> {
    pub unsafe fn assume_init(self) -> Arc<T> {
        //先用ManuallyDrop将self封装以便不对self做drop操作
        //然后取出内部的堆指针形成新的Arc<T>。
        Arc::from_inner(mem::ManuallyDrop::new(self).ptr.cast())
    }
}
```
`Arc<T>`其他方法：
```rust
impl<T: ?Sized> Arc<T> {
    //相当于Arc<T>的leak函数
    pub fn into_raw(this: Self) -> *const T {
        let ptr = Self::as_ptr(&this);
        //把堆内存指针取出后，由调用代码负责释放，
        //本结构体要规避后继的释放操作
        mem::forget(this);
        ptr
    }

    //获得堆内存变量的指针，不会涉及安全问题,注意，这里ptr不是堆内存块的首地址，而是向后有偏移
    //因为ArcInner<T>采用C语言的内存布局，所以value在最后
    pub fn as_ptr(this: &Self) -> *const T {
        let ptr: *mut ArcInner<T> = NonNull::as_ptr(this.ptr);

        unsafe { ptr::addr_of_mut!((*ptr).value) }
    }

    //从堆内存T类型变量的指针重建Arc<T>，注意，这里的ptr一般是调用`Arc<T>::into_raw()获得的裸指针
    //ptr不是堆内存块首地址，需要减去strong和weak的内存大小
    pub unsafe fn from_raw(ptr: *const T) -> Self {
        let offset = unsafe { data_offset(ptr) };

        // 减去偏移量，得到正确的ArcInner堆内存的首地址
        let rc_ptr =
            unsafe { (ptr as *mut ArcInner<T>).set_ptr_value((ptr as *mut u8).offset(-offset)) };

        unsafe { Self::from_ptr(rc_ptr) }
    }
```
into_raw, from_raw要成对使用，否则就必须对这两个方法的内存由清晰的认知。否则极易出现问题。

```rust
 
    pub fn get_mut(this: &mut Self) -> Option<&mut T> {
        if this.is_unique() { unsafe { Some(Arc::get_mut_unchecked(this)) } } else { None }
    }

    pub unsafe fn get_mut_unchecked(this: &mut Self) -> &mut T {
        unsafe { &mut (*this.ptr.as_ptr()).data }
    }

}
```

## `LinkedList<T>`代码分析
双向链表及其他数据结构的代码实现都是经典的实用性及训练性上佳的项目。本书对这些经典数据结构将只分析LinkedList<T>，重点分析RUST与其他语言的不同的部分。如果对LinkedList<T>彻底理解了，那其他数据结构也就不成为问题:
`LinkedList<T>`类型结构定义如下：
```rust
//这个定义表示LinkedList只支持固定长度的T类型
pub struct LinkedList<T> {
    //等同于直接用裸指针，使得代码最方便及简化，但需要对安全性额外投入精力
    //这个实际上与C语言相同，只是用Option增加了安全措施
    head: Option<NonNull<Node<T>>>,
    tail: Option<NonNull<Node<T>>>,
    len: usize,
    //marker说明本结构有一个Box<Node<T>>的所有权，并会负责调用其的drop
    //编译器应做好drop check, 检查与本结构相关的Box<Node<T>>的生命周期及drop
    //marker体现了RUST的独特点
    marker: PhantomData<Box<Node<T>>>,
}

struct Node<T> {
    next: Option<NonNull<Node<T>>>,
    prev: Option<NonNull<Node<T>>>,
    element: T,
}
```
Node方法代码：
```rust
impl<T> Node<T> {
    fn new(element: T) -> Self {
        Node { next: None, prev: None, element }
    }

    fn into_element(self: Box<Self>) -> T {
        //消费了Box，堆内存被释放并将element拷贝到栈
        self.element
    }
}
```
LinkedList的创建及简单的增减方法：
```rust
impl<T> LinkedList<T> {
    //创建一个空的LinkedList
    pub const fn new() -> Self {
        LinkedList { head: None, tail: None, len: 0, marker: PhantomData }
    }
```
在头部增加一个成员及删除一个成员：
```rust    
    //在首部增加一个节点
    pub fn push_front(&mut self, elt: T) {
        //用box从堆内存申请一个节点，push_front_node见后面函数
        self.push_front_node(box Node::new(elt));
    }
    fn push_front_node(&mut self, mut node: Box<Node<T>>) {
        // 整体全是不安全代码
        unsafe {
            node.next = self.head;
            node.prev = None;
            //需要将Box的堆内存leak出来使用。此块内存后继如果还在链表，需要由LinkedList负责drop.后面可以看到LinkedList的drop函数的处理。
            //如果pop出链表，那会重新用这里leak出来的NonNull<Node<T>>生成Box,再由Box释放
            let node = Some(Box::leak(node).into());

            match self.head {
                None => self.tail = node,
                // 注意下面，不能使直接用head.prev，因为那样会复制一个head，导致逻辑错误
                // 此处是RUST语法带来的极易出错的陷阱。
                Some(head) => (*head.as_ptr()).prev = node,
            }
            
            self.head = node;
            self.len += 1;
        }
    }

    //从链表头部删除一个节点
    pub fn pop_front(&mut self) -> Option<T> {
        //Option<T>::map，此函数后，节点的堆内存已经被释放
        //变量被拷贝到栈内存
        self.pop_front_node().map(Node::into_element)
    }
    fn pop_front_node(&mut self) -> Option<Box<Node<T>>> {
        //整体是unsafe
        self.head.map(|node| unsafe {
            //重新生成Box，以便后继可以释放堆内存
            let node = Box::from_raw(node.as_ptr());
            //更换head指针
            self.head = node.next;

            match self.head {
                None => self.tail = None,
                // push_front_node() 已经分析过
                Some(head) => (*head.as_ptr()).prev = None,
            }

            self.len -= 1;
            node
        })
    }
```
在尾部增加一个成员及删除一个成员
```rust
    //从尾部增加一个节点
    pub fn push_back(&mut self, elt: T) {
        //用box从堆内存申请一个节点
        self.push_back_node(box Node::new(elt));
    }

    fn push_back_node(&mut self, mut node: Box<Node<T>>) {
        // 整体不安全
        unsafe {
            node.next = None;
            node.prev = self.tail;
            //需要将Box的堆内存leak出来使用。此块内存后继如果还在链表，需要由LinkedList负责drop.
            //如果pop出链表，那会重新用这里leak出来的NonNull<Node<T>>重新生成Box
            let node = Some(Box::leak(node).into());

            match self.tail {
                None => self.head = node,
                //前面代码已经有分析 
                Some(tail) => (*tail.as_ptr()).next = node,
            }

            self.tail = node;
            self.len += 1;
        }
    }

    //从尾端删除节点
    pub fn pop_back(&mut self) -> Option<T> {
        self.pop_back_node().map(Node::into_element)
    }

    fn pop_back_node(&mut self) -> Option<Box<Node<T>>> {
        self.tail.map(|node| unsafe {
            //重新创建Box以便删除堆内存
            let node = Box::from_raw(node.as_ptr());
            self.tail = node.prev;

            match self.tail {
                None => self.head = None,
                
                Some(tail) => (*tail.as_ptr()).next = None,
            }

            self.len -= 1;
            node
        })
    }
    
    ...
}

//Drop
unsafe impl<#[may_dangle] T> Drop for LinkedList<T> {
    fn drop(&mut self) {
        struct DropGuard<'a, T>(&'a mut LinkedList<T>);

        impl<'a, T> Drop for DropGuard<'a, T> {
            fn drop(&mut self) {
                //为了在下面的循环中出现panic，这里可以继续做释放
                //感觉此处做这个有些不自信
                while self.0.pop_front_node().is_some() {}
            }
        }

        while let Some(node) = self.pop_front_node() {
            let guard = DropGuard(self);
            //显式的drop 获取的Box<Node<T>>
            drop(node);
            //执行到此处，guard认为已经完成，不能再调用guard的drop
            mem::forget(guard);
        }
    }
}
```
以上基本上说明了RUST的LinkedList的设计及代码的一些关键点。

用Iterator来对List进行访问，Iterator的相关结构代码如下：
into_iter()相关结构及方法：
```rust
//变量本身的Iterator的类型
pub struct IntoIter<T> {
    list: LinkedList<T>,
}

impl<T> IntoIterator for LinkedList<T> {
    type Item = T;
    type IntoIter = IntoIter<T>;

    /// 对LinkedList<T> 做消费
    fn into_iter(self) -> IntoIter<T> {
        IntoIter { list: self }
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;

    fn next(&mut self) -> Option<T> {
        //从头部获取变量
        self.list.pop_front()
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        (self.list.len, Some(self.list.len))
    }
}
```
iter_mut()调用相关结构及方法
```rust
//可变引用的Iterator的类型
pub struct IterMut<'a, T: 'a> {
    head: Option<NonNull<Node<T>>>,
    tail: Option<NonNull<Node<T>>>,
    len: usize,
    //这个marker也标示了IterMut对LinkedList有一个可变引用
    //创建IterMut后，与之相关的LinkerList不能在被其他安全的代码修改
    marker: PhantomData<&'a mut Node<T>>,
}

impl <T> LinkedList<T> {
    ...
    pub fn iter_mut(&mut self) -> IterMut<'_, T> {
        IterMut { head: self.head, tail: self.tail, len: self.len, marker: PhantomData }
    }
    ...
}
impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<&'a mut T> {
        if self.len == 0 {
            None
        } else {
            self.head.map(|node| unsafe {
                // 注意，下面代码执行后堆内存已经没有所有权的载体,
                // 此函数的调用代码必须在后继对返回的引用做Box重组
                // 或者直接drop，否则会造成内存泄漏 
                // 但因为没有所有权载体，所以除了这里的可变引用外，
                // 不会有其他访问情况，仍然符合所有权的独占性
                let node = &mut *node.as_ptr();
                self.len -= 1;
                self.head = node.next;
                &mut node.element
            })
        }
    }

    ...
}

//不可变引用的Iterator的类型
pub struct Iter<'a, T: 'a> {
    head: Option<NonNull<Node<T>>>,
    tail: Option<NonNull<Node<T>>>,
    len: usize,
    //对生命周期做标识，也标识了一个对LinkedList的不可变引用
    marker: PhantomData<&'a Node<T>>,
}

impl<T> Clone for Iter<'_, T> {
    fn clone(&self) -> Self {
        //本书中第一次出现这个表述
        Iter { ..*self }
    }
}

//Iterator trait for Iter略
```
LinkedList其他的代码略。

## String 类型分析
String结构定义如下：
```rust
pub struct String {
    vec: Vec<u8>,
}
```
`Vec<u8>`和String的关系可以与[u8]与&str的关系相对比。整个String实际上是一个大的
Adapter 模式，针对Vec<u8>, [u8], &str三者做组合
String的创建函数：
```rust
impl String {
     pub const fn new() -> String {
        String { vec: Vec::new() }
    }

    //将str内容加到String的尾部
    pub fn push_str(&mut self, string: &str) {
        //adapter，直接用Vec::extend_from_slice([u8])
        //具体的细节请参考Vec那节
        self.vec.extend_from_slice(string.as_bytes())
    }
    ...
}

impl ToOwned for str {
    type Owned = String;
    fn to_owned(&self) -> String {
        //这里是个adapter模式，首先从用self.as_bytes()获取[u8], 然后用通用的[u8].to_owned()完成
        //to_owned逻辑,随后从Vec[u8]生成String
        unsafe { String::from_utf8_unchecked(self.as_bytes().to_owned()) }
    }

    
    fn clone_into(&self, target: &mut String) {
        //adapter模式，需要先得到Vec<u8>,因为into_bytes会消费掉String。
        //target不支持，所以需要用take先把所有权转移出来，然后获取Vec<u8>
        //这是RUST的一个通用的技巧
        let mut b = mem::take(target).into_bytes();
        //通用的[u8].clone_into
        self.as_bytes().clone_into(&mut b);
        //把新的String赋给原先的地址
        *target = unsafe { String::from_utf8_unchecked(b) }
    }
}

impl From<&str> for String {
    fn from(s: &str) -> String {
        s.to_owned()
    }
} 


```
解引用方法代码：
```rust
impl ops::Deref for String {
    type Target = str;

    fn deref(&self) -> &str {
        //&self.vec会被强转为&[u8]
        unsafe { str::from_utf8_unchecked(&self.vec) }
    }
}

impl ops::DerefMut for String {
    fn deref_mut(&mut self) -> &mut str {
        //这里直接用&mut self.vec应该也可以，会被强转成&mut [u8]
        unsafe { str::from_utf8_unchecked_mut(&mut *self.vec) }
    }
}
```
运算符重载方法
```rust
impl ops::Index<ops::RangeFull> for String {
    type Output = str;

    fn index(&self, _index: ops::RangeFull) -> &str {
        unsafe { str::from_utf8_unchecked(&self.vec) }
    }
}
impl ops::Index<ops::Range<usize>> for String {
    type Output = str;

    fn index(&self, index: ops::Range<usize>) -> &str {
        //先用Index<RangeFull>取&str, 然后用Index<Range>取子串
        &self[..][index]
    }
}
impl Borrow<str> for String {
    fn borrow(&self) -> &str {
        //自动解引用, 利用Index<RangeFull>完成，代码最简
        &self[..]
    }
}

impl BorrowMut<str> for String {
    fn borrow_mut(&mut self) -> &mut str {
        //自动解引用, 利用Index<RangeFull>完成，代码最简
        &mut self[..]
    }
}

impl Add<&str> for String {
    type Output = String;

    fn add(mut self, other: &str) -> String {
        self.push_str(other);
        self
    }
}

impl AddAssign<&str> for String {
    fn add_assign(&mut self, other: &str) {
        self.push_str(other);
    }
}
```
字符串数组连接方法：
```rust
//此函数主要简化多个字符串的连接
impl<S: Borrow<str>> Concat<str> for [S] {
    type Output = String;

    fn concat(slice: &Self) -> String {
        //见下个方法分析
        Join::join(slice, "")
    }
}

impl<S: Borrow<str>> Join<&str> for [S] {
    type Output = String;

    fn join(slice: &Self, sep: &str) -> String {
        unsafe { String::from_utf8_unchecked(join_generic_copy(slice, sep.as_bytes())) }
    }
}

macro_rules! specialize_for_lengths {
    ($separator:expr, $target:expr, $iter:expr; $($num:expr),*) => {{
        let mut target = $target;
        let iter = $iter;
        let sep_bytes = $separator;
        match $separator.len() {
            $(
                // 如果分隔切片长度符合预设值
                $num => {
                    for s in iter {
                        //拷贝分隔切片到目的切片，且更新目的切片
                        copy_slice_and_advance!(target, sep_bytes);
                        //拷贝内容切片
                        let content_bytes = s.borrow().as_ref();
                        copy_slice_and_advance!(target, content_bytes);
                    }
                },
            )*
            _ => {
                // 如果分隔切片长度不符合预设值，实质也做与上段代码同样的操作
                for s in iter {
                    copy_slice_and_advance!(target, sep_bytes);
                    let content_bytes = s.borrow().as_ref();
                    copy_slice_and_advance!(target, content_bytes);
                }
            }
        }
        target
    }}
}

//完成一个切片拷贝后，切片向前到未拷贝的开始处
macro_rules! copy_slice_and_advance {
    ($target:expr, $bytes:expr) => {
        let len = $bytes.len();
        //将目的切片切分成两段，首段为待拷贝空间，尾端为未拷贝空间
        let (head, tail) = { $target }.split_at_mut(len);
        head.copy_from_slice($bytes);
        $target = tail;
    };
}

//将若干个T类型的切片连接到一起形成一个基于T类型的切片
fn join_generic_copy<B, T, S>(slice: &[S], sep: &[T]) -> Vec<T>
where
    T: Copy, //最基础的成员类型
    B: AsRef<[T]> + ?Sized, //可以表示为最基础成员的切片引用
    S: Borrow<B>, //以B类型作为操作类型，所以S应该能borrow成B类型的引用
{
    let sep_len = sep.len();
    let mut iter = slice.iter();

    // 第一个成员头部没有间隔
    let first = match iter.next() {
        Some(first) => first,
        None => return vec![],
    };

    //计算iter中所有成员的长度，并加上间隔长度乘剩余成员的数目
    //得到总的长度。
    //从这个函数能够发现rust的链式编程的能力
    
    let reserved_len = sep_len
        .checked_mul(iter.len())//这里去掉了slice的首个成员，
        .and_then(|n| {
            //这里的重新重新生成iter，计算了所有的slice的所有成员
            slice.iter().map(|s| s.borrow().as_ref().len()).try_fold(n, usize::checked_add)
        })
        .expect("attempt to join into collection with len > usize::MAX");

    // 创建一个有足够容量的Vec
    let mut result = Vec::with_capacity(reserved_len);
    debug_assert!(result.capacity() >= reserved_len);
    //完成first的内容拷贝
    result.extend_from_slice(first.borrow().as_ref());

    unsafe {
        let pos = result.len();
        let target = result.get_unchecked_mut(pos..reserved_len);
        
        //完成对剩余成员及分隔符拷贝到result
        let remain = specialize_for_lengths!(sep, target, iter; 0, 1, 2, 3, 4);

        //完成长度拷贝。
        let result_len = reserved_len - remain.len();
        result.set_len(result_len);
    }
    result
}
```
## RUST的fmt相关代码
fmt给出RUST实现可变参数的解决方案。
alloc库中给出了format宏，完成对可变参数的格式化输出
format宏代码如下：
```rust
macro_rules! format {
    ($($arg:tt)*) => {{
        //format调用下面紧接所示的函数， 由format_args宏实现可变参数方案
        let res = $crate::fmt::format($crate::__export::format_args!($($arg)*));
        res
    }}
}
```
format_args宏将可变参数组合成Arguments变量，可以作为RUST的可变参数支持的经典案列。
```rust
//因为安全的愿因，下宏由编译器实现，形成Arguments的变量，
macro_rules! format_args {
    ($fmt:expr) => {{ /* compiler built-in */ }};
    ($fmt:expr, $($args:tt)*) => {{ /* compiler built-in */ }};
}
//上面提到的Arguments结构
pub struct Arguments<'a> {
    // 存放需要格式化的参数之间的字符串，对应于每一个格式化参数
    // 此字符串可以为空
    pieces: &'a [&'static str],

    // 针对每个格式化参数的格式描述
    fmt: Option<&'a [rt::v1::Argument]>,

    // 每个参数，以及参数的格式化字符串生成函数
    args: &'a [ArgumentV1<'a>],
}
```
format_args生成Arguments举例如下：
format_args!("ab {:b} cd {:p}", 1, 2) 
结果的Arguments结构中：  
其中pieces有两个成员，为:`"ab ", " cd " ` 
fmt有两个成员，为：`{postion:0, format:{align:UnKnown, flags:0, precision:Implied, width:Implied}}, {position:1, format:{align:UnKnown, flags:4, precision:Implied, width:Implied}}` 
args有两个成员为: `{1, core::fmt::num::Binary::fmt()}, {2, core::fmt::num::Pointer::fmt()}`  
```rust
//rt::v1::Argument
//对非默认格式化参数，每个参数format_args!宏会生成一个Argument变量
pub struct Argument {
    //表示参数的在Arguments中的序号，
    pub position: usize,
    //格式参数，用于格式化输出
    pub format: FormatSpec,
}
pub struct FormatSpec {
    //格式化时需要填充的字符
    pub fill: char,
    pub align: Alignment,
    //FlagV1 按位赋值
    pub flags: u32,
    pub precision: Count,
    pub width: Count,
}

//以下结构可认为是针对每一个参数，都有一个格式化输出的函数与其对应
pub struct ArgumentV1<'a> {
    //类似C语言的va_arg的返回类型，可以认为是void *
    value: &'a Opaque,
    //针对value的格式化输出
    formatter: fn(&Opaque, &mut Formatter<'_>) -> Result,
}

#[derive(Copy, Clone, PartialEq, Eq)]
pub enum Alignment {
    /// 左端对齐
    Left,
    /// 右端对齐
    Right,
    /// 中间对齐
    Center,
    /// 没有对齐
    Unknown,
}

pub enum Count {
    /// 字面量的值
    Is(usize),
    /// Specified using `$` and `*` syntaxes, stores the index into `args`
    Param(usize),
    /// Not specified
    Implied,
}
//flags 的位
enum FlagV1 {
    SignPlus,
    SignMinus,
    Alternate,
    SignAwareZeroPad,
    DebugLowerHex,
    DebugUpperHex,
}

//输入到每个类型的格式化函数
pub struct Formatter<'a> {
    //以下到precision都是由format_arg!宏在发现参数要求非默认的格式化时
    //生成的。
    flags: u32,
    fill: char,
    align: rt::v1::Alignment,
    width: Option<usize>,
    precision: Option<usize>,

    buf: &'a mut (dyn Write + 'a),
}

```
以下为将Arguments 形成字符串的调用代码：
```rust
//Arguments包含了本输出中所有的需要格式化的参数
pub fn format(args: Arguments<'_>) -> string::String {
    //估计了输出字符串长度，尽量减少堆内存的重新申请
    let capacity = args.estimated_capacity();
    //申请足够空间的字符串
    let mut output = string::String::with_capacity(capacity);
    //根据输入的格式化参数，完成对参数的格式化字符串输出
    output.write_fmt(args).expect("a formatting trait implementation returned an error");
    output
}
```
```rust
pub trait Write {
    fn write_str(&mut self, s: &str) -> Result;
    fn write_char(&mut self, c: char) -> Result {
        self.write_str(c.encode_utf8(&mut [0; 4]))
    }

    //格式化的输出
    fn write_fmt(mut self: &mut Self, args: Arguments<'_>) -> Result {
        //见下面紧邻的write函数分析
        write(&mut self, args)
    }
}

//完成格式化输出, output是一个承载格式化输出的类型，现在可以简单的认为是一个String
//但dyn Write实际上提供了扩展性，可以根据代码的需要设计其他类型完成
//Arguments是一个通用的参数结构
pub fn write(output: &mut dyn Write, args: Arguments<'_>) -> Result {
    //格式化的执行类型结构,Formmater也完成了格式化的通用方法
    let mut formatter = Formatter::new(output);
    let mut idx = 0;

    match args.fmt {
        //如果所有参数都是默认格式输出
        None => {
            // 对所有的参数进行轮询
            for (i, arg) in args.args.iter().enumerate() {
                //获取该参数前需要输出的字符串
                let piece = unsafe { args.pieces.get_unchecked(i) };
                if !piece.is_empty() {
                    //向output输出字符串
                    formatter.buf.write_str(*piece)?;
                }
                //调用每个参数的格式化输出函数，向formatter输出格式化参数字符串
                (arg.formatter)(arg.value, &mut formatter)?;
                idx += 1;
            }
        }
        //如果有参数不是默认格式输出
        Some(fmt) => {
            // 对所有参数进行轮询
            for (i, arg) in fmt.iter().enumerate() {
                // 获取该参数前应该输出的字符串
                let piece = unsafe { args.pieces.get_unchecked(i) };
                if !piece.is_empty() {
                    //向output输出字符串
                    formatter.buf.write_str(*piece)?;
                }
                //向formatter 输出格式化参数字符串
                unsafe { run(&mut formatter, arg, args.args) }?;
                idx += 1;
            }
        }
    }

    // 如果还有额外的字符串
    if let Some(piece) = args.pieces.get(idx) {
        //输出该字符串
        formatter.buf.write_str(*piece)?;
    }

    Ok(())
}
//非默认个数输出的格式化字符串输出函数
unsafe fn run(fmt: &mut Formatter<'_>, arg: &rt::v1::Argument, args: &[ArgumentV1<'_>]) -> Result {
    //根据格式化参数的格式完成fmt的格式设置
    fmt.fill = arg.format.fill;
    fmt.align = arg.format.align;
    fmt.flags = arg.format.flags;
    unsafe {
        fmt.width = getcount(args, &arg.format.width);
        fmt.precision = getcount(args, &arg.format.precision);
    }

    debug_assert!(arg.position < args.len());
    //获取格式化参数
    let value = unsafe { args.get_unchecked(arg.position) };

    // 真正的进行格式化
    (value.formatter)(value.value, fmt)
}


impl<'a> Arguments<'a> {
    /// format_args!()完成字符串和参数解析后，如果都是默认格式，用下面的函数创建
    /// Arguments变量
    pub const fn new_v1(pieces: &'a [&'static str], args: &'a [ArgumentV1<'a>]) -> Arguments<'a> {
        if pieces.len() < args.len() || pieces.len() > args.len() + 1 {
            panic!("invalid args");
        }
        Arguments { pieces, fmt: None, args }
    }

    //format_args!()完成字符串和参数解析后，如果格式化格式不是默认格式，用下面的函数创建Arguments
    pub const fn new_v1_formatted(
        pieces: &'a [&'static str],
        args: &'a [ArgumentV1<'a>],
        fmt: &'a [rt::v1::Argument],
        _unsafe_arg: UnsafeArg,
    ) -> Arguments<'a> {
        Arguments { pieces, fmt: Some(fmt), args }
    }

    //预估格式化后字符串长度
    pub fn estimated_capacity(&self) -> usize {
        //计算所有除格式化参数外的长度
        let pieces_length: usize = self.pieces.iter().map(|x| x.len()).sum();

        if self.args.is_empty() {
            pieces_length
        } else if !self.pieces.is_empty() && self.pieces[0].is_empty() && pieces_length < 16 {
            //如果字符串以格式化参数作为起始且除格式化以外的字符小于16
            0
        } else {
            //其他情况，为了防止额外申请堆内存，事先申请更多的内存
            pieces_length.checked_mul(2).unwrap_or(0)
        }
    }
}
```
对输出做格式化是非常复杂的可变参数支持的例子。从对以上的代码分析，在RUST支持可变参数的途径：
1. 首先定义一个支持可变参数的宏，例如foramt_args宏，这个宏将可变参数转变成一个数据结构，数据结构需要根据需要进行设计。
2. 根据数据结构设计方法或函数。

Vec<T>中的vec！宏也是一个典型的可变参数实现，单其用途较单纯，因此也非常简单。可变参数是非常具有直观性及方便的语法。写一些库的时候需要经常用到。

# 茶歇 
与操作系统无关的部分至此告一段落。后继将主要分析 library/std/src目录下的代码。 
std库的代码分为操作系统相关与操作系统无关两个大的部分：
操作系统相关：目录 library/std/src/sys; library/src/src/os 中的代码，其中os目录中的代码只由sys目录中的代码使用。
操作系统无关：其他部分。

其中library/std/src/ffi目录下主要是C语言与RUST语言互操作需要的模块，包括类型，C语言字符串，OS的字符串，可变参数等。

后继分析按照如下思路进行：
1. 按照操作系统的内存管理，进程/线程管理，进程/线程间通信，文件系统/IO/网络/时间，异步编程，杂项的顺序做分析
2. 每部分分析先给出RUST对操作系统定义的统一trait， 给出linux及wasi的操作系统相关部分实现的代码摘要分析，然后对操作系统无关实现部分的摘要分析。

# RUST中与C语言互通
## C语言类型定义适配
代码路径：library/core/ffi/mod.rs  
```rust
macro_rules! type_alias_no_nz {
    {
      $Docfile:tt, $Alias:ident = $Real:ty;
      $( $Cfg:tt )*
    } => {
        #[doc = include_str!($Docfile)]
        $( $Cfg )*
        #[unstable(feature = "core_ffi_c", issue = "94501")]
        pub type $Alias = $Real;
    }
}

macro_rules! type_alias {
    {
      $Docfile:tt, $Alias:ident = $Real:ty, $NZAlias:ident = $NZReal:ty;
      $( $Cfg:tt )*
    } => {
        type_alias_no_nz! { $Docfile, $Alias = $Real; $( $Cfg )* }

        #[doc = concat!("Type alias for `NonZero` version of [`", stringify!($Alias), "`]")]
        #[unstable(feature = "raw_os_nonzero", issue = "82363")]
        $( $Cfg )*
        pub type $NZAlias = $NZReal;
    }
}

//以下的定义，对所有C语言的类型以"c_xxxx"来命名，并用类型别名的形式定义为RUST的类型
//以下做了简化，仅针对linux操作系统
type_alias! { "c_char.md", c_char = c_char_definition::c_char, NonZero_c_char = c_char_definition::NonZero_c_char;
type_alias! { "c_schar.md", c_schar = i8, NonZero_c_schar = NonZeroI8; }
type_alias! { "c_uchar.md", c_uchar = u8, NonZero_c_uchar = NonZeroU8; }
type_alias! { "c_short.md", c_short = i16, NonZero_c_short = NonZeroI16; }
type_alias! { "c_ushort.md", c_ushort = u16, NonZero_c_ushort = NonZeroU16; }
type_alias! { "c_int.md", c_int = i32, NonZero_c_int = NonZeroI32; }
type_alias! { "c_uint.md", c_uint = u32, NonZero_c_uint = NonZeroU32; }
type_alias! { "c_long.md", c_long = i32, NonZero_c_long = NonZeroI32;
type_alias! { "c_ulong.md", c_ulong = u32, NonZero_c_ulong = NonZeroU32;
type_alias! { "c_longlong.md", c_longlong = i64, NonZero_c_longlong = NonZeroI64; }
type_alias! { "c_ulonglong.md", c_ulonglong = u64, NonZero_c_ulonglong = NonZeroU64; }
type_alias_no_nz! { "c_float.md", c_float = f32; }
type_alias_no_nz! { "c_double.md", c_double = f64; 

pub type c_size_t = usize;
pub type c_ptrdiff_t = isize;
pub type c_ssize_t = isize;

mod c_char_definition {
    cfg_if! {
        if #[cfg(any(
            all(
                target_os = "linux",
                any(
                    target_arch = "aarch64",
                    target_arch = "arm",
                    target_arch = "powerpc",
                    target_arch = "powerpc64",
                    target_arch = "s390x",
                    target_arch = "riscv64",
                    target_arch = "riscv32"
                )
            ),
            all(target_os = "fuchsia", target_arch = "aarch64")
        ))] {
            pub type c_char = u8;
            pub type NonZero_c_char = crate::num::NonZeroU8;
        } 
    }        
}
//以下是针对C语言的可变参数 VA_ARG给出的相关RUST匹配
pub enum c_void {
    __variant1,
    __variant2,
}

pub struct VaListImpl<'f> {
    gp_offset: i32,
    fp_offset: i32,
    overflow_arg_area: *mut c_void,
    reg_save_area: *mut c_void,
    _marker: PhantomData<&'f mut &'f c_void>,
}

pub struct VaList<'a, 'f: 'a> {
    inner: &'a mut VaListImpl<'f>,

    _marker: PhantomData<&'a mut VaListImpl<'f>>,
}

impl<'f> VaListImpl<'f> {
    pub fn as_va_list<'a>(&'a mut self) -> VaList<'a, 'f> {
        VaList { inner: self, _marker: PhantomData }
    }
}
impl<'f> Clone for VaListImpl<'f> {
    #[inline]
    fn clone(&self) -> Self {
        let mut dest = crate::mem::MaybeUninit::uninit();
        // SAFETY: we write to the `MaybeUninit`, thus it is initialized and `assume_init` is legal
        unsafe {
            va_copy(dest.as_mut_ptr(), self);
            dest.assume_init()
        }
    }
}

impl<'f> VaListImpl<'f> {
    /// Advance to the next arg.
    #[inline]
    pub unsafe fn arg<T: sealed_trait::VaArgSafe>(&mut self) -> T {
        // SAFETY: the caller must uphold the safety contract for `va_arg`.
        unsafe { va_arg(self) }
    }

    /// Copies the `va_list` at the current location.
    pub unsafe fn with_copy<F, R>(&self, f: F) -> R
    where
        F: for<'copy> FnOnce(VaList<'copy, 'f>) -> R,
    {
        let mut ap = self.clone();
        let ret = f(ap.as_va_list());
        // SAFETY: the caller must uphold the safety contract for `va_end`.
        unsafe {
            va_end(&mut ap);
        }
        ret
    }
}

extern "rust-intrinsic" {
    //以下缺少了C语言va_start的对应，RUST不需要
    /// Destroy the arglist `ap` after initialization with `va_start` or
    /// `va_copy`.
    fn va_end(ap: &mut VaListImpl<'_>);

    /// Copies the current location of arglist `src` to the arglist `dst`.
    fn va_copy<'f>(dest: *mut VaListImpl<'f>, src: &VaListImpl<'f>);

    /// Loads an argument of type `T` from the `va_list` `ap` and increment the
    /// argument `ap` points to.
    fn va_arg<T: sealed_trait::VaArgSafe>(ap: &mut VaListImpl<'_>) -> T;
}

```

## 系统调用的封装
操作系统的系统调用一般出错返回-1, 为了简化对此情况的处理，RUST实现了以下机制:    

```rust
//对系统调用出错的判断
pub trait IsMinusOne {
    fn is_minus_one(&self) -> bool;
}

macro_rules! impl_is_minus_one {
    ($($t:ident)*) => ($(impl IsMinusOne for $t {
        fn is_minus_one(&self) -> bool {
            *self == -1
        }
    })*)
}

//对所有系统调用可能的返回类型实现了出错判断trait
impl_is_minus_one! { i8 i16 i32 i64 isize }

//对系统调用进行出错处理的封装，将错误转换为Result类型
pub fn cvt<T: IsMinusOne>(t: T) -> crate::io::Result<T> {
    //Error是对操作系统错误的封装
    if t.is_minus_one() { Err(crate::io::Error::last_os_error()) } else { Ok(t) }
}

//对于被中断的系统调用做额外的处理封装
pub fn cvt_r<T, F>(mut f: F) -> crate::io::Result<T>
where
    T: IsMinusOne,
    F: FnMut() -> T,
{
    loop {
        match cvt(f()) {
            //如果返回是调用被中断, 则再次执行系统调用
            Err(ref e) if e.kind() == ErrorKind::Interrupted => {}
            other => return other,
        }
    }
}

//对系统调用的返回值进行判断，转换为io::Result的结果
pub fn cvt_nz(error: libc::c_int) -> crate::io::Result<()> {
    if error == 0 { Ok(()) } else { Err(crate::io::Error::from_raw_os_error(error)) }
}

```
## CStr及CString代码分析
代码路径：library/std/src/ffi/c_str.rs  
RUST定义CStr及CString主要的目的就是与C的各种库函数交互。
因此CStr及CString不涉及字符串的迭代器，格式化，加减，分裂，字符查找等等操作。只是负责做String及str与C语言之间的转换及与转换相关及调试相关的若干功能。
之所以设计CString，是因为如果需要保存C语言的字符串，需要用堆内存的方式来完成。同时，传递给C语言的字符串，需要位于堆内存中代码才会比较简单。
一般的处理C语言交互传入的字符串的过程是:首先需要用CStr将字符串进行包装，使得保证后继操作复合RUST的安全规则；如果需要对字符串做保存，那需要用CStr生成一个CString。当然，也可以直接转化为String，要根据具体的情况和需求处理，但仅使用String的方式在某些场景下显然效率不高并且也是不合适的。
如果是需要将RUST的str转换成C语言的字符串，则先转换成CString，

CString及CStr类型结构定义如下：
```rust
pub struct CString {
    // C语言的字符串是一个以0位结尾的字节数组. 通常的，申请的空间大小会大于字符串长度，因此
    // 下面的切片长度不能用于判断字符串长度
    inner: box<[u8]>,
}
pub struct CStr {
    // 此处没有太好的办法，C语言对字符串实际上会存储在一个申请后就固定的字节数组里，然后用指针表示字符串类型
    // 但RUST显然不可能用裸指针来实现，切片类型是最接近的。但要注意C语言中的实际上是个固定的数组
    inner: [c_char],
}
```
CStr主要的需求是对C语言的char*进行封装并定义转换方法，将C语言的字符串安全化。并在需要的时候转化成str或CString类型

```rust

impl CStr {
    //主要的创建方法，这个函数接收一个已经由C语言模块传递过来的char *指针，然后创建RUST
    //需要的CStr引用 并返回
    // 调用代码应该保证传入参数的正确性。此函数返回的引用生命周期由调用代码的上下文决定
    // 生命周期的正确性也由调用代码保证。
    pub unsafe fn from_ptr<'a>(ptr: *const c_char) -> &'a CStr {
        //将* const c_char转换成 &[u8]
        unsafe {
            //调用C语言的库函数libc::strlen获得字符串长度，这里实际可以用RUST自行实现
            let len = sys::strlen(ptr);
            let ptr = ptr as *const u8;
            //先创建&[u8], 然后创建Self类型引用
            Self::_from_bytes_with_nul_unchecked(slice::from_raw_parts(ptr, len as usize + 1))
        }
    }

    pub fn from_bytes_until_nul(bytes: &[u8]) -> Result<&CStr, FromBytesUntilNulError> {
        //core库实现了memchr，查找到字符串尾部字节位置
        let nul_pos = memchr::memchr(0, bytes);
        match nul_pos {
            Some(nul_pos) => {
                // slice仅保留有效的字节.
                let subslice = &bytes[..nul_pos + 1];
                // 见后继的分析
                Ok(unsafe { CStr::from_bytes_with_nul_unchecked(subslice) })
            }
            None => Err(FromBytesUntilNulError(())),
        }
    }
    //从准备好的[u8]创建CStr的引用并返回
    pub const unsafe fn from_bytes_with_nul_unchecked(bytes: &[u8]) -> &CStr {
        debug_assert!(!bytes.is_empty() && bytes[bytes.len() - 1] == 0);
        //见后继的分析
        unsafe { Self::_from_bytes_with_nul_unchecked(bytes) }
    }

    const unsafe fn _from_bytes_with_nul_unchecked(bytes: &[u8]) -> &Self {
        // 利用裸指针转换，注意这里CStr结构定义没有用#[repr(transparent)]或#[repr(C)]，这里直接做转换的根据感觉有些不足，
        //返回的生命周期要小于bytes，但因为bytes基本上是从一个裸指针转换而来的，所以
        //这里的&Self的生命周期的正确性还是要由调用代码负责
        unsafe { &*(bytes as *const [u8] as *const Self) }
    }

    //将CStr转换成C语言的字符串，需要保证复合C语言字符串的规则
    //此函数可能引发一个潜在问题如下例：
    /// use std::ffi::CString;
    ///
    /// let ptr = CString::new("Hello").expect("CString::new failed").as_ptr();
    /// unsafe {
    ///     // 这里会出现悬垂指针,见后面的解释
    ///     *ptr;
    /// }
    /// ```
    ///
    /// 以上悬垂指针是因为`as_ptr`没有生命周期，因为CString创建的变量又没有变量声明与之绑定，所以其在执行完as_ptr后立即被释放。
    /// 可使用如下的方法
    /// ```no_run
    /// # #![allow(unused_must_use)]
    /// use std::ffi::CString;
    /// 
    /// //声明一个变量，生命周期一般会到作用域的尾部。
    /// let hello = CString::new("Hello").expect("CString::new failed");
    /// let ptr = hello.as_ptr();
    /// unsafe {
    ///     // `ptr` is valid because `hello` is in scope
    ///     *ptr;
    /// }
    /// ```
    pub const fn as_ptr(&self) -> *const c_char {
        self.inner.as_ptr()
    }

    //转换成去掉尾部0的[u8]切片引用
    pub fn to_bytes(&self) -> &[u8] {
        let bytes = self.to_bytes_with_nul();
        // SAFETY: to_bytes_with_nul returns slice with length at least 1
        unsafe { bytes.get_unchecked(..bytes.len() - 1) }
    }

    //转换成[u8]切片引用,尾部仍然有0
    pub fn to_bytes_with_nul(&self) -> &[u8] {
        unsafe { &*(&self.inner as *const [c_char] as *const [u8]) }
    }

    //转换成&str
    pub fn to_str(&self) -> Result<&str, str::Utf8Error> {
        str::from_utf8(self.to_bytes())
    }

    //转换成CString
    pub fn into_c_string(self: Box<CStr>) -> CString {
        //将堆内存从Box取出
        let raw = Box::into_raw(self) as *mut [u8];
        //重新形成Box结构，然后创建CString
        CString { inner: unsafe { Box::from_raw(raw) } }
    }
}

```
CString的相关实现如下：
```rust

impl CString {
    pub fn new<T: Into<Vec<u8>>>(t: T) -> Result<CString, NulError> {
        trait SpecNewImpl {
            fn spec_new_impl(self) -> Result<CString, NulError>;
        }

        impl<T: Into<Vec<u8>>> SpecNewImpl for T {
            default fn spec_new_impl(self) -> Result<CString, NulError> {
                let bytes: Vec<u8> = self.into();
                match memchr::memchr(0, &bytes) {
                    Some(i) => Err(NulError(i, bytes)),
                    None => Ok(unsafe { CString::_from_vec_unchecked(bytes) }),
                }
            }
        }

        //此函数用来防止多次申请内存
        fn spec_new_impl_bytes(bytes: &[u8]) -> Result<CString, NulError> {
            // 此处checked_add的优化效率最高，bytes中没有0，所以需要加1
            let capacity = bytes.len().checked_add(1).unwrap();

            // 申请堆内存，并将bytes写入堆内存,此处申请可以防止重复申请，但无论成功与否都会申请内存
            let mut buffer = Vec::with_capacity(capacity);
            //此时还没有给buffer的尾部赋0
            buffer.extend(bytes);

            // 看bytes内是否有0值
            match memchr::memchr(0, bytes) {
                //有0，出错了，将buffer在参数返回，由外部代码处理
                Some(i) => Err(NulError(i, buffer)),
                //无0，生成CString，生成函数中会赋0
                None => Ok(unsafe { CString::_from_vec_unchecked(buffer) }),
            }
        }

        //可以从[u8]切片生成CString
        impl SpecNewImpl for &'_ [u8] {
            fn spec_new_impl(self) -> Result<CString, NulError> {
                spec_new_impl_bytes(self)
            }
        }

        //支持从str生成CString
        impl SpecNewImpl for &'_ str {
            fn spec_new_impl(self) -> Result<CString, NulError> {
                spec_new_impl_bytes(self.as_bytes())
            }
        }

        //支持从可变[u8]生成CString
        impl SpecNewImpl for &'_ mut [u8] {
            fn spec_new_impl(self) -> Result<CString, NulError> {
                spec_new_impl_bytes(self)
            }
        }

        t.spec_new_impl()
    }

    //从Vec创建CString,实际是从String创建的支持函数
    pub unsafe fn from_vec_unchecked(v: Vec<u8>) -> Self {
        debug_assert!(memchr::memchr(0, &v).is_none());
        unsafe { Self::_from_vec_unchecked(v) }
    }

    //Vec<u8>已经完成安全检查，不会出错
    unsafe fn _from_vec_unchecked(mut v: Vec<u8>) -> Self {
        //以下就是增加尾部的0值
        v.reserve_exact(1);
        v.push(0);
        //将堆内存从Vec结构转移至Box结构
        Self { inner: v.into_boxed_slice() }
    }

    //从C语言字符串创建CString, 此时c语言的字符串应该是前期RUST代码申请的堆内存
    //要规避不是RUST申请的堆内存的情况
    pub unsafe fn from_raw(ptr: *mut c_char) -> CString {
        // ptr应该从CString::into_raw得到的,此方法使用后，可以省略一次内存拷贝
        unsafe {
            //得到字符串长度
            let len = sys::strlen(ptr) + 1; // Including the NUL byte
            //形成正确的切片引用 
            let slice = slice::from_raw_parts_mut(ptr, len as usize);
            //形成CString
            CString { inner: Box::from_raw(slice as *mut [c_char] as *mut [u8]) }
        }
    }

    pub fn into_raw(self) -> *mut c_char {
        //CString已经包含了0值
        Box::into_raw(self.into_inner()) as *mut c_char
    }

    //转换成String类型
    pub fn into_string(self) -> Result<String, IntoStringError> {
        String::from_utf8(self.into_bytes()).map_err(|e| IntoStringError {
            error: e.utf8_error(),
            inner: unsafe { Self::_from_vec_unchecked(e.into_bytes()) },
        })
    }

    pub fn into_bytes(self) -> Vec<u8> {
        //消费了CString，Box中的堆内存转移到Vec
        let mut vec = self.into_inner().into_vec();
        //删掉尾部的0值
        let _nul = vec.pop();
        debug_assert_eq!(_nul, Some(0u8));
        vec
    }

    pub fn into_bytes_with_nul(self) -> Vec<u8> {
        //不对尾部的0值做处理
        self.into_inner().into_vec()
    }

    //将CString转换为[u8]切片引用
    pub fn as_bytes(&self) -> &[u8] {
        // 删除尾部的0值
        unsafe { self.inner.get_unchecked(..self.inner.len() - 1) }
    }

    //保留尾部的0值
    pub fn as_bytes_with_nul(&self) -> &[u8] {
        &self.inner
    }

    //转换为CStr的引用
    pub fn as_c_str(&self) -> &CStr {
        &*self
    }

    
    pub fn into_boxed_c_str(self) -> Box<CStr> {
        //Box取出堆内存指针，然后转换，再封装入Box，RUST这个实在是麻烦
        unsafe { Box::from_raw(Box::into_raw(self.into_inner()) as *mut CStr) }
    }

    fn into_inner(self) -> Box<[u8]> {
        //将Box取出，如果直接解封装的方式，因为会调用self的drop函数，会再调用内部的Box的drop。
        //用MannuallyDrop来规避是不想再重构了，这个代码的例子不应学习
        //如果有需要inner，那就不应该用Box<[u8]>这种方式来设计
        //这个设计导致必须用下面这种技巧，带来理解上的复杂性
        let this = mem::ManuallyDrop::new(self);
        unsafe { ptr::read(&this.inner) }
    }

    //从Vec生成CString
    pub unsafe fn from_vec_with_nul_unchecked(v: Vec<u8>) -> Self {
        debug_assert!(memchr::memchr(0, &v).unwrap() + 1 == v.len());
        unsafe { Self::_from_vec_with_nul_unchecked(v) }
    }

    //同上，无需再检查0值
    unsafe fn _from_vec_with_nul_unchecked(v: Vec<u8>) -> Self {
        Self { inner: v.into_boxed_slice() }
    }

    //此函数为从String转换为CString准备
    pub fn from_vec_with_nul(v: Vec<u8>) -> Result<Self, FromVecWithNulError> {
        //确定0值的位置
        let nul_pos = memchr::memchr(0, &v);
        match nul_pos {
            //如果0值的位置正确
            Some(nul_pos) if nul_pos + 1 == v.len() => {
                // 创建CString
                Ok(unsafe { Self::_from_vec_with_nul_unchecked(v) })
            }
            //出错处理
            Some(nul_pos) => Err(FromVecWithNulError {
                error_kind: FromBytesWithNulErrorKind::InteriorNul(nul_pos),
                bytes: v,
            }),
            None => Err(FromVecWithNulError {
                error_kind: FromBytesWithNulErrorKind::NotNulTerminated,
                bytes: v,
            }),
        }
    }
}

//drop函数
impl Drop for CString {
    fn drop(&mut self) {
        unsafe {
            //消费了Box，堆内存已经拷贝到栈，然后将C语言的字符串设置为空字符串。
            *self.inner.get_unchecked_mut(0) = 0;
            
        }
    }
}

impl ops::Deref for CString {
    type Target = CStr;

    fn deref(&self) -> &CStr {
        unsafe { CStr::_from_bytes_with_nul_unchecked(self.as_bytes_with_nul()) }
    }
}
```
CString, CStr其他代码略。

## 代码工程中的一个技巧
在对不同的CPU架构，不同的操作系统进行适配的时候，通常在代码中采用如下的组织方式：
1. 有接口定义文件，在C语言中一般用头文件，在RUST中用mod.rs文件，这个文件负责向其他模块提供一致的API访问界面
2. 每种CPU架构或者每种操作系统各自建立一个目录(模块)
3. 每种CPU架构或者每种操作系统各自实现接口的代码都在此目录下实现
4. 利用编译参数控制特定CPU架构，操作系统仅编译特定目录下的代码
举例如下：
```rust
mod common;

cfg_if::cfg_if! {
    if #[cfg(unix)] {
        mod unix;
        pub use self::unix::*;
    } else if #[cfg(windows)] {
        mod windows;
        pub use self::windows::*;
    } else if #[cfg(target_os = "solid_asp3")] {
        mod solid;
        pub use self::solid::*;
    } else if #[cfg(target_os = "hermit")] {
        mod hermit;
        pub use self::hermit::*;
    } else if #[cfg(target_os = "wasi")] {
        mod wasi;
        pub use self::wasi::*;
    } else if #[cfg(target_family = "wasm")] {
        mod wasm;
        pub use self::wasm::*;
    } else if #[cfg(all(target_vendor = "fortanix", target_env = "sgx"))] {
        mod sgx;
        pub use self::sgx::*;
    } else {
        mod unsupported;
        pub use self::unsupported::*;
    }
}
```
RUST对编译的控制是直接在本身的代码中实现的，利用mod 语法控制了编译的目录。这是RUST的一个优势，对于C，需要在Makefile里面来控制编译那个目录。
RUST用`pub use self::windows::*`的语法，将特定的操作系统的模块重导出为 `std::sys::*`，从而对其他的RUST模块实现了对不同操作系统API接口访问的统一。
类似的设计方式可能会在多种场景下遇到，例如对不同数据库API的适配，对不同3D API的适配等等。
## OsString 代码分析
操作系统系统调用采用的String很可能与C语言不同，也可能与RUST不同。基于与CStr及CString类似的理由，RUST也实现了OsStr及OsString。显然，这个模块包括了操作系统相关及操作系统无关的两个部分：
操作系统无关部分代码路径如下：  
library/src/std/src/ffi/os_str.rs    
操作系统相关部分代码路径如下，(仅列出linux及windows)： 
library/src/std/src/sys/unix/os_str.rs   
library/src/std/src/sys/windows/os_str.rs  

linux操作系统相关部分的接口类型结构定义： 
```rust
#[repr(transparent)]
pub struct Buf {
    pub inner: Vec<u8>,
}

#[repr(transparent)]
pub struct Slice {
    pub inner: [u8],
}
```

windows操作系统相关部分的接口结构定义：
```rust
pub struct Buf {
    pub inner: Wtf8Buf,
}
pub struct Slice {
    pub inner: Wtf8,
}
pub struct Wtf8Buf {
    bytes: Vec<u8>,
}
pub struct Wtf8 {
    bytes: [u8],
}
```

OsString及OsStr的定义：
```rust
pub struct OsString {
    inner: Buf,
}

pub struct OsStr {
    inner: Slice,
}
```
OsString及OsStr实际上是两个适配器，每个方法基本上都是做个透传，如：
```rust
impl OsString {
    pub fn new() -> OsString {
        OsString { inner: Buf::from_string(String::new()) }
    }
    
    pub fn into_string(self) -> Result<String, OsString> {
        self.inner.into_string().map_err(|buf| OsString { inner: buf })
    }
    ...
}
```
OsString及OStr的方法与CString及CStr高度类似，分析略。

# std的内存管理分析
std库与core库在内存管理RUST提供的机制是统一的。即Allocator trait 与 GlobalAlloc trait
std库用System 作为这两个trait的实现载体，core库中使用的是Global，但Global没有实现GlobalAlloc trait：  
```rust
//单元结构体，仅用来作为内存管理的实现载体
pub struct System;

impl System {
    //具体的内存申请实现，与Global类似，可参考前文的解释
    fn alloc_impl(&self, layout: Layout, zeroed: bool) -> Result<NonNull<[u8]>, AllocError> {
        match layout.size() {
            0 => Ok(NonNull::slice_from_raw_parts(layout.dangling(), 0)),
            size => unsafe {
                let raw_ptr = if zeroed {
                    //System也实现了GlobalAlloc trait，这与core库中不同，core库中对GlobalAlloc trait中方法的调用是使用了编译器提供了包装。这使得core库即可以适用于用户态也可以适用于内核态。但std库就是在用户态，所以解决方法更直接。
                    GlobalAlloc::alloc_zeroed(self, layout)
                } else {
                    GlobalAlloc::alloc(self, layout)
                };
                let ptr = NonNull::new(raw_ptr).ok_or(AllocError)?;
                Ok(NonNull::slice_from_raw_parts(ptr, size))
            },
        }
    }

    //内存不足，需要增加空间的申请操作
    unsafe fn grow_impl(
        &self,
        ptr: NonNull<u8>,
        old_layout: Layout,
        new_layout: Layout,
        zeroed: bool,
    ) -> Result<NonNull<[u8]>, AllocError> {
        debug_assert!(
            new_layout.size() >= old_layout.size(),
            "`new_layout.size()` must be greater than or equal to `old_layout.size()`"
        );

        match old_layout.size() {
            //旧的空间是0，那相当于申请一个新空间的操作
            0 => self.alloc_impl(new_layout, zeroed),

            //旧的内存块与新内存块的对齐是一致的
            old_size if old_layout.align() == new_layout.align() => unsafe {
                //直接调用realloc的逻辑即可，
                let new_size = new_layout.size();

                intrinsics::assume(new_size >= old_layout.size());
                //realloc的逻辑保证旧的内存块的内容被保留
                let raw_ptr = GlobalAlloc::realloc(self, ptr.as_ptr(), old_layout, new_size);
                let ptr = NonNull::new(raw_ptr).ok_or(AllocError)?;
                if zeroed {
                    //旧内存块的内容不变，仅对新内存块处理
                    raw_ptr.add(old_size).write_bytes(0, new_size - old_size);
                }
                //形成新的NonNull<[u8]>返回
                Ok(NonNull::slice_from_raw_parts(ptr, new_size))
            },
            
            //旧内存块与新内存块的对齐不一致
            old_size => unsafe {
                //需要按照新的内存布局参数重新申请一块内存
                let new_ptr = self.alloc_impl(new_layout, zeroed)?;
                //将旧内存块的内容拷贝到新内存
                ptr::copy_nonoverlapping(ptr.as_ptr(), new_ptr.as_mut_ptr(), old_size);
                //将旧内存块释放掉
                Allocator::deallocate(&self, ptr, old_layout);
                Ok(new_ptr)
            },
        }
    }
}

// 实现Allocator trait, std库后继会使用此trait完成内存管理操作。
unsafe impl Allocator for System {
    //申请内存块
    fn allocate(&self, layout: Layout) -> Result<NonNull<[u8]>, AllocError> {
        self.alloc_impl(layout, false)
    }
    //申请内存块，并将内存块清零
    fn allocate_zeroed(&self, layout: Layout) -> Result<NonNull<[u8]>, AllocError> {
        self.alloc_impl(layout, true)
    }
    //释放内存块
    unsafe fn deallocate(&self, ptr: NonNull<u8>, layout: Layout) {
        if layout.size() != 0 {
            unsafe { GlobalAlloc::dealloc(self, ptr.as_ptr(), layout) }
        }
    }
    //增长内存空间
    unsafe fn grow(
        &self,
        ptr: NonNull<u8>,
        old_layout: Layout,
        new_layout: Layout,
    ) -> Result<NonNull<[u8]>, AllocError> {
        unsafe { self.grow_impl(ptr, old_layout, new_layout, false) }
    }
    //增长内存空间，增长的部分进行清零
    unsafe fn grow_zeroed(
        &self,
        ptr: NonNull<u8>,
        old_layout: Layout,
        new_layout: Layout,
    ) -> Result<NonNull<[u8]>, AllocError> {
        unsafe { self.grow_impl(ptr, old_layout, new_layout, true) }
    }
    //收缩内存空间，此时必须重新申请内存。
    unsafe fn shrink(
        &self,
        ptr: NonNull<u8>,
        old_layout: Layout,
        new_layout: Layout,
    ) -> Result<NonNull<[u8]>, AllocError> {
        debug_assert!(
            new_layout.size() <= old_layout.size(),
            "`new_layout.size()` must be smaller than or equal to `old_layout.size()`"
        );
        
        match new_layout.size() {
            // 收缩空间至0, 实际上就是释放内存
            0 => unsafe {
                Allocator::deallocate(&self, ptr, old_layout);
                //返回一个dangling的指针表示悬垂指针。此处应该用Option<NonNull<[u8]>>返回才符合
                //rust的习惯用法吧，目前的返回代码后继有额外的判断负担
                Ok(NonNull::slice_from_raw_parts(new_layout.dangling(), 0))
            },

            //如果内存对齐相同
            new_size if old_layout.align() == new_layout.align() => unsafe {
                intrinsics::assume(new_size <= old_layout.size());
                //realloc函数会保留原内存内容
                let raw_ptr = GlobalAlloc::realloc(self, ptr.as_ptr(), old_layout, new_size);
                let ptr = NonNull::new(raw_ptr).ok_or(AllocError)?;
                Ok(NonNull::slice_from_raw_parts(ptr, new_size))
            },

            //对齐不同，必须重新申请内存
            new_size => unsafe {
                let new_ptr = Allocator::allocate(&self, new_layout)?;
                //将原内存内容拷贝入新内存
                ptr::copy_nonoverlapping(ptr.as_ptr(), new_ptr.as_mut_ptr(), new_size);
                //释放原内存
                Allocator::deallocate(&self, ptr, old_layout);
                Ok(new_ptr)
            },
        }
    }
}
```
以上是std库的通用实现,代码位置：library/std/src/alloc.rs

以下是unix操作系统的适配，代码位置：library/std/src/sys/unix/alloc.rs  
不同的操作系统，其内存申请的系统调用都不一致，因此对GlobalAlloc的实现也不一致。  
```rust
unsafe impl GlobalAlloc for System {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        // 用libc来实现内存申请，这里适配的难点在于对齐的适配，libc的对齐实际上是不能指定的。
        // 实际上，在读这个代码前我就从来没有意识到适配会出现这个问题，只有在申请的内存对齐小于MIN_ALIGN以及申请的内存大小大于对齐大小时才能调用libc的malloc做申请
        if layout.align() <= MIN_ALIGN && layout.align() <= layout.size() {
            libc::malloc(layout.size()) as *mut u8
        } else {
            #[cfg(target_os = "macos")]
            {
                if layout.align() > (1 << 31) {
                    return ptr::null_mut();
                }
            }
            aligned_malloc(&layout)
        }
    }

    unsafe fn alloc_zeroed(&self, layout: Layout) -> *mut u8 {
        // 一样需要处理对齐
        if layout.align() <= MIN_ALIGN && layout.align() <= layout.size() {
            libc::calloc(layout.size(), 1) as *mut u8
        } else {
            let ptr = self.alloc(layout);
            //不能用calloc处理时需要清零
            if !ptr.is_null() {
                ptr::write_bytes(ptr, 0, layout.size());
            }
            ptr
        }
    }

    unsafe fn dealloc(&self, ptr: *mut u8, _layout: Layout) {
        libc::free(ptr as *mut libc::c_void)
    }

    unsafe fn realloc(&self, ptr: *mut u8, layout: Layout, new_size: usize) -> *mut u8 {
        //对齐处理
        if layout.align() <= MIN_ALIGN && layout.align() <= new_size {
            libc::realloc(ptr as *mut libc::c_void, new_size) as *mut u8
        } else {
            //无法对齐时的处理，
            realloc_fallback(self, ptr, layout, new_size)
        }
    }
}

pub unsafe fn realloc_fallback(
    alloc: &System,
    ptr: *mut u8,
    old_layout: Layout,
    new_size: usize,
) -> *mut u8 {
    // Docs for GlobalAlloc::realloc require this to be valid:
    let new_layout = Layout::from_size_align_unchecked(new_size, old_layout.align());

    let new_ptr = GlobalAlloc::alloc(alloc, new_layout);
    if !new_ptr.is_null() {
        let size = cmp::min(old_layout.size(), new_size);
        ptr::copy_nonoverlapping(ptr, new_ptr, size);
        GlobalAlloc::dealloc(alloc, ptr, old_layout);
    }
    new_ptr
}
cfg_if::cfg_if! {
    if #[cfg(target_os = "wasi")] {
        //wasi提供aligned_alloc的支持
        unsafe fn aligned_malloc(layout: &Layout) -> *mut u8 {
            libc::aligned_alloc(layout.align(), layout.size()) as *mut u8
        }
    } else {
        //其他需要用posix_memalign来完成
        unsafe fn aligned_malloc(layout: &Layout) -> *mut u8 {
            let mut out = ptr::null_mut();
            // posix_memalign requires that the alignment be a multiple of `sizeof(void*)`.
            // Since these are all powers of 2, we can just use max.
            let align = layout.align().max(crate::mem::size_of::<usize>());
            let ret = libc::posix_memalign(&mut out, align, layout.size());
            if ret != 0 { ptr::null_mut() } else { out as *mut u8 }
        }
    }
}
```
# std库文件描述符代码分析
以linux为例，文件系统实际上成为操作系统的所有资源的管理体系的基础设施。因此，将之放在内存管理之后来做分析。
RUST将文件管理的结构分成：
1. 操作系统文件描述符的封装结构。对于RUST来说，操作系统的文件描述符与堆内存要处理的安全性类似。需要建立类似智能指针的结构完成对其的管理，并纳入到RUST的安全体系内。
2. 1创建的类型结构主要用于解决安全问题，如果在同一个类型加上文件功能，会乱。因此在1的基础上建立RUST自身的文件描述符类型，用于处理文件的读写等功能
3. 在2的基础上，实现普通的文件，目录文件，Socket，Pipe，IO设备文件等逻辑文件类型。
   
本章将讨论1及2，3以后在涉及到各模块时再进行详细分析   
代码目录： library/src/std/src/os/fd/raw.rs   
          library/src/std/src/os/fd/owned.rs  
          library/src/std/src/sys/unix/fd.rs
        
## 操作系统的文件描述符适配所有权封装
RUST当然要使用操作系统调用返回的fd来操作文件，操作系统返回的文件RUST定义类型RawFd。RawFd本身和裸指针一样，是没有所有权概念的，但RUST中文件显然需要具备所有权，RUST在RawFd上定义了封装类型OwnedFd来实现针对RawFd的所有权，又定义了类型BorrowedFd作为OwnedFd的借用类型。  
理解这两个类型，我们可以把RawFd类比与裸指针* const T， OwnedFd类比与 T, BorrowedFd类比于&T。

以linux操作系统为基础进行分析：
```rust
//虽然是int型，但因为表示了系统的资源，所以可以类比于裸指针。后继被标识文件所有权的封装类型所封装后才能进入安全的RUST领域。
pub type RawFd = raw::c_int;

//此trait用于从封装RawFd的类型中获取RawFd, 此时返回的RawFd安全性类似于裸指针。
pub trait AsRawFd {
    fn as_raw_fd(&self) -> RawFd;
}

//从RawFd创建一个封装类型,返回的Self获得了RawFd代表的文件的所有权 
pub trait FromRawFd {
    unsafe fn from_raw_fd(fd: RawFd) -> Self;
}

//将封装类型变量消费掉，并返回RawFd，此时RUST中没有其他变量拥有RawFd代表文件的所有权，后继要由RawFd对close负责，或者将RawFd重新封装入另一个表示所有权的封装类型变量。 
pub trait IntoRawFd {
    fn into_raw_fd(self) -> RawFd;
}

//获取标准输入的RawFd
impl AsRawFd for io::Stdin {
    fn as_raw_fd(&self) -> RawFd {
        //libc的标准输入文件标识宏
        libc::STDIN_FILENO
    }
}

//标准输出的RawFd
impl AsRawFd for io::Stdout {
    fn as_raw_fd(&self) -> RawFd {
        //libc的标准输出宏
        libc::STDOUT_FILENO
    }
}

//标准错误的RawFd
impl AsRawFd for io::Stderr {
    fn as_raw_fd(&self) -> RawFd {
        //libc的标准错误宏
        libc::STDERR_FILENO
    }
}
```
拥有RawFd所有权的OwnedFd类型结构及OwnedFd的借用类型结构BorrowedFd。
```rust
#[repr(transparent)]
pub struct BorrowedFd<'fd> {
    fd: RawFd,
    //用OwnedFd作为RawFd的所有权版本，RawFd实际上可认为是对OwnedFd的借用。
    //但仅用fd无法表达出生命周期和借用关系，
    //这里的PhantomData用OwnedFd的引用及生命周期泛型表示了这个关系
    _phantom: PhantomData<&'fd OwnedFd>,
}

#[repr(transparent)]
pub struct OwnedFd {
    //这个封装仅是一个形式，编译器并没有认为OwnedFd已经拥有了fd所代表文件的所有权。
    //所以，OwnedFd拥有所有权这个事情实际上是代码约定，其他代码务必不能导致用RawFd创建另一份
    //OwnedFd, 也不能另外调用fd的close。
    //调用操作系统的系统调用获得文件Fd后，应该第一时间用OwnedFd进行封装，后继如果要使用，则应该
    //用borrow的方法来借出BorrowedFd，
    fd: RawFd,
}

impl BorrowedFd<'_> {
    //直接在RawFd上生成BorrowFd,注意，这里的RawFd应该已经被一个OwnedFd所包装,否则这里不正确
    pub unsafe fn borrow_raw(fd: RawFd) -> Self {
        assert_ne!(fd, u32::MAX as RawFd);
        //这里的PhantomData的赋值令人疑惑，只能认为是编译器的魔术了 
        unsafe { Self { fd, _phantom: PhantomData } }
    }
}

impl OwnedFd {
    //复制，这里是在操作系统内部复制了一个新的fd，需要调用系统调用完成
    pub fn try_clone(&self) -> crate::io::Result<Self> {
        // 设置复制的功能设定标志
        let cmd = libc::F_DUPFD_CLOEXEC;

        //调用libc库完成复制，返回新的fd
        let fd = cvt(unsafe { libc::fcntl(self.as_raw_fd(), cmd, 0) })?;
        //用新的fd创建新的Owned变量
        Ok(unsafe { Self::from_raw_fd(fd) })
    }
}

impl AsRawFd for BorrowedFd<'_> {
    //此方法应该尽量仅用于调用系统调用的时候使用
    fn as_raw_fd(&self) -> RawFd {
        self.fd
    }
}

impl AsRawFd for OwnedFd {
    //此方法应该尽量仅用于调用系统调用时使用
    fn as_raw_fd(&self) -> RawFd {
        self.fd
    }
}

impl IntoRawFd for OwnedFd {
    fn into_raw_fd(self) -> RawFd {
        let fd = self.fd;
        //必须forget，否则会触发drop调用close(fd)
        forget(self);
        fd
    }
}

impl FromRawFd for OwnedFd {
    //应该只能用这个方法创建OwnedFd
    unsafe fn from_raw_fd(fd: RawFd) -> Self {
        assert_ne!(fd, u32::MAX as RawFd);
        unsafe { Self { fd } }
    }
}

//这个方法证明OwnedFd拥有了操作系统返回的fd的所有权
impl Drop for OwnedFd {
    fn drop(&mut self) {
        unsafe {
            let _ = libc::close(self.fd);
        }
    }
}

//对OwnedFd创建借用的trait
pub trait AsFd {
    fn as_fd(&self) -> BorrowedFd<'_>;
}

impl AsFd for OwnedFd {
    fn as_fd(&self) -> BorrowedFd<'_> {
        //BorrowedFd中的PhantomData从&self中获得
        unsafe { BorrowedFd::borrow_raw(self.as_raw_fd()) }
    }
}
//以下为所有的高层视角资源生成OwnedFd的借用
impl AsFd for fs::File {
    fn as_fd(&self) -> BorrowedFd<'_> {
        //实质是OwnedFd.as_fd
        self.as_inner().as_fd()
    }
}

impl From<fs::File> for OwnedFd {
    fn from(file: fs::File) -> OwnedFd {
        //消费了File
        file.into_inner().into_inner().into_inner()
        //此处不涉及对file的forget
    }
}

impl From<OwnedFd> for fs::File {
    fn from(owned_fd: OwnedFd) -> Self {
        //创建fs::File
        Self::from_inner(FromInner::from_inner(FromInner::from_inner(owned_fd)))
    }
}

impl AsFd for crate::net::TcpStream {
    fn as_fd(&self) -> BorrowedFd<'_> {
        //socket在unix与fd没有区别，也使用OwnedFd和BorrowedFd来做所有权的解决方案
        self.as_inner().socket().as_fd()
    }
}

impl From<crate::net::TcpStream> for OwnedFd {
    fn from(tcp_stream: crate::net::TcpStream) -> OwnedFd {
        //消费掉tcp_stream，具体在tcp stream分析
        tcp_stream.into_inner().into_socket().into_inner().into_inner().into()
    }
}

impl From<OwnedFd> for crate::net::TcpStream {
    fn from(owned_fd: OwnedFd) -> Self {
        //后继在TcpStream章节分析
        Self::from_inner(FromInner::from_inner(FromInner::from_inner(FromInner::from_inner(
            owned_fd,
        ))))
    }
}

impl AsFd for crate::net::TcpListener {
    fn as_fd(&self) -> BorrowedFd<'_> {
        //同TcpStream
        self.as_inner().socket().as_fd()
    }
}

impl From<crate::net::TcpListener> for OwnedFd {
    fn from(tcp_listener: crate::net::TcpListener) -> OwnedFd {
        tcp_listener.into_inner().into_socket().into_inner().into_inner().into()
    }
}

impl From<OwnedFd> for crate::net::TcpListener {
    fn from(owned_fd: OwnedFd) -> Self {
        Self::from_inner(FromInner::from_inner(FromInner::from_inner(FromInner::from_inner(
            owned_fd,
        ))))
    }
}

impl AsFd for crate::net::UdpSocket {
    fn as_fd(&self) -> BorrowedFd<'_> {
        //UDP与TCP类似
        self.as_inner().socket().as_fd()
    }
}

impl From<crate::net::UdpSocket> for OwnedFd {
    fn from(udp_socket: crate::net::UdpSocket) -> OwnedFd {
        udp_socket.into_inner().into_socket().into_inner().into_inner().into()
    }
}

impl From<OwnedFd> for crate::net::UdpSocket {
    fn from(owned_fd: OwnedFd) -> Self {
        Self::from_inner(FromInner::from_inner(FromInner::from_inner(FromInner::from_inner(
            owned_fd,
        ))))
    }
}
```
对于需要调用RUST以外语言实现的第三方库时，都会面临一个从第三方库获取的资源如何在RUST设计其所有权的问题。Unix的fd的方案给出了一个经典的设计方式。即把第三方库获取的资源在逻辑上类似于裸指针，如RawFd。然后用一个封装结构封装用于所有权实现，例如OwnedFd。用另一个封装结构用作借用，例如BorrowedFd。这个设计方案在真正的生产环境中会经常被用到。    

## RUST标准库文件描述符的结构与实现
在OwnedFd的基础上创建的结构，用来完成对文件的通用操作
```rust
pub struct FileDesc(OwnedFd);

impl FileDesc {
    //从文件描述符读出字节流
    pub fn read(&self, buf: &mut [u8]) -> io::Result<usize> {
        let ret = cvt(unsafe {
            //调用libc的read函数
            libc::read(
                //按C语言的调用进行转换
                self.as_raw_fd(),
                //转换成void *指针
                buf.as_mut_ptr() as *mut c_void,
                //不能超过buf，也不能超过一次读的最大长度
                cmp::min(buf.len(), READ_LIMIT),
            )
        })?;
        Ok(ret as usize)
    }

    //对应于libc的iovec读的方式,具体请参考libc的说明
    pub fn read_vectored(&self, bufs: &mut [IoSliceMut<'_>]) -> io::Result<usize> {
        let ret = cvt(unsafe {
            libc::readv(
                self.as_raw_fd(),
                bufs.as_ptr() as *const libc::iovec,
                cmp::min(bufs.len(), max_iov()) as c_int,
            )
        })?;
        Ok(ret as usize)
    }

    //一直读到文件结束
    pub fn read_to_end(&self, buf: &mut Vec<u8>) -> io::Result<usize> {
        let mut me = self;
        (&mut me).read_to_end(buf)
    }

    //从文件的某一个位置开始读，请参考libc的pread64的man
    pub fn read_at(&self, buf: &mut [u8], offset: u64) -> io::Result<usize> {
        use libc::pread64;

        unsafe {
            cvt(pread64(
                self.as_raw_fd(),
                buf.as_mut_ptr() as *mut c_void,
                cmp::min(buf.len(), READ_LIMIT),
                offset as i64,
            ))
            .map(|n| n as usize)
        }
    }

    //读到buffer中的某一个位置
    pub fn read_buf(&self, buf: &mut ReadBuf<'_>) -> io::Result<()> {
        let ret = cvt(unsafe {
            libc::read(
                self.as_raw_fd(),
                buf.unfilled_mut().as_mut_ptr() as *mut c_void,
                cmp::min(buf.remaining(), READ_LIMIT),
            )
        })?;

        //原有的空间是MaybeUninit，读到内容后需要进行初始化标注
        unsafe {
            buf.assume_init(ret as usize);
        }
        //更新内容长度
        buf.add_filled(ret as usize);
        Ok(())
    }

    //向文件描述符写入字节流
    pub fn write(&self, buf: &[u8]) -> io::Result<usize> {
        let ret = cvt(unsafe {
            libc::write(
                self.as_raw_fd(),
                buf.as_ptr() as *const c_void,
                cmp::min(buf.len(), READ_LIMIT),
            )
        })?;
        Ok(ret as usize)
    }

    //iovec的方式写入
    pub fn write_vectored(&self, bufs: &[IoSlice<'_>]) -> io::Result<usize> {
        let ret = cvt(unsafe {
            libc::writev(
                self.as_raw_fd(),
                bufs.as_ptr() as *const libc::iovec,
                cmp::min(bufs.len(), max_iov()) as c_int,
            )
        })?;
        Ok(ret as usize)
    }

    //在文件的某一位置写入字节流
    pub fn write_at(&self, buf: &[u8], offset: u64) -> io::Result<usize> {
        use libc::pwrite64;

        unsafe {
            cvt(pwrite64(
                self.as_raw_fd(),
                buf.as_ptr() as *const c_void,
                cmp::min(buf.len(), READ_LIMIT),
                offset as i64,
            ))
            .map(|n| n as usize)
        }
    }

    //获取FD_CLOEXEC，具体请参考libc的相关手册
    pub fn get_cloexec(&self) -> io::Result<bool> {
        unsafe { Ok((cvt(libc::fcntl(self.as_raw_fd(), libc::F_GETFD))? & libc::FD_CLOEXEC) != 0) }
    }

    //设置FD_CLOEXEC的属性，一般会在打开文件时完成设置，否则要注意不同线程竞争问题
    pub fn set_cloexec(&self) -> io::Result<()> {
        unsafe {
            let previous = cvt(libc::fcntl(self.as_raw_fd(), libc::F_GETFD))?;
            let new = previous | libc::FD_CLOEXEC;
            if new != previous {
                cvt(libc::fcntl(self.as_raw_fd(), libc::F_SETFD, new))?;
            }
            Ok(())
        }
    }

    //设置为非阻塞
    pub fn set_nonblocking(&self, nonblocking: bool) -> io::Result<()> {
        unsafe {
            let v = nonblocking as c_int;
            cvt(libc::ioctl(self.as_raw_fd(), libc::FIONBIO, &v))?;
            Ok(())
        }
    }

    //复制文件描述符
    pub fn duplicate(&self) -> io::Result<FileDesc> {
        Ok(Self(self.0.try_clone()?))
    }

    //后继可以加入其他需要的通用文件操作方法
}

impl AsInner<OwnedFd> for FileDesc {
    //不消费FileDesc获取内部引用
    fn as_inner(&self) -> &OwnedFd {
        &self.0
    }
}

impl IntoInner<OwnedFd> for FileDesc {
    fn into_inner(self) -> OwnedFd {
        //消费self，获得内部OwnedFd
        //不必做其他资源释放操作
        self.0
    }
}

impl FromInner<OwnedFd> for FileDesc {
    //从参数创建FileDesc类型变量
    fn from_inner(owned_fd: OwnedFd) -> Self {
        Self(owned_fd)
    }
}

impl AsFd for FileDesc {
    //创建一个引用
    fn as_fd(&self) -> BorrowedFd<'_> {
        self.0.as_fd()
    }
}

impl AsRawFd for FileDesc {
    //简化代码
    fn as_raw_fd(&self) -> RawFd {
        self.0.as_raw_fd()
    }
}

impl IntoRawFd for FileDesc {
    fn into_raw_fd(self) -> RawFd {
        //见OwnedFd::into_raw_fd
        self.0.into_raw_fd()
    }
}

impl FromRawFd for FileDesc {
    //见OwnedFd::from_raw_fd
    unsafe fn from_raw_fd(raw_fd: RawFd) -> Self {
        Self(FromRawFd::from_raw_fd(raw_fd))
    }
}

```
文件描述符实际上代表了操作系统的资源，是后继个模块分析的一个基础。
# std库进程管理代码分析
描述进程管理的需求以一个linux的shell命令比较合适：
例如： cat 序言.md | more
在shell程序执行这条命令时，做了以下的工作：
1. 创建1个管道
2. fork第一个子进程
3. 指定第一个子进程的标准输入是管道的读出端
4. 第一个子进程用execv执行more的可执行文件
5. fork第二个子进程
6. 打开"序言.md"文件
7. 将管段的写入端作为第二个子进程的标准输出，将"序言.md"作为进程的标准输入
8. 第二个子进程用execv执行cat可执行文件
9. wait两个子进程结束
上例如果用RUST来实现，代码简略如下：
```rust
    let child_more = Command::new("more")
                           .stdin(Stdio::piped())
                           .spawn()
                           .expect("error more");
    let child_cat = Command::new("cat")
                          .arg("引言.md")
                          .stdout(child_more.stdin.unwrap())
                          .spawn().expect("cat error");              
```
可以看到，RUST大幅减轻了代码实现。  
以上基本简要包含了RUST进程管理的任务。      
在创建一个进程及真正执行进程的二进制文件之间，父进程可以对子进程完成一些进程参数控制，通常就是对标准输入/标准输出/标准错误做重定向设置。匿名管道是专门为这个场景准备的进程间通信机制，用于将两个进程的标准输出及标准输入连接。   
其他专用于父子进程通信的手段还有:父进程可以等待子进程结束，向子进程发送信号，获取子进程退出的返回值等。  

进程间通信还有很多其他手段，放在后继专门章节做分析。本章只分析匿名管道。   

操作系统无关的代码路径：library/src/std/src/process.rs   
                      library/src/std/src/syscommon/process.rs
操作系统相关的代码路径：library/src/std/src/sys/unix/process/*  
                      library/src/std/src/os/linux/process.rs   
                      library/src/std/src/sys/unix/pipe.rs

## 匿名管道
匿名管道被设计用来在父子进程或者同一个父进程创建的子进程之间进行通信。一般只用于标准输入及输出的重定向。
```rust
//匿名管道的资源用文件描述符表示
pub struct AnonPipe(FileDesc);

pub fn anon_pipe() -> io::Result<(AnonPipe, AnonPipe)> {
    //匿名管道会创建两个文件描述符
    let mut fds = [0; 2];

    unsafe {
        //pipe2系统调用，fds的类型RUST做了推断，O_CLOEXEC表示后继exec调用的时候会自动close
        cvt(libc::pipe2(fds.as_mut_ptr(), libc::O_CLOEXEC))?;
        //返回两个匿名管道
        Ok((AnonPipe(FileDesc::from_raw_fd(fds[0])), AnonPipe(FileDesc::from_raw_fd(fds[1]))))
    }
}
//FileDesc的adapter
impl AnonPipe {
    pub fn read(&self, buf: &mut [u8]) -> io::Result<usize> {
        self.0.read(buf)
    }

    pub fn read_vectored(&self, bufs: &mut [IoSliceMut<'_>]) -> io::Result<usize> {
        self.0.read_vectored(bufs)
    }

    pub fn is_read_vectored(&self) -> bool {
        self.0.is_read_vectored()
    }

    pub fn write(&self, buf: &[u8]) -> io::Result<usize> {
        self.0.write(buf)
    }

    pub fn write_vectored(&self, bufs: &[IoSlice<'_>]) -> io::Result<usize> {
        self.0.write_vectored(bufs)
    }

    pub fn is_write_vectored(&self) -> bool {
        self.0.is_write_vectored()
    }
}

impl IntoInner<FileDesc> for AnonPipe {
    fn into_inner(self) -> FileDesc {
        self.0
    }
}

pub fn read2(p1: AnonPipe, v1: &mut Vec<u8>, p2: AnonPipe, v2: &mut Vec<u8>) -> io::Result<()> {
    // 对两个同时读出，并将输出合并
    let p1 = p1.into_inner();
    let p2 = p2.into_inner();
    //需要设置成非阻塞，用异步读的方式
    p1.set_nonblocking(true)?;
    p2.set_nonblocking(true)?;

    //准备libc的poll调用参数，具体的细节请参考libc的手册
    let mut fds: [libc::pollfd; 2] = unsafe { mem::zeroed() };
    fds[0].fd = p1.as_raw_fd();
    fds[0].events = libc::POLLIN;
    fds[1].fd = p2.as_raw_fd();
    fds[1].events = libc::POLLIN;
    loop {
        // poll调用，当任意一个fd有内容可读事件，则返回，否则阻塞
        cvt_r(|| unsafe { libc::poll(fds.as_mut_ptr(), 2, -1) })?;

        //fds[0]读到v1
        if fds[0].revents != 0 && read(&p1, v1)? {
            //出错，需要读出p2，此时重新设置成阻塞
            p2.set_nonblocking(false)?;
            //drop是因为返回为单元类型而做
            return p2.read_to_end(v2).map(drop);
        }
        if fds[1].revents != 0 && read(&p2, v2)? {
            //出错，需要读出p2,此时需要重新设置阻塞
            p1.set_nonblocking(false)?;
            return p1.read_to_end(v1).map(drop);
        }
    }

    fn read(fd: &FileDesc, dst: &mut Vec<u8>) -> Result<bool, io::Error> {
        //一直读完
        match fd.read_to_end(dst) {
            //读到内容
            Ok(_) => Ok(true),
            Err(e) => {
                if e.raw_os_error() == Some(libc::EWOULDBLOCK)
                    || e.raw_os_error() == Some(libc::EAGAIN)
                {
                    //没有读到内容，但是指示需要阻塞
                    Ok(false)
                } else {
                    //读出错
                    Err(e)
                }
            }
        }
    }
    //必然会在前面返回。这里会对p1及p2做所有权释放，会close掉创建的管道文件
}

impl AsRawFd for AnonPipe {
    fn as_raw_fd(&self) -> RawFd {
        self.0.as_raw_fd()
    }
}

impl AsFd for AnonPipe {
    fn as_fd(&self) -> BorrowedFd<'_> {
        self.0.as_fd()
    }
}

impl IntoRawFd for AnonPipe {
    fn into_raw_fd(self) -> RawFd {
        self.0.into_raw_fd()
    }
}

impl FromRawFd for AnonPipe {
    unsafe fn from_raw_fd(raw_fd: RawFd) -> Self {
        Self(FromRawFd::from_raw_fd(raw_fd))
    }
}
```
由以上的代码可以发现，涉及到操作系统的系统调用层面编程时，RUST实际需要大量C编程知识。讲述C编程超出了本书的本意，因此后面将不再更多的解释C调用相关的内容。

## 标准输入输出重定向类型及实现
```rust
//如果进程需要做标准输入/输出/错误重定向，应该申请此类型为参数并设置
pub struct StdioPipes {
    //None表示不重定向，Some()表示重定向到匿名管道
    pub stdin: Option<AnonPipe>,
    pub stdout: Option<AnonPipe>,
    pub stderr: Option<AnonPipe>,
}

//对子进程的标准输入输出做设置
pub struct ChildPipes {
    pub stdin: ChildStdio,
    pub stdout: ChildStdio,
    pub stderr: ChildStdio,
}
//对子进程标准输入/输出/错误进行指定,此时fd已经准备好已经创建
pub enum ChildStdio {
    //继承父进程的标准输入输出错误
    Inherit,
    //直接是unix的fd
    Explicit(c_int),
    //RUST包装后的句柄 
    Owned(FileDesc),
}
//此类型用于在子进程准备阶段对子进程的标准输入/输出/错误做配置用
//RUST的进程管理会根据Stdio的类型完成子进程的准备,仅仅是配置
pub enum Stdio {
    //继承父进程
    Inherit,
    //设置为Null
    Null,
    //创建匿名管道作为标准输入/输出/错误
    MakePipe,
    //标准输入/输出/错误使用给出的文件描述符
    Fd(FileDesc),
}

//针对enum实现方法的例子
impl Stdio {
    //此方法在创建子进程时根据self的设置完成对子进程的标准输入/输出/错误的准备。
    //并返回准备好的标准输入/输出/错误
    pub fn to_child_stdio(&self, readable: bool) -> io::Result<(ChildStdio, Option<AnonPipe>)> {
        match *self {
            //指定为继承父进程，子进程为继承父进程，父进程没有与之相关的管道
            Stdio::Inherit => Ok((ChildStdio::Inherit, None)),

            //配置为使用指定的文件描述符
            Stdio::Fd(ref fd) => {
                if fd.as_raw_fd() >= 0 && fd.as_raw_fd() <= libc::STDERR_FILENO {
                    //如果指定的文件描述符是标准输入/输出/错误，则复制后返回Owned,
                    //父进程没有管道与子进程连接
                    Ok((ChildStdio::Owned(fd.duplicate()?), None))
                } else {
                    //子进程使用指定的文件描述符，此时因为不能获取FileDesc的所有权，所以只能使用RawFd，父进程没有管道与之连接
                    Ok((ChildStdio::Explicit(fd.as_raw_fd()), None))
                }
            }

            //配置为创建管道连接父子进程
            Stdio::MakePipe => {
                //创建管道
                let (reader, writer) = pipe::anon_pipe()?;
                //根据读写标志设置自身管道的文件描述符, readable指子进程是读方
                let (ours, theirs) = if readable { (writer, reader) } else { (reader, writer) };
                //返回创建的管道描述符
                Ok((ChildStdio::Owned(theirs.into_inner()), Some(ours)))
            }

            //指定为dev/null
            Stdio::Null => {
                let mut opts = OpenOptions::new();
                opts.read(readable);
                opts.write(!readable);
                let path = unsafe { CStr::from_ptr(DEV_NULL.as_ptr() as *const _) };
                //需要先打开/dev/null
                let fd = File::open_c(&path, &opts)?;
                //输出为/dev/null
                Ok((ChildStdio::Owned(fd.into_inner()), None))
            }
        }
    }
}

impl From<AnonPipe> for Stdio {
    fn from(pipe: AnonPipe) -> Stdio {
        Stdio::Fd(pipe.into_inner())
    }
}

impl From<File> for Stdio {
    fn from(file: File) -> Stdio {
        Stdio::Fd(file.into_inner())
    }
}
//直接获取RawFd用于操作系统调用
impl ChildStdio {
    pub fn fd(&self) -> Option<c_int> {
        match *self {
            ChildStdio::Inherit => None,
            ChildStdio::Explicit(fd) => Some(fd),
            ChildStdio::Owned(ref fd) => Some(fd.as_raw_fd()),
        }
    }
}

```

## 进程管理
使用RUST的创建进程的代码举例如下：
```rust
use std::process::Command;

Command::new("ls")
        .arg("-l")
        .arg("-a")
        .stdout(Stdio::piped())
        .spawn()
        .expect("ls command failed to start");
```
可以看到，RUST把C语言库中分散的进程准备及执行相关的内容整体组织进了Command结构的实现中，并利用函数式编程的链式调用使其语法易于理解，从后面的实现中也可以看出Command对程序员是一个巨大的福利。
Command的具体使用方式请参考官方标准库文档获得指导。

RUST中操作系统无关与操作系统相关的接口类型结构及实现：  
RUST在进程管理中，没有使用trait的表达方式，而是直接使用了各操作系统统一实现相同名称的类型，再对此类型实现相同的方法。  
   
```rust
//所有操作系统都有pub struct Process的类型, 类型结构定义各操作系统可以不同，但需要实现同样的方法。
//unix的Process, unix的通常定义
pub struct Process {
    //unix的进程pid
    pid: pid_t,
    //退出的状态
    status: Option<ExitStatus>,
    // Linux上，每个process与一个fd相关联。
    #[cfg(target_os = "linux")]
    pidfd: Option<PidFd>,
}
//unix的Process的方法实现
impl Process {
    //linux的创建方法，应该在fork函数调用以后才能调用此方法
    #[cfg(target_os = "linux")]
    unsafe fn new(pid: pid_t, pidfd: pid_t) -> Self {
        use crate::os::unix::io::FromRawFd;
        use crate::sys_common::FromInner;
        // Safety: If `pidfd` is nonnegative, we assume it's valid and otherwise unowned.
        let pidfd = (pidfd >= 0).then(|| PidFd::from_inner(sys::fd::FileDesc::from_raw_fd(pidfd)));
        Process { pid, status: None, pidfd }
    }

    //父进程调用此函数杀掉子进程
    pub fn kill(&mut self) -> io::Result<()> {
        // 假如子进程已经是退出状态，那子进程的pid可能已经分配给其他进程使用，此时需要返回错误。
        if self.status.is_some() {
            Err(io::const_io_error!(
                ErrorKind::InvalidInput,
                "invalid argument: can't kill an exited process",
            ))
        } else {
            //libc库的kill调用
            cvt(unsafe { libc::kill(self.pid, libc::SIGKILL) }).map(drop)
        }
    }

    //等待子进程结束，会阻塞当前线程
    pub fn wait(&mut self) -> io::Result<ExitStatus> {
        use crate::sys::cvt_r;
        //如果已经退出，返回
        if let Some(status) = self.status {
            return Ok(status);
        }

        let mut status = 0 as c_int;
        //调用libc的waitpid等待返回
        cvt_r(|| unsafe { libc::waitpid(self.pid, &mut status, 0) })?;
        //子进程已经退出，设置合适的状态
        self.status = Some(ExitStatus::new(status));
        Ok(ExitStatus::new(status))
    }

    //非阻塞的等待子进程退出
    pub fn try_wait(&mut self) -> io::Result<Option<ExitStatus>> {
        if let Some(status) = self.status {
            return Ok(Some(status));
        }
        let mut status = 0 as c_int;
        //用libc::WNOHANG表示非阻塞
        let pid = cvt(unsafe { libc::waitpid(self.pid, &mut status, libc::WNOHANG) })?;
        if pid == 0 {
            //没有退出
            Ok(None)
        } else {
            //已经退出
            self.status = Some(ExitStatus::new(status));
            Ok(Some(ExitStatus::new(status)))
        }
    }
}
```
因为wasi进程管理与unix类似，因此下面再给出windows的代码做一下对比：
```rust
pub struct Process {
    //windows句柄
    handle: Handle,
}

impl Process {
    pub fn kill(&mut self) -> io::Result<()> {
        //使用windows的系统调用
        cvt(unsafe { c::TerminateProcess(self.handle.as_raw_handle(), 1) })?;
        Ok(())
    }

    pub fn wait(&mut self) -> io::Result<ExitStatus> {
        unsafe {
            //windows使用如下调用来等待进程退出
            let res = c::WaitForSingleObject(self.handle.as_raw_handle(), c::INFINITE);
            if res != c::WAIT_OBJECT_0 {
                return Err(Error::last_os_error());
            }
            let mut status = 0;
            //额外调用来获取退出码
            cvt(c::GetExitCodeProcess(self.handle.as_raw_handle(), &mut status))?;
            Ok(ExitStatus(status))
        }
    }

    pub fn try_wait(&mut self) -> io::Result<Option<ExitStatus>> {
        unsafe {
            //不等待
            match c::WaitForSingleObject(self.handle.as_raw_handle(), 0) {
                c::WAIT_OBJECT_0 => {}
                c::WAIT_TIMEOUT => {
                    return Ok(None);
                }
                _ => return Err(io::Error::last_os_error()),
            }
            //获取退出码
            let mut status = 0;
            cvt(c::GetExitCodeProcess(self.handle.as_raw_handle(), &mut status))?;
            Ok(Some(ExitStatus(status)))
        }
    }
}
```
由上可见，不同的操作系统实现了相同的类型名及类型方法。后继其他类型将只关注unix的实现。
其他进程管理相关的类型结构及方法实现：  
```rust
//Command完成进程准备的所有参数，进程启动等
//以下为unix的定义
pub struct Command {
    //进程的可执行文件名，由这个定义看，RUST对中文作为可执行文件名的支持存疑, 这个需要用代码做验证
    program: CString,
    //进程的命令行参数,同上，是否支持中文？  
    args: Vec<CString>,
    /// 传递给`execvp`的参数, 第一个参数应该是`program`, 然后是
    /// `args`, 最后应该是`null`. 修改时需要注意这三个参数的联动性
    argv: Argv,
    env: CommandEnv,

    //当前目录
    cwd: Option<CString>,
    //unix的uid
    uid: Option<uid_t>,
    //unix的gid
    gid: Option<gid_t>,
    // 对CString参数的输入是否存在0做标识
    saw_nul: bool,
    closures: Vec<Box<dyn FnMut() -> io::Result<()> + Send + Sync>>,
    groups: Option<Box<[gid_t]>>,
    //标准输入配置
    stdin: Option<Stdio>,
    //标准输出配置
    stdout: Option<Stdio>,
    //标准错误配置
    stderr: Option<Stdio>,
    #[cfg(target_os = "linux")]
    create_pidfd: bool,
    pgroup: Option<pid_t>,
}

impl Command {
    pub fn new(program: &OsStr) -> Command {
        let mut saw_nul = false;
        //OsStr转换为CStr,并返回OsStr是否存在尾值0
        let program = os2c(program, &mut saw_nul);
        //用program创建默认的Command结构体
        Command {
            //argv尾部必须有一个null指针
            argv: Argv(vec![program.as_ptr(), ptr::null()]),
            args: vec![program.clone()],
            //同名参数赋值
            program,
            env: Default::default(),
            cwd: None,
            uid: None,
            gid: None,
            saw_nul,
            closures: Vec::new(),
            groups: None,
            stdin: None,
            stdout: None,
            stderr: None,
            create_pidfd: false,
            pgroup: None,
        }
    }

    //这个方法设置program进入两个arg参数
    pub fn set_arg_0(&mut self, arg: &OsStr) {
        // Set a new arg0
        let arg = os2c(arg, &mut self.saw_nul);
        debug_assert!(self.argv.0.len() > 1);
        self.argv.0[0] = arg.as_ptr();
        self.args[0] = arg;
    }

    //此方法增加增加一个命令行的参数
    pub fn arg(&mut self, arg: &OsStr) {
        let arg = os2c(arg, &mut self.saw_nul);
        //增加参数到argv
        self.argv.0[self.args.len()] = arg.as_ptr();
        self.argv.0.push(ptr::null());

        //增加参数到args
        self.args.push(arg);
    }

    //根据标准输入/输出/错误的配置完成动作
    pub fn setup_io(
        &self,
        default: Stdio,
        needs_stdin: bool,
    ) -> io::Result<(StdioPipes, ChildPipes)> {
        let null = Stdio::Null;
        let default_stdin = if needs_stdin { &default } else { &null };
        //没有配置的话就使用默认配置
        let stdin = self.stdin.as_ref().unwrap_or(default_stdin);
        let stdout = self.stdout.as_ref().unwrap_or(&default);
        let stderr = self.stderr.as_ref().unwrap_or(&default);
        //创建标准输入的子进程文件
        let (their_stdin, our_stdin) = stdin.to_child_stdio(true)?;
        //创建标准输出的子进程文件
        let (their_stdout, our_stdout) = stdout.to_child_stdio(false)?;
        //创建标准错误的子进程文件 
        let (their_stderr, our_stderr) = stderr.to_child_stdio(false)?;
        //完成本进程的设置
        let ours = StdioPipes { stdin: our_stdin, stdout: our_stdout, stderr: our_stderr };
        //完成子进程的设置
        let theirs = ChildPipes { stdin: their_stdin, stdout: their_stdout, stderr: their_stderr };
        Ok((ours, theirs))
    }
```
以下为创建进程的方法实现，从以下代码可见，在面向操作系统时，RUST并不比C更轻松，也不比C会有少犯错误的空间
```rust
    //创建进程的具体执行方法
    pub fn spawn(
        &mut self,
        default: Stdio,
        needs_stdin: bool,
    ) -> io::Result<(Process, StdioPipes)> {
        const CLOEXEC_MSG_FOOTER: [u8; 4] = *b"NOEX";

        //完成环境变量创建
        let envp = self.capture_env();

        //命令行出错处理
        if self.saw_nul() {
            return Err(io::const_io_error!(
                ErrorKind::InvalidInput,
                "nul byte found in provided data",
            ));
        }

        //完成标准输入/输出/错误文件的创建与准备
        let (ours, theirs) = self.setup_io(default, needs_stdin)?;

        //利用posix的api来创建进程
        if let Some(ret) = self.posix_spawn(&theirs, envp.as_ref())? {
            return Ok((ret, ours));
        }

        //创建一个匿名管道,这个匿名管道用来捕捉exec的错误
        let (input, output) = sys::pipe::anon_pipe()?;

        //此时要对环境参数加锁
        let env_lock = sys::os::env_read_lock();
        //fork新的进程, 新进程复制所有的老进程的参数和栈
        let (pid, pidfd) = unsafe { self.do_fork()? };

        if pid == 0 {
            //新创建的子进程, 总是abort退出
            crate::panic::always_abort();
            //不用理会env_lock，父进程会处理,这个细节RUST和C是一样容易出错的
            mem::forget(env_lock);
            //子进程不使用input
            drop(input);
            //执行二进制可执行文件,执行成功不会返回, exec执行成功后,因为output设置了FD_CLOEXEC，
            //所以output会被关闭
            let Err(err) = unsafe { self.do_exec(theirs, envp.as_ref()) };
            //exec执行失败,做错误处理
            let errno = err.raw_os_error().unwrap_or(libc::EINVAL) as u32;
            let errno = errno.to_be_bytes();
            let bytes = [
                errno[0],
                errno[1],
                errno[2],
                errno[3],
                CLOEXEC_MSG_FOOTER[0],
                CLOEXEC_MSG_FOOTER[1],
                CLOEXEC_MSG_FOOTER[2],
                CLOEXEC_MSG_FOOTER[3],
            ];
            // 将exec的错误写入管道，使得父进程能够获得
            // 这里是一个细致的考虑
            rtassert!(output.write(&bytes).is_ok());
            unsafe { libc::_exit(1) }
        }

        //对env_lock进行处理
        drop(env_lock);
        //不需要output，要显式drop
        drop(output);

        //根据fork的返回创建RUST的子进程结构
        let mut p = unsafe { Process::new(pid, pidfd) };
        let mut bytes = [0; 8];

        loop {
            //如果子进程exec成功，则管道对端会关闭，此处read会返回
            match input.read(&mut bytes) {
                //子进程成功执行
                Ok(0) => return Ok((p, ours)),
                //子进程exec失败，返回错误
                Ok(8) => {
                    let (errno, footer) = bytes.split_at(4);
                    assert_eq!(
                        CLOEXEC_MSG_FOOTER, footer,
                        "Validation on the CLOEXEC pipe failed: {:?}",
                        bytes
                    );
                    let errno = i32::from_be_bytes(errno.try_into().unwrap());
                    //子进程失败时做个等待
                    assert!(p.wait().is_ok(), "wait() should either return Ok or panic");
                    return Err(Error::from_raw_os_error(errno));
                }
                //以下为其他失败情况
                Err(ref e) if e.kind() == ErrorKind::Interrupted => {}
                Err(e) => {
                    assert!(p.wait().is_ok(), "wait() should either return Ok or panic");
                    panic!("the CLOEXEC pipe failed: {e:?}")
                }
                Ok(..) => {
                    // pipe I/O up to PIPE_BUF bytes should be atomic
                    assert!(p.wait().is_ok(), "wait() should either return Ok or panic");
                    panic!("short read on the CLOEXEC pipe")
                }
            }
        }
    }

    //fork系统调用的RUST版本,不解释，实际上基本等同于C代码的RUST翻译
    unsafe fn do_fork(&mut self) -> Result<(pid_t, pid_t), io::Error> {
        use crate::sync::atomic::{AtomicBool, Ordering};

        static HAS_CLONE3: AtomicBool = AtomicBool::new(true);
        const CLONE_PIDFD: u64 = 0x00001000;

        #[repr(C)]
        struct clone_args {
            flags: u64,
            pidfd: u64,
            child_tid: u64,
            parent_tid: u64,
            exit_signal: u64,
            stack: u64,
            stack_size: u64,
            tls: u64,
            set_tid: u64,
            set_tid_size: u64,
            cgroup: u64,
        }

        raw_syscall! {
            fn clone3(cl_args: *mut clone_args, len: libc::size_t) -> libc::c_long
        }

        // Bypassing libc for `clone3` can make further libc calls unsafe,
        // so we use it sparingly for now. See #89522 for details.
        // Some tools (e.g. sandboxing tools) may also expect `fork`
        // rather than `clone3`.
        let want_clone3_pidfd = self.get_create_pidfd();

        // If we fail to create a pidfd for any reason, this will
        // stay as -1, which indicates an error.
        let mut pidfd: pid_t = -1;

        // Attempt to use the `clone3` syscall, which supports more arguments
        // (in particular, the ability to create a pidfd). If this fails,
        // we will fall through this block to a call to `fork()`
        if want_clone3_pidfd && HAS_CLONE3.load(Ordering::Relaxed) {
            let mut args = clone_args {
                flags: CLONE_PIDFD,
                pidfd: &mut pidfd as *mut pid_t as u64,
                child_tid: 0,
                parent_tid: 0,
                exit_signal: libc::SIGCHLD as u64,
                stack: 0,
                stack_size: 0,
                tls: 0,
                set_tid: 0,
                set_tid_size: 0,
                cgroup: 0,
            };

            let args_ptr = &mut args as *mut clone_args;
            let args_size = crate::mem::size_of::<clone_args>();

            let res = cvt(clone3(args_ptr, args_size));
            match res {
                Ok(n) => return Ok((n as pid_t, pidfd)),
                Err(e) => match e.raw_os_error() {
                    // Multiple threads can race to execute this store,
                    // but that's fine - that just means that multiple threads
                    // will have tried and failed to execute the same syscall,
                    // with no other side effects.
                    Some(libc::ENOSYS) => HAS_CLONE3.store(false, Ordering::Relaxed),
                    // Fallback to fork if `EPERM` is returned. (e.g. blocked by seccomp)
                    Some(libc::EPERM) => {}
                    _ => return Err(e),
                },
            }
        }

        // Generally, we just call `fork`. If we get here after wanting `clone3`,
        // then the syscall does not exist or we do not have permission to call it.
        cvt(libc::fork()).map(|res| (res, pidfd))
    }

    // 执行二进制文件,值得注意的是，调用execvp, 函数将不再返回，从而导致父进程已经申请的内存及
    // 文件描述符资源会错误的被处理，因此，这个函数整体的安全规则仍然与C语言调用execvp的规则一致，
    // RUST的安全规则在此处起不到更大的帮助
    unsafe fn do_exec(
        &mut self,
        stdio: ChildPipes,
        maybe_envp: Option<&CStringArray>,
    ) -> Result<!, io::Error> {
        use crate::sys::{self, cvt_r};

        //用dup2系统调用设置本进程的标准输入/输出/错误
        if let Some(fd) = stdio.stdin.fd() {
            cvt_r(|| libc::dup2(fd, libc::STDIN_FILENO))?;
        }
        if let Some(fd) = stdio.stdout.fd() {
            cvt_r(|| libc::dup2(fd, libc::STDOUT_FILENO))?;
        }
        if let Some(fd) = stdio.stderr.fd() {
            cvt_r(|| libc::dup2(fd, libc::STDERR_FILENO))?;
        }

        //设置进程其他参数，具体请参考操作系统的编程书籍
        {
            if let Some(_g) = self.get_groups() {
                cvt(libc::setgroups(_g.len().try_into().unwrap(), _g.as_ptr()))?;
            }
            if let Some(u) = self.get_gid() {
                cvt(libc::setgid(u as gid_t))?;
            }
            if let Some(u) = self.get_uid() {
                if libc::getuid() == 0 && self.get_groups().is_none() {
                    cvt(libc::setgroups(0, ptr::null()))?;
                }
                cvt(libc::setuid(u as uid_t))?;
            }
        }
        if let Some(ref cwd) = *self.get_cwd() {
            cvt(libc::chdir(cwd.as_ptr()))?;
        }

        if let Some(pgroup) = self.get_pgroup() {
            cvt(libc::setpgid(0, pgroup))?;
        }

        {
            //对进程接收的信号进行设置
            use crate::mem::MaybeUninit;
            //注意这里用MaybeUninit申请了一个栈变量，后继需要给C函数使用
            let mut set = MaybeUninit::<libc::sigset_t>::uninit();
            cvt(sigemptyset(set.as_mut_ptr()))?;
            cvt(libc::pthread_sigmask(libc::SIG_SETMASK, set.as_ptr(), ptr::null_mut()))?;

            {
                let ret = sys::signal(libc::SIGPIPE, libc::SIG_DFL);
                if ret == libc::SIG_ERR {
                    return Err(io::Error::last_os_error());
                }
            }
        }

        //用于在执行二进制之前做些其他的操作,如统计信息和log日志之类
        //考虑很完善
        for callback in self.get_closures().iter_mut() {
            callback()?;
        }

        //以下用于在exec出错的时候恢复环境变量,这个地方是体现RUST优越性的地方
        //请细心体会以下下面利用生命周期的错误处理方式
        let mut _reset = None;
        if let Some(envp) = maybe_envp {
            struct Reset(*const *const libc::c_char);

            impl Drop for Reset {
                fn drop(&mut self) {
                    unsafe {
                        *sys::os::environ() = self.0;
                    }
                }
            }

            _reset = Some(Reset(*sys::os::environ()));
            *sys::os::environ() = envp.as_ptr();
        }

        //调用execvp执行二进制文件
        libc::execvp(self.get_program_cstr().as_ptr(), self.get_argv().as_ptr());
        Err(io::Error::last_os_error())
    }
}
```

RUST标准库中操作系统无关的进程管理结构及实现：
```rust
use crate::sys::process as imp;
// Child用来保存创建的子进程的信息
pub struct Child {
    //系统分配的子进程的标识句柄
    pub(crate) handle: imp::Process,

    //子进程标准输入句柄,这里实际上是保存向子进程写输入的文件
    pub stdin: Option<ChildStdin>,

    //子进程标准输出句柄,这里实际上是保存从子进程读输出的文件
    pub stdout: Option<ChildStdout>,

    //子进程标准错误句柄,实际上是从子进程读错误的文件
    pub stderr: Option<ChildStderr>,
}

pub struct ChildStdin {
    //见操作系统相关部分代码
    inner: AnonPipe,
}
pub struct ChildStdout {
    inner: AnonPipe,
}
pub struct ChildStderr {
    inner: AnonPipe,
}

//Command封装了所有进程管理的API
pub struct Command {
    inner: imp::Command,
}
impl Command {
    //输入进程的二进制可执行文件的路径及名称，
    pub fn new<S: AsRef<OsStr>>(program: S) -> Command {
        Command { inner: imp::Command::new(program.as_ref()) }
    }

    //输入一个进程命令的参数
    pub fn arg<S: AsRef<OsStr>>(&mut self, arg: S) -> &mut Command {
        self.inner.arg(arg.as_ref());
        self
    }

    //输入若干进程命令的参数
    pub fn args<I, S>(&mut self, args: I) -> &mut Command
    where
        I: IntoIterator<Item = S>,
        S: AsRef<OsStr>,
    {
        for arg in args {
            self.arg(arg.as_ref());
        }
        self
    }

    //针对进程插入或设置一个环境参数
    pub fn env<K, V>(&mut self, key: K, val: V) -> &mut Command
    where
        K: AsRef<OsStr>,
        V: AsRef<OsStr>,
    {
        self.inner.env_mut().set(key.as_ref(), val.as_ref());
        self
    }

    //插入或设置若干个环境参数
    pub fn envs<I, K, V>(&mut self, vars: I) -> &mut Command
    where
        I: IntoIterator<Item = (K, V)>,
        K: AsRef<OsStr>,
        V: AsRef<OsStr>,
    {
        for (ref key, ref val) in vars {
            self.inner.env_mut().set(key.as_ref(), val.as_ref());
        }
        self
    }

    //清除一个环境参数
    pub fn env_remove<K: AsRef<OsStr>>(&mut self, key: K) -> &mut Command {
        self.inner.env_mut().remove(key.as_ref());
        self
    }

    //清除所有环境参数
    pub fn env_clear(&mut self) -> &mut Command {
        self.inner.env_mut().clear();
        self
    }

    //获取当前目录
    pub fn current_dir<P: AsRef<Path>>(&mut self, dir: P) -> &mut Command {
        self.inner.cwd(dir.as_ref().as_ref());
        self
    }

    //配置标准输入
    pub fn stdin<T: Into<Stdio>>(&mut self, cfg: T) -> &mut Command {
        self.inner.stdin(cfg.into().0);
        self
    }

    //配置标准输出
    pub fn stdout<T: Into<Stdio>>(&mut self, cfg: T) -> &mut Command {
        self.inner.stdout(cfg.into().0);
        self
    }
    
    //配置标准错误
    pub fn stderr<T: Into<Stdio>>(&mut self, cfg: T) -> &mut Command {
        self.inner.stderr(cfg.into().0);
        self
    }

    //正式按照Command的参数创建进程
    pub fn spawn(&mut self) -> io::Result<Child> {
        self.inner.spawn(imp::Stdio::Inherit, true).map(Child::from_inner)
    }

    //创建子进程，并等待子进程结束,返回Output结构变量
    pub fn output(&mut self) -> io::Result<Output> {
        self.inner
            .spawn(imp::Stdio::MakePipe, false)
            .map(Child::from_inner)
            .and_then(|p| p.wait_with_output())
    }

    //创建子进程，等待进程结束，返回进程退出码
    pub fn status(&mut self) -> io::Result<ExitStatus> {
        self.inner
            .spawn(imp::Stdio::Inherit, true)
            .map(Child::from_inner)
            .and_then(|mut p| p.wait())
    }

    ...    
}

//进程命令的参数集合
pub struct CommandArgs<'a> {
    inner: imp::CommandArgs<'a>,
}

//保存子进程结束后的输出
pub struct Output {
    //子进程退出的返回码
    pub status: ExitStatus,
    //子进程的标准输出输出的内容
    pub stdout: Vec<u8>,
    //子进程标准错误输出的内容
    pub stderr: Vec<u8>,
}

//标准输入输出的类型结构
pub struct Stdio(imp::Stdio);

impl Stdio {
    //创建一个管道的标准输入输出
    pub fn piped() -> Stdio {
        Stdio(imp::Stdio::MakePipe)
    }
    //继承父进程
    pub fn inherit() -> Stdio {
        Stdio(imp::Stdio::Inherit)
    }

    //使用/dev/null作为进程标准输入输出
    pub fn null() -> Stdio {
        Stdio(imp::Stdio::Null)
    }
}

//对子进程的API
impl Child {
    //杀掉子进程
    pub fn kill(&mut self) -> io::Result<()> {
        self.handle.kill()
    }

    //获取子进程的id值
    pub fn id(&self) -> u32 {
        self.handle.id()
    }

    //等待子进程结束，需要提前释放子进程的标准输入，否则子进程
    //可能不会退出
    pub fn wait(&mut self) -> io::Result<ExitStatus> {
        drop(self.stdin.take());
        self.handle.wait().map(ExitStatus)
    }


    //尝试等子进程退出
    pub fn try_wait(&mut self) -> io::Result<Option<ExitStatus>> {
        Ok(self.handle.try_wait()?.map(ExitStatus))
    }

    //等待子进程退出并获取所有输出
    pub fn wait_with_output(mut self) -> io::Result<Output> {
        //需要先关闭子进程的标准输入
        drop(self.stdin.take());

        //申请缓存
        let (mut stdout, mut stderr) = (Vec::new(), Vec::new());
        match (self.stdout.take(), self.stderr.take()) {
            (None, None) => {}

            (Some(mut out), None) => {
                //读子进程标准输出到缓存
                let res = out.read_to_end(&mut stdout);
                res.unwrap();
            }
            (None, Some(mut err)) => {
                //读标准错误到缓存
                let res = err.read_to_end(&mut stderr);
                res.unwrap();
            }
            (Some(out), Some(err)) => {
                //读两个，此函数应该专门为这个目的做的设计
                let res = read2(out.inner, &mut stdout, err.inner, &mut stderr);
                res.unwrap();
            }
        }

        //等待子进程结束
        let status = self.wait()?;
        //创建Output返回
        Ok(Output { status, stdout, stderr })
    }
}
//从进程退出并返回一个值，父进程会获得这个值
pub fn exit(code: i32) -> ! {
    crate::rt::cleanup();
    crate::sys::os::exit(code)
}

//异常退出，与exit相比较，不会处理资源释放操作
pub fn abort() -> ! {
    crate::sys::abort_internal();
}
```
不熟悉操作系统的进程操作，基本就没有办法理解进程管理，因此，任何系统语言最关键的仍然是对操作系统系统调用的理解，而操作系统调用又离不开C语言的理解。所以, RUST的标准库最适合学习的是C程序员。

# 并发编程相关类型结构代码分析
并发编程主要包括线程及线程间通信内容。  
因为并发编程是各种语言教材的重点部分，所以，对于基础概念，本节不多赘述。同时，为了代码理解的方便，本节将先从各种锁结构的代码分析开始。    
本节将先将与操作系统相关的各种锁的代码分析完毕，再在这个基础上分析RUST在这些锁的基础上实现的具有语言特点的各种临界区类型。   

## linux的各种锁机制的实现  
代码路径： library/std/src/sys/unix/locks/*.rs     
          library/std/src/sys/unix/futex.rs     
### 锁的基础设施FUTEX代码分析
FUTEX是一种高效率的互斥机制，其本质是将原先分配给内核的部分锁处理代码转移到用户态处理。   
所有的临界区锁保护实际上是基于一个原子变量的检测及赋值，当这个原子变量处于一个范围时，允许代码执行，否则，就进入等待队列。当处于临界区的代码执行完毕后，会恢复原子变量唤醒等待队列等待的线程或进程。   
传统上，对锁的原子变量的检测是放在内核态的，内核检测及修改原子变量并完成等待阻塞操作或唤醒操作。这显然是一种效率很低的做法，因为大部分时候临界区是不会发生冲突的。每次检测都执行一次内核态与用户态的切换没有必要。因此，诞生了将原子变量检测与赋值放在用户态完成的思路。      
这个思路需要完成的需求整体如下：
1. 用户态声明一个原子变量作为能否进入临界区的锁。如果该值被设置为某值，代表不能进入临界区，其他值代表可以进入临界区。
2. 访问临界区的线程或进程修改原子变量表示要进入临界区，然后探测原子变量，看是否能够进入，能够进入则执行临界区访问
3. 不能进入则调用futex的系统调用进入内核，内核会重新做原子变量判断并决定是否进入等待，这个过程是不可被其他线程打断的。
4. 位于临界区的线程或进程执行完毕临界区代码后，在用户态修改原子变量，并唤醒一个或所有等待的进程或线程
5. 可以满足跨进程
   
futex即是满足这个需求的设计。 显然，传统的锁机制如pthread_mutex可以用futex方案重新设计。所以futex实际上是作为传统锁机制的基础设施存在。但对其做了解对于系统级编程仍然是有必要的。   

unix家族futex的RUST的界面如下：
```rust
pub fn futex_wait(futex: &AtomicI32, expected: i32, timeout: Option<Duration>) -> bool {
    use super::time::Timespec;
    use crate::ptr::null;
    use crate::sync::atomic::Ordering::Relaxed;

    // 计算超时
    let timespec =
        timeout.and_then(|d| Some(Timespec::now(libc::CLOCK_MONOTONIC).checked_add_duration(&d)?));

    loop {
        // 仅当原子变量是期待的值的时候才进入等待
        if futex.load(Relaxed) != expected {
            return true;
        }

        // 以下说明，RUST仅仅支持线程间的同步, 具体的系统调用请参考man手册
        let r = unsafe {
            libc::syscall(
                libc::SYS_futex,
                futex as *const AtomicI32,
                //FUTEX_PRIVATE_FLAG说明仅支持本进程线程间的futex
                libc::FUTEX_WAIT_BITSET | libc::FUTEX_PRIVATE_FLAG,
                //传入的futex是这个值时就阻塞
                expected,
                timespec.as_ref().map_or(null(), |t| &t.t as *const libc::timespec),
                null::<u32>(), // This argument is unused for FUTEX_WAIT_BITSET.
                !0u32,         // A full bitmask, to make it behave like a regular FUTEX_WAIT.
            )
        };

        //错误处理
        match (r < 0).then(super::os::errno) {
            Some(libc::ETIMEDOUT) => return false,
            Some(libc::EINTR) => continue,
            _ => return true,
        }
    }
}

//唤醒一个等待futex的执行者
pub fn futex_wake(futex: &AtomicI32) -> bool {
    unsafe {
        libc::syscall(
            libc::SYS_futex,
            futex as *const AtomicI32,
            libc::FUTEX_WAKE | libc::FUTEX_PRIVATE_FLAG,
            1,
        ) > 0
    }
}

/// 唤醒所有等待futex的所有者.
pub fn futex_wake_all(futex: &AtomicI32) {
    unsafe {
        libc::syscall(
            libc::SYS_futex,
            futex as *const AtomicI32,
            libc::FUTEX_WAKE | libc::FUTEX_PRIVATE_FLAG,
            i32::MAX,
        );
    }
}
```
FUTEX是一个让人惊讶的特性，惊讶之处在于为什么如此之晚这一特性才被实现。
### Mutex源代码分析
Mutex作为传统的临界区保护机制，在linux上，RUST利用futex重新实现了Mutex库而没有使用pthread_mutex_t。

```rust
pub struct Mutex {
    /// 0: unlocked
    /// 1: locked, no other threads waiting
    /// 2: locked, and other threads waiting (contended)
    /// 用作锁判断的原始变量,值见上面英文注释
    futex: AtomicI32,
}

impl Mutex {
    pub const fn new() -> Self {
        //初始化为不加锁
        Self { futex: AtomicI32::new(0) }
    }

    pub unsafe fn init(&mut self) {}

    pub unsafe fn destroy(&self) {}

    //如果临界区操作非常快，可以用try_lock, 获取到锁后
    //即可快速完成操作，作为整体不进入内核的方案。
    //try_lock如果返回false，需要调用lock等待锁被打开
    pub unsafe fn try_lock(&self) -> bool {
        //利用原子操作试图加锁，如果futex不为0,会失败
        //如果成功，此时还没有其他线程进入临界区
        self.futex.compare_exchange(0, 1, Acquire, Relaxed).is_ok()
    }

    //如果临界区操作耗时长，则应该直接调用这个代码，不使用try_lock
    pub unsafe fn lock(&self) {
        if self.futex.compare_exchange(0, 1, Acquire, Relaxed).is_err() {
            //如果失败，说明已经由其他线程占有了锁，本线程需要等待
            self.lock_contended();
        }
    }

    //执行后会等待其他线程临界区操作结束
    fn lock_contended(&self) {
        // 这里处理另一个线程并不操作临界区，或者临界区操作非常快的情况
        let mut state = self.spin();

        // 自旋等待后，如果锁已经打开，获取锁 
        if state == 0 {
            match self.futex.compare_exchange(0, 1, Acquire, Relaxed) {
                Ok(_) => return, // Locked!
                Err(s) => state = s,
            }
        }

        //临界区操作时间长，需要调用系统内核进入等待状态
        loop {
            // 如果没有其他线程，更新futex为2，表示临界区操作很长
            // 有线程需要被唤醒
            if state != 2 && self.futex.swap(2, Acquire) == 0 {
                // 获取到了锁，可以操作临界区了
                return;
            }

            // 如果state是2，那需要等待正在处理临界区的线程释放锁.
            // 如果futex是2，则阻塞
            futex_wait(&self.futex, 2, None);

            // 退出后仍然需要判断是否
            state = self.spin();
        }
    }

    // 在临界区操作非常快的情况下，不进入操作系统内核的解决方案 
    fn spin(&self) -> i32 {
        let mut spin = 100;
        loop {
            // 读取原子变量
            let state = self.futex.load(Relaxed);

            // 如果锁已开，或者明确的进入临界区的指示,立刻退出循环
            if state != 1 || spin == 0 {
                return state;
            }

            //做个CPU自旋
            crate::hint::spin_loop();
            spin -= 1;
        }
    }

    //解锁，lock调用后及try_lock调用返回为真时，需要调用这个函数。
    pub unsafe fn unlock(&self) {
        if self.futex.swap(0, Release) == 2 {
            // 如果是2，说明有线程通过操作系统内核等待，需要wake
            self.wake();
        }
    }

    #[cold]
    fn wake(&self) {
        //通知操作系统内核，唤醒一个线程
        futex_wake(&self.futex);
    }
}
```

### Convar分析
RUST标准库的条件变量Condvar解决方案：  
条件变量本身是一种信号机制， 由A线程向B线程通知事件发生。  
条件变量需要与一个Mutex的变量配合，来完成包含条件变量操作的临界区方案。
```rust
pub struct Condvar {
    // 本身是一个原子变量，值的变化表示条件变化
    futex: AtomicI32,
}

impl Condvar {
    //初始化
    pub const fn new() -> Self {
        Self { futex: AtomicI32::new(0) }
    }

    pub unsafe fn init(&mut self) {}

    pub unsafe fn destroy(&self) {}


    //通知一个线程条件已经变化，调用此函数前，应该对关联
    //mutex加锁，调用后，应释放锁
    pub unsafe fn notify_one(&self) {
        //简单的改变值，表示条件已经变化
        self.futex.fetch_add(1, Relaxed);
        //用futex操作系统调用完成通知,仅唤醒一个线程
        futex_wake(&self.futex);
    }

    //调用前应对关联mutex加锁，调用后应释放锁
    pub unsafe fn notify_all(&self) {
        self.futex.fetch_add(1, Relaxed);
        //唤醒所有等待线程
        futex_wake_all(&self.futex);
    }

    pub unsafe fn wait(&self, mutex: &Mutex) {
        self.wait_optional_timeout(mutex, None);
    }

    pub unsafe fn wait_timeout(&self, mutex: &Mutex, timeout: Duration) -> bool {
        self.wait_optional_timeout(mutex, Some(timeout))
    }

    //等待条件变化,调用此函数前，应该将关联mutex上锁保护self.futex及其他临界区的操作,
    //调用后，应释放锁
    unsafe fn wait_optional_timeout(&self, mutex: &Mutex, timeout: Option<Duration>) -> bool {
        // 外部应该先将mutex上锁，防止其他线程改变self.futex，此处获取当前值 
        let futex_value = self.futex.load(Relaxed);

        // 释放锁。 
        mutex.unlock();

        // 如果条件不变，则进入等待 
        let r = futex_wait(&self.futex, futex_value, timeout);

        // 条件已经变化，加锁保护self.futex的值及其他的临界区的操作 
        mutex.lock();

        r
    }
}
```
### RWLock源代码分析
读写锁适用的场景如下：    
临界区允许多个读同时存在，但读写不能同时存在。临界区读的时候要设置锁处于读锁，此时允许读不允许写。如果有线程要写，需要等待。
临界区写的时候要设置写锁，此时和正常的锁是一致的，所有其他试图访问临界区的线程都需要等待。
```rust
pub struct RwLock {
    // 从0到30位用来作为锁的计数:
    //   0: Unlocked
    //   1..=0x3FFF_FFFE: 作为读线程的计数，并作为读锁
    //   0x3FFF_FFFF: 写锁
    // Bit 30: 有其他读线程在等待，此时应该是写锁.
    // Bit 31: 有其他写线程在等待，此时读锁及写锁都有可能.
    state: AtomicI32,
    // 利用这个值的变化做信号的通知，类似与CondVar的操作.
    writer_notify: AtomicI32,
}

const READ_LOCKED: i32 = 1;
const MASK: i32 = (1 << 30) - 1;
const WRITE_LOCKED: i32 = MASK;
const MAX_READERS: i32 = MASK - 1;
const READERS_WAITING: i32 = 1 << 30;
const WRITERS_WAITING: i32 = 1 << 31;

fn is_unlocked(state: i32) -> bool {
    state & MASK == 0
}

fn is_write_locked(state: i32) -> bool {
    state & MASK == WRITE_LOCKED
}

fn has_readers_waiting(state: i32) -> bool {
    state & READERS_WAITING != 0
}

fn has_writers_waiting(state: i32) -> bool {
    state & WRITERS_WAITING != 0
}

//这个函数用来判断是否可以进入临界区读
fn is_read_lockable(state: i32) -> bool {
    // 只有在读线程的数量小于最大值，且没有其他线程在等着读或等着写，此时可以更新读锁
    // 否则应该进入读等待队列，
    // 只要有线程等着写，要后继的读线程就需要阻塞在读锁中
    state & MASK < MAX_READERS && !has_readers_waiting(state) && !has_writers_waiting(state)
}

fn has_reached_max_readers(state: i32) -> bool {
    state & MASK == MAX_READERS
}

impl RwLock {
    pub const fn new() -> Self {
        Self { state: AtomicI32::new(0), writer_notify: AtomicI32::new(0) }
    }

    pub unsafe fn destroy(&self) {}

    //试图读，一般如果需要做读锁，应直接调用read，此函数用作不希望阻塞的情况
    pub unsafe fn try_read(&self) -> bool {
        // 如果判断可读，则对读锁加1,然后进入临界区做读操作，如果失败，可以调用read等待读
        self.state
            .fetch_update(Acquire, Relaxed, |s| is_read_lockable(s).then(|| s + READ_LOCKED))
            .is_ok()
    }

    //获取读锁或阻塞等待到能获取
    pub unsafe fn read(&self) {
        let state = self.state.load(Relaxed);
        if !is_read_lockable(state)
            //此时更新读锁失败，证明锁已经被改变
            || self
                .state
                .compare_exchange_weak(state, state + READ_LOCKED, Acquire, Relaxed)
                .is_err()
        {
            //复杂的读锁获取或者进入等待
            self.read_contended();
        }
    }

    //解锁读
    pub unsafe fn read_unlock(&self) {
        //更新读锁
        let state = self.state.fetch_sub(READ_LOCKED, Release) - READ_LOCKED;

        // 除非有写线程在等待，否则此时不应该有线程在等待读. 写线程优先于读
        debug_assert!(!has_readers_waiting(state) || has_writers_waiting(state));

        // 如果有线程等着写，那就做唤醒操作，解锁读时，一定是有等着写的线程才能导致读线程被阻塞.
        if is_unlocked(state) && has_writers_waiting(state) {
            self.wake_writer_or_readers(state);
        }
    }

    //读线程阻塞处理
    fn read_contended(&self) {
        //自旋读，主要处理同时读的读锁更细冲突导致的不一致情况
        let mut state = self.spin_read();

        loop {
            // 再次尝试获取读锁
            if is_read_lockable(state) {
                match self.state.compare_exchange_weak(state, state + READ_LOCKED, Acquire, Relaxed)
                {
                    //成功
                    Ok(_) => return, // Locked!
                    //如果是读锁不一致，那就再次尝试
                    Err(s) => {
                        state = s;
                        continue;
                    }
                }
            }

            // 如果达到最大的读线程数目, 可能是有线程忘记解锁 
            if has_reached_max_readers(state) {
                panic!("too many active read locks on RwLock");
            }

            // 确保等待读的标志已经被设置.
            if !has_readers_waiting(state) {
                if let Err(s) =
                    self.state.compare_exchange(state, state | READERS_WAITING, Relaxed, Relaxed)
                {
                    //这里，上段的is_read_lockable(state)执行是失败的，所以不会有读锁被增加的问题
                    //失败说明读锁又在被并发修改，所以此时可能可以读了，要再次尝试
                    state = s;
                    continue;
                }
            }

            // 终于可以阻塞了，如果此时state没变，就会阻塞 
            futex_wait(&self.state, state | READERS_WAITING, None);

            // 此处或者是阻塞失败，或者被唤醒，两者都要重新看是否能够获得锁或者继续阻塞 
            state = self.spin_read();
        }
    }

    // 试图读，应该在不想阻塞的情况下调用, 此方法返回成功代表已经获取了写锁
    pub unsafe fn try_write(&self) -> bool {
        self.state
            //读的时候必须处于Unlocked状态
            .fetch_update(Acquire, Relaxed, |s| is_unlocked(s).then(|| s + WRITE_LOCKED))
            .is_ok()
    }

    //此函数用于获取写锁或阻塞等待到能获取
    pub unsafe fn write(&self) {
        if self.state.compare_exchange_weak(0, WRITE_LOCKED, Acquire, Relaxed).is_err() {
            //进入更复杂的锁获取或等待
            self.write_contended();
        }
    }

    //解锁写
    pub unsafe fn write_unlock(&self) {
        //更新值
        let state = self.state.fetch_sub(WRITE_LOCKED, Release) - WRITE_LOCKED;

        //还没有唤醒，应该没有其他线程冲突
        debug_assert!(is_unlocked(state));

        //有等待线程的话，就唤醒
        if has_writers_waiting(state) || has_readers_waiting(state) {
            self.wake_writer_or_readers(state);
        }
    }

    //进入写等待队列的处理
    fn write_contended(&self) {
        let mut state = self.spin_write();

        //假定当前没有线程等待写, 后继处理如果更新，
        //则说明有两个以上的线程在竞争获取写锁，所以
        //设置state时需要同时更新写等待标志位
        let mut other_writers_waiting = 0;

        loop {
            //  如果unlocked，那试图获取写锁
            if is_unlocked(state) {
                match self.state.compare_exchange_weak(
                    state,
                    state | WRITE_LOCKED | other_writers_waiting,
                    Acquire,
                    Relaxed,
                ) {
                    //获取成功
                    Ok(_) => return, // Locked!
                    Err(s) => {
                        //获取失败，再次循环
                        state = s;
                        continue;
                    }
                }
            }

            // 不为unlock，进入写等待队列并更新写等待标志
            if !has_writers_waiting(state) {
                if let Err(s) =
                    self.state.compare_exchange(state, state | WRITERS_WAITING, Relaxed, Relaxed)
                {
                    //更新失败，说明有同时访问者，需要重新试图获取写锁
                    state = s;
                    continue;
                }
            }

            // 不为unlock，且写等待标志位已经设置
            // 已经有其他写线程在等待.
            other_writers_waiting = WRITERS_WAITING;

            // 获取写锁解除的通知变量
            let seq = self.writer_notify.load(Acquire);

            let s = self.state.load(Relaxed);
            //这个地方错了，本意估计是is_unlocked(s), state此时已经确定
            //为lock了(注:作者在github提交了issue，目前此错误已经被修改)
            if is_unlocked(state) || !has_writers_waiting(s) {
                //这里如果又有变化，那么再次试图获得写锁
                state = s;
                continue;
            }

            // 阻塞，等待解锁通知 
            futex_wait(&self.writer_notify, seq, None);

            // 失败或者唤醒，重新再次试图获取写锁 
            state = self.spin_write();
        }
    }

    /// 唤醒等待线程
    fn wake_writer_or_readers(&self, mut state: i32) {
        assert!(is_unlocked(state));

        // 仅有写现程在等.
        if state == WRITERS_WAITING {
            //改变写等待标记，此时可能会形成一个冲突，所以write_contended会等待之前再做
            //一次判断
            match self.state.compare_exchange(state, 0, Relaxed, Relaxed) {
                Ok(_) => {
                    self.wake_writer();
                    return;
                }
                Err(s) => {
                    // 有冲突，更新state，此时只有可能是读线程在更新.
                    state = s;
                }
            }
        }

        // 即有读线程在等，也有写线程在等
        if state == READERS_WAITING + WRITERS_WAITING {
            //清写等待标志，此时0到30位肯定是0
            if self.state.compare_exchange(state, READERS_WAITING, Relaxed, Relaxed).is_err() {
                // 不应该出错，如果出错，那锁状态已经不对，无能为力了，而且不知道错误在哪里.
                // 感觉还是应该panic一下
                return;
            }
            //唤醒等待的写线程
            if self.wake_writer() {
                return;
            }
            // 执行到这里，证明没有写线程在等，那接下来处理读线程。此时直接修改state就可以，因为不可能有其他
            // 线程修改state了
            state = READERS_WAITING;
        }

        // 唤醒等待的读线程 
        if state == READERS_WAITING {
            if self.state.compare_exchange(state, 0, Relaxed, Relaxed).is_ok() {
                //唤醒所有读，这里读线程被唤醒后，仍然可能会有写线程插入，但不会出现问题
                futex_wake_all(&self.state);
            }
        }
    }

    /// 唤醒写线程
    fn wake_writer(&self) -> bool {
        //类似CondVar的处理方式，修改唤醒标志，然后唤醒即可
        self.writer_notify.fetch_add(1, Release);
        futex_wake(&self.writer_notify)
    }

    /// 自旋，处理一些在很短的时间内的状态修改使得锁可以获取，规避进入内核.
    fn spin_until(&self, f: impl Fn(i32) -> bool) -> i32 {
        let mut spin = 100; // Chosen by fair dice roll.
        loop {
            let state = self.state.load(Relaxed);
            //满足函数要求或自旋时间到，退出
            if f(state) || spin == 0 {
                return state;
            }
            crate::hint::spin_loop();
            spin -= 1;
        }
    }

    fn spin_write(&self) -> i32 {
        // 如果为unlock状态或者已经明确有写线程在等待并做了写等待置位.
        self.spin_until(|state| is_unlocked(state) || has_writers_waiting(state))
    }

    fn spin_read(&self) -> i32 {
        // 如果没有写锁，或者写等待或者读等待已经被置位 
        self.spin_until(|state| {
            !is_write_locked(state) || has_readers_waiting(state) || has_writers_waiting(state)
        })
    }
}

```
### 可重入的Mutex
如果一个线程调用lock获取锁之后，允许其在临界区的代码又对该锁调用lock，这个锁是Reentrant mutex。用futex处理这种情况效率不高，因为需要多次进入内核获取线程信息。因此，使用已有的libc的pthread_mutex_t的机制.
```rust
pub struct ReentrantMutex {
    //仅用来给libc使用
    inner: UnsafeCell<libc::pthread_mutex_t>,
}

unsafe impl Send for ReentrantMutex {}
unsafe impl Sync for ReentrantMutex {}

impl ReentrantMutex {
    //因为这个初始化没有完成可重入锁的设置，所以实际上没有初始化
    //但因为libc的限制，必须要先创建变量才能后完成初始化，所以需要此关联函数
    //调用此函数后，再调用init方法完成初始化
    pub const unsafe fn uninitialized() -> ReentrantMutex {
        ReentrantMutex { inner: UnsafeCell::new(libc::PTHREAD_MUTEX_INITIALIZER) }
    }

    //完成可重入锁的设置
    pub unsafe fn init(&self) {
        //栈中定义一块内存
        let mut attr = MaybeUninit::<libc::pthread_mutexattr_t>::uninit();
        //利用C函数做初始化，
        cvt_nz(libc::pthread_mutexattr_init(attr.as_mut_ptr())).unwrap();
        //创建类型变量
        let attr = PthreadMutexAttr(&mut attr);
        //设置attr属性
        cvt_nz(libc::pthread_mutexattr_settype(attr.0.as_mut_ptr(), libc::PTHREAD_MUTEX_RECURSIVE))
            .unwrap();
        //完成初始化
        cvt_nz(libc::pthread_mutex_init(self.inner.get(), attr.0.as_ptr())).unwrap();
    }

    //lock调用
    pub unsafe fn lock(&self) {
        //简单的调用libc函数
        let result = libc::pthread_mutex_lock(self.inner.get());
        debug_assert_eq!(result, 0);
    }

    //不想阻塞时的调用
    pub unsafe fn try_lock(&self) -> bool {
        //简单的调用libc
        libc::pthread_mutex_trylock(self.inner.get()) == 0
    }

    //解锁
    pub unsafe fn unlock(&self) {
        let result = libc::pthread_mutex_unlock(self.inner.get());
        debug_assert_eq!(result, 0);
    }

    //释放锁
    pub unsafe fn destroy(&self) {
        let result = libc::pthread_mutex_destroy(self.inner.get());
        debug_assert_eq!(result, 0);
    }
}
```

## RUST的锁机制
代码路径： library/std/src/sys_common/mutex.rs|condvar.rs|rwlock.rs
### 适用于静态变量的锁
因为所有权的定义，如果锁作为静态变量存在，则其初始化函数必须在编译期执行，即为const fn。静态锁主要用于保护静态变量形成的临界区。 

静态Mutex结构代码如下：

```rust
//用于static变量
pub struct StaticMutex(imp::Mutex);

unsafe impl Sync for StaticMutex {}

impl StaticMutex {
    /// const 函数可以用于static变量赋值 
    pub const fn new() -> Self {
        Self(imp::Mutex::new())
    }

    //上锁后用锁的引用形成封装结构返回，StaticMutexGuard见下文分析
    pub unsafe fn lock(&'static self) -> StaticMutexGuard {
        self.0.lock();
        StaticMutexGuard(&self.0)
    }
    //没有设计unlock方法
}

//此结构设计主要是充分利用RUST的编译器来简化解锁代码，
//使用StaticMutexGuard后可以不必在考虑解锁这件事
pub struct StaticMutexGuard(&'static imp::Mutex);

impl Drop for StaticMutexGuard {
    //生命周期终止时做unlock
    fn drop(&mut self) {
        unsafe {
            self.0.unlock();
        }
    }
}
```
对静态锁的支持仅限于StaticMutex
#### 适用于静态变量的读写锁
```rust
pub struct StaticRwLock(imp::RwLock);

impl StaticRwLock {
    /// const fn以初始化静态读写锁.
    pub const fn new() -> Self {
        Self(imp::RwLock::new())
    }

    /// 返回
    pub fn read(&'static self) -> StaticRwLockReadGuard {
        unsafe { self.0.read() };
        StaticRwLockReadGuard(&self.0)
    }

    pub fn write(&'static self) -> StaticRwLockWriteGuard {
        unsafe { self.0.write() };
        StaticRwLockWriteGuard(&self.0)
    }
}

pub struct StaticRwLockReadGuard(&'static imp::RwLock);

impl Drop for StaticRwLockReadGuard {
    fn drop(&mut self) {
        unsafe {
            self.0.read_unlock();
        }
    }
}

pub struct StaticRwLockWriteGuard(&'static imp::RwLock);

impl Drop for StaticRwLockWriteGuard {
    fn drop(&mut self) {
        unsafe {
            self.0.write_unlock();
        }
    }
}
```
### 适用于非静态变量的锁

#### MovableMutex
代码如下：
```rust
//imp::MoveableMutex在linux即imp::Mutex，其他系统基本也一样
pub struct MovableMutex(imp::MovableMutex);

unsafe impl Sync for MovableMutex {}

impl MovableMutex {
    /// 创建锁 
    pub fn new() -> Self {
        let mut mutex = imp::MovableMutex::from(imp::Mutex::new());
        //需要调用init(), 这里区别与StaticMutex
        unsafe { mutex.init() };
        Self(mutex)
    }

    pub(super) fn raw(&self) -> &imp::Mutex {
        &self.0
    }

    //获取锁
    pub fn raw_lock(&self) {
        unsafe { self.0.lock() }
    }

    //不希望阻塞获取锁
    pub fn try_lock(&self) -> bool {
        unsafe { self.0.try_lock() }
    }

    //释放锁
    pub unsafe fn raw_unlock(&self) {
        self.0.unlock()
    }
}

impl Drop for MovableMutex {
    fn drop(&mut self) {
        //用pthread_mutex_t需要
        unsafe { self.0.destroy() };
    }
}
```

#### 条件变量
```rust
//对Condvar的关联Mutex做check
type CondvarCheck = <imp::MovableMutex as check::CondvarCheck>::Check;

/// 对操作系统的Condvar做的封装.
pub struct Condvar {
    //就是imp::Condvar
    inner: imp::MovableCondvar,
    check: CondvarCheck,
}

impl Condvar {
    /// 创建新的Condvar.
    pub fn new() -> Self {
        let mut c = imp::MovableCondvar::from(imp::Condvar::new());
        unsafe { c.init() };
        Self { inner: c, check: CondvarCheck::new() }
    }

    /// 发信号唤醒一个等待此Condvar的线程.
    pub fn notify_one(&self) {
        unsafe { self.inner.notify_one() };
    }

    /// 发信号唤醒所有等待此信号的线程.
    pub fn notify_all(&self) {
        unsafe { self.inner.notify_all() };
    }

    /// 等待信号.
    pub unsafe fn wait(&self, mutex: &MovableMutex) {
        //确保始终使用同一个关联Mutex
        self.check.verify(mutex);
        // 阻塞并等待信号
        self.inner.wait(mutex.raw())
    }

    //超时等待
    pub unsafe fn wait_timeout(&self, mutex: &MovableMutex, dur: Duration) -> bool {
        self.check.verify(mutex);
        self.inner.wait_timeout(mutex.raw(), dur)
    }
}

impl Drop for Condvar {
    fn drop(&mut self) {
        unsafe { self.inner.destroy() };
    }
}

```

#### RWLock 
```rust
pub struct MovableRwLock(imp::MovableRwLock);

impl MovableRwLock {
    pub fn new() -> Self {
        Self(imp::MovableRwLock::from(imp::RwLock::new()))
    }

    pub fn read(&self) {
        unsafe { self.0.read() }
    }

    pub fn try_read(&self) -> bool {
        unsafe { self.0.try_read() }
    }

    pub fn write(&self) {
        unsafe { self.0.write() }
    }

    pub fn try_write(&self) -> bool {
        unsafe { self.0.try_write() }
    }

    pub unsafe fn read_unlock(&self) {
        self.0.read_unlock()
    }

    pub unsafe fn write_unlock(&self) {
        self.0.write_unlock()
    }
}

impl Drop for MovableRwLock {
    fn drop(&mut self) {
        unsafe { self.0.destroy() };
    }
}

```
## RUST中锁的杂项

### 加锁返回的统一类型
```rust
//lock方法的返回结果
pub type LockResult<Guard> = Result<Guard, PoisonError<Guard>>;
//try_lock方法的返回结果
pub type TryLockResult<Guard> = Result<Guard, TryLockError<Guard>>;

//错误类型
pub struct PoisonError<T> {
    guard: T,
}

impl<T> PoisonError<T> {
    //创建一个错误变量
    pub fn new(guard: T) -> PoisonError<T> {
        PoisonError { guard }
    }

    //从Error中获取导致错误的变量
    pub fn into_inner(self) -> T {
        self.guard
    }

    //获取导致错误变量的引用
    pub fn get_ref(&self) -> &T {
        &self.guard
    }

    //获取可变引用
    pub fn get_mut(&mut self) -> &mut T {
        &mut self.guard
    }
}

//Mutex<T> try_lock错误返回
pub enum TryLockError<T> {
    //线程异常返回
    Poisoned(PoisonError<T>),
    //临界区已经被锁，需要阻塞
    WouldBlock,
}
```

### Poison状态
如果一个线程在获取一个锁的期间发生了panic，则锁保护的临界区的数据已经不能认为是正确的。因为panic导致锁在一个未期望的代码位置解锁。此时，需要对锁标志一个状态，RUST名词称为锁处于Poison状态，并设计了Flag来表示这个状态。    
代码如下：
```rust
// 用于标识线程在加锁的状态下panic退出
pub struct Flag {
    failed: AtomicBool,
}

impl Flag {
    //初始的退出状态时候为假
    pub const fn new() -> Flag {
        Flag { failed: AtomicBool::new(false) }
    }

    //加锁的时候被调用，此时如果本线程已经panic，则需要返回错误
    pub fn borrow(&self) -> LockResult<Guard> {
        //获取本线程的panic状态, 相关部分在Thread一节会再解释
        let ret = Guard { panicking: thread::panicking() };
        //如果Flag已经是真，则返回Err并给出本线程状态，否则返回Ok
        if self.get() { Err(PoisonError::new(ret)) } else { Ok(ret) }
    }

    //释放锁的时候被调用，如果本线程panic，则会更新Flag为true
    pub fn done(&self, guard: &Guard) {
        if !guard.panicking && thread::panicking() {
            self.failed.store(true, Ordering::Relaxed);
        }
    }

    //获得Flag的值
    pub fn get(&self) -> bool {
        self.failed.load(Ordering::Relaxed)
    }
}

```
## RUST的临界区变量实现
代码路径: library/std/src/sync/*.rs
### `Mutex<T>`的实现
`Mutex<T>`是最典型的临界区变量。RUST的设计是使得`Mutex<T>`与其他的类型变量在使用中类似。代码中不必关注其的跨线程操作的安全性。这一设计思路实际在`RefCell<T>`, `Rc<T>`, `Arc<T>`等安全封装结构时是一脉相承的：   
1. 设计一个基础类型结构，将要操作的真实类型变量封装在其内，并拥有其所有权，
2. 设计一个借用类型结构，由基础类型结构的某一方法生成，在此方法中完成附加安全操作，如计数增加，加锁等。
3. 借用类型结构实现解引用方法，返回真实类型变量的引用或可变引用，由此可以对真实变类型变量进行访问，修改操作
4. 借用类型结构的drop方法会执行安全逆操作，如减少计数或解锁。

`Mutex<T>`的设计如下：   
1. 基本类型`Mutex<T>`，负责临界区的数据存储及Mutex锁     
2. `MutexGuard<'a, T>`作为`Mutex<T>`的借用类型结构, lock()作为借用方法，返回`MutexGuard<T>`，可以直接对其解引用后获得内部变量的引用/可变引用，随后执行临界区数据操作及读写。生命周期结束后，`MutexGuard<T>`的drop会解锁操作，从而使得加锁解锁操作实际上代码不必关心。lock()本身完全可以等同于一个borrow()的调用。       
3. `Mutex<T>`本身是一个内部可变型的类型, 实现多处共享且可修改     
4. 线程panic时的Poison处理,使得其他语言极少关注的情况在RUST中自然得解。 

```rust
pub struct Mutex<T: ?Sized> {
    //临界区的锁
    inner: sys::MovableMutex,
    //标识Mutex在线程panic时处于锁状态
    poison: poison::Flag,
    //临界区数据, Mutex本身是一个内部可变性的类型
    data: UnsafeCell<T>,
}

unsafe impl<T: ?Sized + Send> Send for Mutex<T> {}
unsafe impl<T: ?Sized + Send> Sync for Mutex<T> {}

```

用于`Mutex<T>`配合的借用封装类型结构`MutexGuard`如下：
```rust
//用于lock调用后的对原始变量的访问引用。并包含了poison用于在生命周期终结的时候
//更新Mutex<T>的Flag
pub struct MutexGuard<'a, T: ?Sized + 'a> {
    lock: &'a Mutex<T>,
    poison: poison::Guard,
}

//标识MutexGuard的当前线程状态
pub struct Guard {
    panicking: bool,
}

//支持函数
pub fn map_result<T, U, F>(result: LockResult<T>, f: F) -> LockResult<U>
where
    F: FnOnce(T) -> U,
{
    match result {
        Ok(t) => Ok(f(t)),
        Err(PoisonError { guard }) => Err(PoisonError::new(f(guard))),
    }
}

//MutexGuard创建关联函数
impl<'mutex, T: ?Sized> MutexGuard<'mutex, T> {
    unsafe fn new(lock: &'mutex Mutex<T>) -> LockResult<MutexGuard<'mutex, T>> {
        //代码见上面的函数，这里，如果Mutex<T>的poison为假，即使本线程已经panic，也返回Ok类型
        //因为不是在加锁时遇到panic，所以临界区数据还是好的。
        poison::map_result(lock.poison.borrow(), |guard| MutexGuard { lock, poison: guard })
    }
}

//deref，返回临界区数据的引用
impl<T: ?Sized> Deref for MutexGuard<'_, T> {
    type Target = T;

    fn deref(&self) -> &T {
        //利用UnsafeCell获得内部可变性
        unsafe { &*self.lock.data.get() }
    }
}

//返回临界区数据的可变引用
impl<T: ?Sized> DerefMut for MutexGuard<'_, T> {
    fn deref_mut(&mut self) -> &mut T {
        unsafe { &mut *self.lock.data.get() }
    }
}

//drop方法
impl<T: ?Sized> Drop for MutexGuard<'_, T> {
    fn drop(&mut self) {
        unsafe {
            //更新Mutex<T>的Flag,一般的，如果在上锁的状态下panic
            //此方法会被调用，从而能更新poison为true
            self.lock.poison.done(&self.poison);
            //解锁
            self.lock.inner.raw_unlock();
        }
    }
}

//获取Mutex
pub fn guard_lock<'a, T: ?Sized>(guard: &MutexGuard<'a, T>) -> &'a sys::MovableMutex {
    &guard.lock.inner
}

//获取线程panic状态
pub fn guard_poison<'a, T: ?Sized>(guard: &MutexGuard<'a, T>) -> &'a poison::Flag {
    &guard.lock.poison
}

```
在`Mutex<T>`结构中，poison的更新一般在发生panic时，线程终结`MutexGuard<T>`时进行值的更新。因为poison的考虑，`Mutex<T>`的lock后返回的结构复杂。     

`Mutex<T>`的代码分析如下：   
```rust 
//只能创建固定尺寸类型的临界区
impl<T> Mutex<T> {
    //对数据创建一个临界区
    pub fn new(t: T) -> Mutex<T> {
        Mutex {
            //创建系统MovableMutex类型
            inner: sys::MovableMutex::new(),
            //poison为false
            poison: poison::Flag::new(),
            //必须用内部可变性类型
            data: UnsafeCell::new(t),
        }
    }
}

impl<T: ?Sized> Mutex<T> {
    // 获取锁，允许阻塞，返回guard结构用于访问临界区数据及处理锁的释放
    pub fn lock(&self) -> LockResult<MutexGuard<'_, T>> {
        unsafe {
            //先做锁操作
            self.inner.raw_lock();
            //在MutexGuard的new中处理线程panic问题
            MutexGuard::new(self)
        }
    }

    //试图获取锁，不希望阻塞的时候调用
    pub fn try_lock(&self) -> TryLockResult<MutexGuard<'_, T>> {
        unsafe {
            if self.inner.try_lock() {
                //上锁成功，生成MutexGuard
                Ok(MutexGuard::new(self)?)
            } else {
                //失败，提示应该阻塞
                Err(TryLockError::WouldBlock)
            }
        }
    }

    //立即解锁，不希望等待guard生命周期终结，
    pub fn unlock(guard: MutexGuard<'_, T>) {
        drop(guard);
    }

    //是否有线程在panic时锁住了临界区
    pub fn is_poisoned(&self) -> bool {
        self.poison.get()
    }

    //消费Mutex<T>,并获取临界区数据,如果进入此方法内部
    //证明没有锁存在。
    pub fn into_inner(self) -> LockResult<T>
    where
        T: Sized,
    {
        //获取临界区数据
        let data = self.data.into_inner();
        //根据是否有线程在panic时加锁,
        poison::map_result(self.poison.borrow(), |_| data)
    }

    //获取临界区数据的可变引用，此时应该保证没有锁存在，否则
    //可能导致数据竞争。
    pub fn get_mut(&mut self) -> LockResult<&mut T> {
        let data = self.data.get_mut();
        poison::map_result(self.poison.borrow(), |_| data)
    }
}

```
RUST的`Mutex<T>`几乎是极限简单化了临界区访问的代码，并使得使用`Mutex<T>`的程序员不必再熟悉锁的概念就能写出安全的代码。

### `Condvar`实现分析
Condvar在这里主要因为与其配合的成为了`MutexGuard<'a, T>`临界区变量，所以有针对性的处理。此处已经把Condvar与临界区数据做了关联。  

```rust
//仅仅是对操作系统的Condvar的一个封装
pub struct Condvar {
    inner: sys::Condvar,
}

impl Condvar {
    
    pub fn new() -> Condvar {
        Condvar { inner: sys::Condvar::new() }
    }

    // 等待信号通知，并可能进入阻塞,因为用MutexGuard, 保证了必须有关联的Mutex，
    // 且Mutex一定已经lock，且临界区数据包括在内，使得Condvar更易被理解及使用
    pub fn wait<'a, T>(&self, guard: MutexGuard<'a, T>) -> LockResult<MutexGuard<'a, T>> {
        let poisoned = unsafe {
            //获取关联的imp::Mutex
            let lock = mutex::guard_lock(&guard);
            self.inner.wait(lock);
            //获取Mutex的poison
            mutex::guard_poison(&guard).get()
        };
        if poisoned { Err(PoisonError::new(guard)) } else { Ok(guard) }
    }

    //增值函数，在临界区数据满足某个条件一直等待信号
    pub fn wait_while<'a, T, F>(
        &self,
        mut guard: MutexGuard<'a, T>,
        mut condition: F,
    ) -> LockResult<MutexGuard<'a, T>>
    where
        F: FnMut(&mut T) -> bool,
    {
        while condition(&mut *guard) {
            guard = self.wait(guard)?;
        }
        Ok(guard)
    }

    // 简化以毫秒计数的超时等待
    pub fn wait_timeout_ms<'a, T>(
        &self,
        guard: MutexGuard<'a, T>,
        ms: u32,
    ) -> LockResult<(MutexGuard<'a, T>, bool)> {
        let res = self.wait_timeout(guard, Duration::from_millis(ms as u64));
        poison::map_result(res, |(a, b)| (a, !b.timed_out()))
    }

    // 超时等待
    pub fn wait_timeout<'a, T>(
        &self,
        guard: MutexGuard<'a, T>,
        dur: Duration,
    ) -> LockResult<(MutexGuard<'a, T>, WaitTimeoutResult)> {
        let (poisoned, result) = unsafe {
            let lock = mutex::guard_lock(&guard);
            let success = self.inner.wait_timeout(lock, dur);
            (mutex::guard_poison(&guard).get(), WaitTimeoutResult(!success))
        };
        if poisoned { Err(PoisonError::new((guard, result))) } else { Ok((guard, result)) }
    }

    //wait_while的超时版本
    pub fn wait_timeout_while<'a, T, F>(
        &self,
        mut guard: MutexGuard<'a, T>,
        dur: Duration,
        mut condition: F,
    ) -> LockResult<(MutexGuard<'a, T>, WaitTimeoutResult)>
    where
        F: FnMut(&mut T) -> bool,
    {
        let start = Instant::now();
        loop {
            if !condition(&mut *guard) {
                return Ok((guard, WaitTimeoutResult(false)));
            }
            let timeout = match dur.checked_sub(start.elapsed()) {
                Some(timeout) => timeout,
                None => return Ok((guard, WaitTimeoutResult(true))),
            };
            guard = self.wait_timeout(guard, timeout)?.0;
        }
    }

    //信号通知，唤醒一个线程
    pub fn notify_one(&self) {
        self.inner.notify_one()
    }

    //信号通知，唤醒所有线程
    pub fn notify_all(&self) {
        self.inner.notify_all()
    }
}

```
### `RWLock<T>`分析
与`Mutex<T>`的设计采用了一致的方案：

代码分析如下：
```rust
//与Mutex<T>几乎同样的成员
pub struct RwLock<T: ?Sized> {
    inner: sys::MovableRwLock,
    poison: poison::Flag,
    data: UnsafeCell<T>,
}

unsafe impl<T: ?Sized + Send> Send for RwLock<T> {}
unsafe impl<T: ?Sized + Send + Sync> Sync for RwLock<T> {}

//用read锁后的借用封装类型结构
pub struct RwLockReadGuard<'a, T: ?Sized + 'a> {
    lock: &'a RwLock<T>,
}

impl<T: ?Sized> !Send for RwLockReadGuard<'_, T> {}

unsafe impl<T: ?Sized + Sync> Sync for RwLockReadGuard<'_, T> {}

//用write锁后的借用封装类型结构
pub struct RwLockWriteGuard<'a, T: ?Sized + 'a> {
    lock: &'a RwLock<T>,
    poison: poison::Guard,
}

impl<T: ?Sized> !Send for RwLockWriteGuard<'_, T> {}

unsafe impl<T: ?Sized + Sync> Sync for RwLockWriteGuard<'_, T> {}

impl<T> RwLock<T> {
    pub fn new(t: T) -> RwLock<T> {
        RwLock {
            inner: sys::MovableRwLock::new(),
            poison: poison::Flag::new(),
            data: UnsafeCell::new(t),
        }
    }
}

impl<T: ?Sized> RwLock<T> {
    //读上锁，返回一个读锁的临界区借用封装
    pub fn read(&self) -> LockResult<RwLockReadGuard<'_, T>> {
        unsafe {
            self.inner.read();
            RwLockReadGuard::new(self)
        }
    }

    //不希望阻塞时做调用
    pub fn try_read(&self) -> TryLockResult<RwLockReadGuard<'_, T>> {
        unsafe {
            if self.inner.try_read() {
                Ok(RwLockReadGuard::new(self)?)
            } else {
                Err(TryLockError::WouldBlock)
            }
        }
    }

    //写锁，返回一个写锁的借用封装
    pub fn write(&self) -> LockResult<RwLockWriteGuard<'_, T>> {
        unsafe {
            self.inner.write();
            RwLockWriteGuard::new(self)
        }
    }

    //不希望阻塞时的写锁调用
    pub fn try_write(&self) -> TryLockResult<RwLockWriteGuard<'_, T>> {
        unsafe {
            if self.inner.try_write() {
                Ok(RwLockWriteGuard::new(self)?)
            } else {
                Err(TryLockError::WouldBlock)
            }
        }
    }

    //是否中毒
    pub fn is_poisoned(&self) -> bool {
        self.poison.get()
    }

    //消费掉锁，此时如果有读锁或写锁，编译器会告警
    pub fn into_inner(self) -> LockResult<T>
    where
        T: Sized,
    {
        let data = self.data.into_inner();
        poison::map_result(self.poison.borrow(), |_| data)
    }

    //此时如果有读锁或写锁，编译器会告警,针对self
    pub fn get_mut(&mut self) -> LockResult<&mut T> {
        let data = self.data.get_mut();
        poison::map_result(self.poison.borrow(), |_| data)
    }
}

impl<T> From<T> for RwLock<T> {
    /// Creates a new instance of an `RwLock<T>` which is unlocked.
    /// This is equivalent to [`RwLock::new`].
    fn from(t: T) -> Self {
        RwLock::new(t)
    }
}

impl<'rwlock, T: ?Sized> RwLockReadGuard<'rwlock, T> {
    unsafe fn new(lock: &'rwlock RwLock<T>) -> LockResult<RwLockReadGuard<'rwlock, T>> {
        poison::map_result(lock.poison.borrow(), |_| RwLockReadGuard { lock })
    }
}

impl<'rwlock, T: ?Sized> RwLockWriteGuard<'rwlock, T> {
    unsafe fn new(lock: &'rwlock RwLock<T>) -> LockResult<RwLockWriteGuard<'rwlock, T>> {
        poison::map_result(lock.poison.borrow(), |guard| RwLockWriteGuard { lock, poison: guard })
    }
}

impl<T: ?Sized> Deref for RwLockReadGuard<'_, T> {
    type Target = T;

    fn deref(&self) -> &T {
        unsafe { &*self.lock.data.get() }
    }
}

impl<T: ?Sized> Deref for RwLockWriteGuard<'_, T> {
    type Target = T;

    fn deref(&self) -> &T {
        unsafe { &*self.lock.data.get() }
    }
}

impl<T: ?Sized> DerefMut for RwLockWriteGuard<'_, T> {
    fn deref_mut(&mut self) -> &mut T {
        unsafe { &mut *self.lock.data.get() }
    }
}

impl<T: ?Sized> Drop for RwLockReadGuard<'_, T> {
    fn drop(&mut self) {
        unsafe {
            self.lock.inner.read_unlock();
        }
    }
}

impl<T: ?Sized> Drop for RwLockWriteGuard<'_, T> {
    fn drop(&mut self) {
        self.lock.poison.done(&self.poison);
        unsafe {
            self.lock.inner.write_unlock();
        }
    }
}
```
### Barrier 类型临界变量
```rust
pub struct Barrier {
    lock: Mutex<BarrierState>,
    cvar: Condvar,
    num_threads: usize,
}

// The inner state of a double barrier
struct BarrierState {
    count: usize,
    generation_id: usize,
}

/// A `BarrierWaitResult` is returned by [`Barrier::wait()`] when all threads
/// in the [`Barrier`] have rendezvoused.
///
/// # Examples
///
/// ```
/// use std::sync::Barrier;
///
/// let barrier = Barrier::new(1);
/// let barrier_wait_result = barrier.wait();
/// ```
pub struct BarrierWaitResult(bool);


impl Barrier {
    /// Creates a new barrier that can block a given number of threads.
    ///
    /// A barrier will block `n`-1 threads which call [`wait()`] and then wake
    /// up all threads at once when the `n`th thread calls [`wait()`].
    ///
    /// [`wait()`]: Barrier::wait
    ///
    /// # Examples
    ///
    /// ```
    /// use std::sync::Barrier;
    ///
    /// let barrier = Barrier::new(10);
    /// ```
    pub fn new(n: usize) -> Barrier {
        Barrier {
            lock: Mutex::new(BarrierState { count: 0, generation_id: 0 }),
            cvar: Condvar::new(),
            num_threads: n,
        }
    }

    /// Blocks the current thread until all threads have rendezvoused here.
    ///
    /// Barriers are re-usable after all threads have rendezvoused once, and can
    /// be used continuously.
    ///
    /// A single (arbitrary) thread will receive a [`BarrierWaitResult`] that
    /// returns `true` from [`BarrierWaitResult::is_leader()`] when returning
    /// from this function, and all other threads will receive a result that
    /// will return `false` from [`BarrierWaitResult::is_leader()`].
    ///
    /// # Examples
    ///
    /// ```
    /// use std::sync::{Arc, Barrier};
    /// use std::thread;
    ///
    /// let mut handles = Vec::with_capacity(10);
    /// let barrier = Arc::new(Barrier::new(10));
    /// for _ in 0..10 {
    ///     let c = Arc::clone(&barrier);
    ///     // The same messages will be printed together.
    ///     // You will NOT see any interleaving.
    ///     handles.push(thread::spawn(move|| {
    ///         println!("before wait");
    ///         c.wait();
    ///         println!("after wait");
    ///     }));
    /// }
    /// // Wait for other threads to finish.
    /// for handle in handles {
    ///     handle.join().unwrap();
    /// }
    /// ```
    pub fn wait(&self) -> BarrierWaitResult {
        let mut lock = self.lock.lock().unwrap();
        let local_gen = lock.generation_id;
        lock.count += 1;
        if lock.count < self.num_threads {
            // We need a while loop to guard against spurious wakeups.
            // https://en.wikipedia.org/wiki/Spurious_wakeup
            while local_gen == lock.generation_id {
                lock = self.cvar.wait(lock).unwrap();
            }
            BarrierWaitResult(false)
        } else {
            lock.count = 0;
            lock.generation_id = lock.generation_id.wrapping_add(1);
            self.cvar.notify_all();
            BarrierWaitResult(true)
        }
    }
}

impl BarrierWaitResult {
    /// Returns `true` if this thread is the "leader thread" for the call to
    /// [`Barrier::wait()`].
    ///
    /// Only one thread will have `true` returned from their result, all other
    /// threads will have `false` returned.
    ///
    /// # Examples
    ///
    /// ```
    /// use std::sync::Barrier;
    ///
    /// let barrier = Barrier::new(1);
    /// let barrier_wait_result = barrier.wait();
    /// println!("{:?}", barrier_wait_result.is_leader());
    /// ```
    #[stable(feature = "rust1", since = "1.0.0")]
    #[must_use]
    pub fn is_leader(&self) -> bool {
        self.0
    }
}

```
### Once 类型分析

```rust
type Masked = ();

/// A synchronization primitive which can be used to run a one-time global
/// initialization. Useful for one-time initialization for FFI or related
/// functionality. This type can only be constructed with [`Once::new()`].
///
/// # Examples
///
/// ```
/// use std::sync::Once;
///
/// static START: Once = Once::new();
///
/// START.call_once(|| {
///     // run initialization here
/// });
/// ```
#[stable(feature = "rust1", since = "1.0.0")]
pub struct Once {
    // `state_and_queue` is actually a pointer to a `Waiter` with extra state
    // bits, so we add the `PhantomData` appropriately.
    state_and_queue: AtomicPtr<Masked>,
    _marker: marker::PhantomData<*const Waiter>,
}

// The `PhantomData` of a raw pointer removes these two auto traits, but we
// enforce both below in the implementation so this should be safe to add.
unsafe impl Sync for Once {}
unsafe impl Send for Once {}

impl UnwindSafe for Once {}

impl RefUnwindSafe for Once {}

pub struct OnceState {
    poisoned: bool,
    set_state_on_drop_to: Cell<*mut Masked>,
}

pub const ONCE_INIT: Once = Once::new();

// Four states that a Once can be in, encoded into the lower bits of
// `state_and_queue` in the Once structure.
const INCOMPLETE: usize = 0x0;
const POISONED: usize = 0x1;
const RUNNING: usize = 0x2;
const COMPLETE: usize = 0x3;

// Mask to learn about the state. All other bits are the queue of waiters if
// this is in the RUNNING state.
const STATE_MASK: usize = 0x3;

// Representation of a node in the linked list of waiters, used while in the
// RUNNING state.
// Note: `Waiter` can't hold a mutable pointer to the next thread, because then
// `wait` would both hand out a mutable reference to its `Waiter` node, and keep
// a shared reference to check `signaled`. Instead we hold shared references and
// use interior mutability.
#[repr(align(4))] // Ensure the two lower bits are free to use as state bits.
struct Waiter {
    thread: Cell<Option<Thread>>,
    signaled: AtomicBool,
    next: *const Waiter,
}

// Head of a linked list of waiters.
// Every node is a struct on the stack of a waiting thread.
// Will wake up the waiters when it gets dropped, i.e. also on panic.
struct WaiterQueue<'a> {
    state_and_queue: &'a AtomicPtr<Masked>,
    set_state_on_drop_to: *mut Masked,
}

impl Once {
    /// Creates a new `Once` value.
    #[inline]
    #[stable(feature = "once_new", since = "1.2.0")]
    #[rustc_const_stable(feature = "const_once_new", since = "1.32.0")]
    #[must_use]
    pub const fn new() -> Once {
        Once {
            state_and_queue: AtomicPtr::new(ptr::invalid_mut(INCOMPLETE)),
            _marker: marker::PhantomData,
        }
    }

    /// Performs an initialization routine once and only once. The given closure
    /// will be executed if this is the first time `call_once` has been called,
    /// and otherwise the routine will *not* be invoked.
    ///
    /// This method will block the calling thread if another initialization
    /// routine is currently running.
    ///
    /// When this function returns, it is guaranteed that some initialization
    /// has run and completed (it might not be the closure specified). It is also
    /// guaranteed that any memory writes performed by the executed closure can
    /// be reliably observed by other threads at this point (there is a
    /// happens-before relation between the closure and code executing after the
    /// return).
    ///
    /// If the given closure recursively invokes `call_once` on the same [`Once`]
    /// instance the exact behavior is not specified, allowed outcomes are
    /// a panic or a deadlock.
    ///
    /// # Examples
    ///
    /// ```
    /// use std::sync::Once;
    ///
    /// static mut VAL: usize = 0;
    /// static INIT: Once = Once::new();
    ///
    /// // Accessing a `static mut` is unsafe much of the time, but if we do so
    /// // in a synchronized fashion (e.g., write once or read all) then we're
    /// // good to go!
    /// //
    /// // This function will only call `expensive_computation` once, and will
    /// // otherwise always return the value returned from the first invocation.
    /// fn get_cached_val() -> usize {
    ///     unsafe {
    ///         INIT.call_once(|| {
    ///             VAL = expensive_computation();
    ///         });
    ///         VAL
    ///     }
    /// }
    ///
    /// fn expensive_computation() -> usize {
    ///     // ...
    /// # 2
    /// }
    /// ```
    ///
    /// # Panics
    ///
    /// The closure `f` will only be executed once if this is called
    /// concurrently amongst many threads. If that closure panics, however, then
    /// it will *poison* this [`Once`] instance, causing all future invocations of
    /// `call_once` to also panic.
    ///
    /// This is similar to [poisoning with mutexes][poison].
    ///
    /// [poison]: struct.Mutex.html#poisoning
    #[stable(feature = "rust1", since = "1.0.0")]
    #[track_caller]
    pub fn call_once<F>(&self, f: F)
    where
        F: FnOnce(),
    {
        // Fast path check
        if self.is_completed() {
            return;
        }

        let mut f = Some(f);
        self.call_inner(false, &mut |_| f.take().unwrap()());
    }

    /// Performs the same function as [`call_once()`] except ignores poisoning.
    ///
    /// Unlike [`call_once()`], if this [`Once`] has been poisoned (i.e., a previous
    /// call to [`call_once()`] or [`call_once_force()`] caused a panic), calling
    /// [`call_once_force()`] will still invoke the closure `f` and will _not_
    /// result in an immediate panic. If `f` panics, the [`Once`] will remain
    /// in a poison state. If `f` does _not_ panic, the [`Once`] will no
    /// longer be in a poison state and all future calls to [`call_once()`] or
    /// [`call_once_force()`] will be no-ops.
    ///
    /// The closure `f` is yielded a [`OnceState`] structure which can be used
    /// to query the poison status of the [`Once`].
    ///
    /// [`call_once()`]: Once::call_once
    /// [`call_once_force()`]: Once::call_once_force
    ///
    /// # Examples
    ///
    /// ```
    /// use std::sync::Once;
    /// use std::thread;
    ///
    /// static INIT: Once = Once::new();
    ///
    /// // poison the once
    /// let handle = thread::spawn(|| {
    ///     INIT.call_once(|| panic!());
    /// });
    /// assert!(handle.join().is_err());
    ///
    /// // poisoning propagates
    /// let handle = thread::spawn(|| {
    ///     INIT.call_once(|| {});
    /// });
    /// assert!(handle.join().is_err());
    ///
    /// // call_once_force will still run and reset the poisoned state
    /// INIT.call_once_force(|state| {
    ///     assert!(state.is_poisoned());
    /// });
    ///
    /// // once any success happens, we stop propagating the poison
    /// INIT.call_once(|| {});
    /// ```
    #[stable(feature = "once_poison", since = "1.51.0")]
    pub fn call_once_force<F>(&self, f: F)
    where
        F: FnOnce(&OnceState),
    {
        // Fast path check
        if self.is_completed() {
            return;
        }

        let mut f = Some(f);
        self.call_inner(true, &mut |p| f.take().unwrap()(p));
    }

    /// Returns `true` if some [`call_once()`] call has completed
    /// successfully. Specifically, `is_completed` will return false in
    /// the following situations:
    ///   * [`call_once()`] was not called at all,
    ///   * [`call_once()`] was called, but has not yet completed,
    ///   * the [`Once`] instance is poisoned
    ///
    /// This function returning `false` does not mean that [`Once`] has not been
    /// executed. For example, it may have been executed in the time between
    /// when `is_completed` starts executing and when it returns, in which case
    /// the `false` return value would be stale (but still permissible).
    ///
    /// [`call_once()`]: Once::call_once
    ///
    /// # Examples
    ///
    /// ```
    /// use std::sync::Once;
    ///
    /// static INIT: Once = Once::new();
    ///
    /// assert_eq!(INIT.is_completed(), false);
    /// INIT.call_once(|| {
    ///     assert_eq!(INIT.is_completed(), false);
    /// });
    /// assert_eq!(INIT.is_completed(), true);
    /// ```
    ///
    /// ```
    /// use std::sync::Once;
    /// use std::thread;
    ///
    /// static INIT: Once = Once::new();
    ///
    /// assert_eq!(INIT.is_completed(), false);
    /// let handle = thread::spawn(|| {
    ///     INIT.call_once(|| panic!());
    /// });
    /// assert!(handle.join().is_err());
    /// assert_eq!(INIT.is_completed(), false);
    /// ```
    #[stable(feature = "once_is_completed", since = "1.43.0")]
    #[inline]
    pub fn is_completed(&self) -> bool {
        // An `Acquire` load is enough because that makes all the initialization
        // operations visible to us, and, this being a fast path, weaker
        // ordering helps with performance. This `Acquire` synchronizes with
        // `Release` operations on the slow path.
        self.state_and_queue.load(Ordering::Acquire).addr() == COMPLETE
    }

    // This is a non-generic function to reduce the monomorphization cost of
    // using `call_once` (this isn't exactly a trivial or small implementation).
    //
    // Additionally, this is tagged with `#[cold]` as it should indeed be cold
    // and it helps let LLVM know that calls to this function should be off the
    // fast path. Essentially, this should help generate more straight line code
    // in LLVM.
    //
    // Finally, this takes an `FnMut` instead of a `FnOnce` because there's
    // currently no way to take an `FnOnce` and call it via virtual dispatch
    // without some allocation overhead.
    #[cold]
    #[track_caller]
    fn call_inner(&self, ignore_poisoning: bool, init: &mut dyn FnMut(&OnceState)) {
        let mut state_and_queue = self.state_and_queue.load(Ordering::Acquire);
        loop {
            match state_and_queue.addr() {
                COMPLETE => break,
                POISONED if !ignore_poisoning => {
                    // Panic to propagate the poison.
                    panic!("Once instance has previously been poisoned");
                }
                POISONED | INCOMPLETE => {
                    // Try to register this thread as the one RUNNING.
                    let exchange_result = self.state_and_queue.compare_exchange(
                        state_and_queue,
                        ptr::invalid_mut(RUNNING),
                        Ordering::Acquire,
                        Ordering::Acquire,
                    );
                    if let Err(old) = exchange_result {
                        state_and_queue = old;
                        continue;
                    }
                    // `waiter_queue` will manage other waiting threads, and
                    // wake them up on drop.
                    let mut waiter_queue = WaiterQueue {
                        state_and_queue: &self.state_and_queue,
                        set_state_on_drop_to: ptr::invalid_mut(POISONED),
                    };
                    // Run the initialization function, letting it know if we're
                    // poisoned or not.
                    let init_state = OnceState {
                        poisoned: state_and_queue.addr() == POISONED,
                        set_state_on_drop_to: Cell::new(ptr::invalid_mut(COMPLETE)),
                    };
                    init(&init_state);
                    waiter_queue.set_state_on_drop_to = init_state.set_state_on_drop_to.get();
                    break;
                }
                _ => {
                    // All other values must be RUNNING with possibly a
                    // pointer to the waiter queue in the more significant bits.
                    assert!(state_and_queue.addr() & STATE_MASK == RUNNING);
                    wait(&self.state_and_queue, state_and_queue);
                    state_and_queue = self.state_and_queue.load(Ordering::Acquire);
                }
            }
        }
    }
}

fn wait(state_and_queue: &AtomicPtr<Masked>, mut current_state: *mut Masked) {
    // Note: the following code was carefully written to avoid creating a
    // mutable reference to `node` that gets aliased.
    loop {
        // Don't queue this thread if the status is no longer running,
        // otherwise we will not be woken up.
        if current_state.addr() & STATE_MASK != RUNNING {
            return;
        }

        // Create the node for our current thread.
        let node = Waiter {
            thread: Cell::new(Some(thread::current())),
            signaled: AtomicBool::new(false),
            next: current_state.with_addr(current_state.addr() & !STATE_MASK) as *const Waiter,
        };
        let me = &node as *const Waiter as *const Masked as *mut Masked;

        // Try to slide in the node at the head of the linked list, making sure
        // that another thread didn't just replace the head of the linked list.
        let exchange_result = state_and_queue.compare_exchange(
            current_state,
            me.with_addr(me.addr() | RUNNING),
            Ordering::Release,
            Ordering::Relaxed,
        );
        if let Err(old) = exchange_result {
            current_state = old;
            continue;
        }

        // We have enqueued ourselves, now lets wait.
        // It is important not to return before being signaled, otherwise we
        // would drop our `Waiter` node and leave a hole in the linked list
        // (and a dangling reference). Guard against spurious wakeups by
        // reparking ourselves until we are signaled.
        while !node.signaled.load(Ordering::Acquire) {
            // If the managing thread happens to signal and unpark us before we
            // can park ourselves, the result could be this thread never gets
            // unparked. Luckily `park` comes with the guarantee that if it got
            // an `unpark` just before on an unparked thread it does not park.
            thread::park();
        }
        break;
    }
}

impl Drop for WaiterQueue<'_> {
    fn drop(&mut self) {
        // Swap out our state with however we finished.
        let state_and_queue =
            self.state_and_queue.swap(self.set_state_on_drop_to, Ordering::AcqRel);

        // We should only ever see an old state which was RUNNING.
        assert_eq!(state_and_queue.addr() & STATE_MASK, RUNNING);

        // Walk the entire linked list of waiters and wake them up (in lifo
        // order, last to register is first to wake up).
        unsafe {
            // Right after setting `node.signaled = true` the other thread may
            // free `node` if there happens to be has a spurious wakeup.
            // So we have to take out the `thread` field and copy the pointer to
            // `next` first.
            let mut queue =
                state_and_queue.with_addr(state_and_queue.addr() & !STATE_MASK) as *const Waiter;
            while !queue.is_null() {
                let next = (*queue).next;
                let thread = (*queue).thread.take().unwrap();
                (*queue).signaled.store(true, Ordering::Release);
                // ^- FIXME (maybe): This is another case of issue #55005
                // `store()` has a potentially dangling ref to `signaled`.
                queue = next;
                thread.unpark();
            }
        }
    }
}

impl OnceState {
    /// Returns `true` if the associated [`Once`] was poisoned prior to the
    /// invocation of the closure passed to [`Once::call_once_force()`].
    ///
    /// # Examples
    ///
    /// A poisoned [`Once`]:
    ///
    /// ```
    /// use std::sync::Once;
    /// use std::thread;
    ///
    /// static INIT: Once = Once::new();
    ///
    /// // poison the once
    /// let handle = thread::spawn(|| {
    ///     INIT.call_once(|| panic!());
    /// });
    /// assert!(handle.join().is_err());
    ///
    /// INIT.call_once_force(|state| {
    ///     assert!(state.is_poisoned());
    /// });
    /// ```
    ///
    /// An unpoisoned [`Once`]:
    ///
    /// ```
    /// use std::sync::Once;
    ///
    /// static INIT: Once = Once::new();
    ///
    /// INIT.call_once_force(|state| {
    ///     assert!(!state.is_poisoned());
    /// });
    #[stable(feature = "once_poison", since = "1.51.0")]
    pub fn is_poisoned(&self) -> bool {
        self.poisoned
    }

    /// Poison the associated [`Once`] without explicitly panicking.
    // NOTE: This is currently only exposed for the `lazy` module
    pub(crate) fn poison(&self) {
        self.set_state_on_drop_to.set(ptr::invalid_mut(POISONED));
    }
}
```

## 线程管理分析
RUST的线程主要由以下几部分组成：
1. 线程属性设置，创建管理，join，是操作系统系统调用的RUST延伸
2. 线程局部存储, 是操作系统系统调用的RUST延伸
3. 线程panic管理, 是RUST的异常处理方案一部分
4. RUST运行时,是RUST语言自身的特性 
5. 为在线程中借用环境变量的Scope方案，是RUST语言自身的特性
   
### 操作系统相关的线程代码分析
均以linux为例，wasi与linux基本相同
```rust
// Thread结构，pthread函数需要用此结构作为参数调用pthread的API
// 这里 id没有象fd那样实现RawFd, OwnedFd, BorrowedFd，估计主要是因为所有权不象fd那样特别重要
pub struct Thread {
    id: libc::pthread_t,
}

// Thread当然应该支持Send 及 Sync
unsafe impl Send for Thread {}
unsafe impl Sync for Thread {}

impl Thread {
    // 熟悉C语言的会发现这个函数很容易理解，基本和C可以映射。大量的libc的调用直接导致用unsafe
    // 标记这个函数 
    pub unsafe fn new(stack: usize, p: Box<dyn FnOnce()>) -> io::Result<Thread> {
        //申请一个堆内存存放Box<dyn FnOnce()>, 并解封，将p直接置为申请的堆内存地址
        // 实质就是一个C的指针
        let p = Box::into_raw(box p);
        //等于pthread_t native = 0;
        let mut native: libc::pthread_t = mem::zeroed();
        let mut attr: libc::pthread_attr_t = mem::zeroed();
        //pthread_attr_init出错是有可能的，此处标准库偷了懒
        assert_eq!(libc::pthread_attr_init(&mut attr), 0);

        //对线程栈进行设置，一般不必设置
        {
            //线程栈不能小于允许最小的栈，实际上最大栈也应该有限制
            let stack_size = cmp::max(stack, min_stack_size(&attr));

            //设置线程栈大小
            match libc::pthread_attr_setstacksize(&mut attr, stack_size) {
                0 => {}
                n => {
                    assert_eq!(n, libc::EINVAL);
                    // 仅在参数不是内存页整数倍的情况会执行下面代码,重新调整栈空间,并设置，
                    // 此时的设置不应该再出错
                    let page_size = os::page_size();
                    let stack_size =
                        (stack_size + page_size - 1) & (-(page_size as isize - 1) as usize - 1);
                    assert_eq!(libc::pthread_attr_setstacksize(&mut attr, stack_size), 0);
                }
            };
        }

        //创建线程，thread_start是线程主函数，见后面分析
        //输入的闭包p作为thread_start的参数，attr当前只处理栈大小，成功后native会被赋值
        let ret = libc::pthread_create(&mut native, &attr, thread_start, p as *mut _);
        // attr任务完成，释放其申请的资源，C编程的时候这一步经常被忽略，这也是RUST的一个安全体现
        // 此处RUST认为不会失败 
        assert_eq!(libc::pthread_attr_destroy(&mut attr), 0);

        return if ret != 0 {
            // 失败
            // 重新建立Box以便释放申请的堆内存
            drop(Box::from_raw(p));
            //获取操作系统的错误
            Err(io::Error::from_raw_os_error(ret))
        } else {
            //成功，创建Thread返回
            Ok(Thread { id: native })
        };

        //所有RUST线程的主函数
        extern "C" fn thread_start(main: *mut libc::c_void) -> *mut libc::c_void {
            unsafe {
                // 这个是线程栈保护机制，如果线程出现栈溢出，可以用这个机制探测到， 
                // 这个是C语言编写大的服务器应用如数据库等积累下来的经验
                // 对C来说必要性是很大的，具体的代码分析略
                let _handler = stack_overflow::Handler::new();
                // 先将传入的堆内存重组为Box，然后自动解双层引用消费掉两个Box并运行真正的
                //线程函数 
                Box::from_raw(main as *mut Box<dyn FnOnce()>)();
            }
            //C函数的返回值
            ptr::null_mut()
        }
    }

    pub fn yield_now() {
        //让出CPU
        let ret = unsafe { libc::sched_yield() };
        debug_assert_eq!(ret, 0);
    }

    pub fn set_name(name: &CStr) {
        const PR_SET_NAME: libc::c_int = 15;
        // 更改线程名字，具体的系统调用请参考libc库 
        unsafe {
            libc::prctl(
                PR_SET_NAME,
                name.as_ptr(),
                0 as libc::c_ulong,
                0 as libc::c_ulong,
                0 as libc::c_ulong,
            );
        }
    }

    //线程睡眠,可以认为是glibc的sleep函数的RUST版本
    pub fn sleep(dur: Duration) {
        let mut secs = dur.as_secs();
        let mut nsecs = dur.subsec_nanos() as _;

        unsafe {
            //一次睡不到位就多睡几次, 另外也可能被中途打断，那需要再接着睡，
            // 以下这段代码值得注意，nanosleep让以前简单的调用一次usleep的我深感惭愧
            while secs > 0 || nsecs > 0 {
                //准备C语言时间变量
                let mut ts = libc::timespec {
                    tv_sec: cmp::min(libc::time_t::MAX as u64, secs) as libc::time_t,
                    tv_nsec: nsecs,
                };
                secs -= ts.tv_sec as u64;
                let ts_ptr = &mut ts as *mut _;
                if libc::nanosleep(ts_ptr, ts_ptr) == -1 {
                    assert_eq!(os::errno(), libc::EINTR);
                    //中途被打断的话，ts_ptr会放置还剩余的时间
                    //因此要把时间重新加入，下一次循环再睡
                    //真的是容易被忽视的返回值
                    secs += ts.tv_sec as u64;
                    nsecs = ts.tv_nsec;
                } else {
                    //重新置值
                    nsecs = 0;
                }
            }
        }
    }

    //等待目标线程结束
    pub fn join(self) {
        unsafe {
            let ret = libc::pthread_join(self.id, ptr::null_mut());
            //默认是非join，此处的作用是取消drop导致的默认pthread_detach调用
            mem::forget(self);
            assert!(ret == 0, "failed to join thread: {}", io::Error::from_raw_os_error(ret));
        }
    }

    pub fn id(&self) -> libc::pthread_t {
        //此处所有权没有转移。一般用于做pthread的系统调用临时使用
        //但不能用这个返回的id调用pthread_detach或者pthread_join及类似功能的pthread
        //C函数
        self.id
    }

    pub fn into_id(self) -> libc::pthread_t {
        let id = self.id;
        //pthread_detach应该由调用此函数的代码负责。
        //所有权已经转移
        mem::forget(self);
        id
    }
}

impl Drop for Thread {
    //主要用于父线程不必等待子线程结束的情况，默认为不等待
    fn drop(&mut self) {
        let ret = unsafe { libc::pthread_detach(self.id) };
        debug_assert_eq!(ret, 0);
    }
}
```

RUST在操作系统线程的基础上，实现了线程栈内存的溢出检查:
```rust
//线程栈守卫
pub mod guard {
    // 内存页大小
    static PAGE_SIZE: AtomicUsize = AtomicUsize::new(0);

    pub type Guard = Range<usize>;

    //获取线程栈栈底(栈溢出)内存地址,实际上可以看做是一个C函数，
    // 请参考具体的C语言pthread库说明
    unsafe fn get_stack_start() -> Option<*mut libc::c_void> {
        let mut ret = None;
        let mut attr: libc::pthread_attr_t = crate::mem::zeroed();
        let e = libc::pthread_getattr_np(libc::pthread_self(), &mut attr);
        if e == 0 {
            let mut stackaddr = crate::ptr::null_mut();
            let mut stacksize = 0;
            assert_eq!(libc::pthread_attr_getstack(&attr, &mut stackaddr, &mut stacksize), 0);
            ret = Some(stackaddr);
        }
        if e == 0 || cfg!(target_os = "freebsd") {
            assert_eq!(libc::pthread_attr_destroy(&mut attr), 0);
        }
        ret
    }

    // 获取与线程栈栈底(栈溢出)与内存页对齐的地址
    unsafe fn get_stack_start_aligned() -> Option<*mut libc::c_void> {
        let page_size = PAGE_SIZE.load(Ordering::Relaxed);
        assert!(page_size != 0);
        let stackptr = get_stack_start()?;
        let stackaddr = stackptr.addr();

        // 以下计算从栈底向栈顶方向找到第一个对齐地址
        let remainder = stackaddr % page_size;
        Some(if remainder == 0 {
            stackptr
        } else {
            stackptr.with_addr(stackaddr + page_size - remainder)
        })
    }

    pub unsafe fn init() -> Option<Guard> {
        //获得内存页大小
        let page_size = os::page_size();
        PAGE_SIZE.store(page_size, Ordering::Relaxed);

        {
            // Linux 内核已经做了栈守护，所以使用内核的机制
            let stackptr = get_stack_start_aligned()?;
            let stackaddr = stackptr.addr();
            Some(stackaddr - page_size..stackaddr)
        } 
    }
    //获取当前栈守护地址
    pub unsafe fn current() -> Option<Guard> {
        let mut ret = None;
        let mut attr: libc::pthread_attr_t = crate::mem::zeroed();
        let e = libc::pthread_getattr_np(libc::pthread_self(), &mut attr);
        if e == 0 {
            let mut guardsize = 0;
            assert_eq!(libc::pthread_attr_getguardsize(&attr, &mut guardsize), 0);
            if guardsize == 0 {
                    panic!("there is no guard page");
            }
            let mut stackptr = crate::ptr::null_mut::<libc::c_void>();
            let mut size = 0;
            assert_eq!(libc::pthread_attr_getstack(&attr, &mut stackptr, &mut size), 0);

            let stackaddr = stackptr.addr();
            let ret = {
                Some(stackaddr - guardsize..stackaddr + guardsize)
            }
        }
        ret
    }
}

fn min_stack_size(attr: *const libc::pthread_attr_t) -> usize {
    //用动态链接获取库函数
    dlsym!(fn __pthread_get_minstack(*const libc::pthread_attr_t) -> libc::size_t);

    match __pthread_get_minstack.get() {
        None => libc::PTHREAD_STACK_MIN,
        Some(f) => unsafe { f(attr) },
    }
}

// 专用于处理栈溢出的结构及实现
pub struct Handler {
    data: *mut libc::c_void,
}

impl Handler {
    pub unsafe fn new() -> Handler {
        //主要用于完成溢出处理函数的栈处置
        make_handler()
    }

    fn null() -> Handler {
        Handler { data: crate::ptr::null_mut() }
    }
}

impl Drop for Handler {
    fn drop(&mut self) {
        unsafe {
            drop_handler(self.data);
        }
    }
}

//stack_overflow模块
mod imp {
    // 对SIGSEGV及SIGBUS的信号处理函数。这两个函数会在线程出现栈溢出时被触发
    unsafe extern "C" fn signal_handler(
        signum: libc::c_int,
        info: *mut libc::siginfo_t,
        _data: *mut libc::c_void,
    ) {
        let guard = thread_info::stack_guard().unwrap_or(0..0);
        let addr = (*info).si_addr() as usize;

        // 判断是否访问了栈保护端的地址，如果是，则输出告警信息
        if guard.start <= addr && addr < guard.end {
            rtprintpanic!(
                "\nthread '{}' has overflowed its stack\n",
                thread::current().name().unwrap_or("<unknown>")
            );
            rtabort!("stack overflow");
        } else {
            // 否则执行默认操作.
            let mut action: sigaction = mem::zeroed();
            action.sa_sigaction = SIG_DFL;
            sigaction(signum, &action, ptr::null_mut());
        }
    }

    static MAIN_ALTSTACK: AtomicPtr<libc::c_void> = AtomicPtr::new(ptr::null_mut());
    static NEED_ALTSTACK: AtomicBool = AtomicBool::new(false);

    //初始化信号函数注册，此函数似乎,在sys::init中被调用
    pub unsafe fn init() {
        let mut action: sigaction = mem::zeroed();
        for &signal in &[SIGSEGV, SIGBUS] {
            sigaction(signal, ptr::null_mut(), &mut action);
            // 配置保护内存访问的信号处理函数.
            if action.sa_sigaction == SIG_DFL {
                action.sa_flags = SA_SIGINFO | SA_ONSTACK;
                action.sa_sigaction = signal_handler as sighandler_t;
                sigaction(signal, &action, ptr::null_mut());
                NEED_ALTSTACK.store(true, Ordering::Relaxed);
            }
        }

        let handler = make_handler();
        MAIN_ALTSTACK.store(handler.data, Ordering::Relaxed);
        mem::forget(handler);
    }

    pub unsafe fn cleanup() {
        drop_handler(MAIN_ALTSTACK.load(Ordering::Relaxed));
    }

    //下面这段函数是将段保护的内存设置成用户态写入会触发缺页中断，从而触发信号
    unsafe fn get_stackp() -> *mut libc::c_void {
        let flags = MAP_PRIVATE | MAP_ANON | libc::MAP_STACK;
        //mmap一段内存作为信号处理函数的栈，额外一个page用作保护
        let stackp =
            mmap(ptr::null_mut(), SIGSTKSZ + page_size(), PROT_READ | PROT_WRITE, flags, -1, 0);
        if stackp == MAP_FAILED {
            panic!("failed to allocate an alternative stack: {}", io::Error::last_os_error());
        }
        // 最低的一个page用来作为保护
        let guard_result = libc::mprotect(stackp, page_size(), PROT_NONE);
        if guard_result != 0 {
            panic!("failed to set up alternative stack guard page: {}", io::Error::last_os_error());
        }
        // 真正的栈从底部向上一个page开始
        stackp.add(page_size())
    }

    unsafe fn get_stack() -> libc::stack_t {
        libc::stack_t { ss_sp: get_stackp(), ss_flags: 0, ss_size: SIGSTKSZ }
    }

    //用于对每个线程设置线程信号处理的栈
    pub unsafe fn make_handler() -> Handler {
        if !NEED_ALTSTACK.load(Ordering::Relaxed) {
            return Handler::null();
        }
        let mut stack = mem::zeroed();
        sigaltstack(ptr::null(), &mut stack);
        // 设置信号处理函数的栈 
        if stack.ss_flags & SS_DISABLE != 0 {
            //设置用于信号处理的栈
            stack = get_stack();
            // 设置栈，长度为SIGSTKSZ
            sigaltstack(&stack, ptr::null_mut());
            Handler { data: stack.ss_sp as *mut libc::c_void }
        } else {
            Handler::null()
        }
    }

    pub unsafe fn drop_handler(data: *mut libc::c_void) {
        if !data.is_null() {
            let stack = libc::stack_t {
                ss_sp: ptr::null_mut(),
                ss_flags: SS_DISABLE,
                ss_size: SIGSTKSZ,
            };
            //删除信号处理函数专用栈
            sigaltstack(&stack, ptr::null_mut());
            // unmap用于信号处理的内存.
            munmap(data.sub(page_size()), SIGSTKSZ + page_size());
        }
    }
}

```

线程的本地全局变量Thread Local Key的实现。线程的本地存储解决一类问题如下：
在线程代码中，有时希望多个线程共享同一个变量名的全局变量，以简化编码。但希望这个变量在不同的线程有各自的拷贝，彼此不影响。典型的例子就是前文的线程栈guard空间。如果每个线程都共享同一个变量名，那代码会少很多啰嗦。
具体的代码分析如下:    
```rust
//pthread_key_t请参考libc的pthread编程手册
pub type Key = libc::pthread_key_t;

//dtor用于对创建的key做释放操作, 返回的Key可以被进程中的线程共享使用
pub unsafe fn create(dtor: Option<unsafe extern "C" fn(*mut u8)>) -> Key {
    let mut key = 0;
    assert_eq!(libc::pthread_key_create(&mut key, mem::transmute(dtor)), 0);
    key
}

// 各线程可以将key设置成自己需要的内存块，这个内存块的所有权属于Key，
// 这是个代码规定，编译器不知道，所以安全上需要程序员负责
pub unsafe fn set(key: Key, value: *mut u8) {
    let r = libc::pthread_setspecific(key, value as *mut _);
    debug_assert_eq!(r, 0);
}

// 用key将内存块获得，实际上是获得一个引用
pub unsafe fn get(key: Key) -> *mut u8 {
    libc::pthread_getspecific(key) as *mut u8
}

// 删除掉key，需要所有线程都删除，调用此函数会导致调用内存块的dtor函数，也即drop
pub unsafe fn destroy(key: Key) {
    let r = libc::pthread_key_delete(key);
    debug_assert_eq!(r, 0);
}

```
### 标准库线程支持层代码分析
代码路径：library/std/src/sys_common/thread.rs    
         library/std/src/sys_common/thread_local.rs
         library/std/src/sys_common/thread_info.rs
    
在操作系统的线程概念与RUST作为API提供的线程之间的一层代码。主要处理一些RUST的语法导致的一些需要额外在操作系统的线程结构做一些包装的基础层。   

对Thread Local Key做类型封装，以屏蔽不同操作系统除API外的差异。    
代码如下：
```rust
//适用与作为静态变量的Thread Local Key结构
pub struct StaticKey {
    /// 仅仅是一个数值，为0的时候代表此时无意义
    key: AtomicUsize,
    ///对key的析构函数 
    dtor: Option<unsafe extern "C" fn(*mut u8)>,
}

//一般作为StaticKey的初始化赋值
pub const INIT: StaticKey = StaticKey::new(None);

impl StaticKey {
    pub const fn new(dtor: Option<unsafe extern "C" fn(*mut u8)>) -> StaticKey {
        //key为0，代表此时没有创建thread local key
        StaticKey { key: atomic::AtomicUsize::new(0), dtor }
    }

    //如果没有创建thread local key, 此方法会创建一个
    pub unsafe fn get(&self) -> *mut u8 {
        //获取key的指针
        //调用self.key会在无thread local key时创建一个
        imp::get(self.key())
    }

    //如果没有创建thread local key, 此方法会创建一个
    pub unsafe fn set(&self, val: *mut u8) {
        imp::set(self.key(), val)
    }

    //获得thread local key的key值
    unsafe fn key(&self) -> imp::Key {
        match self.key.load(Ordering::Relaxed) {
            //如果为0，表示thread local key没有创建，需要创建一个
            0 => self.lazy_init() as imp::Key,
            //不为0，则返回key
            n => n as imp::Key,
        }
    }

    //创建一个thread local key
    unsafe fn lazy_init(&self) -> usize {
        // 为特殊的操作系统准备
        if imp::requires_synchronized_create() {
            // 需要加锁保护，因为保护静态变量，所以要使用StaticMutex
            // INIT_LOCK所有线程共享
            static INIT_LOCK: StaticMutex = StaticMutex::new();
            let _guard = INIT_LOCK.lock();
            let mut key = self.key.load(Ordering::SeqCst);
            if key == 0 {
                //创建Key
                key = imp::create(self.dtor) as usize;
                self.key.store(key, Ordering::SeqCst);
            }
            rtassert!(key != 0);
            return key;
            //_guard生命周期结束会释放INIT_LOCK
        }

        //unix系统有可能分配为0的thread local key
        let key1 = imp::create(self.dtor);
        let key = if key1 != 0 {
            key1
        } else {
            //如果是0，需要重新再申请一个新的key
            let key2 = imp::create(self.dtor);
            imp::destroy(key1);
            key2
        };
        rtassert!(key != 0);
        match self.key.compare_exchange(0, key as usize, Ordering::SeqCst, Ordering::SeqCst) {
            //这里也作为方法的返回
            Ok(_) => key as usize,
            // 如果有其他的值，那就用那个值， 
            Err(n) => {
                imp::destroy(key);
                n
            }
        }
    }
}

//非静态变量的Thread Local Key
pub struct Key {
    key: imp::Key,
}

impl Key {
    // 创建一个thread local key
    pub fn new(dtor: Option<unsafe extern "C" fn(*mut u8)>) -> Key {
        Key { key: unsafe { imp::create(dtor) } }
    }

    // 获取key相关的内存，可能为空，
    pub fn get(&self) -> *mut u8 {
        unsafe { imp::get(self.key) }
    }

    //设置key相关的内存
    pub fn set(&self, val: *mut u8) {
        unsafe { imp::set(self.key, val) }
    }
}

impl Drop for Key {
    fn drop(&mut self) {
        // Right now Windows doesn't support TLS key destruction, but this also
        // isn't used anywhere other than tests, so just leak the TLS key.
        // unsafe { imp::destroy(self.key) }
    }
}

```
### 标准库线程局部变量外部接口(Thread Local)
操作系统的Thread Local Key使用起来明显非常繁琐，且很容易出错。RUST标准库对其进行了符合`rust`的类型封装。这一类型需要完成的工作如下：   
1. 对所有的Thread Local Key存放的真正的数据类型需要声明key时做定义。
2. 向使用者屏蔽key的创建及销毁过程，key与真实数据的捆绑与获取过程，使得key的使用类似于通用类型结构的使用。   

RUST采用了宏及数据类型相结合的方案，下面的代码说明了RUST的local key的使用。
```rust
 use std::cell::RefCell;
 thread_local! {
     pub static FOO: RefCell<u32> = RefCell::new(1);

     static BAR: RefCell<f32> = RefCell::new(1.0);
 }
 fn main() {FOO.with(|f|{*f.borrow_mut() = 2})}
```
以上的`thread_local`将一个普通的变量定义转换为Thread Local Key的变量. 并且可以在随后的with方法内可以正常的的使用该普通变量。用这种方法，RUST使得Thread Local Key的变量使用与普通变量基本上做到了相一致。 (其他语言多用set(),get()方法完成Thread local的操作，RUST采用了更近一步的设计)      
具体的代码实现如下：   
```rust
//LocalKey只能用于静态变量
pub struct LocalKey<T: 'static> {
    //只能用于静态变量
    //inner是一个支持泛型的函数类型
    inner: unsafe fn(Option<&mut Option<T>>) -> Option<&'static T>,
}

pub struct AccessError;

impl Error for AccessError {}

impl<T: 'static> LocalKey<T> {
    // 这里仅仅做一个内存占位
    pub const unsafe fn new(
        inner: unsafe fn(Option<&mut Option<T>>) -> Option<&'static T>,
    ) -> LocalKey<T> {
        LocalKey { inner }
    }

    //所有的Thread Local的操作都在witch参数的闭包中，
    // with会将Thread Local的可变引用输入闭包的参数
    pub fn with<F, R>(&'static self, f: F) -> R
    where
        F: FnOnce(&T) -> R,
    {
        self.try_with(f).expect(
            "cannot access a Thread Local Storage value \
             during or after destruction",
        )
    }

    pub fn try_with<F, R>(&'static self, f: F) -> Result<R, AccessError>
    where
        F: FnOnce(&T) -> R,
    {
        unsafe {
            //获取Thread Local内存指针后，调用闭包
            let thread_local = (self.inner)(None).ok_or(AccessError)?;
            Ok(f(thread_local))
        }
    }

    //对Thread Local做初始化，然后再执行操作
    fn initialize_with<F, R>(&'static self, init: T, f: F) -> R
    where
        F: FnOnce(Option<T>, &T) -> R,
    {
        unsafe {
            let mut init = Some(init);
            let reference = (self.inner)(Some(&mut init)).expect(
                "cannot access a Thread Local Storage value \
                 during or after destruction",
            );
            f(init, reference)
        }
    }
}

//LocalKey通常会与内部可变性变量配合,设计方法来简化使用者的代码
impl<T: 'static> LocalKey<Cell<T>> {
    //对内部可变性变量赋值
    pub fn set(&'static self, value: T) {
        self.initialize_with(Cell::new(value), |value, cell| {
            if let Some(value) = value {
                // value输入的Cell变量参数，cell是Thread Local的引用
                // 对cell做出更新,并消费掉value.
                cell.set(value.into_inner());
            }
        });
    }

    //只能在T实现Copy trait的情况下支持，否则会出现
    //双份所有权
    pub fn get(&'static self) -> T
    where
        T: Copy,
    {
        self.with(|cell| cell.get())
    }

    //获取Thread Local的变量所有权，并将Thread Local置为默认
    pub fn take(&'static self) -> T
    where
        T: Default,
    {
        self.with(|cell| cell.take())
    }

    //替换
    pub fn replace(&'static self, value: T) -> T {
        self.with(|cell| cell.replace(value))
    }
}

//提供Thread Local是内部可变性的基础
impl<T: 'static> LocalKey<RefCell<T>> {
    //borrow的对应简化
    pub fn with_borrow<F, R>(&'static self, f: F) -> R
    where
        F: FnOnce(&T) -> R,
    {
        self.with(|cell| f(&cell.borrow()))
    }

    //borrow_mut的对应简化
    pub fn with_borrow_mut<F, R>(&'static self, f: F) -> R
    where
        F: FnOnce(&mut T) -> R,
    {
        self.with(|cell| f(&mut cell.borrow_mut()))
    }

    //修改值
    pub fn set(&'static self, value: T) {
        self.initialize_with(RefCell::new(value), |value, cell| {
            if let Some(value) = value {
                *cell.borrow_mut() = value.into_inner();
            }
        });
    }

    //获取所有权
    pub fn take(&'static self) -> T
    where
        T: Default,
    {
        self.with(|cell| cell.take())
    }

    //替换
    pub fn replace(&'static self, value: T) -> T {
        self.with(|cell| cell.replace(value))
    }
}

/// 惰性初始化.
mod lazy {
    use crate::cell::UnsafeCell;
    use crate::hint;
    use crate::mem;

    pub struct LazyKeyInner<T> {
        //None作为未初始化的标志
        inner: UnsafeCell<Option<T>>,
    }

    impl<T> LazyKeyInner<T> {
        //new一个未初始化变量
        pub const fn new() -> LazyKeyInner<T> {
            LazyKeyInner { inner: UnsafeCell::new(None) }
        }

        pub unsafe fn get(&self) -> Option<&'static T> {
            // 返回内部变量的引用
            unsafe { (*self.inner.get()).as_ref() }
        }

        // 真正的初始化
        pub unsafe fn initialize<F: FnOnce() -> T>(&self, init: F) -> &'static T {
            let value = init();
            let ptr = self.inner.get();

            // 如果用*ptr = Some(value)，会导致编译器对上一个变量做drop处理,对Thread Local
            // 的drop实际上有些复杂，所以此处用一个replace
            unsafe {
                let _ = mem::replace(&mut *ptr, Some(value));
            }

            unsafe {
                // 返回Some内变量的引用.
                match *ptr {
                    Some(ref x) => x,
                    None => hint::unreachable_unchecked(),
                }
            }
        }

        //take语义
        pub unsafe fn take(&mut self) -> Option<T> {
            unsafe { (*self.inner.get()).take() }
        }
    }
}

//利用llvm的Thread Local方案，不直接使用操作系统系统调用的thread local key
pub mod fast {
    use super::lazy::LazyKeyInner;
    use crate::cell::Cell;
    use crate::fmt;
    use crate::mem;
    use crate::sys::thread_local_dtor::register_dtor;

    #[derive(Copy, Clone)]
    enum DtorState {
        //没有初始化
        Unregistered,
        //已经初始化完成
        Registered,
        //Local Key已经被destroy
        RunningOrHasRun,
    }

    pub struct Key<T> {
        //  放置key存储的变量, None表示变量没有初始化。与dtor_state配合完成对
        //  Key的状态判断
        inner: LazyKeyInner<T>,

        // Local Key的destroy函数状态 
        dtor_state: Cell<DtorState>,
    }


    impl<T> Key<T> {
        pub const fn new() -> Key<T> {
            //实际做内存占位
            Key { inner: LazyKeyInner::new(), dtor_state: Cell::new(DtorState::Unregistered) }
        }

        // 证明仍然使用操作系统的thread local key的destory机制 
        pub unsafe fn register_dtor(a: *mut u8, dtor: unsafe extern "C" fn(*mut u8)) {
            unsafe {
                register_dtor(a, dtor);
            }
        }

        pub unsafe fn get<F: FnOnce() -> T>(&self, init: F) -> Option<&'static T> {
            // 如果已经初始化，则取用值
            // 如果没有初始化，则进行初始化
            unsafe {
                match self.inner.get() {
                    Some(val) => Some(val),
                    None => self.try_initialize(init),
                }
            }
        }

        #[inline(never)]
        unsafe fn try_initialize<F: FnOnce() -> T>(&self, init: F) -> Option<&'static T> {
            if !mem::needs_drop::<T>() || unsafe { self.try_register_dtor() } {
                // 只用变量不需要drop，或者注册destroy函数成功的情况下才做初始化.
                Some(unsafe { self.inner.initialize(init) })
            } else {
                None
            }
        }

        unsafe fn try_register_dtor(&self) -> bool {
            match self.dtor_state.get() {
                DtorState::Unregistered => {
                    // 注册destroy函数
                    unsafe { register_dtor(self as *const _ as *mut u8, destroy_value::<T>) };
                    self.dtor_state.set(DtorState::Registered);
                    true
                }
                DtorState::Registered => {
                    // 被递归初始化
                    true
                }
                DtorState::RunningOrHasRun => false,
            }
        }
    }

    unsafe extern "C" fn destroy_value<T>(ptr: *mut u8) {
        let ptr = ptr as *mut Key<T>;

        unsafe {
            //将变量所有权获得
            let value = (*ptr).inner.take();
            //设置destroy状态
            (*ptr).dtor_state.set(DtorState::RunningOrHasRun);
            //对变量做drop
            drop(value);
        }
    }
}

//利用操作系统的StaticKey
pub mod os {
    use super::lazy::LazyKeyInner;
    use crate::cell::Cell;
    use crate::fmt;
    use crate::marker;
    use crate::ptr;
    use crate::sys_common::thread_local_key::StaticKey as OsStaticKey;

    pub struct Key<T> {
        // 操作系统的静态Key.
        os: OsStaticKey,
        //指示本结构有一个Cell<T>的所有权
        marker: marker::PhantomData<Cell<T>>,
    }

    unsafe impl<T> Sync for Key<T> {}

    struct Value<T: 'static> {
        inner: LazyKeyInner<T>,
        key: &'static Key<T>,
    }

    impl<T: 'static> Key<T> {
        pub const fn new() -> Key<T> {
            //创建一个StaticKey
            Key { os: OsStaticKey::new(Some(destroy_value::<T>)), marker: marker::PhantomData }
        }

        pub unsafe fn get(&'static self, init: impl FnOnce() -> T) -> Option<&'static T> {
            // thread local key的get操作.
            let ptr = unsafe { self.os.get() as *mut Value<T> };
            if ptr.addr() > 1 {
                //有值
                if let Some(ref value) = unsafe { (*ptr).inner.get() } {
                    return Some(value);
                }
            }
            // 进行初始化.
            unsafe { self.try_initialize(init) }
        }

        unsafe fn try_initialize(&'static self, init: impl FnOnce() -> T) -> Option<&'static T> {
            let ptr = unsafe { self.os.get() as *mut Value<T> };
            if ptr.addr() == 1 {
                // 被destroy了
                return None;
            }

            let ptr = if ptr.is_null() {
                // 从堆上申请内存.
                let ptr: Box<Value<T>> = box Value { inner: LazyKeyInner::new(), key: self };
                //获取申请的堆内存地址
                let ptr = Box::into_raw(ptr);
                // 将地址与操作系统的key相关联
                unsafe {
                    self.os.set(ptr as *mut u8);
                }
                ptr
            } else {
                // 递归初始化，返回已有的ptr 
                ptr
            };

            // 初始化变量.
            unsafe { Some((*ptr).inner.initialize(init)) }
        }
    }

    unsafe extern "C" fn destroy_value<T: 'static>(ptr: *mut u8) {
        unsafe {
            //恢复Box以便释放堆内存
            let ptr = Box::from_raw(ptr as *mut Value<T>);
            let key = ptr.key;
            //将thread local key设置为1
            key.os.set(ptr::invalid_mut(1));
            //释放Box
            drop(ptr);
            // key可以重新用于与新的内存相关
            key.os.set(ptr::null_mut());
        }
    }
}
pub use self::local::fast::Key as __FastLocalKeyInner;
pub use self::local::os::Key as __OsLocalKeyInner;

// Thread Local声明宏
macro_rules! thread_local {
    // empty (base case for the recursion)
    () => {};

    // init 是一个const 修饰的block
    ($(#[$attr:meta])* $vis:vis static $name:ident: $t:ty = const { $init:expr }; $($rest:tt)*) => (
        $crate::__thread_local_inner!($(#[$attr])* $vis $name, $t, const $init);
        $crate::thread_local!($($rest)*);
    );

    ($(#[$attr:meta])* $vis:vis static $name:ident: $t:ty = const { $init:expr }) => (
        $crate::__thread_local_inner!($(#[$attr])* $vis $name, $t, const $init);
    );

    //  init不是block
    ($(#[$attr:meta])* $vis:vis static $name:ident: $t:ty = $init:expr; $($rest:tt)*) => (
        $crate::__thread_local_inner!($(#[$attr])* $vis $name, $t, $init);
        $crate::thread_local!($($rest)*);
    );

    ($(#[$attr:meta])* $vis:vis static $name:ident: $t:ty = $init:expr) => (
        $crate::__thread_local_inner!($(#[$attr])* $vis $name, $t, $init);
    );
}

macro_rules! __thread_local_inner {
    ($(#[$attr:meta])* $vis:vis $name:ident, $t:ty, $($init:tt)*) => {
        //定义了一个LocalKey的变量
        $(#[$attr])* $vis const $name: $crate::thread::LocalKey<$t> =
            $crate::__thread_local_inner!(@key $t, $($init)*);
    }
    //对const init的处理
    (@key $t:ty, const $init:expr) => {{
        //LocalKey的创建函数
        unsafe fn __getit(
            _init: $crate::option::Option<&mut $crate::option::Option<$t>>,
        ) -> $crate::option::Option<&'static $t> {
            //不可变变量定义
            const INIT_EXPR: $t = $init;

            {
                #[thread_local]
                //静态全局变量, 应用llvm的Thread Local Storage, 需要操作系统支持
                static mut VAL: $t = INIT_EXPR;

                // 判断是否需要有drop函数
                if !$crate::mem::needs_drop::<$t>() {
                    //如果不需要drop，返回VAL的引用即可
                    unsafe {
                        return $crate::option::Option::Some(&VAL)
                    }
                }

                // 0 == dtor not registered
                // 1 == dtor registered, dtor not run
                // 2 == dtor registered and is running or has run
                #[thread_local]
                //释放函数注册状态
                static mut STATE: $crate::primitive::u8 = 0;

                //释放函数
                unsafe extern "C" fn destroy(ptr: *mut $crate::primitive::u8) {
                    let ptr = ptr as *mut $t;

                    unsafe {
                        $crate::debug_assert_eq!(STATE, 1);
                        STATE = 2;
                        $crate::ptr::drop_in_place(ptr);
                    }
                }

                unsafe {
                    match STATE {
                        // 0 == 需要注册释放函数.
                        0 => {
                            //fast::Key::register_dtor，见下文分析
                            $crate::thread::__FastLocalKeyInner::<$t>::register_dtor(
                                $crate::ptr::addr_of_mut!(VAL) as *mut $crate::primitive::u8,
                                destroy,
                            );
                            STATE = 1;
                            $crate::option::Option::Some(&VAL)
                        }
                        // 1 == 释放函数已经注册，直接返回Key
                        1 => $crate::option::Option::Some(&VAL),
                        // 释放函数已经运行，返回.
                        _ => $crate::option::Option::None,
                    }
                }
            }
        }

        unsafe {
            //生成LocalKey
            $crate::thread::LocalKey::new(__getit)
        }
    }};

    // 非const的init的处理
    (@key $t:ty, $init:expr) => {
        {
            fn __init() -> $t { $init }

            //LocalKey的创建函数
            unsafe fn __getit(
                init: $crate::option::Option<&mut $crate::option::Option<$t>>,
            ) -> $crate::option::Option<&'static $t> {
                //利用llvm编译器属性定义变量为thread local变量
                #[thread_local]
                static __KEY: $crate::thread::__FastLocalKeyInner<$t> =
                    $crate::thread::__FastLocalKeyInner::new();


                //初始化 
                unsafe {
                    __KEY.get(move || {
                        if let $crate::option::Option::Some(init) = init {
                            if let $crate::option::Option::Some(value) = init.take() {
                                return value;
                            } else if $crate::cfg!(debug_assertions) {
                                $crate::unreachable!("missing default value");
                            }
                        }
                        __init()
                    })
                }
            }

            unsafe {
                $crate::thread::LocalKey::new(__getit)
            }
        }
    };
}
```
Thread Local的RUST标准库内容颇为复杂，但提供了非常方便的对外使用。

### Thread Info 分析
Thead Info利用Thread Local存储一些Thread的信息。
```rust
struct ThreadInfo {
    stack_guard: Option<Guard>,
    thread: Thread,
}

thread_local! { static THREAD_INFO: RefCell<Option<ThreadInfo>> = const { RefCell::new(None) } }

impl ThreadInfo {
    fn with<R, F>(f: F) -> Option<R>
    where
        F: FnOnce(&mut ThreadInfo) -> R,
    {
        THREAD_INFO
            .try_with(move |thread_info| {
                let mut thread_info = thread_info.borrow_mut();
                let thread_info = thread_info.get_or_insert_with(|| ThreadInfo {
                    stack_guard: None,
                    thread: Thread::new(None),
                });
                f(thread_info)
            })
            .ok()
    }
}

pub fn current_thread() -> Option<Thread> {
    ThreadInfo::with(|info| info.thread.clone())
}

pub fn stack_guard() -> Option<Guard> {
    ThreadInfo::with(|info| info.stack_guard.clone()).and_then(|o| o)
}

pub fn set(stack_guard: Option<Guard>, thread: Thread) {
    THREAD_INFO.with(move |thread_info| {
        let mut thread_info = thread_info.borrow_mut();
        rtassert!(thread_info.is_none());
        *thread_info = Some(ThreadInfo { stack_guard, thread });
    });
}

```
### 标准库线程外部接口分析
RUST在操作系统的线程支持之上，实现了语言自身的线程概念，首要的，就是为每一个线程分配一个ID做标识。
即ThreadId, 代码如下：   
```rust
//用一个非零的64位整数
pub struct ThreadId(NonZeroU64);

impl ThreadId {
    // 分配一个新的线程ID 
    fn new() -> ThreadId {
        // 线程ID是一个全局静态变量，且是一个临界区访问，此处只能用StaticMutex锁
        static GUARD: mutex::StaticMutex = mutex::StaticMutex::new();
        //声明静态变量,是全局的静态变量，但声明在这里限制对其的访问, 初始值为1
        static mut COUNTER: u64 = 1;

        unsafe {
            //防止竞争
            let guard = GUARD.lock();

            // 加入到达最大值，panic处理。这里默认一个进程不可能创建超过u64::MAX的线程.
            // 这个稍微有些不严谨，因为即使线程终止，ID也不能重用。可以认为，一个进程不可能运行到这个时间
            if COUNTER == u64::MAX {
                drop(guard); //panic之前显式drop，以避免影响其他线程 
                //错误处理
                panic!("failed to generate unique thread ID: bitspace exhausted");
            }

            //分配新线程ID
            let id = COUNTER;
            COUNTER += 1;

            ThreadId(NonZeroU64::new(id).unwrap())
            //guard生命周期结束，会调用drop()
        }
    }
    ...
}
```
Thread park的实现，Thread park是一种将线程自身陷入阻塞，等待别的线程做唤醒的机制。是一种比较简单的多个线程间的同步机制，通常用于线程指令执行过程中有顺序要求但不必临界区的情况
```rust
const PARKED: i32 = -1;
const EMPTY: i32 = 0;
const NOTIFIED: i32 = 1;

pub struct Parker {
    state: AtomicI32,
}

// Parker 利用原子变量操作中的内存顺序规则完成.
impl Parker {
    #[inline]
    pub const fn new() -> Self {
        Parker { state: AtomicI32::new(EMPTY) }
    }

    pub unsafe fn park(&self) {
        // 利用Acquire顺序获取当前状态 
        if self.state.fetch_sub(1, Acquire) == NOTIFIED {
            return;
        }
        loop {
            // 如果state是PARKED，阻塞等待 
            futex_wait(&self.state, PARKED, None);
            // 被唤醒，将状态重新置为EMPTY,并检测是否为NOTIFIED.
            if self.state.compare_exchange(NOTIFIED, EMPTY, Acquire, Acquire).is_ok() {
                return;
            } else {
                // 不是NOTIFIED，其他park的线程已经执行，再次循环.
            }
        }
    }

    // 超时.
    pub unsafe fn park_timeout(&self, timeout: Duration) {
        if self.state.fetch_sub(1, Acquire) == NOTIFIED {
            return;
        }
        // 直接等待，设置超时，此时不再循环，因为循环会导致超时不准.
        futex_wait(&self.state, PARKED, Some(timeout));
        if self.state.swap(EMPTY, Acquire) == NOTIFIED {
            // 被unpark()唤醒
        } else {
            // 超时或者其他唤醒.
        }
    }

    pub fn unpark(&self) {
        // 将state更换到NOTIFIED
        if self.state.swap(NOTIFIED, Release) == PARKED {
            //唤醒阻塞的线程
            futex_wake(&self.state);
        }
    }
}
```
RUST标准库的Thread的对外结构:
```rust
/// 事实上的Thread结构
struct Inner {
    name: Option<CString>, //需要与外部语言库交互 
    id: ThreadId,
    // 用于park
    parker: Parker,
}

//Thread的管理类型
#[derive(Clone)]
pub struct Thread {
    //需要被多个线程共享
    inner: Arc<Inner>,
}

impl Thread {
    // 仅创建一个结构用于管理，此时尚没有与线程相关联.
    pub(crate) fn new(name: Option<CString>) -> Thread {
        Thread { inner: Arc::new(Inner { name, id: ThreadId::new(), parker: Parker::new() }) }
    }

    //对thread做unpark操作，使得park的thread结束阻塞。
    pub fn unpark(&self) {
        self.inner.parker.unpark();
    }

    pub fn id(&self) -> ThreadId {
        self.inner.id
    }

    pub fn name(&self) -> Option<&str> {
        self.cname().map(|s| unsafe { str::from_utf8_unchecked(s.to_bytes()) })
    }

    fn cname(&self) -> Option<&CStr> {
        self.inner.name.as_deref()
    }
}
```
RUST的进程创建返回类型 JoinHandle：
```rust
struct JoinInner<'scope, T> {
    native: imp::Thread,
    thread: Thread,
    packet: Arc<Packet<'scope, T>>,
}

impl<'scope, T> JoinInner<'scope, T> {
    fn join(mut self) -> Result<T> {
        //等待线程退出
        self.native.join();
        //获取线程退出的结果或者异常信息
        Arc::get_mut(&mut self.packet).unwrap().result.get_mut().take().unwrap()
    }
}

//调用spawn后返回JoinHandle，JoinHandle作为线程外部对该线程操作的标识类型结构,如park，join等
pub struct JoinHandle<T>(JoinInner<'static, T>);

unsafe impl<T> Send for JoinHandle<T> {}
unsafe impl<T> Sync for JoinHandle<T> {}


impl<T> JoinHandle<T> {
    //获取线程的Thread结构引用
    pub fn thread(&self) -> &Thread {
        &self.0.thread
    }

    //等待线程结束
    pub fn join(self) -> Result<T> {
        self.0.join()
    }

    //判断线程是否已经终止
    pub fn is_finished(&self) -> bool {
        Arc::strong_count(&self.0.packet) == 1
    }
}

```
RUST线程创建工厂类型：
```rust
//用于非默认属性的线程创建
pub struct Builder {
    // 名字，线程默认没有名字
    name: Option<String>,
    // 线程堆栈大小，默认堆栈为2M bytes
    stack_size: Option<usize>,
}

// 线程创建方法实现
impl Builder {
    //创建一个默认的builder，一般的，用于需要对name及stack_size做修改
    pub fn new() -> Builder {
        Builder { name: None, stack_size: None }
    }

    //给线程设置名称，目前仅用于线程panic时的信息输出
    pub fn name(mut self, name: String) -> Builder {
        self.name = Some(name);
        self
    }

    //设置线程的堆栈空间
    pub fn stack_size(mut self, size: usize) -> Builder {
        self.stack_size = Some(size);
        self
    }

    // 利用Builder属性参数创建一个新线程。如果不是在主线程执行这个函数，
    // 新线程的生命周期可能长于创建它的线程，此时创建线程可以用JoinHandle来等待
    // 新线程结束。
    pub fn spawn<F, T>(self, f: F) -> io::Result<JoinHandle<T>>
    where
        F: FnOnce() -> T,
        F: Send + 'static,
        T: Send + 'static,
    {
        unsafe { self.spawn_unchecked(f) }
    }

    //不安全的spawn
    pub unsafe fn spawn_unchecked<'a, F, T>(self, f: F) -> io::Result<JoinHandle<T>>
    where
        F: FnOnce() -> T,
        F: Send + 'a,
        T: Send + 'a,
    {
        Ok(JoinHandle(unsafe { self.spawn_unchecked_(f, None) }?))
    }

    //真正的spawn执行函数，此处与进程的spawn函数有些类似
    unsafe fn spawn_unchecked_<'a, 'scope, F, T>(
        self,
        f: F,
        scope_data: Option<&'scope scoped::ScopeData>,
    ) -> io::Result<JoinInner<'scope, T>>
    where
        F: FnOnce() -> T,
        F: Send + 'a,
        T: Send + 'a,
        'scope: 'a,
    {
        let Builder { name, stack_size } = self;

        //不能小于规定的最小堆栈
        let stack_size = stack_size.unwrap_or_else(thread::min_stack);

        //创建一个Thread的变量
        let my_thread = Thread::new(name.map(|name| {
            CString::new(name).expect("thread name may not contain interior null bytes")
        }));
        //增加Arc计数，用于转移到创建的线程代码
        let their_thread = my_thread.clone();

        let my_packet: Arc<Packet<'scope, T>> =
            Arc::new(Packet { scope: scope_data, result: UnsafeCell::new(None) });
        //Arc计数增加，子线程写，父线程读
        let their_packet = my_packet.clone();

        //捕获panic输出的缓存空间设置,是Thread Local变量
        let output_capture = crate::io::set_output_capture(None);
        crate::io::set_output_capture(output_capture.clone());

        //所有线程的主函数, 可以认为是线程的runtime
        let main = move || {
            //their_thread已经转移到创建线程
            if let Some(name) = their_thread.cname() {
                //设置线程名字
                imp::Thread::set_name(name);
            }

            //设置本线程的panic捕获空间
            crate::io::set_output_capture(output_capture);

            // 完成thread_info的线程本地初始化.
            thread_info::set(unsafe { imp::guard::current() }, their_thread);
            // 执行f，如果f内部发生panic，调用栈会输出
            let try_result = panic::catch_unwind(panic::AssertUnwindSafe(|| {
                crate::sys_common::backtrace::__rust_begin_short_backtrace(f)
            }));
            //  将线程退出信息设置到their_packet中
            unsafe { *their_packet.result.get() = Some(try_result) };
        };

        if let Some(scope_data) = scope_data {
            scope_data.increment_num_running_threads();
        }

        //真正的创建线程
        Ok(JoinInner {
            //创建线程
            native: unsafe {
                imp::Thread::new(
                    stack_size,
                    mem::transmute::<Box<dyn FnOnce() + 'a>, Box<dyn FnOnce() + 'static>>(
                        Box::new(main),
                    ),
                )?
            },
            thread: my_thread,
            packet: my_packet,
        })
    }
}

//无须指定参数的简易线程启动函数
pub fn spawn<F, T>(f: F) -> JoinHandle<T>
where
    F: FnOnce() -> T,
    F: Send + 'static,
    T: Send + 'static,
{
    Builder::new().spawn(f).expect("failed to spawn thread")
}

//线程自身的结构变量获取
pub fn current() -> Thread {
    thread_info::current_thread().expect(
        "use of std::thread::current() is not possible \
         after the thread's local data has been destroyed",
    )
}

//出让CPU
pub fn yield_now() {
    imp::Thread::yield_now()
}

//本线程是否已经panic
pub fn panicking() -> bool {
    panicking::panicking()
}

//阻塞，等待其他线程唤醒
pub fn park() {
    unsafe {
        current().inner.parker.park();
    }
}

pub fn park_timeout(dur: Duration) {
    unsafe {
        current().inner.parker.park_timeout(dur);
    }
}

pub type Result<T> = crate::result::Result<T, Box<dyn Any + Send + 'static>>;

// 用来获取线程的退出值
// 需要在线程间共享
struct Packet<'scope, T> {
    scope: Option<&'scope scoped::ScopeData>,
    result: UnsafeCell<Option<Result<T>>>,
}

// 使用了UnsafeCell， 需要声明实现Sync
unsafe impl<'scope, T: Sync> Sync for Packet<'scope, T> {}

impl<'scope, T> Drop for Packet<'scope, T> {
    fn drop(&mut self) {
        // If this packet was for a thread that ran in a scope, the thread
        // panicked, and nobody consumed the panic payload, we make sure
        // the scope function will panic.
        let unhandled_panic = matches!(self.result.get_mut(), Some(Err(_)));
        // Drop the result without causing unwinding.
        // This is only relevant for threads that aren't join()ed, as
        // join() will take the `result` and set it to None, such that
        // there is nothing left to drop here.
        // If this panics, we should handle that, because we're outside the
        // outermost `catch_unwind` of our thread.
        // We just abort in that case, since there's nothing else we can do.
        // (And even if we tried to handle it somehow, we'd also need to handle
        // the case where the panic payload we get out of it also panics on
        // drop, and so on. See issue #86027.)
        if let Err(_) = panic::catch_unwind(panic::AssertUnwindSafe(|| {
            *self.result.get_mut() = None;
        })) {
            rtabort!("thread result panicked on drop");
        }
        // Book-keeping so the scope knows when it's done.
        if let Some(scope) = self.scope {
            // 在scope spawn线程时，在此处保证唤醒park的主线程
            scope.decrement_num_running_threads(unhandled_panic);
        }
    }
}
```

RUST线程 scope 结构,因为线程的主函数是闭包函数，对所有环境变量都是以借用引入，这会导致
因为线程不知道何时结束而出现环境变量的生命周期问题，利用scope使得线程闭包可以正常借用环境变量
```rust
pub struct Scope<'scope, 'env: 'scope> {
    //主要用来做生命周期的保证
    data: ScopeData,
    //指示线程的生命周期
    scope: PhantomData<&'scope mut &'scope ()>,
    //指示环境变量的生命周期
    env: PhantomData<&'env mut &'env ()>,
}

/// JoinHandle的scoped 版本
pub struct ScopedJoinHandle<'scope, T>(JoinInner<'scope, T>);

pub(super) struct ScopeData {
    num_running_threads: AtomicUsize,
    a_thread_panicked: AtomicBool,
    main_thread: Thread,
}

impl ScopeData {
    pub(super) fn increment_num_running_threads(&self) {
        //spawn线程时增加计数
        if self.num_running_threads.fetch_add(1, Ordering::Relaxed) > usize::MAX / 2 {
            self.decrement_num_running_threads(false);
            panic!("too many running threads in thread scope");
        }
    }
    pub(super) fn decrement_num_running_threads(&self, panic: bool) {
        if panic {
            self.a_thread_panicked.store(true, Ordering::Relaxed);
        }
        //减少线程计数
        if self.num_running_threads.fetch_sub(1, Ordering::Release) == 1 {
            //唤醒主线程
            self.main_thread.unpark();
        }
    }
}

// socpe内部创建的线程可以借用非静态变量
// 其中，'env是环境变量的生命周期，'scope是线程的生命周期
pub fn scope<'env, F, T>(f: F) -> T
where
    F: for<'scope> FnOnce(&'scope Scope<'scope, 'env>) -> T,
{
    let scope = Scope {
        data: ScopeData {
            num_running_threads: AtomicUsize::new(0),
            main_thread: current(),
            a_thread_panicked: AtomicBool::new(false),
        },
        env: PhantomData,
        scope: PhantomData,
    };

    // Run `f`, but catch panics so we can make sure to wait for all the threads to join.
    let result = catch_unwind(AssertUnwindSafe(|| f(&scope)));

    // 等待所有的线程都退出，保证线程的生命周期小于本函数的生命周期.
    while scope.data.num_running_threads.load(Ordering::Acquire) != 0 {
        park();
    }

    // Throw any panic from `f`, or the return value of `f` if no thread panicked.
    match result {
        Err(e) => resume_unwind(e),
        Ok(_) if scope.data.a_thread_panicked.load(Ordering::Relaxed) => {
            panic!("a scoped thread panicked")
        }
        Ok(result) => result,
    }
}

impl<'scope, 'env> Scope<'scope, 'env> {
    //用于在scope中创建新线程
    pub fn spawn<F, T>(&'scope self, f: F) -> ScopedJoinHandle<'scope, T>
    where
        F: FnOnce() -> T + Send + 'scope,
        T: Send + 'scope,
    {
        Builder::new()
            .spawn_scoped(self, f)
            .expect("failed to spawn thread")
    }
}

impl Builder {
    // 在scope情况下创建线程
    pub fn spawn_scoped<'scope, 'env, F, T>(
        self,
        scope: &'scope Scope<'scope, 'env>,
        f: F,
    ) -> io::Result<ScopedJoinHandle<'scope, T>>
    where
        //设置了生命周期，使得f可以使用环境变量引用
        F: FnOnce() -> T + Send + 'scope,
        T: Send + 'scope,
    {
        Ok(ScopedJoinHandle(unsafe {
            self.spawn_unchecked_(f, Some(&scope.data))
        }?))
    }
}

//利用生命周期的标注来使用环境变量引用
impl<'scope, T> ScopedJoinHandle<'scope, T> {
    
    pub fn thread(&self) -> &Thread {
        &self.0.thread
    }

    pub fn join(self) -> Result<T> {
        self.0.join()
    }

    pub fn is_finished(&self) -> bool {
        Arc::strong_count(&self.0.packet) == 1
    }
}
```
## RUST线程间消息通信
在网络操作系统中，线程间使用消息通信被广泛采用，甚至线程间通信仅使用消息机制。主要因为如果线程之间仅使用消息机制的话，即基本可以保证没有临界区，从而减少内存安全问题的情况。一般的，针对每个线程创建一个多个生产者，单个消费者的消息队列，消费者绑定在这个线程上，其他需要与此线程通信的线程是生产者。这种系统一般会确定一个通用的消息协议格式。
消息通信的方式需要尽量规避过长的消息内容。  

代码路径：library/std/src/sync/mpsc/*.*     

本书将只讨论mpsc这一机制，spsc的分析留给读者。   
RUST将通信分成了三种情况：  
1. 最初建立连接时，默认为仅做一次发送，接收，即oneshot通道形式
2. 如果发送多于一个包，但收线程及发线程都固定为同一个，则升级为stream通道形式
3. 如果发送线程多于一个，则升级为shared通道形式

采用如此复杂的情况，虽然有合理的成分，但感觉标准库的作者实际上是在炫技，并且不想被人轻易的理解其思路及想法。实际上，统一使用shared的形式即可靠，又简单。因为升级这个过程实际上极易引发问题。
mpsc模块中复杂的主要结构类型如下：
1. Queue结构，用于消费者及接受者之间存储消息的队列，是满足Sync的类型结构   
2. SignalToken/WaitToken结构，用于解除接收线程的阻塞信号     
3. `oneshot::Packet<T>` oneshot类型的channel机制     
4. `shared::Packet<T>` shared类型的channel机制      
5. `Sender<Flavor<T>>`, `Receiver<Flavor<T>>`是接收及发送的端口   

我们将分节对其进行介绍
### 消息队列数据结构实现
多于一个消息包的时候，需要消息队列，RUST用于消息包的队列结构是一个无锁的，无阻塞的临界区队列，非常巧妙的设计，是需要牢记在心的, 充分体现了RUST标准库开发人员高超的编程技巧。
```rust
//以下是简单的FIFO的队列实现
pub enum PopResult<T> {
    //返回队列成员
    Data(T),
    //队列为空
    Empty,
    //在有些时刻会出现瞬间的不一致情况
    Inconsistent,
}

//节点结构
struct Node<T> {
    //next指针,利用原子指针实现多线程的Sync，值得牢记
    next: AtomicPtr<Node<T>>,
    value: Option<T>,
}

///  能够被多个线程操作的队列
pub struct Queue<T> {
    //利用原子指针操作实现多线程的Sync，极大简化了代码
    head: AtomicPtr<Node<T>>,
    //从后面的代码看，这里实际上是队列的头部，这个Queue的代码搞得奇怪
    tail: UnsafeCell<*mut Node<T>>,
}

unsafe impl<T: Send> Send for Queue<T> {}
unsafe impl<T: Send> Sync for Queue<T> {}

impl<T> Node<T> {
    unsafe fn new(v: Option<T>) -> *mut Node<T> {
        //申请堆内存后，将堆内存的指针提取出来
        Box::into_raw(box Node { next: AtomicPtr::new(ptr::null_mut()), value: v })
    }
}

impl<T> Queue<T> {
    pub fn new() -> Queue<T> {
        let stub = unsafe { Node::new(None) };
        //生成一个空元素的节点列表
        Queue { head: AtomicPtr::new(stub), tail: UnsafeCell::new(stub) }
    }

    //在头部
    pub fn push(&self, t: T) {
        unsafe {
            let n = Node::new(Some(t));
            //换成C的话，就是head->next = n; head = n
            //对于空队列来说，是tail = head; head->next = n; head = n; 
            //现在tail实际上是队列头部，head是尾部。tail的next是第一个有意义的成员 
            let prev = self.head.swap(n, Ordering::AcqRel);
            //要考虑在两个赋值中间加入了其他线程的操作是否会出问题,
            //这里面有一个复杂的分析，
            //假设原队列为head, 有两个线程分别插入新节点n,m
            //当n先执行，而m在这个代码位置插入，则m插入前prev_n = pre_head, head = n
            //m插入后，prev_m = n, head = m。如果n先执行下面的语句，执行完后 
            // pre_head->next = n, n->next = null，然后m执行完下面语句
            // pre_head->next = n, n->next = m, head = m，队列是正确的。
            // 如果m先执行，执行完后 pre_head->next = null, n->next = m, head = m;
            // 然后n执行，执行完成后 pre_head->next = n, n->next = m, head =m， 队列是正确的。
            // 换成多个线程实际上也一样是正确的。这个地方处理十分巧妙，这是系统级编程语言的魅
            //力, 当然，实际上是裸指针编程的魅力  
            //当然，在这个过程中会出现Inconsistent          
            (*prev).next.store(n, Ordering::Release);
            
        }
    }

    //仅有一个线程在pop
    pub fn pop(&self) -> PopResult<T> {
        unsafe {
            //tail实际上是队列头，value是None
            let tail = *self.tail.get();
            //tail的next是第一个有意义的成员
            let next = (*tail).next.load(Ordering::Acquire);

            //next如果为空，说明队列是空队列
            if !next.is_null() {
                //此处原tail会被drop，tail被赋成next
                //因为push只可能改变next，所以这里不会有线程冲突问题
                //这个语句完成后，队列是完整及一致的 
                *self.tail.get() = next;
                assert!((*tail).value.is_none());
                assert!((*next).value.is_some());
                //将value的所有权转移出来，*next的value又重新置为None
                //当tail == head的时候 就又都是stub了
                let ret = (*next).value.take().unwrap();
                //恢复Box，以便以后释放堆内存
                let _: Box<Node<T>> = Box::from_raw(tail);
                return Data(ret);
            }

            // 此时如果head不是tail，一般说明有线程正在push，出现了不一致的情况,但这个不一致
            // 随着另一线程插入的结束会终结
            if self.head.load(Ordering::Acquire) == tail { Empty } else { Inconsistent }
        }
    }
}

impl<T> Drop for Queue<T> {
    fn drop(&mut self) {
        unsafe {
            //空队列的stub也要释放
            let mut cur = *self.tail.get();
            while !cur.is_null() {
                let next = (*cur).next.load(Ordering::Relaxed);
                //恢复Box并消费掉，释放堆内存
                let _: Box<Node<T>> = Box::from_raw(cur);
                cur = next;
            }
        }
    }
}
```
### 线程间简单的阻塞及唤醒信号机制
消息通信时，消息发送端需要有一个机制通知消息接收端消息已经发出。Condvar可以完成这一工作，但RUST的消息机制决定用无锁设计，所以做了新的实现。   
下面的设计具有通用性，正如上节的Queue。基本思路是：
1. 设计多个线程间的信号结构, 只允许一个线程等待在信号上，可以有多个线程触发信号解锁 
2. 利用原子变量的变化来做等待及信号等待

代码如下：
```rust
//线程间共享的信号结构
struct Inner {
    //指明执行信号等待的线程
    thread: Thread,
    //标志解除等待信号发送
    woken: AtomicBool,
}

unsafe impl Send for Inner {}
unsafe impl Sync for Inner {}

//信号发送端结构
pub struct SignalToken {
    inner: Arc<Inner>,
}

//信号接收端结构
pub struct WaitToken {
    inner: Arc<Inner>,
}

impl !Send for WaitToken {}

impl !Sync for WaitToken {}

//信号对创建函数,由信号等待端线程创建
pub fn tokens() -> (WaitToken, SignalToken) {
    //初始为无信号
    let inner = Arc::new(Inner { thread: thread::current(), woken: AtomicBool::new(false) });
    // wait由线程本身使用
    let wait_token = WaitToken { inner: inner.clone() };
    // signal由其他线程使用
    let signal_token = SignalToken { inner };
    (wait_token, signal_token)
}

impl SignalToken {
    //发送信号以便唤醒等待线程
    pub fn signal(&self) -> bool {
        //更改原子变量，看是否处于等待信号状态
        let wake = self
            .inner
            .woken
            .compare_exchange(false, true, Ordering::SeqCst, Ordering::SeqCst)
            .is_ok();
        if wake {
            //更改成功，接收线程会调用park阻塞，unpark解除接收线程阻塞
            self.inner.thread.unpark();
        }
        wake
    }
   
    //传递给其他线程以便用来生成SignalToken，此处只能用
    //裸指针，这里是传递没有被智能指针封装的堆内存指针
    pub unsafe fn to_raw(self) -> *mut u8 {
        Arc::into_raw(self.inner) as *mut u8
    }
    
    //从to_raw生成的堆内存指针恢复为SignalToken,由发送线程完成
    pub unsafe fn from_raw(signal_ptr: *mut u8) -> SignalToken {
        SignalToken { inner: Arc::from_raw(signal_ptr as *mut Inner) }
    }
}

impl WaitToken {
    //接收线程等待发送端信号
    pub fn wait(self) {
        //必须先对woken做过设置
        while !self.inner.woken.load(Ordering::SeqCst) {
            thread::park()
        }
    }

    //设置超时的等待, 请参考线程锁那一节的park内容
    pub fn wait_max_until(self, end: Instant) -> bool {
        while !self.inner.woken.load(Ordering::SeqCst) {
            let now = Instant::now();
            if now >= end {
                return false;
            }
            thread::park_timeout(end - now)
        }
        true
    }
}
```
### oneshot通道机制实现
oneshot专门为收发一次消息包而优化的结构。
```rust
//以下用于标识通道的状态
//没有数据包
const EMPTY: *mut u8 = ptr::invalid_mut::<u8>(0); 
//有数据包等待被接收
const DATA: *mut u8 = ptr::invalid_mut::<u8>(1); 
//中断
const DISCONNECTED: *mut u8 = ptr::invalid_mut::<u8>(2); 
// 其他值(ptr)代表接收者信号结构变量的指针, 说明有接收者在等待接收

//消息包结构, 因为只有一次收及一次发，所以结构中除state外
//其他不涉及数据竞争
pub struct Packet<T> {
    // 通道状态，取值为EMPTY/DATA/DISCONNECTED/ptr
    state: AtomicPtr<u8>,
    // 通道内的数据, 此数据需要从发送者拷贝到此处，再拷贝到接受者，但因为仅有一个包
    // 所以性能不是关注要点
    data: UnsafeCell<Option<T>>,
    // 当发送第二个包，或者对Sender做clone时，需要进行升级,此处放置新的通道接收Receiver结构
    // 拥有所有权
    upgrade: UnsafeCell<MyUpgrade<T>>,
}

//接收时发生的错误类型结构
pub enum Failure<T> {
    //空错误
    Empty,
    //连接中断
    Disconnected,
    //升级中,发送线程会把ReceiverT发送过来
    Upgraded(Receiver<T>),
}

pub enum UpgradeResult {
    //已经成功升级为其他类型的通道
    UpSuccess,
    // 升级遇到Disconnected
    UpDisconnected,
    //接收线程阻塞及期望接收的信号
    UpWoke(SignalToken),
}

enum MyUpgrade<T> {
    //通道内没有包，可以不升级
    NothingSent,
    //通道内已经发送过包，需要考虑升级
    SendUsed,
    //通道已经被通知需要升级，升级后的端口在参数中
    GoUp(Receiver<T>),
}

impl<T> Packet<T> {
    //创建一个通道,所有的内容都是初始化值
    pub fn new() -> Packet<T> {
        Packet {
            data: UnsafeCell::new(None),
            upgrade: UnsafeCell::new(NothingSent),
            state: AtomicPtr::new(EMPTY),
        }
    }

    //发送线程通过Sender端口发送包,发送线程应保证只调用一次 
    pub fn send(&self, t: T) -> Result<(), T> {
        unsafe {
            //检查是否已经有包发过了 
            match *self.upgrade.get() {
                //没有包发送过，则继续执行
                NothingSent => {}
                //不应该执行到此处，应该先升级再发送
                _ => panic!("sending on a oneshot that's already sent on "),
            }
            assert!((*self.data.get()).is_none());
            //拷贝消息包内容
            ptr::write(self.data.get(), Some(t));
            //设置upgrade为已经发送过包，
            ptr::write(self.upgrade.get(), SendUsed);

            //更新state
            match self.state.swap(DATA, Ordering::SeqCst) {
                // 此时可以正常发送, state设置为有数据状态
                EMPTY => Ok(()),

                // 表明接收端已经destroy通道，
                DISCONNECTED => {
                    //需要state恢复成中断
                    self.state.swap(DISCONNECTED, Ordering::SeqCst);
                    //需要恢复upgrade，
                    ptr::write(self.upgrade.get(), NothingSent);
                    //需要将消息包数据回收,并返回发送出错
                    Err((&mut *self.data.get()).take().unwrap())
                }

                // 不应该到达这一步 
                DATA => unreachable!(),

                // 有线程等待接收.
                ptr => {
                    //通知接收线程解除阻塞
                    SignalToken::from_raw(ptr).signal();
                    Ok(())
                }
            }
        }
    }

    // 测试是否已经发过消息包
    pub fn sent(&self) -> bool {
        unsafe { !matches!(*self.upgrade.get(), NothingSent) }
    }

    //接收线程通过Receiver接收
    pub fn recv(&self, deadline: Option<Instant>) -> Result<T, Failure<T>> {
        // 尽量不阻塞线程
        if self.state.load(Ordering::SeqCst) == EMPTY {
            //消息为空, 需要阻塞，生成信号通知对
            let (wait_token, signal_token) = blocking::tokens();
            //获取信号发送端的堆内存
            let ptr = unsafe { signal_token.to_raw() };

            // 设置状态为有线程在等待接收
            if self.state.compare_exchange(EMPTY, ptr, Ordering::SeqCst, Ordering::SeqCst).is_ok() {
                //设置成功，判断是否有超时
                if let Some(deadline) = deadline {
                    //设置超时，阻塞
                    let timed_out = !wait_token.wait_max_until(deadline);
                    // 判断是否超时
                    if timed_out {
                        //如果超时，做清理，如果发送端通知升级，则形成Upgraded(Receiver<T>)
                        // 这里的map_err(Upgraded)构建了Upgraded(Receiver<T>)，需要记住
                        self.abort_selection().map_err(Upgraded)?;
                    }
                    //被接收线程唤醒
                } else {
                    //没有设置超时，一直阻塞等待
                    wait_token.wait();
                    debug_assert!(self.state.load(Ordering::SeqCst) != EMPTY);
                }
            } else {
                //失败，清理信号
                drop(unsafe { SignalToken::from_raw(ptr) });
            }
            //wait_token及signal_token都生命周期终止
        }

        //此时已经有数据了
        self.try_recv()
    }

    pub fn try_recv(&self) -> Result<T, Failure<T>> {
        unsafe {
            match self.state.load(Ordering::SeqCst) {
                //数据为空，返回错误
                EMPTY => Err(Empty),

                //发现数据
                DATA => {
                    //修改state为EMPTY
                    let _ = self.state.compare_exchange(
                        DATA,
                        EMPTY,
                        Ordering::SeqCst,
                        Ordering::SeqCst,
                    );
                    //将数据读出
                    match (&mut *self.data.get()).take() {
                        Some(data) => Ok(data),
                        None => unreachable!(),
                    }
                }

                //中断状态时，可能此通道已经被升级，要检查是否还有数据
                DISCONNECTED => match (&mut *self.data.get()).take() {
                    //有数据,读出数据即可
                    Some(data) => Ok(data),
                    //没有数据，更新upgrade状态
                    None => match ptr::replace(self.upgrade.get(), SendUsed) {
                        //不是通知升级，则发送端已经关闭，返回Disconnected信息 
                        SendUsed | NothingSent => Err(Disconnected),
                        //通知升级,将Receiver<T>包装到返回变量返回 
                        GoUp(upgrade) => Err(Upgraded(upgrade)),
                    },
                },

                // 不可能的分支
                _ => unreachable!(),
            }
        }
    }

    // 升级管道到其他类型，由发送线程调用 
    pub fn upgrade(&self, up: Receiver<T>) -> UpgradeResult {
        unsafe {
            let prev = match *self.upgrade.get() {
                //可正常升级
                NothingSent => NothingSent,
                SendUsed => SendUsed,
                //其他状态表示已经升级完成
                _ => panic!("upgrading again"),
            };
            // 将升级到的Receiver写入self.upgrade 
            ptr::write(self.upgrade.get(), GoUp(up));

            //后继不会再使用self传递消息，更新状态为DISCONNECTED
            match self.state.swap(DISCONNECTED, Ordering::SeqCst) {
                // 原状态为DATA及EMPTY，返回升级成功
                // 此时有可能消息还没有被接收
                // 返回后，发送端端口Sender会生命周期终结
                DATA | EMPTY => UpSuccess,

                //  如果已经DISCONNECT，则需要撤回本次请求
                DISCONNECTED => {
                    ptr::replace(self.upgrade.get(), prev);
                    // 升级时通道已经中断
                    UpDisconnected
                }

                // 如果有线程在等待接收， 需要将唤醒信号返回
                ptr => UpWoke(SignalToken::from_raw(ptr)),
            }
        }
    }

    //删除通道, 由发送线程在Sender被drop时调用
    pub fn drop_chan(&self) {
        //更新状态
        match self.state.swap(DISCONNECTED, Ordering::SeqCst) {
            //原状态为下面的值可以不做操作
            DATA | DISCONNECTED | EMPTY => {}

            // 如果有等待线程，则发送信号唤醒
            ptr => unsafe {
                SignalToken::from_raw(ptr).signal();
            },
        }
    }

    //删除端口,由接收线程在Receiver被drop时调用
    pub fn drop_port(&self) {
        //更新状态
        match self.state.swap(DISCONNECTED, Ordering::SeqCst) {
            DISCONNECTED | EMPTY => {}

            // 如果有数据，需要删除它
            DATA => unsafe {
                (&mut *self.data.get()).take().unwrap();
                //数据包生命周期终止
            },

            // 接收线程才能调用这个函数
            _ => unreachable!(),
        }
    }

    // 阻塞超时处理.
    pub fn abort_selection(&self) -> Result<bool, Receiver<T>> {
        //获取state
        let state = match self.state.load(Ordering::SeqCst) {
            // 这些状态不用处理 
            s @ (EMPTY | DATA | DISCONNECTED) => s,

            // ptr是本线程设置的，切换回EMPTY状态, 并把信号指针带回
            ptr => self
                .state
                .compare_exchange(ptr, EMPTY, Ordering::SeqCst, Ordering::SeqCst)
                .unwrap_or_else(|x| x),
        };

        match state {
            //不应该出现这个情况
            EMPTY => unreachable!(),
            //有数据 
            DATA => Ok(true),

            // 发送端中断
            DISCONNECTED => unsafe {
                //收到数据
                if (*self.data.get()).is_some() {
                    Ok(true)
                } else {
                    //看是否需要升级
                    match ptr::replace(self.upgrade.get(), SendUsed) {
                        //升级调用，返回升级到的端口Reciver<T>
                        GoUp(port) => Err(port),
                        _ => Ok(true),
                    }
                }
            },

            // 没有其他线程发送数据
            ptr => unsafe {
                //删除信号
                drop(SignalToken::from_raw(ptr));
                //没有接收数据
                Ok(false)
            },
        }
    }
}

impl<T> Drop for Packet<T> {
    fn drop(&mut self) {
        assert_eq!(self.state.load(Ordering::SeqCst), DISCONNECTED);
    }
}

```
### Shared的通道
当oneshot的tx做clone操作时，oneshot的通道升级到Shared类型通道:
```rust 
//用发送包的技术来表示通道的状态
//通道中断计数标志
const DISCONNECTED: isize = isize::MIN;
//最大能支持的通道数
const FUDGE: isize = 1024;
const MAX_REFCOUNT: usize = (isize::MAX) as usize;
//最多能计数的无阻塞收包数目
const MAX_STEALS: isize = 1 << 20;
const EMPTY: *mut u8 = ptr::null_mut(); // initial state: no data, no blocked receiver

pub struct Packet<T> {
    //消息包的queue
    queue: mpsc::Queue<T>,
    //发送的包总数,每次阻塞或接收包数目到达限值会设置为-1。
    // -1作为有阻塞，需要发送信号的标记
    cnt: AtomicIsize,
    //接收的包总数,每次阻塞，或接收包数目达到限值会清零
    steals: UnsafeCell<isize>,
    //唤醒的信号SingleToken指针

    //接收线程阻塞时期待的信号量
    to_wake: AtomicPtr<u8>,

    //初始最少有两个使用者,每多一个发送线程就加1
    channels: AtomicUsize,

    //接收端关闭通道的标志
    port_dropped: AtomicBool,
    //发送端发现接收端中断，确定清理线程的辅助结构
    sender_drain: AtomicIsize,

    //使用单元类型的Mutex，将Mutex仅做锁的场景，不包含临界区,通常这个锁的临界区是一段代码操作
    select_lock: Mutex<()>,
}

pub enum Failure {
    Empty,
    Disconnected,
}

enum StartResult {
    Installed,
    Abort,
}

impl<T> Packet<T> {
    //新建一个通道，随后必须紧跟postinit_lock及inherit_blocker后才能做其他
    //通道操作
    pub fn new() -> Packet<T> {
        Packet {
            //包队列
            queue: mpsc::Queue::new(),
            //发送的包总数,每次阻塞或接收包数目到达限值会清零。 
            cnt: AtomicIsize::new(0),
            //接收的包总数,每次阻塞，或接收包数目达到限值会清零
            steals: UnsafeCell::new(0),
            //唤醒接收线程的信号
            to_wake: AtomicPtr::new(EMPTY),
            //初始最少有两个使用者,每多一个发送线程就加1
            channels: AtomicUsize::new(2),
            //接收端关闭通道的标志
            port_dropped: AtomicBool::new(false),
            //发送端发现接收端中断，确定清理线程的辅助结构
            sender_drain: AtomicIsize::new(0),
            //用于创建时的临界区代码保户
            select_lock: Mutex::new(()),
        }
    }

    // 必须在new之后第一时间调用，在封装self的Arc还没有clone之前
    pub fn postinit_lock(&self) -> MutexGuard<'_, ()> {
        self.select_lock.lock().unwrap()
    }

    // 这个函数处理升级前的通道遗留的阻塞线程场景,guard是调用postinit_lock的返回
    pub fn inherit_blocker(&self, token: Option<SignalToken>, guard: MutexGuard<'_, ()>) {
        //判断是否有接收线程阻塞
        if let Some(token) = token {
            assert_eq!(self.cnt.load(Ordering::SeqCst), 0);
            assert_eq!(self.to_wake.load(Ordering::SeqCst), EMPTY);
            //将阻塞信号设置到to_wake中
            self.to_wake.store(unsafe { token.to_raw() }, Ordering::SeqCst);
            //有接收线程阻塞，导致发第一个包的时候，才会去唤醒接收线程，接收线程才可能
            //做升级，然后才能接收数据包。这个-1作为阻塞的标志
            //这个设计方式过于复杂，不是一个好的设计，
            self.cnt.store(-1, Ordering::SeqCst);

            unsafe {
                // cnt为-1，steals也需要设置为-1
                *self.steals.get() = -1;
            }
        }

        //解锁
        drop(guard);
    }

    pub fn send(&self, t: T) -> Result<(), T> {
        //看接收端口Receiver是否已经关闭
        if self.port_dropped.load(Ordering::SeqCst) {
            return Err(t);
        }

        //判断通道是否中断,因为每个线程发送都可能会造成计数加1，所以最大值
        //是DISCONNECTED+FUDGE,这个区间可认为通道已经被设置为中断
        if self.cnt.load(Ordering::SeqCst) < DISCONNECTED + FUDGE {
            return Err(t);
        }

        //消息入队列
        self.queue.push(t);
        //增加队列计数,每次push队列都要先对cnt增加值来反映此操作
        //但此时此时接收端口Receiver可能生命周期终止，导致cnt被设置为DISCONNECT
        match self.cnt.fetch_add(1, Ordering::SeqCst) {
            //原值为-1，是发送的第一个包，且接收端在等待信号
            //其他线程不会得到-1, 只有-1的发送线程来发送信号
            -1 => {
                //发信号通知接收线程退出阻塞,工作结束,
                //这个机制搞的有些复杂
                self.take_to_wake().signal();
            }

            // 消息入队列后，通道被中断，此时需要把数据包撤回.
            n if n < DISCONNECTED + FUDGE => {
                //重新设置cnt为中断状态
                self.cnt.store(DISCONNECTED, Ordering::SeqCst);

                //判断我们是否是第一个sender_drain
                if self.sender_drain.fetch_add(1, Ordering::SeqCst) == 0 {
                    //是，负责删除队列里面的所有消息包
                    loop {
                        //循环直到queue为空
                        loop {
                            match self.queue.pop() {
                                mpsc::Data(..) => {}
                                mpsc::Empty => break,
                                mpsc::Inconsistent => thread::yield_now(),
                            }
                        }
                        
                        if self.sender_drain.fetch_sub(1, Ordering::SeqCst) == 1 {
                            //确定所有线程都已经被处理
                            break;
                        }
                        //还有其他线程做了sender_drain的add，那再循环
                    }

                    // 本线程push到queue的包确定已经删除
                }
            }

            _ => {}
        }

        Ok(())
    }

    pub fn recv(&self, deadline: Option<Instant>) -> Result<T, Failure> {
        //尽量不阻塞
        match self.try_recv() {
            Err(Empty) => {}
            data => return data,
        }

        //需要阻塞
        //生成通知信号
        let (wait_token, signal_token) = blocking::tokens();
        //因为try_recv到此处可能会有其他线程发包，需要做些
        //处理看是否需要阻塞
        if self.decrement(signal_token) == Installed {
            //确定要阻塞
            if let Some(deadline) = deadline {
                //有超时要去,做一个超时等待
                let timed_out = !wait_token.wait_max_until(deadline);
                if timed_out {
                    //如果超时，需要做清理工作
                    self.abort_selection(false);
                }
            } else {
                //阻塞至包来到
                wait_token.wait();
            }
        }

        //当前已经有数据包
        match self.try_recv() {
            data @ Ok(..) => unsafe {
                //反应阻塞收包统计,try_recv会加1,这里减掉
                //有可能没有阻塞，但按照阻塞来计算
                //这里是为了对冲在阻塞时对cnt多减1
                //无论如何，利用这个来实现对阻塞与否的判断我认为不是一个好主意
                *self.steals.get() -= 1;
                data
            },
            data => data,
        }
    }

    //判断是否应该阻塞
    fn decrement(&self, token: SignalToken) -> StartResult {
        unsafe {
            assert_eq!(
                self.to_wake.load(Ordering::SeqCst),
                EMPTY,
                "This is a known bug in the Rust standard library. See https://github.com/rust-lang/rust/issues/39364"
            );
            // 设置收线程阻塞信号到通道
            let ptr = token.to_raw();
            self.to_wake.store(ptr, Ordering::SeqCst);

            //进入阻塞时对steals做清零
            let steals = ptr::replace(self.steals.get(), 0);

            //cnt需要把上次阻塞到本次阻塞之间的收包数目减掉，然后再减1,以便cnt成为-1
            match self.cnt.fetch_sub(1 + steals, Ordering::SeqCst) {
                //如果减法之前发送侧已经中断
                DISCONNECTED => {
                    //将cnt恢复为中断
                    self.cnt.store(DISCONNECTED, Ordering::SeqCst);
                }
                
                //不是中断，原来至少应该发送过一个包，cnt应该不小于0
                n => {
                    assert!(n >= 0);
                    //在两次取值间可能有其他通道已经发包过来，那不应该阻塞
                    //如果没有其他包，则阻塞
                    //发送的包减掉接收的包不大于0，表示没有线程竞争发包
                    if n - steals <= 0 {
                        //正常阻塞
                        return Installed;
                    }
                }
            }

            //此时队列已经有包或者DISCONNECT了，不需要阻塞
            //撤掉信号
            self.to_wake.store(EMPTY, Ordering::SeqCst);
            drop(SignalToken::from_raw(ptr));
            Abort
        }
    }

    //接收数据包
    pub fn try_recv(&self) -> Result<T, Failure> {
        //从队列取得一个包
        let ret = match self.queue.pop() {
            //成功
            mpsc::Data(t) => Some(t),
            //不成功
            mpsc::Empty => None,

            // 此时处于一个临界状态.可以做个自旋等待一下
            mpsc::Inconsistent => {
                let data;
                //默认为肯定会获得数据
                loop {
                    //这里等待一个操作系统调度周期
                    //试图让发送线程工作
                    //但等待时间不定
                    thread::yield_now();
                    match self.queue.pop() {
                        //收到数据
                        mpsc::Data(t) => {
                            data = t;
                            break;
                        }
                        //不应有这种情况
                        mpsc::Empty => panic!("inconsistent => empty"),
                        //继续等待
                        mpsc::Inconsistent => {}
                    }
                }
                Some(data)
            }
        };
        match ret {
            //接收到数据
            Some(data) => unsafe {
                //如果非阻塞收包已经大于MAX_STEALS
                if *self.steals.get() > MAX_STEALS {
                    //将cnt清零
                    match self.cnt.swap(0, Ordering::SeqCst) {
                        //原cnt是DISCONNECTED
                        DISCONNECTED => {
                            //重新置为DISCONNECTED
                            self.cnt.store(DISCONNECTED, Ordering::SeqCst);
                        }
                        n => {
                            //这里在cnt及steals上共同减去两者之间小者
                            //取值小者
                            let m = cmp::min(n, *self.steals.get());
                            //steals及cnt都减去最小值
                            *self.steals.get() -= m;
                            //实际上是原cnt减去m
                            self.bump(n - m);
                        }
                    }
                    assert!(*self.steals.get() >= 0);
                }
                //steals增加
                *self.steals.get() += 1;
                Ok(data)
            },

            //没有收到数据
            None => {
                match self.cnt.load(Ordering::SeqCst) {
                    //如果通道没有中断，返回异常的队列空
                    n if n != DISCONNECTED => Err(Empty),
                    //其他 只可能是DISCONNECTED
                    _ => {
                        //再接收一次
                        match self.queue.pop() {
                            //没有对self.steals做操作
                            mpsc::Data(t) => Ok(t),
                            //空，认为已经中断
                            mpsc::Empty => Err(Disconnected),
                            // 不应有这种情况
                            mpsc::Inconsistent => unreachable!(),
                        }
                        //丢弃这个包，所以不必更新计数
                    }
                }
            }
        }
    }

    // Sender<T>做clone时的支撑函数 
    pub fn clone_chan(&self) {
        //channel数目增加
        let old_count = self.channels.fetch_add(1, Ordering::SeqCst);

        if old_count > MAX_REFCOUNT {
            abort();
        }
    }

    // 发送线程关闭通道
    pub fn drop_chan(&self) {
        //减少channel计数
        match self.channels.fetch_sub(1, Ordering::SeqCst) {
            //需要做清理
            1 => {}
            //还有其他发送线程，不必处理
            n if n > 1 => return,
            //不应该发生这种情况
            n => panic!("bad number of channels left {n}"),
        }

        //所有发送线程均已关闭，发端置中断状态
        match self.cnt.swap(DISCONNECTED, Ordering::SeqCst) {
            // 有接收线程阻塞
            -1 => {
                //发信号解除阻塞
                self.take_to_wake().signal();
            }
            DISCONNECTED => {}
            n => {
                assert!(n >= 0);
            }
        }
    }

    //接收线程关闭通道
    pub fn drop_port(&self) {
        //置标志
        self.port_dropped.store(true, Ordering::SeqCst);
        //获取上次阻塞以来接收的数据包
        let mut steals = unsafe { *self.steals.get() };
        while {
            //这个block是while的条件语句
            //当发送数据包与接收数据包相同时，设置中断
            match self.cnt.compare_exchange(
                steals,
                DISCONNECTED,
                Ordering::SeqCst,
                Ordering::SeqCst,
            ) {
                //成功,退出循环
                Ok(_) => false,
                //old是DISCONNECT时，退出循环，否则进入循环
                Err(old) => old != DISCONNECTED,
            }
        } {
            //这个循环把队列清空
            loop {
                //收包
                match self.queue.pop() {
                    mpsc::Data(..) => {
                        steals += 1;
                    }
                    mpsc::Empty | mpsc::Inconsistent => break,
                }
                //生命周期终结，释放包
            }
        }
    }

    // 重组阻塞信号结构 
    fn take_to_wake(&self) -> SignalToken {
        let ptr = self.to_wake.load(Ordering::SeqCst);
        self.to_wake.store(EMPTY, Ordering::SeqCst);
        assert!(ptr != EMPTY);
        unsafe { SignalToken::from_raw(ptr) }
    }

    //一次性给cnt增加若干值
    fn bump(&self, amt: isize) -> isize {
        //一次增加cnt输入参数
        match self.cnt.fetch_add(amt, Ordering::SeqCst) {
            //如果原值是DISCONNECT
            DISCONNECTED => {
                //cnt恢复为DISCONNECT
                self.cnt.store(DISCONNECTED, Ordering::SeqCst);
                DISCONNECTED
            }
            n => n,
        }
    }

    //接收线程阻塞超时时做处理
    pub fn abort_selection(&self, _was_upgrade: bool) -> bool {
        //加锁，保护下面的临界区代码
        {
            let _guard = self.select_lock.lock().unwrap();
        }

        let steals = {
            //这个程序员愿意用block作为表达式结果
            //获取cnt
            let cnt = self.cnt.load(Ordering::SeqCst);
            //发送端没有中断，cnt应该是阻塞超时的次数
            //只能是-1或者0
            if cnt < 0 && cnt != DISCONNECTED { -cnt } else { 0 }
        };
        //cnt增加，清除超时,每次超时如果有包，则steals加1
        let prev = self.bump(steals + 1);

        //发送端已经中断
        if prev == DISCONNECTED {
            //更新等待信号为空, 
            assert_eq!(self.to_wake.load(Ordering::SeqCst), EMPTY);
            //后继退出收包
            true
        } else {
            //当前的发包计数
            let cur = prev + steals + 1;
            assert!(cur >= 0);
            if prev < 0 {
                //没有发包导致，drop掉接收等待信号
                drop(self.take_to_wake());
            } else {
                //发送端马上应该发送信号，等一下
                while self.to_wake.load(Ordering::SeqCst) != EMPTY {
                    thread::yield_now();
                }
            }
            unsafe {
                let old = self.steals.get();
                //steals只可能是0或1
                assert!(*old == 0 || *old == -1);
                //更新self.steals,实际上是steals加1
                *old = steals;
                prev >= 0
            }
        }
    }
}

impl<T> Drop for Packet<T> {
    fn drop(&mut self) {
        //确保Packet已经清理完毕
        assert_eq!(self.cnt.load(Ordering::SeqCst), DISCONNECTED);
        assert_eq!(self.to_wake.load(Ordering::SeqCst), EMPTY);
        assert_eq!(self.channels.load(Ordering::SeqCst), 0);
    }
}
```
shared 类型的通道设计最奇怪的地方是用了复杂的发包计数来作为阻塞标记。导致该处代码不易理解。
### mpsc的对外函数及接口

通道相关的类型结构及函数：
```rust
pub fn channel<T>() -> (Sender<T>, Receiver<T>) {
    //初始时创建oneshot的通道
    let a = Arc::new(oneshot::Packet::new());
    //对onshot通道做clone，然后创建Sender及Receiver
    (Sender::new(Flavor::Oneshot(a.clone())), Receiver::new(Flavor::Oneshot(a)))
}

//发送端端口
pub struct Sender<T> {
    inner: UnsafeCell<Flavor<T>>,
}

//接收端端口
pub struct Receiver<T> {
    inner: UnsafeCell<Flavor<T>>,
}

//用来实现可升级的通道，因为有带参数的成员，RUST没有使用dyn trait这种设计
//对于认为以后通道类型不会再扩张时，采用enum的设计方式更易控制
//但如果预计后继还会有很多通道方式，则应该采用dyn Packet<T>的设计方式
enum Flavor<T> {
    //只发送单一通信包的通道
    Oneshot(Arc<oneshot::Packet<T>>),
    //一对一的多通信包的通道,当发端发送第二个包的时候
    //要创建并切换到这个通道
    Stream(Arc<stream::Packet<T>>),
    //多对一的通道，当发端做clone操作的时候
    //要创建并切换到这个通道
    Shared(Arc<shared::Packet<T>>),
    //同步通道，本书不分析
    Sync(Arc<sync::Packet<T>>),
}

//Sender及Receiver内部访问支持trait
trait UnsafeFlavor<T> {
    fn inner_unsafe(&self) -> &UnsafeCell<Flavor<T>>;
    unsafe fn inner_mut(&self) -> &mut Flavor<T> {
        &mut *self.inner_unsafe().get()
    }
    unsafe fn inner(&self) -> &Flavor<T> {
        &*self.inner_unsafe().get()
    }
}
impl<T> UnsafeFlavor<T> for Sender<T> {
    fn inner_unsafe(&self) -> &UnsafeCell<Flavor<T>> {
        &self.inner
    }
}
impl<T> UnsafeFlavor<T> for Receiver<T> {
    fn inner_unsafe(&self) -> &UnsafeCell<Flavor<T>> {
        &self.inner
    }
}
```
Sender的方法：
```rust
impl<T> Sender<T> {
    //创建包含通道的Sender
    fn new(inner: Flavor<T>) -> Sender<T> {
        Sender { inner: UnsafeCell::new(inner) }
    }

    //发送一个数据包
    pub fn send(&self, t: T) -> Result<(), SendError<T>> {
        //相当于新创建了一个Flavor的变量, 此时要注意drop是否发生了两次
        //这里对解引用的match因为没有引发赋值，不会导致所有权转移
        let (new_inner, ret) = match *unsafe { self.inner() } {
            //必须是ref，否则会导致所有权转移
            Flavor::Oneshot(ref p) => {
                //判断是否还能发送包，此时只能发一个包
                if !p.sent() {
                    return p.send(t).map_err(SendError);
                } else {
                    //多于一个包，创建stream的通道来进行升级
                    let a = Arc::new(stream::Packet::new());
                    //基于新的通道创建新的Receiver
                    let rx = Receiver::new(Flavor::Stream(a.clone()));
                    //通知rx端进行升级操作
                    match p.upgrade(rx) {
                        //升级成功
                        oneshot::UpSuccess => {
                            //发送报文
                            let ret = a.send(t);
                            //将新的通道赋值
                            (a, ret)
                        }
                        //接收已经DISCONNECT，将数据包及新通道共同返回
                        oneshot::UpDisconnected => (a, Err(t)),
                        //接收线程阻塞,需要做唤醒
                        oneshot::UpWoke(token) => {
                            //先将包发送
                            a.send(t).ok().unwrap();
                            //唤醒接收线程
                            token.signal();
                            //返回新通道
                            (a, Ok(()))
                        }
                    }
                }
            }
            //已经是Stream，正常发送包的逻辑，直接返回，不修改self
            Flavor::Stream(ref p) => return p.send(t).map_err(SendError),
            //已经是Shared，正常的发送逻辑，直接返回，不修改self
            Flavor::Shared(ref p) => return p.send(t).map_err(SendError),
            //不可能到达这个代码位置
            Flavor::Sync(..) => unreachable!(),
        };

        unsafe {
            //只有oneshot会进入此处
            //新建Sender，并将新的Sender及老的Sender进行内存替换
            //此处要注意，enum的不同成员不保证内存相同，但这里是没有问题的
            let tmp = Sender::new(Flavor::Stream(new_inner));
            mem::swap(self.inner_mut(), tmp.inner_mut());
            //此处,tmp会生命周期终结，tmp当前是oneshot的类型。
            //要注意收端是怎么终结的
        }
        ret.map_err(SendError)
    }
}

impl<T> Clone for Sender<T> {
    /// clone代表进入了多发一收的模式，需要升级到shared类型的通道 
    fn clone(&self) -> Sender<T> {
        let packet = match *unsafe { self.inner() } {
            Flavor::Oneshot(ref p) => {
                //创建shared类型通道
                let a = Arc::new(shared::Packet::new());
                {
                    //创建后首先lock
                    let guard = a.postinit_lock();
                    //创建Receiver
                    let rx = Receiver::new(Flavor::Shared(a.clone()));
                    //进行升级
                    let sleeper = match p.upgrade(rx) {
                        oneshot::UpSuccess | oneshot::UpDisconnected => None,
                        oneshot::UpWoke(task) => Some(task),
                    };
                    //完成通道设置
                    a.inherit_blocker(sleeper, guard);
                }
                //置值
                a
            }
            //进入一对一的多包发送
            Flavor::Stream(ref p) => {
                //仍然创建shared类型通道
                let a = Arc::new(shared::Packet::new());
                {
                    //首先lock
                    let guard = a.postinit_lock();
                    //创建Receiver
                    let rx = Receiver::new(Flavor::Shared(a.clone()));
                    //升级
                    let sleeper = match p.upgrade(rx) {
                        stream::UpSuccess | stream::UpDisconnected => None,
                        stream::UpWoke(task) => Some(task),
                    };
                    //完成通道设置
                    a.inherit_blocker(sleeper, guard);
                }
                //置值
                a
            }
            Flavor::Shared(ref p) => {
                //先做clone_chan
                p.clone_chan();
                //创建新的Sender,并返回
                return Sender::new(Flavor::Shared(p.clone()));
            }
            //不会到达这个地方
            Flavor::Sync(..) => unreachable!(),
        };

        unsafe {
            //创建新的Sender
            let tmp = Sender::new(Flavor::Shared(packet.clone()));
            //替换现有的Sender
            mem::swap(self.inner_mut(), tmp.inner_mut());
            //原有的Flavor生命周期终止并被drop
        }
        //创建新的Sender,并返回
        Sender::new(Flavor::Shared(packet))
    }
}

impl<T> Drop for Sender<T> {
    fn drop(&mut self) {
        //行为一致，都是中断通道
        match *unsafe { self.inner() } {
            Flavor::Oneshot(ref p) => p.drop_chan(),
            Flavor::Stream(ref p) => p.drop_chan(),
            Flavor::Shared(ref p) => p.drop_chan(),
            Flavor::Sync(..) => unreachable!(),
        }
    }
}
```
Receiver的方法：
```rust
impl<T> Receiver<T> {
    fn new(inner: Flavor<T>) -> Receiver<T> {
        Receiver { inner: UnsafeCell::new(inner) }
    }

    //不阻塞的收包
    pub fn try_recv(&self) -> Result<T, TryRecvError> {
        loop {
            let new_port = match *unsafe { self.inner() } {
                Flavor::Oneshot(ref p) => match p.try_recv() {
                    //非升级的情况都直接返回
                    Ok(t) => return Ok(t),
                    Err(oneshot::Empty) => return Err(TryRecvError::Empty),
                    Err(oneshot::Disconnected) => return Err(TryRecvError::Disconnected),
                    //升级的情况将rx置值到new_port
                    Err(oneshot::Upgraded(rx)) => rx,
                },
                Flavor::Stream(ref p) => match p.try_recv() {
                    //非升级的情况都直接返回
                    Ok(t) => return Ok(t),
                    Err(stream::Empty) => return Err(TryRecvError::Empty),
                    Err(stream::Disconnected) => return Err(TryRecvError::Disconnected),
                    //升级的情况将rx置值到new_port
                    Err(stream::Upgraded(rx)) => rx,
                },
                Flavor::Shared(ref p) => match p.try_recv() {
                    Ok(t) => return Ok(t),
                    Err(shared::Empty) => return Err(TryRecvError::Empty),
                    Err(shared::Disconnected) => return Err(TryRecvError::Disconnected),
                    //不应该出现升级的情况
                },
                Flavor::Sync(ref p) => match p.try_recv() {
                    Ok(t) => return Ok(t),
                    Err(sync::Empty) => return Err(TryRecvError::Empty),
                    Err(sync::Disconnected) => return Err(TryRecvError::Disconnected),
                },
            };
            unsafe {
                //直接用new_port替换原来的Flavor
                mem::swap(self.inner_mut(), new_port.inner_mut());
            }
            //new_port生命周期终结，原有的Flavor被调用drop
        }
    }

    //阻塞收包，替换逻辑与try_recv相同
    pub fn recv(&self) -> Result<T, RecvError> {
        loop {
            let new_port = match *unsafe { self.inner() } {
                Flavor::Oneshot(ref p) => match p.recv(None) {
                    Ok(t) => return Ok(t),
                    Err(oneshot::Disconnected) => return Err(RecvError),
                    Err(oneshot::Upgraded(rx)) => rx,
                    Err(oneshot::Empty) => unreachable!(),
                },
                Flavor::Stream(ref p) => match p.recv(None) {
                    Ok(t) => return Ok(t),
                    Err(stream::Disconnected) => return Err(RecvError),
                    Err(stream::Upgraded(rx)) => rx,
                    Err(stream::Empty) => unreachable!(),
                },
                Flavor::Shared(ref p) => match p.recv(None) {
                    Ok(t) => return Ok(t),
                    Err(shared::Disconnected) => return Err(RecvError),
                    Err(shared::Empty) => unreachable!(),
                },
                Flavor::Sync(ref p) => return p.recv(None).map_err(|_| RecvError),
            };
            unsafe {
                mem::swap(self.inner_mut(), new_port.inner_mut());
            }
        }
    }

    //设置超时的阻塞收包
    pub fn recv_timeout(&self, timeout: Duration) -> Result<T, RecvTimeoutError> {
        // Do an optimistic try_recv to avoid the performance impact of
        // Instant::now() in the full-channel case.
        match self.try_recv() {
            Ok(result) => Ok(result),
            Err(TryRecvError::Disconnected) => Err(RecvTimeoutError::Disconnected),
            //没有包的时候才进入超时
            Err(TryRecvError::Empty) => match Instant::now().checked_add(timeout) {
                //调用超时接收
                Some(deadline) => self.recv_deadline(deadline),
                None => self.recv().map_err(RecvTimeoutError::from),
            },
        }
    }

    //真正的超时接收,与recv基本相同，仅增加了超时参数
    pub fn recv_deadline(&self, deadline: Instant) -> Result<T, RecvTimeoutError> {
        use self::RecvTimeoutError::*;

        loop {
            let port_or_empty = match *unsafe { self.inner() } {
                Flavor::Oneshot(ref p) => match p.recv(Some(deadline)) {
                    Ok(t) => return Ok(t),
                    Err(oneshot::Disconnected) => return Err(Disconnected),
                    Err(oneshot::Upgraded(rx)) => Some(rx),
                    Err(oneshot::Empty) => None,
                },
                Flavor::Stream(ref p) => match p.recv(Some(deadline)) {
                    Ok(t) => return Ok(t),
                    Err(stream::Disconnected) => return Err(Disconnected),
                    Err(stream::Upgraded(rx)) => Some(rx),
                    Err(stream::Empty) => None,
                },
                Flavor::Shared(ref p) => match p.recv(Some(deadline)) {
                    Ok(t) => return Ok(t),
                    Err(shared::Disconnected) => return Err(Disconnected),
                    Err(shared::Empty) => None,
                },
                Flavor::Sync(ref p) => match p.recv(Some(deadline)) {
                    Ok(t) => return Ok(t),
                    Err(sync::Disconnected) => return Err(Disconnected),
                    Err(sync::Empty) => None,
                },
            };

            if let Some(new_port) = port_or_empty {
                unsafe {
                    mem::swap(self.inner_mut(), new_port.inner_mut());
                }
            }

            // If we're already passed the deadline, and we're here without
            // data, return a timeout, else try again.
            if Instant::now() >= deadline {
                return Err(Timeout);
            }
        }
    }

    //函数式编程，用iterator来简化rx的动作
    pub fn iter(&self) -> Iter<'_, T> {
        Iter { rx: self }
    }

    //不会阻塞的iterator
    pub fn try_iter(&self) -> TryIter<'_, T> {
        TryIter { rx: self }
    }
}
```
针对Receiver的迭代器举例：
```rust
//只是为了函数式编程及利用Iterator的基础设施
pub struct Iter<'a, T: 'a> {
    rx: &'a Receiver<T>,
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = T;

    fn next(&mut self) -> Option<T> {
        self.rx.recv().ok()
    }
}

```
# RUST的RUNTIME
RUST的runtime及程序的主线程初始化      
路径：library/std/src/rt.rs：    
      library/std/src/unix/mod.rs       
      library/std/src/panic.rs   
      library/std/src/panicking.rs   
      library/std/src/panic/*.rs   
RUST程序execv以后，最初是由std::rt::lang_start进入RUST的RUNTIME    :
代码如下：
```rust
//RUST应用的代码入口点
fn lang_start<T: crate::process::Termination + 'static>(
    main: fn() -> T,
    argc: isize,
    argv: *const *const u8,
) -> isize {
    //调用了lang_start_internal
    let Ok(v) = lang_start_internal(
        //__rust_begin_short_backtrace(main)标识栈顶,同时也调用了main
        &move || crate::sys_common::backtrace::__rust_begin_short_backtrace(main).report().to_i32(),
        argc,
        argv,
    );
    v
}

fn lang_start_internal(
    main: &(dyn Fn() -> i32 + Sync + crate::panic::RefUnwindSafe),
    argc: isize,
    argv: *const *const u8,
) -> Result<isize, !> {
    use crate::{mem, panic};
    let rt_abort = move |e| {
        mem::forget(e);
        rtabort!("initialization or cleanup bug");
    };
    //完成执行main之前的准备,具体见后面的init函数，用catch_unwind捕获init函数执行中的panic信息
    panic::catch_unwind(move || unsafe { init(argc, argv) }).map_err(rt_abort)?;
    //执行main函数，同样，用catch_unwind捕获所有可能的panic信息
    let ret_code = panic::catch_unwind(move || panic::catch_unwind(main).unwrap_or(101) as isize)
        .map_err(move |e| {
            mem::forget(e);
            rtabort!("drop of the panic payload panicked");
        });
    //完成所有的清理工作,一样的catch_unwind
    panic::catch_unwind(cleanup).map_err(rt_abort)?;
    ret_code
}
```
进入main函数之前的初始化内容                    
```rust
//此函数在main函数之前被调用完成标准输入/输出/错误，线程栈保护等设置，
//然后控制权交给main
unsafe fn init(argc: isize, argv: *const *const u8) {
    unsafe {
        //见下面的代码分析，完成进入main的各项初始化
        sys::init(argc, argv);

        //以下是对主线程的线程runtime的初始化,可对比线程的spawn函数
        //设置主线程的栈保护
        let main_guard = sys::thread::guard::init();
        //设置当前的线程为主线程
        let thread = Thread::new(Some(rtunwrap!(Ok, CString::new("main"))));
        //设置栈保护地址与线程的信息, 使用了thread_local_key的方式使得此info仅与当前线程相关
        thread_info::set(main_guard, thread);
    }
}

//linux系统的上文sys::init实现
pub unsafe fn init(argc: isize, argv: *const *const u8) {
    // 见下文说明.
    sanitize_standard_fds();

    // 将 SIGPIPE 设置为ignore
    reset_sigpipe();

    //进程栈溢出初始化,系统调用sigaltstack()支持设置一个内存空间，当访问这个空间地址的时候
    //发送一个信号给进程，stack_overflow即利用这个机制完成了对当前线程的该信号的设置及处理
    //这个对所有线程的堆栈溢出的处理做了初始化
    stack_overflow::init();
    //对命令行的输入完成RUST的结构转化
    args::init(argc, argv);

    unsafe fn sanitize_standard_fds() {
        //仅linux
        {
            {
                use crate::sys::os::errno;
                //轮询stdin,stdout,stderr的文件描述符
                let pfds: &mut [_] = &mut [
                    libc::pollfd { fd: 0, events: 0, revents: 0 },
                    libc::pollfd { fd: 1, events: 0, revents: 0 },
                    libc::pollfd { fd: 2, events: 0, revents: 0 },
                ];
                //从poll结果获得文件描述符是否已经关闭
                while libc::poll(pfds.as_mut_ptr(), 3, 0) == -1 {
                    if errno() == libc::EINTR {
                        continue;
                    }
                    //此处说明未知错误需要退出
                    libc::abort();
                }
                for pfd in pfds {
                    if pfd.revents & libc::POLLNVAL == 0 {
                        //文件描述符已经打开
                        continue;
                    }
                    //文件描述符关闭, 则用/dev/null作为文件描述符，注意下面直接用str转换为CStr的
                    //代码,因为此循环的fd最小，所以下面这个open如果调用成功，返回的fd即为当前的///被关闭的fd.从而达到了重新将标准输入/输出/错误文件描述符打开的目的
                    if libc::open("/dev/null\0".as_ptr().cast(), libc::O_RDWR, 0) == -1 {
                        // 无法打开文件，则应退出程序
                        libc::abort();
                    }
                }
            } 
        }
    }

    //设置对SIGPIPE的处理为IGNORE
    unsafe fn reset_sigpipe() {
        rtassert!(signal(libc::SIGPIPE, libc::SIG_IGN) != libc::SIG_ERR);
    }
}
```
对panic的捕获函数：
```rust
//对f的panic做unwind操作并捕获
pub fn catch_unwind<F: FnOnce() -> R + UnwindSafe, R>(f: F) -> Result<R> {
    //编译器的try catch机制
    unsafe { panicking::r#try(f) }
}

//常用于前面已经调用过catch_unwind，但需要继续panic过程
pub fn resume_unwind(payload: Box<dyn Any + Send>) -> ! {
    panicking::rust_panic_without_hook(payload)
}
``` 
RUST的RUNTIME主要是完成一些安全机制及异常处理机制。了解RUNTIME可以使得我们对如何构建一个强健的，易于排查错误的应用有更深的了解。

# 标准库文件系统分析
## linux的操作系统相关的文件系统实现
```rust
// 操作系统无关界面接口结构
pub struct File(FileDesc);

// 这个宏仅仅在Linux下有效
macro_rules! cfg_has_statx {
    ($($then_tt:tt)*) => {
        cfg_if::cfg_if! {
                $($then_tt)*
        }
    };
    ($($block_inner:tt)*) => {
        {
            $($block_inner)*
        }
    };
}

//操作系统支持statx结构,删除了一些与linux无关的内容
cfg_has_statx! {{
    #[derive(Clone)]
    //文件属性
    pub struct FileAttr {
        stat: stat64,
        statx_extra_fields: Option<StatxExtraFields>,
    }

    #[derive(Clone)]
    struct StatxExtraFields {
        stx_mask: u32,
        stx_btime: libc::statx_timestamp,
    }

    //linux上，statx包含了最全面的信息
    unsafe fn try_statx(
        fd: c_int,
        path: *const c_char,
        flags: i32,
        mask: u32,
    ) -> Option<io::Result<FileAttr>> {
        use crate::sync::atomic::{AtomicU8, Ordering};

        syscall! {
            fn statx(
                fd: c_int,
                pathname: *const c_char,
                flags: c_int,
                mask: libc::c_uint,
                statxbuf: *mut libc::statx
            ) -> c_int
        }

        let mut buf: libc::statx = mem::zeroed();
        if let Err(err) = cvt(statx(fd, path, flags, mask, &mut buf)) {
            return Some(Err(err));
        }

        // 需要用stat64返回，所以做一下翻译.
        let mut stat: stat64 = mem::zeroed();
        // `c_ulong` on gnu-mips, `dev_t` otherwise
        stat.st_dev = libc::makedev(buf.stx_dev_major, buf.stx_dev_minor) as _;
        stat.st_ino = buf.stx_ino as libc::ino64_t;
        stat.st_nlink = buf.stx_nlink as libc::nlink_t;
        stat.st_mode = buf.stx_mode as libc::mode_t;
        stat.st_uid = buf.stx_uid as libc::uid_t;
        stat.st_gid = buf.stx_gid as libc::gid_t;
        stat.st_rdev = libc::makedev(buf.stx_rdev_major, buf.stx_rdev_minor) as _;
        stat.st_size = buf.stx_size as off64_t;
        stat.st_blksize = buf.stx_blksize as libc::blksize_t;
        stat.st_blocks = buf.stx_blocks as libc::blkcnt64_t;
        stat.st_atime = buf.stx_atime.tv_sec as libc::time_t;
        // `i64` on gnu-x86_64-x32, `c_ulong` otherwise.
        stat.st_atime_nsec = buf.stx_atime.tv_nsec as _;
        stat.st_mtime = buf.stx_mtime.tv_sec as libc::time_t;
        stat.st_mtime_nsec = buf.stx_mtime.tv_nsec as _;
        stat.st_ctime = buf.stx_ctime.tv_sec as libc::time_t;
        stat.st_ctime_nsec = buf.stx_ctime.tv_nsec as _;

        let extra = StatxExtraFields {
            stx_mask: buf.stx_mask,
            stx_btime: buf.stx_btime,
        };

        Some(Ok(FileAttr { stat, statx_extra_fields: Some(extra) }))
    }

}} 
// all DirEntry's will have a reference to this struct
struct InnerReadDir {
    dirp: Dir,
    root: PathBuf,
}

pub struct ReadDir {
    inner: Arc<InnerReadDir>,
}

struct Dir(*mut libc::DIR);

unsafe impl Send for Dir {}
unsafe impl Sync for Dir {}

pub struct DirEntry {
    dir: Arc<InnerReadDir>,
    entry: dirent64_min,
    // We need to store an owned copy of the entry name on platforms that use
    // readdir() (not readdir_r()), because a) struct dirent may use a flexible
    // array to store the name, b) it lives only until the next readdir() call.
    name: CString,
}

// Define a minimal subset of fields we need from `dirent64`, especially since
// we're not using the immediate `d_name` on these targets. Keeping this as an
// `entry` field in `DirEntry` helps reduce the `cfg` boilerplate elsewhere.
struct dirent64_min {
    d_ino: u64,
    #[cfg(not(any(target_os = "solaris", target_os = "illumos")))]
    d_type: u8,
}

#[derive(Clone, Debug)]
pub struct OpenOptions {
    // generic
    read: bool,
    write: bool,
    append: bool,
    truncate: bool,
    create: bool,
    create_new: bool,
    // system-specific
    custom_flags: i32,
    mode: mode_t,
}

#[derive(Clone, PartialEq, Eq, Debug)]
pub struct FilePermissions {
    mode: mode_t,
}

#[derive(Copy, Clone, PartialEq, Eq, Hash, Debug)]
pub struct FileType {
    mode: mode_t,
}

#[derive(Debug)]
pub struct DirBuilder {
    mode: mode_t,
}

cfg_has_statx! {{
    impl FileAttr {
        fn from_stat64(stat: stat64) -> Self {
            Self { stat, statx_extra_fields: None }
        }
    }
} else {
    impl FileAttr {
        fn from_stat64(stat: stat64) -> Self {
            Self { stat }
        }
    }
}}

impl FileAttr {
    pub fn size(&self) -> u64 {
        self.stat.st_size as u64
    }
    pub fn perm(&self) -> FilePermissions {
        FilePermissions { mode: (self.stat.st_mode as mode_t) }
    }

    pub fn file_type(&self) -> FileType {
        FileType { mode: self.stat.st_mode as mode_t }
    }
}

impl FileAttr {
    pub fn modified(&self) -> io::Result<SystemTime> {
        Ok(SystemTime::from(libc::timespec {
            tv_sec: self.stat.st_mtime as libc::time_t,
            tv_nsec: self.stat.st_mtime_nsec as _,
        }))
    }

    pub fn created(&self) -> io::Result<SystemTime> {
        cfg_has_statx! {
            if let Some(ext) = &self.statx_extra_fields {
                return if (ext.stx_mask & libc::STATX_BTIME) != 0 {
                    Ok(SystemTime::from(libc::timespec {
                        tv_sec: ext.stx_btime.tv_sec as libc::time_t,
                        tv_nsec: ext.stx_btime.tv_nsec as _,
                    }))
                } else {
                    Err(io::const_io_error!(
                        io::ErrorKind::Uncategorized,
                        "creation time is not available for the filesystem",
                    ))
                };
            }
        }

        Err(io::const_io_error!(
            io::ErrorKind::Unsupported,
            "creation time is not available on this platform \
                            currently",
        ))
    }
}

impl AsInner<stat64> for FileAttr {
    fn as_inner(&self) -> &stat64 {
        &self.stat
    }
}

impl FilePermissions {
    pub fn readonly(&self) -> bool {
        // check if any class (owner, group, others) has write permission
        self.mode & 0o222 == 0
    }

    pub fn set_readonly(&mut self, readonly: bool) {
        if readonly {
            // remove write permission for all classes; equivalent to `chmod a-w <file>`
            self.mode &= !0o222;
        } else {
            // add write permission for all classes; equivalent to `chmod a+w <file>`
            self.mode |= 0o222;
        }
    }
    pub fn mode(&self) -> u32 {
        self.mode as u32
    }
}

impl FileType {
    pub fn is_dir(&self) -> bool {
        self.is(libc::S_IFDIR)
    }
    pub fn is_file(&self) -> bool {
        self.is(libc::S_IFREG)
    }
    pub fn is_symlink(&self) -> bool {
        self.is(libc::S_IFLNK)
    }

    pub fn is(&self, mode: mode_t) -> bool {
        self.mode & libc::S_IFMT == mode
    }
}

impl FromInner<u32> for FilePermissions {
    fn from_inner(mode: u32) -> FilePermissions {
        FilePermissions { mode: mode as mode_t }
    }
}

impl fmt::Debug for ReadDir {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        // This will only be called from std::fs::ReadDir, which will add a "ReadDir()" frame.
        // Thus the result will be e g 'ReadDir("/home")'
        fmt::Debug::fmt(&*self.inner.root, f)
    }
}

impl Iterator for ReadDir {
    type Item = io::Result<DirEntry>;

    fn next(&mut self) -> Option<io::Result<DirEntry>> {
        unsafe {
            loop {
                // As of POSIX.1-2017, readdir() is not required to be thread safe; only
                // readdir_r() is. However, readdir_r() cannot correctly handle platforms
                // with unlimited or variable NAME_MAX.  Many modern platforms guarantee
                // thread safety for readdir() as long an individual DIR* is not accessed
                // concurrently, which is sufficient for Rust.
                super::os::set_errno(0);
                let entry_ptr = readdir64(self.inner.dirp.0);
                if entry_ptr.is_null() {
                    // null can mean either the end is reached or an error occurred.
                    // So we had to clear errno beforehand to check for an error now.
                    return match super::os::errno() {
                        0 => None,
                        e => Some(Err(Error::from_raw_os_error(e))),
                    };
                }

                // Only d_reclen bytes of *entry_ptr are valid, so we can't just copy the
                // whole thing (#93384).  Instead, copy everything except the name.
                let mut copy: dirent64 = mem::zeroed();
                // Can't dereference entry_ptr, so use the local entry to get
                // offsetof(struct dirent, d_name)
                let copy_bytes = &mut copy as *mut _ as *mut u8;
                let copy_name = &mut copy.d_name as *mut _ as *mut u8;
                let name_offset = copy_name.offset_from(copy_bytes) as usize;
                let entry_bytes = entry_ptr as *const u8;
                let entry_name = entry_bytes.add(name_offset);
                ptr::copy_nonoverlapping(entry_bytes, copy_bytes, name_offset);

                let entry = dirent64_min {
                    d_ino: copy.d_ino as u64,
                    #[cfg(not(any(target_os = "solaris", target_os = "illumos")))]
                    d_type: copy.d_type as u8,
                };

                let ret = DirEntry {
                    entry,
                    // d_name is guaranteed to be null-terminated.
                    name: CStr::from_ptr(entry_name as *const _).to_owned(),
                    dir: Arc::clone(&self.inner),
                };
                if ret.name_bytes() != b"." && ret.name_bytes() != b".." {
                    return Some(Ok(ret));
                }
            }
        }
    }

}

impl Drop for Dir {
    fn drop(&mut self) {
        let r = unsafe { libc::closedir(self.0) };
        debug_assert_eq!(r, 0);
    }
}

impl DirEntry {
    pub fn path(&self) -> PathBuf {
        self.dir.root.join(self.file_name_os_str())
    }

    pub fn file_name(&self) -> OsString {
        self.file_name_os_str().to_os_string()
    }

    pub fn metadata(&self) -> io::Result<FileAttr> {
        let fd = cvt(unsafe { dirfd(self.dir.dirp.0) })?;
        let name = self.name_cstr().as_ptr();

        cfg_has_statx! {
            if let Some(ret) = unsafe { try_statx(
                fd,
                name,
                libc::AT_SYMLINK_NOFOLLOW | libc::AT_STATX_SYNC_AS_STAT,
                libc::STATX_ALL,
            ) } {
                return ret;
            }
        }

        let mut stat: stat64 = unsafe { mem::zeroed() };
        cvt(unsafe { fstatat64(fd, name, &mut stat, libc::AT_SYMLINK_NOFOLLOW) })?;
        Ok(FileAttr::from_stat64(stat))
    }

    pub fn file_type(&self) -> io::Result<FileType> {
        match self.entry.d_type {
            libc::DT_CHR => Ok(FileType { mode: libc::S_IFCHR }),
            libc::DT_FIFO => Ok(FileType { mode: libc::S_IFIFO }),
            libc::DT_LNK => Ok(FileType { mode: libc::S_IFLNK }),
            libc::DT_REG => Ok(FileType { mode: libc::S_IFREG }),
            libc::DT_SOCK => Ok(FileType { mode: libc::S_IFSOCK }),
            libc::DT_DIR => Ok(FileType { mode: libc::S_IFDIR }),
            libc::DT_BLK => Ok(FileType { mode: libc::S_IFBLK }),
            _ => self.metadata().map(|m| m.file_type()),
        }
    }

    pub fn ino(&self) -> u64 {
        self.entry.d_ino as u64
    }

    fn name_bytes(&self) -> &[u8] {
        self.name_cstr().to_bytes()
    }

    fn name_cstr(&self) -> &CStr {
        &self.name
    }

    pub fn file_name_os_str(&self) -> &OsStr {
        OsStr::from_bytes(self.name_bytes())
    }
}

impl OpenOptions {
    pub fn new() -> OpenOptions {
        OpenOptions {
            // generic
            read: false,
            write: false,
            append: false,
            truncate: false,
            create: false,
            create_new: false,
            // system-specific
            custom_flags: 0,
            mode: 0o666,
        }
    }

    pub fn read(&mut self, read: bool) {
        self.read = read;
    }
    pub fn write(&mut self, write: bool) {
        self.write = write;
    }
    pub fn append(&mut self, append: bool) {
        self.append = append;
    }
    pub fn truncate(&mut self, truncate: bool) {
        self.truncate = truncate;
    }
    pub fn create(&mut self, create: bool) {
        self.create = create;
    }
    pub fn create_new(&mut self, create_new: bool) {
        self.create_new = create_new;
    }

    pub fn custom_flags(&mut self, flags: i32) {
        self.custom_flags = flags;
    }
    pub fn mode(&mut self, mode: u32) {
        self.mode = mode as mode_t;
    }

    fn get_access_mode(&self) -> io::Result<c_int> {
        match (self.read, self.write, self.append) {
            (true, false, false) => Ok(libc::O_RDONLY),
            (false, true, false) => Ok(libc::O_WRONLY),
            (true, true, false) => Ok(libc::O_RDWR),
            (false, _, true) => Ok(libc::O_WRONLY | libc::O_APPEND),
            (true, _, true) => Ok(libc::O_RDWR | libc::O_APPEND),
            (false, false, false) => Err(Error::from_raw_os_error(libc::EINVAL)),
        }
    }

    fn get_creation_mode(&self) -> io::Result<c_int> {
        match (self.write, self.append) {
            (true, false) => {}
            (false, false) => {
                if self.truncate || self.create || self.create_new {
                    return Err(Error::from_raw_os_error(libc::EINVAL));
                }
            }
            (_, true) => {
                if self.truncate && !self.create_new {
                    return Err(Error::from_raw_os_error(libc::EINVAL));
                }
            }
        }

        Ok(match (self.create, self.truncate, self.create_new) {
            (false, false, false) => 0,
            (true, false, false) => libc::O_CREAT,
            (false, true, false) => libc::O_TRUNC,
            (true, true, false) => libc::O_CREAT | libc::O_TRUNC,
            (_, _, true) => libc::O_CREAT | libc::O_EXCL,
        })
    }
}

impl File {
    pub fn open(path: &Path, opts: &OpenOptions) -> io::Result<File> {
        let path = cstr(path)?;
        File::open_c(&path, opts)
    }

    pub fn open_c(path: &CStr, opts: &OpenOptions) -> io::Result<File> {
        let flags = libc::O_CLOEXEC
            | opts.get_access_mode()?
            | opts.get_creation_mode()?
            | (opts.custom_flags as c_int & !libc::O_ACCMODE);
        // The third argument of `open64` is documented to have type `mode_t`. On
        // some platforms (like macOS, where `open64` is actually `open`), `mode_t` is `u16`.
        // However, since this is a variadic function, C integer promotion rules mean that on
        // the ABI level, this still gets passed as `c_int` (aka `u32` on Unix platforms).
        let fd = cvt_r(|| unsafe { open64(path.as_ptr(), flags, opts.mode as c_int) })?;
        Ok(File(unsafe { FileDesc::from_raw_fd(fd) }))
    }

    pub fn file_attr(&self) -> io::Result<FileAttr> {
        let fd = self.as_raw_fd();

        cfg_has_statx! {
            if let Some(ret) = unsafe { try_statx(
                fd,
                b"\0" as *const _ as *const c_char,
                libc::AT_EMPTY_PATH | libc::AT_STATX_SYNC_AS_STAT,
                libc::STATX_ALL,
            ) } {
                return ret;
            }
        }

        let mut stat: stat64 = unsafe { mem::zeroed() };
        cvt(unsafe { fstat64(fd, &mut stat) })?;
        Ok(FileAttr::from_stat64(stat))
    }

    pub fn fsync(&self) -> io::Result<()> {
        cvt_r(|| unsafe { os_fsync(self.as_raw_fd()) })?;
        return Ok(());

        unsafe fn os_fsync(fd: c_int) -> c_int {
            libc::fsync(fd)
        }
    }

    pub fn datasync(&self) -> io::Result<()> {
        cvt_r(|| unsafe { os_datasync(self.as_raw_fd()) })?;
        return Ok(());

        unsafe fn os_datasync(fd: c_int) -> c_int {
            libc::fdatasync(fd)
        }
    }

    pub fn truncate(&self, size: u64) -> io::Result<()> {
        use crate::convert::TryInto;
        let size: off64_t =
            size.try_into().map_err(|e| io::Error::new(io::ErrorKind::InvalidInput, e))?;
        cvt_r(|| unsafe { ftruncate64(self.as_raw_fd(), size) }).map(drop)
    }

    pub fn read(&self, buf: &mut [u8]) -> io::Result<usize> {
        self.0.read(buf)
    }

    pub fn read_vectored(&self, bufs: &mut [IoSliceMut<'_>]) -> io::Result<usize> {
        self.0.read_vectored(bufs)
    }

    #[inline]
    pub fn is_read_vectored(&self) -> bool {
        self.0.is_read_vectored()
    }

    pub fn read_at(&self, buf: &mut [u8], offset: u64) -> io::Result<usize> {
        self.0.read_at(buf, offset)
    }

    pub fn read_buf(&self, buf: &mut ReadBuf<'_>) -> io::Result<()> {
        self.0.read_buf(buf)
    }

    pub fn write(&self, buf: &[u8]) -> io::Result<usize> {
        self.0.write(buf)
    }

    pub fn write_vectored(&self, bufs: &[IoSlice<'_>]) -> io::Result<usize> {
        self.0.write_vectored(bufs)
    }

    #[inline]
    pub fn is_write_vectored(&self) -> bool {
        self.0.is_write_vectored()
    }

    pub fn write_at(&self, buf: &[u8], offset: u64) -> io::Result<usize> {
        self.0.write_at(buf, offset)
    }

    pub fn flush(&self) -> io::Result<()> {
        Ok(())
    }

    pub fn seek(&self, pos: SeekFrom) -> io::Result<u64> {
        let (whence, pos) = match pos {
            // Casting to `i64` is fine, too large values will end up as
            // negative which will cause an error in `lseek64`.
            SeekFrom::Start(off) => (libc::SEEK_SET, off as i64),
            SeekFrom::End(off) => (libc::SEEK_END, off),
            SeekFrom::Current(off) => (libc::SEEK_CUR, off),
        };
        let n = cvt(unsafe { lseek64(self.as_raw_fd(), pos, whence) })?;
        Ok(n as u64)
    }

    pub fn duplicate(&self) -> io::Result<File> {
        self.0.duplicate().map(File)
    }

    pub fn set_permissions(&self, perm: FilePermissions) -> io::Result<()> {
        cvt_r(|| unsafe { libc::fchmod(self.as_raw_fd(), perm.mode) })?;
        Ok(())
    }
}

impl DirBuilder {
    pub fn new() -> DirBuilder {
        DirBuilder { mode: 0o777 }
    }

    pub fn mkdir(&self, p: &Path) -> io::Result<()> {
        let p = cstr(p)?;
        cvt(unsafe { libc::mkdir(p.as_ptr(), self.mode) })?;
        Ok(())
    }

    pub fn set_mode(&mut self, mode: u32) {
        self.mode = mode as mode_t;
    }
}

fn cstr(path: &Path) -> io::Result<CString> {
    Ok(CString::new(path.as_os_str().as_bytes())?)
}

impl AsInner<FileDesc> for File {
    fn as_inner(&self) -> &FileDesc {
        &self.0
    }
}

impl AsInnerMut<FileDesc> for File {
    fn as_inner_mut(&mut self) -> &mut FileDesc {
        &mut self.0
    }
}

impl IntoInner<FileDesc> for File {
    fn into_inner(self) -> FileDesc {
        self.0
    }
}

impl FromInner<FileDesc> for File {
    fn from_inner(file_desc: FileDesc) -> Self {
        Self(file_desc)
    }
}

impl AsFd for File {
    fn as_fd(&self) -> BorrowedFd<'_> {
        self.0.as_fd()
    }
}

impl AsRawFd for File {
    fn as_raw_fd(&self) -> RawFd {
        self.0.as_raw_fd()
    }
}

impl IntoRawFd for File {
    fn into_raw_fd(self) -> RawFd {
        self.0.into_raw_fd()
    }
}

impl FromRawFd for File {
    unsafe fn from_raw_fd(raw_fd: RawFd) -> Self {
        Self(FromRawFd::from_raw_fd(raw_fd))
    }
}

impl fmt::Debug for File {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        #[cfg(any(target_os = "linux", target_os = "netbsd"))]
        fn get_path(fd: c_int) -> Option<PathBuf> {
            let mut p = PathBuf::from("/proc/self/fd");
            p.push(&fd.to_string());
            readlink(&p).ok()
        }

        fn get_mode(fd: c_int) -> Option<(bool, bool)> {
            let mode = unsafe { libc::fcntl(fd, libc::F_GETFL) };
            if mode == -1 {
                return None;
            }
            match mode & libc::O_ACCMODE {
                libc::O_RDONLY => Some((true, false)),
                libc::O_RDWR => Some((true, true)),
                libc::O_WRONLY => Some((false, true)),
                _ => None,
            }
        }

        let fd = self.as_raw_fd();
        let mut b = f.debug_struct("File");
        b.field("fd", &fd);
        if let Some(path) = get_path(fd) {
            b.field("path", &path);
        }
        if let Some((read, write)) = get_mode(fd) {
            b.field("read", &read).field("write", &write);
        }
        b.finish()
    }
}

pub fn readdir(p: &Path) -> io::Result<ReadDir> {
    let root = p.to_path_buf();
    let p = cstr(p)?;
    unsafe {
        let ptr = libc::opendir(p.as_ptr());
        if ptr.is_null() {
            Err(Error::last_os_error())
        } else {
            let inner = InnerReadDir { dirp: Dir(ptr), root };
            Ok(ReadDir {
                inner: Arc::new(inner),
            })
        }
    }
}

pub fn unlink(p: &Path) -> io::Result<()> {
    let p = cstr(p)?;
    cvt(unsafe { libc::unlink(p.as_ptr()) })?;
    Ok(())
}

pub fn rename(old: &Path, new: &Path) -> io::Result<()> {
    let old = cstr(old)?;
    let new = cstr(new)?;
    cvt(unsafe { libc::rename(old.as_ptr(), new.as_ptr()) })?;
    Ok(())
}

pub fn set_perm(p: &Path, perm: FilePermissions) -> io::Result<()> {
    let p = cstr(p)?;
    cvt_r(|| unsafe { libc::chmod(p.as_ptr(), perm.mode) })?;
    Ok(())
}

pub fn rmdir(p: &Path) -> io::Result<()> {
    let p = cstr(p)?;
    cvt(unsafe { libc::rmdir(p.as_ptr()) })?;
    Ok(())
}

pub fn readlink(p: &Path) -> io::Result<PathBuf> {
    let c_path = cstr(p)?;
    let p = c_path.as_ptr();

    let mut buf = Vec::with_capacity(256);

    loop {
        let buf_read =
            cvt(unsafe { libc::readlink(p, buf.as_mut_ptr() as *mut _, buf.capacity()) })? as usize;

        unsafe {
            buf.set_len(buf_read);
        }

        if buf_read != buf.capacity() {
            buf.shrink_to_fit();

            return Ok(PathBuf::from(OsString::from_vec(buf)));
        }

        // Trigger the internal buffer resizing logic of `Vec` by requiring
        // more space than the current capacity. The length is guaranteed to be
        // the same as the capacity due to the if statement above.
        buf.reserve(1);
    }
}

pub fn symlink(original: &Path, link: &Path) -> io::Result<()> {
    let original = cstr(original)?;
    let link = cstr(link)?;
    cvt(unsafe { libc::symlink(original.as_ptr(), link.as_ptr()) })?;
    Ok(())
}

pub fn link(original: &Path, link: &Path) -> io::Result<()> {
    let original = cstr(original)?;
    let link = cstr(link)?;
    cfg_if::cfg_if! {
        {
            // Where we can, use `linkat` instead of `link`; see the comment above
            // this one for details on why.
            cvt(unsafe { libc::linkat(libc::AT_FDCWD, original.as_ptr(), libc::AT_FDCWD, link.as_ptr(), 0) })?;
        }
    }
    Ok(())
}

pub fn stat(p: &Path) -> io::Result<FileAttr> {
    let p = cstr(p)?;

    cfg_has_statx! {
        if let Some(ret) = unsafe { try_statx(
            libc::AT_FDCWD,
            p.as_ptr(),
            libc::AT_STATX_SYNC_AS_STAT,
            libc::STATX_ALL,
        ) } {
            return ret;
        }
    }

    let mut stat: stat64 = unsafe { mem::zeroed() };
    cvt(unsafe { stat64(p.as_ptr(), &mut stat) })?;
    Ok(FileAttr::from_stat64(stat))
}

pub fn lstat(p: &Path) -> io::Result<FileAttr> {
    let p = cstr(p)?;

    cfg_has_statx! {
        if let Some(ret) = unsafe { try_statx(
            libc::AT_FDCWD,
            p.as_ptr(),
            libc::AT_SYMLINK_NOFOLLOW | libc::AT_STATX_SYNC_AS_STAT,
            libc::STATX_ALL,
        ) } {
            return ret;
        }
    }

    let mut stat: stat64 = unsafe { mem::zeroed() };
    cvt(unsafe { lstat64(p.as_ptr(), &mut stat) })?;
    Ok(FileAttr::from_stat64(stat))
}

pub fn canonicalize(p: &Path) -> io::Result<PathBuf> {
    let path = CString::new(p.as_os_str().as_bytes())?;
    let buf;
    unsafe {
        let r = libc::realpath(path.as_ptr(), ptr::null_mut());
        if r.is_null() {
            return Err(io::Error::last_os_error());
        }
        buf = CStr::from_ptr(r).to_bytes().to_vec();
        libc::free(r as *mut _);
    }
    Ok(PathBuf::from(OsString::from_vec(buf)))
}

fn open_from(from: &Path) -> io::Result<(crate::fs::File, crate::fs::Metadata)> {
    use crate::fs::File;
    use crate::sys_common::fs::NOT_FILE_ERROR;

    let reader = File::open(from)?;
    let metadata = reader.metadata()?;
    if !metadata.is_file() {
        return Err(NOT_FILE_ERROR);
    }
    Ok((reader, metadata))
}

fn open_to_and_set_permissions(
    to: &Path,
    reader_metadata: crate::fs::Metadata,
) -> io::Result<(crate::fs::File, crate::fs::Metadata)> {
    use crate::fs::OpenOptions;
    use crate::os::unix::fs::{OpenOptionsExt, PermissionsExt};

    let perm = reader_metadata.permissions();
    let writer = OpenOptions::new()
        // create the file with the correct mode right away
        .mode(perm.mode())
        .write(true)
        .create(true)
        .truncate(true)
        .open(to)?;
    let writer_metadata = writer.metadata()?;
    if writer_metadata.is_file() {
        // Set the correct file permissions, in case the file already existed.
        // Don't set the permissions on already existing non-files like
        // pipes/FIFOs or device nodes.
        writer.set_permissions(perm)?;
    }
    Ok((writer, writer_metadata))
}

pub fn copy(from: &Path, to: &Path) -> io::Result<u64> {
    let (mut reader, reader_metadata) = open_from(from)?;
    let max_len = u64::MAX;
    let (mut writer, _) = open_to_and_set_permissions(to, reader_metadata)?;

    use super::kernel_copy::{copy_regular_files, CopyResult};

    match copy_regular_files(reader.as_raw_fd(), writer.as_raw_fd(), max_len) {
        CopyResult::Ended(bytes) => Ok(bytes),
        CopyResult::Error(e, _) => Err(e),
        CopyResult::Fallback(written) => match io::copy::generic_copy(&mut reader, &mut writer) {
            Ok(bytes) => Ok(bytes + written),
            Err(e) => Err(e),
        },
    }
}

pub fn chown(path: &Path, uid: u32, gid: u32) -> io::Result<()> {
    let path = cstr(path)?;
    cvt(unsafe { libc::chown(path.as_ptr(), uid as libc::uid_t, gid as libc::gid_t) })?;
    Ok(())
}

pub fn fchown(fd: c_int, uid: u32, gid: u32) -> io::Result<()> {
    cvt(unsafe { libc::fchown(fd, uid as libc::uid_t, gid as libc::gid_t) })?;
    Ok(())
}

pub fn lchown(path: &Path, uid: u32, gid: u32) -> io::Result<()> {
    let path = cstr(path)?;
    cvt(unsafe { libc::lchown(path.as_ptr(), uid as libc::uid_t, gid as libc::gid_t) })?;
    Ok(())
}

pub fn chroot(dir: &Path) -> io::Result<()> {
    let dir = cstr(dir)?;
    cvt(unsafe { libc::chroot(dir.as_ptr()) })?;
    Ok(())
}

pub use remove_dir_impl::remove_dir_all;

mod remove_dir_impl {
    use super::{cstr, lstat, Dir, DirEntry, InnerReadDir, ReadDir};
    use crate::ffi::CStr;
    use crate::io;
    use crate::os::unix::io::{AsRawFd, FromRawFd, IntoRawFd};
    use crate::os::unix::prelude::{OwnedFd, RawFd};
    use crate::path::{Path, PathBuf};
    use crate::sync::Arc;
    use crate::sys::{cvt, cvt_r};

    use libc::{fdopendir, openat, unlinkat};

    pub fn openat_nofollow_dironly(parent_fd: Option<RawFd>, p: &CStr) -> io::Result<OwnedFd> {
        let fd = cvt_r(|| unsafe {
            openat(
                parent_fd.unwrap_or(libc::AT_FDCWD),
                p.as_ptr(),
                libc::O_CLOEXEC | libc::O_RDONLY | libc::O_NOFOLLOW | libc::O_DIRECTORY,
            )
        })?;
        Ok(unsafe { OwnedFd::from_raw_fd(fd) })
    }

    fn fdreaddir(dir_fd: OwnedFd) -> io::Result<(ReadDir, RawFd)> {
        let ptr = unsafe { fdopendir(dir_fd.as_raw_fd()) };
        if ptr.is_null() {
            return Err(io::Error::last_os_error());
        }
        let dirp = Dir(ptr);
        // file descriptor is automatically closed by libc::closedir() now, so give up ownership
        let new_parent_fd = dir_fd.into_raw_fd();
        // a valid root is not needed because we do not call any functions involving the full path
        // of the DirEntrys.
        let dummy_root = PathBuf::new();
        Ok((
            ReadDir {
                inner: Arc::new(InnerReadDir { dirp, root: dummy_root }),
            },
            new_parent_fd,
        ))
    }

    fn is_dir(ent: &DirEntry) -> Option<bool> {
        match ent.entry.d_type {
            libc::DT_UNKNOWN => None,
            libc::DT_DIR => Some(true),
            _ => Some(false),
        }
    }

    fn remove_dir_all_recursive(parent_fd: Option<RawFd>, path: &CStr) -> io::Result<()> {
        // try opening as directory
        let fd = match openat_nofollow_dironly(parent_fd, &path) {
            Err(err) if err.raw_os_error() == Some(libc::ENOTDIR) => {
                // not a directory - don't traverse further
                return match parent_fd {
                    // unlink...
                    Some(parent_fd) => {
                        cvt(unsafe { unlinkat(parent_fd, path.as_ptr(), 0) }).map(drop)
                    }
                    // ...unless this was supposed to be the deletion root directory
                    None => Err(err),
                };
            }
            result => result?,
        };

        // open the directory passing ownership of the fd
        let (dir, fd) = fdreaddir(fd)?;
        for child in dir {
            let child = child?;
            let child_name = child.name_cstr();
            match is_dir(&child) {
                Some(true) => {
                    remove_dir_all_recursive(Some(fd), child_name)?;
                }
                Some(false) => {
                    cvt(unsafe { unlinkat(fd, child_name.as_ptr(), 0) })?;
                }
                None => {
                    // POSIX specifies that calling unlink()/unlinkat(..., 0) on a directory can succeed
                    // if the process has the appropriate privileges. This however can causing orphaned
                    // directories requiring an fsck e.g. on Solaris and Illumos. So we try recursing
                    // into it first instead of trying to unlink() it.
                    remove_dir_all_recursive(Some(fd), child_name)?;
                }
            }
        }

        // unlink the directory after removing its contents
        cvt(unsafe {
            unlinkat(parent_fd.unwrap_or(libc::AT_FDCWD), path.as_ptr(), libc::AT_REMOVEDIR)
        })?;
        Ok(())
    }

    fn remove_dir_all_modern(p: &Path) -> io::Result<()> {
        // We cannot just call remove_dir_all_recursive() here because that would not delete a passed
        // symlink. No need to worry about races, because remove_dir_all_recursive() does not recurse
        // into symlinks.
        let attr = lstat(p)?;
        if attr.file_type().is_symlink() {
            crate::fs::remove_file(p)
        } else {
            remove_dir_all_recursive(None, &cstr(p)?)
        }
    }

    pub fn remove_dir_all(p: &Path) -> io::Result<()> {
        remove_dir_all_modern(p)
    }
}

```
# RUST不采用类的理由
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

# 栈和堆

## 为什么有栈及堆
栈存在的必要性：
在程序中调用一个函数代码实际做如下工作：
1. 函数调用的返回指令地址入栈 (栈指针减)
2. 函数调用的参数入栈(假设没有参数使用寄存器)
3. 指令跳转到函数入口地址
4. 函数对每一个局部变量在栈内分配地址，栈指针减
5. 函数利用栈指针+偏移量访问每一个变量及参数
6. 函数返回时，栈指针加回，获取返回指令地址，指令跳转到上一级函数

以上能够看出栈的主要解决的问题，这基本上是最经济及性能最高的方案，早期内存及寻址都是很贵的。      
这个设计方案实际上有很大问题，直接导致了利用栈溢出获取系统控制权成了最牛的黑客技术之一。 
 

堆存在的必要性：
1. 希望变量能够跨越函数
2. 变量的生命周期是动态的，希望经济的使用内存