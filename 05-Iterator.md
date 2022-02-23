# RUST的Iterator实现代码分析 
Iterator在函数式编程中是居于最核心的地位。在函数式编程中，最关键的就是把问题的解决方式设计成能够使用Iterator方案来解决。RUST基本上可以说是原生的Iterator语言，几乎所有的核心关键类型的方法库都依托于Iterator。

## RUST的Iterator与其他语言Iterator比较
RUST定义了三种迭代器Trait:
1. 对容器内的变量进行操作的迭代器：
```rust
pub trait IntoIterator {
    type Item;
    type IntoIter: Iterator<Item = Self::Item>;
    fn into_iter(self) -> Self::IntoIter;
}
```
into_iter返回的迭代器迭代时，会消费容器，完全迭代后容器将被释放。此种迭代器适用于类似生产者-消费者队列的程序设计
2. 对容器内的变量不可用引用进行操作的迭代器：
这个一般不做Trait，而是容器类型实现一个方法：
```pub fn iter(&self) -> I:Iterator```
此方法返回一个迭代器，这种迭代器适用的一个例子是对网络接口做遍历以获得统计值
3. 对容器内的变量可变引用进行操作的迭代器：
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
定义如下：
```rust 
impl<A: Step> Iterator for ops::Range<A> {
        ...
    }
```
只有实现`Step Trait`的Range类型才实现了Iterator, `Step Trait`的定义如下：
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
照此，可以实现一个自定义类型的类型, 并支持Step Trait，如此，即可使用Range的符号。例如，一个二维的点的range，三维的点的range，数列等。
一个为所有的无符号类型整数实现的Step Trait中的一个函数：
```rust
                fn forward_checked(start: Self, n: usize) -> Option<Self> {
                    match Self::try_from(n) {
                        Ok(n) => start.checked_add(n),
                        Err(_) => None, // if n is out of range, `unsigned_start + n` is too
                    }
                }
```
从这个函数中可以看到，使用了try_from做了不同的无符号整数类型之间的变换，checked_add来规避溢出，都是RUST的安全性的具体体现，这是使用rust编码与其他语言编码的不同之处,在编码的时候即强制消除了易忽视的整数变量溢出bug产生。

Range Iterator的底层实现Trait RangeIteratorImpl
```rust
impl<A: Step> RangeIteratorImpl for ops::Range<A> {
    type Item = A;

    default fn spec_next(&mut self) -> Option<A> {
        if self.start < self.end {
            //self.start.clone()是为了不转移self.start的所有权
            let n =
                Step::forward_checked(self.start.clone(), 1).expect("`Step` invariants not upheld");
            //mem::replace将self.start赋值为n，返回self.start的值，这个方式适用于任何类型
            Some(mem::replace(&mut self.start, n))
        } else {
            None
        }
    }

    ...
}
```
从代码分析中可见，rust在代码上的技巧性实际上和C在思想上很类似。都是基于对内存的深刻理解。

## slice的Iterator实现

首先定义了适合&[T]的Iter结构：
```rust
pub struct Iter<'a, T: 'a> {
    //当前元素的指针
    ptr: NonNull<T>,
    //尾元素指针，用ptr == end以快速检测iterator是否为空
    end: *const T, 
    //用来表示iterator与容器类型的生命周期关系
    _marker: PhantomData<&'a T>, 
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

//用宏实现Iterator Trait
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

        // 0元素next的移动
        macro_rules! zst_shrink {
            ($self: ident, $n: ident) => {
                //0元素数组因为不能移动指针，所以移动尾指针
                $self.end = ($self.end as * $raw_mut u8).wrapping_offset(-$n) as * $raw_mut T;
            }
        }
        
        //具体的方法实现
        impl<'a, T> $name<'a, T> {
            // 从Iterator获得切片.
            fn make_slice(&self) -> &'a [T] {
                // ptr::from_raw_parts，由内存首地址和切片长度创建切片指针，然后转换为引用
                unsafe { from_raw_parts(self.ptr.as_ptr(), len!(self)) }
            }

            //实质的next
            unsafe fn post_inc_start(&mut self, offset: isize) -> * $raw_mut T {
                if mem::size_of::<T>() == 0 {
                    //0字节元素偏移实现，调整end的值，ptr不变
                    zst_shrink!(self, offset);
                    self.ptr.as_ptr()
                } else {
                    //非0字节元素，返回首地址，首地址然后后移正确的字节
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

        //Iterator的实现
        impl<'a, T> Iterator for $name<'a, T> {
            type Item = $elem;

            #[inline]
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


impl<'a, T> Iter<'a, T> {
    pub(super) fn new(slice: &'a [T]) -> Self {
        let ptr = slice.as_ptr();
        // SAFETY: Similar to `IterMut::new`.
        unsafe {
            assume(!ptr.is_null()); 

            let end = if mem::size_of::<T>() == 0 {
                //如果切片元素是0字节类型，end 为 首地址加 切片长度字节。即每个元素一个字节
                (ptr as *const u8).wrapping_add(slice.len()) as *const T 
            } else {
                //end为slice.len() * mem::<T>::size_of()
                ptr.add(slice.len()) 
            };
            
            //PhantomData会做类型推断，带入T的类型和生命周期
            Self { ptr: NonNull::new_unchecked(ptr as *mut T), end, _marker: PhantomData }
        }
    }
    
    ...
    ...
}
//宏声明
iterator! {struct Iter -> *const T, &'a T, const, {/* no mut */}, {}}
```
基本上，一个容器类的Iterator的实现基本上是必须要用ptr及mem模块的函数。

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
    ///
    /// 这个数组中，只有data[alive]是有效的，访问其他的部分，即data[..alive.start]
    /// 及data[end..]会发生UB
    /// 
    data: [MaybeUninit<T>; N],

    /// 表明数组中有效的范围.
    ///
    /// 必须满足:
    /// - `alive.start <= alive.end`
    /// - `alive.end <= N`
    alive: Range<usize>,
}
```
从后继的代码可以看出，一旦使用了Iterator, 数组便被IntoIter所代替。
```rust
impl<T, const N: usize> IntoIter<T, N> {
    pub fn new(array: [T; N]) -> Self {
        // 
        // 因为RUST特性目前还不支持数组的transmute，所以用了内存跨类型的transmute_copy，此函数将从栈中申请一块内存。
        // 拷贝完毕后，原数组的所有权已经转移到data中，且data也完成了初始化。此时，需要调用mem::forget反应所有权已经失去。
        // mem::forget不会导致内存泄漏。
        unsafe {
            let iter = Self { data: mem::transmute_copy(&array), alive: 0..N };
            mem::forget(array);
            iter
        }
    }

    pub fn as_slice(&self) -> &[T] {
        // SAFETY: We know that all elements within `alive` are properly initialized.
        unsafe {
            //此处调用SliceIndex::<Range>::get_unchecked
            let slice = self.data.get_unchecked(self.alive.clone());
            MaybeUninit::slice_assume_init_ref(slice)
        }
    }

    pub fn as_mut_slice(&mut self) -> &mut [T] {
        unsafe {
            //此处调用SliceIndex::<Range>::get_unchecked_mut
            let slice = self.data.get_unchecked_mut(self.alive.clone());
            MaybeUninit::slice_assume_init_mut(slice)
        }
    }
}

impl<T, const N: usize> Iterator for IntoIter<T, N> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        // 下面使用Range的Iterator特性实现next. alive的start会变化，从而导致start之前的数组元素无法再被访问。
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
    // 因为IntoIter使用的内存是调用MaybeUninit::uninit()从栈中获得的, 感觉不释放似乎也没有内存泄漏问题。
    // 此处的必要性还需要再思考。
    fn drop(&mut self) {
        // as_mut_slice()获得所有具有所有权的元素，这些元素需要调用drop来释放。这里，data变量中的元素始终封装在MaybeUninit
        unsafe { ptr::drop_in_place(self.as_mut_slice()) }
    }
}

```
以上需要特别注意所有权的转移和内存drop调用，这是RUST需要特别注意训练的点。
```rust
impl<T, const N: usize> IntoIterator for [T; N] {
    type Item = T;
    type IntoIter = IntoIter<T, N>;

    /// Creates a consuming iterator, that is, one that moves each value out of
    /// the array (from start to end). The array cannot be used after calling
    /// this unless `T` implements `Copy`, so the whole array is copied.
    /// 创建消费型的iterator, 如果T不实现`Copy`, 则调用此函数后，数组不可再被访问。
    fn into_iter(self) -> Self::IntoIter {
        IntoIter::new(self)
    }
}

#[stable(feature = "rust1", since = "1.0.0")]
impl<'a, T, const N: usize> IntoIterator for &'a [T; N] {
    type Item = &'a T;
    type IntoIter = Iter<'a, T>;
    
    //调用了slice::iter(), &[T;N]实质是slice结构[T]
    fn into_iter(self) -> Iter<'a, T> {
        self.iter()
    }
}

#[stable(feature = "rust1", since = "1.0.0")]
impl<'a, T, const N: usize> IntoIterator for &'a mut [T; N] {
    type Item = &'a mut T;
    type IntoIter = IterMut<'a, T>;
    
    //同上
    fn into_iter(self) -> IterMut<'a, T> {
        self.iter_mut()
    }
}
```
对于数组类型，iterator的实现耗费了大量的资源并有很多内存操作。对于数组，尽可能的使用引用的Iterator。

## 切片排序

```rust
/// 插入排序, 复杂度O(n^2).
fn insertion_sort<T, F>(v: &mut [T], is_less: &mut F)
where
    F: FnMut(&T, &T) -> bool,
{
    //排序场景下，基本不能使用iterator
    for i in 1..v.len() {
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
由上可见，即使是简单的排序算法，也必须使用mem及ptr和unsafe代码。而排序实际上是最基本的编程，因此若果ptr和mem模块是必须深刻理解的。

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
    a: Option<A>,
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
Iterator的adapter还有很多，如StedBy, Filter, Zip, Intersperse等等。具体请参考标准库手册。基本上所有的adapter都是遵循Adapter的设计模式来实现的。
### 小结
RUST的Iterater的adapter是突出的体现RUST的语法优越性的特性，借助Trait和强大的泛型机制，与c/c++/java/python/go相比较，RUST以很少的代码在标准库就实现了最丰富的adapter。而其他语言往往需要语言基础之上的框架去支持，会导致额外的学习努力。
函数式编程的基础框架之一便是基于Iterator和闭包实现丰富的adapter。这也凸显了RUST在语言级别对函数式编程的良好支持。

## Option的Iterator实现代码分析
Option实现Iterator是比较令人疑惑的，毕竟用Iterator肯定代码更多，逻辑也复杂。主要目的应该是为了重用Iterator构建的各种adapter，及为了函数式编程的需要。仅分析了IntoIterator Trait所涉及的结构及方法
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

    //创建Iterator的实现结构体，所有权传入
    fn into_iter(self) -> IntoIter<T> {
        IntoIter { inner: Item { opt: self } }
    }
}

//具体实现者
impl<A> Iterator for Item<A> {
    type Item = A;

    fn next(&mut self) -> Option<A> {
        self.opt.take()
    }

    fn size_hint(&self) -> (usize, Option<usize>) {
        match self.opt {
            Some(_) => (1, Some(1)),
            None => (0, Some(0)),
        }
    }
}

//变量Iterator, 
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
`
Result<T,E>的 Iterator与Option<T>的Iterator非常相似，略
