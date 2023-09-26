## 语法相关

### uintptr和unsafe.Pointer的区别

unsafe.Pointer只是单纯的通用指针类型，用于转换不同类型指针，它不可以参与指针运算；

而uintptr是用于指针运算的，GC 不把 uintptr 当指针，也就是说 uintptr 无法持有对象， uintptr 类型的目标会被回收；

unsafe.Pointer 可以和 普通指针 进行相互转换；

unsafe.Pointer 可以和 uintptr 进行相互转换。

