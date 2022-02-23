# RUST标准库的基础Trait

## RUST中直接针对泛型实现方法定义和Trait
实现方法或Trait时，可以针对泛型来做实现，并可以给泛型添加限制。可以添加的泛型限制有：无限制，泛型的原生指针（可变/不可变）类型，泛型的引用(可变/不可变)类型，泛型的数组类型，泛型的切片类型，泛型的切片引用（可变/不可变）类型，泛型的元组（可变/不可变）类型，函数类型， 需实现特定的Trait限制等。
对某个具有泛型的类型结构体实现方法和Trait时，也可以对类型结构体的泛型做出如上所述的限制。
举例说明：
```rust
//为所有的类型实现了Borrow<T> Trait
impl<T: ?Sized> Borrow<T> for T {
    fn borrow(&self) -> &T {
        self
    }
}
//为所有的引用类型实现Receiver Trait
impl<T: ?Sized> Receiver for &T {}

//为所有可变引用类型实现Receiver Trait
impl<T: ?Sized> Receiver for &mut T {}

//为所有原生不可变指针类型实现Clone Trait
impl<T: ?Sized> Clone for *const T {
     fn clone(&self) -> Self {
         *self
     }
 }
//为原生可变指针类型实现Clone Trait
impl<T: ?Sized> Clone for *mut T {
     fn clone(&self) -> Self {
         *self
     }
}

//为切片类型[T]实现AsRef<[T]> Trait
impl<T> AsRef<[T]> for [T] {
    fn as_ref(&self) -> &[T] {
        self
    }
}

// 切片的方法 具体实现的摘要
// [T;N]代表数组类型，[T]代表切片类型，
// 下面是针对所有切片类型的统一方法。
impl<T> [T] {
    pub const fn len(&self) -> usize {
        //使用* const T的元数据获得len
        unsafe { crate::ptr::PtrRepr { const_ptr: self }.components.metadata }
    }

    pub const fn is_empty(&self) -> bool {
        self.len() == 0
    }
    
    //以下代码展示了RUST代码的极简洁及易理解
    pub const fn first(&self) -> Option<&T> {
        //从以下代码可以学习及体会RUST的编码技巧。利用绑定语法完成
        //空切片的判断。同时，因为不能从切片中转移所有权，
        // let [first, ..] = self 实际上是 let [ref x, ..] = self的一种简化表示
        if let [first, ..] = self { Some(first) } else { None }
    }

    pub const fn split_first(&self) -> Option<(&T, &[T])> {
        //利用绑定语义，极简高效的完成数组分离，秒杀其他语言
        if let [first, tail @ ..] = self { Some((first, tail)) } else { None }
    }

    ...
    ...
}

//原生指针 具体方法的片段
impl<T: ?Sized> *const T {
    pub const fn is_null(self) -> bool {
        //这里要注意*const T是有元数据的，转化为*const u8后元数据为空，只比较address的部分
        (self as *const u8).guaranteed_eq(null())
    }

    /// Casts to a pointer of another type.
    pub const fn cast<U>(self) -> *const U {
        //注意"_"的使用技巧
        self as _
    }

    pub unsafe fn as_ref<'a>(self) -> Option<&'a T> {
        // else后直接将原生指针转化为&T，调用的代码需要确实的保证安全性
        // *const T指向的内存不应在返回的&T生命周期结束前被释放
        if self.is_null() { None } else { unsafe { Some(&*self) } }
    }
    ...
    ...
}

//原生数组指针 具体方法实现片段。需要注意，原生数组指针也是原生指针，
//此处不应该再实现 *const T的同名函数。
impl<T> *const [T] {
    pub const fn len(self) -> usize {
        metadata(self)
    }

    pub const fn as_ptr(self) -> *const T {
        //注意，转换后metadata函数会返回T类型的对应元数据，不再是切片长度值
        self as *const T
    }
    ...
    ...
}
//如果T实现了Default Trait, 则为Box<T>实现Default Trait
impl<T: Default> Default for Box<T> {
    /// Creates a `Box<T>`, with the `Default` value for T.
    fn default() -> Self {
        box T::default()
    }
}
```
有效使用针对限制的泛型实现方法和Trait可以大幅的减少代码，受限制的泛型是RUST提供的强大的泛型语法武器。

## 编译器内置Trait代码分析
代码路径：
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\core\src\marker.rs
源代码主要部分是标准库指南文档。这里简要给出一些需注意的点
```rust
//Send Trait
pub unsafe auto trait Send {
    // empty.
}
//原生指针不支持Send Trait实现，利用"!"表示明确的对Trait的不支持
impl<T: ?Sized> !Send for *const T {}
impl<T: ?Sized> !Send for *mut T {}

mod impls {
    // 实现Sync的类型的类型不可变引用支持Send
    unsafe impl<T: Sync + ?Sized> Send for &T {}
    // 实现Send的类型的类型可变引用支持 Send
    unsafe impl<T: Send + ?Sized> Send for &mut T {}
}

// Sync Trait
pub unsafe auto trait Sync {
    // Empty
}
//原生指针不支持Sync Trait
impl<T: ?Sized> !Sync for *const T {}
impl<T: ?Sized> !Sync for *mut T {}

pub trait Sized {
    // Empty.
}

//如果一个Sized的类型要强制转换为动态大小类型，那必须实现Unsize Trait
//例如 [T;N] 实现了 Unsize<[T]>
pub trait Unsize<T: ?Sized> {
    // Empty.
}

//模式匹配表达式匹配时编译器需要使用的Trait，如果一个结构实现了PartialEq，该Trait会自动被实现。
//推测这个结构应该是内存按位比较
pub trait StructuralPartialEq {
    // Empty.
}

//主要用于模式匹配，如果一个结构实现了Eq, 该Trait会自动被实现。
pub trait StructuralEq {
    // Empty.
}

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

//PhantomData被用来在类型结构体重做一个标记，标记某结构中需要但不必有实体的另一类型属性
//PhantomData具体阐述请参考标准库文档。 
//PhantomData是个单元结构体，单元结构体的变量名就是单元结构体的类型名。
//所以使用的时候直接使用PhantomData即可，编译器会将泛型的类型实例化信息自动带入PhantomData中
pub struct PhantomData<T: ?Sized>;
```



## ops 运算符 Trait 代码分析
代码路径如下：
%USER%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library\core\src\ops\*.rs

RUST中，所有的运算符号都可以重载。
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
//"==" "!="的实现Trait，实现该Trait的类型可以只部分相等
pub trait PartialEq<Rhs: ?Sized = Self> {
    /// “==” 重载方法
    fn eq(&self, other: &Rhs) -> bool;

    ///`!=` 重载方法
    fn ne(&self, other: &Rhs) -> bool {
        !self.eq(other)
    }
}

/// 实现Derive属性的过程宏
pub macro PartialEq($item:item) {
    /* compiler built-in */
}

//实现该Trait的类型必须完全相等
pub trait Eq: PartialEq<Self> {
    fn assert_receiver_is_total_eq(&self) {}
}

/// 实现Derive属性的过程宏
pub macro Eq($item:item) {
    /* compiler built-in */
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
    pub const fn is_eq(self) -> bool {
        matches!(self, Equal)
    }
    
    //以下函数体实现略
    pub const fn is_ne(self) -> bool 
    pub const fn is_lt(self) -> bool 
    pub const fn is_gt(self) -> bool 
    pub const fn is_le(self) -> bool
    pub const fn is_ge(self) -> bool

    //做反转操作
    pub const fn reverse(self) -> Ordering {
        match self {
            Less => Greater,
            Equal => Equal,
            Greater => Less,
        }
    }

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


// "<" ">" ">=" "<=" 运算符重载结构
pub trait PartialOrd<Rhs: ?Sized = Self>: PartialEq<Rhs> {
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
}

//实现Derive的过程宏
pub macro PartialOrd($item:item) {
    /* compiler built-in */
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

//以下代码易理解，分析略
pub trait Ord: Eq + PartialOrd<Self> {
    fn cmp(&self, other: &Self) -> Ordering;

    fn max(self, other: Self) -> Self
    where
        Self: Sized,
    {
        max_by(self, other, Ord::cmp)
    }

    fn min(self, other: Self) -> Self
    where
        Self: Sized,
    {
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
//实现Drive 属性过程宏
pub macro Ord($item:item) {
    /* compiler built-in */
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

//对于实现了PartialOrd的类型实现一个Ord的反转，这是一个
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

impl<T: Ord> Ord for Reverse<T> {
    fn cmp(&self, other: &Reverse<T>) -> Ordering {
        other.0.cmp(&self.0)
    }

    //其他方法，略
    ...
}

impl<T: Clone> Clone for Reverse<T> {
    fn clone(&self) -> Reverse<T> {
        Reverse(self.0.clone())
    }

    fn clone_from(&mut self, other: &Self) {
        self.0.clone_from(&other.0)
    }
}

// 具体的实现宏 
mod impls {
    use crate::cmp::Ordering::{self, Equal, Greater, Less};
    use crate::hint::unreachable_unchecked;
    
    //PartialEq在原生类型上的实现
    macro_rules! partial_eq_impl {
        ($($t:ty)*) => ($(
            impl PartialEq for $t {
                fn eq(&self, other: &$t) -> bool { (*self) == (*other) }
                fn ne(&self, other: &$t) -> bool { (*self) != (*other) }
            }
        )*)
    }

    impl PartialEq for () {
        fn eq(&self, _other: &()) -> bool {
            true
        }
        fn ne(&self, _other: &()) -> bool {
            false
        }
    }

    partial_eq_impl! {
        bool char usize u8 u16 u32 u64 u128 isize i8 i16 i32 i64 i128 f32 f64
    }

    // Eq，PartialOrd, Ord在原生类型上的实现，略
    ...
    ...
    
    impl PartialEq for ! {
        fn eq(&self, _: &!) -> bool {
            *self
        }
    }

    impl Eq for ! {}

    impl PartialOrd for ! {
        fn partial_cmp(&self, _: &!) -> Option<Ordering> {
            *self
        }
    }

    impl Ord for ! {
        fn cmp(&self, _: &!) -> Ordering {
            *self
        }
    }

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
    impl<A: ?Sized, B: ?Sized> PartialOrd<&B> for &A
    where
        A: PartialOrd<B>,
    {
        fn partial_cmp(&self, other: &&B) -> Option<Ordering> {
            PartialOrd::partial_cmp(*self, *other)
        }
        fn lt(&self, other: &&B) -> bool {
            PartialOrd::lt(*self, *other)
        }
        ...
        ...
    }

    //如果A实现了Ord Trait, 对&A实现Ord Trait
    impl<A: ?Sized> Ord for &A
    where
        A: Ord,
    {
        fn cmp(&self, other: &Self) -> Ordering {
            Ord::cmp(*self, *other)
        }
    }
    impl<A: ?Sized> Eq for &A where A: Eq {}

    impl<A: ?Sized, B: ?Sized> PartialEq<&mut B> for &mut A
    where
        A: PartialEq<B>,
    {
        fn eq(&self, other: &&mut B) -> bool {
            PartialEq::eq(*self, *other)
        }
        fn ne(&self, other: &&mut B) -> bool {
            PartialEq::ne(*self, *other)
        }
    }

    impl<A: ?Sized, B: ?Sized> PartialOrd<&mut B> for &mut A
    where
        A: PartialOrd<B>,
    {
        fn partial_cmp(&self, other: &&mut B) -> Option<Ordering> {
            PartialOrd::partial_cmp(*self, *other)
        }
        fn lt(&self, other: &&mut B) -> bool {
            PartialOrd::lt(*self, *other)
        }
        ...
        ...
    }
    impl<A: ?Sized> Ord for &mut A
    where
        A: Ord,
    {
        fn cmp(&self, other: &Self) -> Ordering {
            Ord::cmp(*self, *other)
        }
    }
    impl<A: ?Sized> Eq for &mut A where A: Eq {}

    impl<A: ?Sized, B: ?Sized> PartialEq<&mut B> for &A
    where
        A: PartialEq<B>,
    {
        fn eq(&self, other: &&mut B) -> bool {
            PartialEq::eq(*self, *other)
        }
        fn ne(&self, other: &&mut B) -> bool {
            PartialEq::ne(*self, *other)
        }
    }

    impl<A: ?Sized, B: ?Sized> PartialEq<&B> for &mut A
    where
        A: PartialEq<B>,
    {
        fn eq(&self, other: &&B) -> bool {
            PartialEq::eq(*self, *other)
        }
        fn ne(&self, other: &&B) -> bool {
            PartialEq::ne(*self, *other)
        }
    }
}

```
以上较完整的给出了关系运算Trait的代码，可以看到，RUST标准库除了对原生类型做了Trait的实现，也针对受限制的泛型尽可能的做了关系运算符 Trait的实现，以便最大的减少后继的开发量。程序员需要精通RUST的标准库已经针对那些泛型类型做好了实现，避免再重复的造轮子。

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

    /// 从Self::Output返回值类型中获得原类型的值
    /// 函数必须符合下面代码的原则，
    /// `Try::from_output(x).branch() --> ControlFlow::Continue(x)`.
    /// 例子：
    /// ```
    /// #![feature(try_trait_v2)]
    /// use std::ops::Try;
    ///
    /// assert_eq!(<Result<_, String> as Try>::from_output(3), Ok(3));
    /// assert_eq!(<Option<_> as Try>::from_output(4), Some(4));
    /// assert_eq!(
    ///     <std::ops::ControlFlow<String, _> as Try>::from_output(5),
    ///     std::ops::ControlFlow::Continue(5),
    /// );
    fn from_output(output: Self::Output) -> Self;

    /// branch函数会返回ControlFlow，据此决定流程继续还是提前返回
    /// 例子：
    /// ```p
    /// #![feature(try_trait_v2)]
    /// use std::ops::{ControlFlow, Try};
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
    /// 该函数从提前返回的值中获取原始值
    ///
    /// 此函数必须符合下面代码的原则
    /// `FromResidual::from_residual(r).branch() --> ControlFlow::Break(r)`.
    /// 例子：
    /// ```
    /// #![feature(try_trait_v2)]
    /// use std::ops::{ControlFlow, FromResidual};
    ///
    /// assert_eq!(Result::<String, i64>::from_residual(Err(3_u8)), Err(3));
    /// assert_eq!(Option::<String>::from_residual(None), None);
    /// assert_eq!(
    ///     ControlFlow::<_, String>::from_residual(ControlFlow::Break(5)),
    ///     ControlFlow::Break(5),
    /// );
    /// ```
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
    /// Move on to the next phase of the operation as normal.
    Continue(C),
    /// Exit the operation without running subsequent phases.
    Break(B),
    // Yes, the order of the variants doesn't match the type parameters.
    // They're in this order so that `ControlFlow<A, B>` <-> `Result<B, A>`
    // is a no-op conversion in the `Try` implementation.
}

impl<B, C> ops::Try for ControlFlow<B, C> {
    type Output = C;
    // convert::Infallible表示类型不会被使用，可认为是忽略类型,
    //对于Residual只会用ControlFlow::Break(B), 所以用Infallible明确说明不会用到此类型
    type Residual = ControlFlow<B, convert::Infallible>;

    fn from_output(output: Self::Output) -> Self {
        ControlFlow::Continue(output)
    }

    fn branch(self) -> ControlFlow<Self::Residual, Self::Output> {
        match self {
            ControlFlow::Continue(c) => ControlFlow::Continue(c),
            ControlFlow::Break(b) => ControlFlow::Break(ControlFlow::Break(b)),
        }
}

impl<B, C> ops::FromResidual for ControlFlow<B, C> {
    // Infallible表示类型不会被用到
    fn from_residual(residual: ControlFlow<B, convert::Infallible>) -> Self {
        match residual {
            ControlFlow::Break(b) => ControlFlow::Break(b),
        }
    }
}
```
Option的Try Trait实现：
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
    #[inline]
    fn from_residual(residual: Option<convert::Infallible>) -> Self {
        match residual {
            None => None,
        }
    }
}
```
一个Iterator::try_fold()的实现：
```rust
    fn try_fold<B, F, R>(&mut self, init: B, mut f: F) -> R
    where
        Self: Sized,
        F: FnMut(B, Self::Item) -> R,
        R: Try<Output = B>,
    {
        let mut accum = init;
        while let Some(x) = self.next() {
            accum = f(accum, x)?;
        }
        //try 关键字将accum转化为R类型变量
        try { accum }
    }
```
Result<T,E>类型的Try Trait请自行分析

#### 小结
利用Try Trait，程序员可以实现自定义类型的?，提供函数式编程的有力手段并简化代码，提升代码的理解度。

### Range 运算符代码分析
Range是符号 .. , start..end , start.. , ..end , ..=end，start..=end 形式
#### Range相关的边界结构Bound
源代码：
```rust
pub enum Bound<T> {
    /// An inclusive bound.
    /// 边界包括
    Included(T),
    /// An exclusive bound.
    /// 边界不包括
    Excluded(T),
    /// An infinite endpoint. Indicates that there is no bound in this direction.
    /// 边界不存在
    Unbounded,
}
```
Include边界的值包含，Exclued边界的值不包含，Unbounded边界值不存在
#### RangeFull
` ..  `的数据结构。
#### Range<Idx>
`start.. end`的数据结构
#### RangeFrom<Idx>
`start..`的数据结构
#### RangeTo<Idx>
`.. end`的数据结构
#### RangeInclusive<Idx>
`start..=end`的数据结构
#### RangeToInclusive<Idx>
`..=end`的数据结构

#### RangeBounds<T: ?Sized>
所有Range统一实现的Trait。
```rust
pub trait RangeBounds<T: ?Sized> {
    /// 获取范围的起始值
    ///
    /// 例子
    /// # fn main() {
    /// use std::ops::Bound::*;
    /// use std::ops::RangeBounds;
    ///
    /// assert_eq!((..10).start_bound(), Unbounded);
    /// assert_eq!((3..10).start_bound(), Included(&3));
    /// # }
    /// ```
    fn start_bound(&self) -> Bound<&T>;

    /// 获取范围的终止值.
    /// 例子
    /// # fn main() {
    /// use std::ops::Bound::*;
    /// use std::ops::RangeBounds;
    ///
    /// assert_eq!((3..).end_bound(), Unbounded);
    /// assert_eq!((3..10).end_bound(), Excluded(&10));
    /// # }
    /// ```
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

#### Range的独立性
Range操作符多用于与Index运算符结合或与Iterator Trait结合使用。在后继的Index运算符和Iterator中会研究Range是如何与他们结合的。

#### 小结
基于泛型的Range类型提供了非常好的语法手段，只要某类型支持排序，那就可以定义一个在此类型基础上实现的Range类型。再结合Index和Iterator, 将高效的实现极具冲击力的代码。

### RUST的Index 运算符代码分析
数组下标符号[]由Index, IndexMut两个Trait完成重载。数组下标符号重载使得程序更有可读性。两个Trait如下定义：
```rust
pub trait Index<Idx: ?Sized> {
    /// The returned type after indexing.
    type Output: ?Sized;

    /// 若果传入的参数超过内存界限将马上引发panic
    fn index(&self, index: Idx) -> &Self::Output;
}

pub trait IndexMut<Idx: ?Sized>: Index<Idx> {
    fn index_mut(&mut self, index: Idx) -> &mut Self::Output;
}
```
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
需要依赖SliceIndex Trait实现[T]的ops::Index。SliceIndex主要是为了实现下标即支持用usize类型取出单一元素，又支持用Range类型取出子slice。
显然，针对不同的Index<Idx>中的泛型Idx，需要实现不同的处理逻辑。SliceIndex的引入是典型的处理这个需求的设计方式。即如果对某一类型实现一个具有泛型的Trait时，如果对于Trait的泛型实例化不同类型，会带来处理逻辑的不同。那就再定义一个辅助Trait，为前Trait的实例化类型实现辅助Trait，在这个辅助Trait的实现中实现不同的处理逻辑。辅助Trait和Trait之间的定义相关性即可参考SliceIndex和Index的定义相关性。

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
    /// 此类型通常为T或者T的引用，切片，原生指针类型
    type Output: ?Sized;

    // 从slice变量中用self获取Option<Output>变量
    fn get(self, slice: &T) -> Option<&Self::Output>;

    fn get_mut(self, slice: &mut T) -> Option<&mut Self::Output>;

    //slice是序列的头指针，后面的具体实现会看到为什么用 *const T
    unsafe fn get_unchecked(self, slice: *const T) -> *const Self::Output;

    unsafe fn get_unchecked_mut(self, slice: *mut T) -> *mut Self::Output;
    
    //如果self超出slice的安全范围，会panic
    fn index(self, slice: &T) -> &Self::Output;

    fn index_mut(self, slice: &mut T) -> &mut Self::Output;
}

unsafe impl<T> SliceIndex<[T]> for usize {
    type Output = T;

    fn get(self, slice: &[T]) -> Option<&T> {
        // 必须转化为*const T才能够用指针加方式来获取
        if self < slice.len() { unsafe { Some(&*self.get_unchecked(slice)) } } else { None }
    }

    fn get_mut(self, slice: &mut [T]) -> Option<&mut T> {
        if self < slice.len() { unsafe { Some(&mut *self.get_unchecked_mut(slice)) } } else { None }
    }

    unsafe fn get_unchecked(self, slice: *const [T]) -> *const T {
        //相当于C的指针加操作
        unsafe { slice.as_ptr().add(self) }
    }

    unsafe fn get_unchecked_mut(self, slice: *mut [T]) -> *mut T {
        unsafe { slice.as_mut_ptr().add(self) }
    }

    fn index(self, slice: &[T]) -> &T {
        // N.B., use intrinsic indexing 此处应该可以用get,但应该是编译器内置支持，为了效率直接使用了内置的数组下标表示。
        &(*slice)[self]
    }

    fn index_mut(self, slice: &mut [T]) -> &mut T {
        // N.B., use intrinsic indexing
        &mut (*slice)[self]
    }
}
```
以上就是针对[T]的以无符号数作为下标取出单一元素的ops::Index 及 ops::IndexMut的底层实现，从slice中取出单一元素必须应用ptr的操作。

```rust
unsafe impl<T> SliceIndex<[T]> for ops::Range<usize> {
    type Output = [T];

    fn get(self, slice: &[T]) -> Option<&[T]> {
        if self.start > self.end || self.end > slice.len() {
            None
        } else {
            unsafe { Some(&*self.get_unchecked(slice)) }
        }
    }

    fn get_mut(self, slice: &mut [T]) -> Option<&mut [T]> {
        if self.start > self.end || self.end > slice.len() {
            None
        } else {
            unsafe { Some(&mut *self.get_unchecked_mut(slice)) }
        }
    }

    unsafe fn get_unchecked(self, slice: *const [T]) -> *const [T] {
        // 利用ptr的内存操作形成* const [T] 原生指针
        unsafe { ptr::slice_from_raw_parts(slice.as_ptr().add(self.start), self.end - self.start) }
    }

    unsafe fn get_unchecked_mut(self, slice: *mut [T]) -> *mut [T] {
        unsafe {
            ptr::slice_from_raw_parts_mut(slice.as_mut_ptr().add(self.start), self.end - self.start)
        }
    }

    fn index(self, slice: &[T]) -> &[T] {
        if self.start > self.end {
            slice_index_order_fail(self.start, self.end);
        } else if self.end > slice.len() {
            slice_end_index_len_fail(self.end, slice.len());
        }
        //将* const [T]转化为切片引用
        unsafe { &*self.get_unchecked(slice) }
    }

    fn index_mut(self, slice: &mut [T]) -> &mut [T] {
        if self.start > self.end {
            slice_index_order_fail(self.start, self.end);
        } else if self.end > slice.len() {
            slice_end_index_len_fail(self.end, slice.len());
        }
        unsafe { &mut *self.get_unchecked_mut(slice) }
    }
}
```
以上是实现用Range从slice中取出子slice的实现。其他如RangeTo等与Range大同小异。

##### 小结
对于切片的Index, 整体上，需要将切片类型的引用转换为slice元素类型的原生指针，然后对原生指针做加减操作，再根据需要重新建立元素类型或作切片类型的原生指针，然后将原生指针转换为引用。由此可见，ptr和mem模块的熟练使用是必须掌握的。

#### 数组数据结构[T;N]的ops::Index实现

```rust
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
