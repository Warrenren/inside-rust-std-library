# inside-rust-std-library
本书主要对RUST的标准库代码进行分析。
首先分析的是标准库的内存部分，完全理解标准库的内存部分代码，RUST的最难点就被攻克。
随后将分析一些基本类型，如整形大小端如何变化，Option/Result类型有哪些解封装方式，各类型针对函数式编程有些什么设计等等
然后是基本的Trait, Ops Trait中包括index如何重载，Range如何实现，Try Trait如何简化代码等等
Iterator是现代化语言及函数式编程的关键，Range/切片/数组/字符串对Iterator的实现将充分的展示RUST的编码技巧
Borrow Trait/Cell/RefCell代码揭示了内部可变性的秘密
智能指针的代码实现给出了如何用rust编写数据结构的实例
...
...
