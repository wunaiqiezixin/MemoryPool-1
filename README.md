>几乎所有用到 `std::allocator<T>` 的地方都可以用本项目的 `MemoryPool<T>` 完美替换  


# 什么是内存池 ？  
  
**内存池是一种预分配内存并进行重复利用的技术**，通过减少频繁的动态内存分配与释放操作，从而提高程序运行效率。    

内存池通常预先分配一块大的内存区域，将其划分为多个小块，每次需要分配内存时直接从这块区域中分配，而不是调用系统的动态分配函数（如`new`或`malloc`）。   

简单来说就是申请一块较大的内存块(不够继续申请)，之后将这块内存的管理放在应用层执行，减少系统调用带来的开销。    

# 什么是 C++ 的 `std::allocator<T>` ?   

`std::allocator<T>`  
C++ 标准库的默认内存分配器，是所有 STL 容器（`vector/string/map`等）的默认内存管理组件，核心设计是**分离内存分配与对象生命周期**。  

**核心接口:**  
- `allocate(n)`：分配可容纳 `n` 个 `T` 对象的原始未初始化内存，返回 `T*` 指针，不执行对象构造  
- `deallocate(p, n)`：释放 `p` 指向的内存块，仅回收内存、不析构对象，`n` 必须与分配时的数量严格一致  
- `construct(p, args...) / destroy(p)`：原地构造 / 析构对象（C++17 起从类内移除，统一由 `std::allocator_traits` 提供）  

**关键特性:**
- 无状态设计：默认实现无成员数据，同类型分配器可互相释放对方分配的内存   
- 底层默认通过 `::operator new` / `::operator delete` 从堆分配内存  
- 可插拔：所有 STL 容器支持替换为自定义分配器，常用于内存池、内存调试、定制内存对齐等场景  

**设计价值:**
>将 “内存申请” 与 “对象创建” 解耦，让容器可以预分配冗余内存（减少系统调用次数），同时为自定义内存管理提供了统一的标准接口。

# 项目介绍：

- **首先**，这是一个 `allocator` 类，所有使用到 `std::allocator` 的地方都可以用 `MemoryPool` 代替。  
```CPP
std::vector<int, MemoryPool<int>> memoryPoolVec;
std::list<int, MemoryPool<int>> memoryPoolList;
```

- **其次**，`MemoyPool` 的成员函数 `newElement/deleteElement` 可以替代 `new/dlelete` 为对象分配内存。
```CPP
MemoryPool<int> memoryPool;  
int* p = memoryPool.newElement();  
memoryPool.deleteElement(p);  
```


**内存管理逻辑：**  
```  
分配流程 (`allocate`):  
`freeSlots_` ——非空——> 直接从空闲链表取  
     │  
     ↓ 空  
`currentSlot_ < lastSlot_` ——是——> 取`currentSlot_`并++  
     │  
     ↓ 否  
`allocateBlock()` ——> 更新管理指针 ——> 返回新槽  
  
释放流程 (`deallocate`):  
指针`p` ——> 转换为`slot_pointer_` ——> 插入`freeSlots_`头部 ——> 等待复用  
```  
