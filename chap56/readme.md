# 《Chrome V8 源码》56. GC 垃圾回收 1 
# 1 介绍  
我在垃圾堆里的日日夜夜，本文讲解垃圾回收用到的 Handle 机制。  
Handle 注册在 GC 上，用于跟踪并记录 V8 的堆上对象，说白了就是记录对象的地址，它是对象的拥有者与 GC 之间的桥梁。我们知道，GC 回收内存时会移动活跃的对象，以腾出更大的连续空间。既然对象被移动了，就要通知对象的拥有者。我们知道，JS 的业务逻辑非常复杂，实时通知是不现实的，更是不能接受的，所以有了 Handle 机制。GC 移动对象的同时也把新地址写到它的 Handle 中，这样，无论对象被移动到哪里，拥有者只需记住 Handle，就可以随时找到这个对象，Handle 是对象拥有者和 GC 都知道的公用位置，起到桥梁的作用。
Handle 很像 C++ 中的指针，他的左边是对象的拥有者，拥有者只需知道 Handle，无需关心对象在哪。他的右边是 GC，GC 移动对象后只要更新 Handle 管理的对象地址就可以了，不用通知对象的拥有者。  
# 2 Handle   
V8 官方给出了 Handle 的类型，包括：局部 Handle、全局 Handle、外部 Handle 等，每种类型的区别和使用场景也有详细介绍，本文不再赘述。这几种 Handle 中，局部 Handle 是最常用的，也是本文主要讲解的内容。Handle 源码如下：  
```c++
1.  class Handle final : public HandleBase {
2.   public://省略..............
3.    class ObjectRef {
4.     public:
5.      T* operator->() { return &object_; }
6.     private:
7.    V8_INLINE ObjectRef operator->() const { return ObjectRef{**this}; }
8.    V8_INLINE T operator*() const {
9.      SLOW_DCHECK(IsDereferenceAllowed());
10.      return T::unchecked_cast(Object(*location()));
11.    }
12.    template <typename S>
13.    inline static const Handle<T> cast(Handle<S> that);
14.    static const Handle<T> null() { return Handle<T>(); }
15.    bool equals(Handle<T> other) const { return address() == other.address(); }
16.    void PatchValue(T new_value) {
17.      SLOW_DCHECK(location_ != nullptr && IsDereferenceAllowed());
18.      *location_ = new_value.ptr();
19.    }
20.    struct equal_to {
21.      V8_INLINE bool operator()(Handle<T> lhs, Handle<T> rhs) const {
22.        return lhs.equals(rhs);
23.      }
24.    };
25.    struct hash {
26.      V8_INLINE size_t operator()(Handle<T> const& handle) const {
27.        return base::hash<Address>()(handle.address());
28.      }
29. };};
```   
上述代码中，除了创建 Handle 之外，最常用的是*和->两个操作，其中 *是用来返回此 Handle 管理的对象地址；->用于访问对象的成员方法，见上述代码中相应的定义。  
第 30 行代码，equals 操作时判断对象地址是否相同，因为两个 Handle 本身的比较不能说明他们管理的对象之间的关系。再者，两个 Handle 本身地址不同，但他们同时指向同一个对象，这是很常见的情况。Handle 的作用是管理对象，所以 equals 操作是对被管理 Ojbect 地址的比较。    
下面是基类 HandleBase 源码：   
```c++
1.  class HandleBase {
2.   public://省略很多...............
3.    V8_INLINE bool is_identical_to(const HandleBase that) const;
4.    V8_INLINE bool is_null() const { return location_ == nullptr; }
5.    V8_INLINE Address address() const { return bit_cast<Address>(location_); }
6.    V8_INLINE Address* location() const {
7.      SLOW_DCHECK(location_ == nullptr || IsDereferenceAllowed());
8.      return location_;
9.    }
10.    Address* location_;
11.  };
```  
成员变量 location_ 保存被管理对象的地址；Adress() 和 location() 都可以返回对象地址，区别是 SLOW_DCHECK。  
局部 Handle 是最常用的 Handle，为了统一管理，V8 给出来了 HandleScope 这个概念。  
# 3 HandleScope  
HandleScope 管理同一个作用域下的所有 Handle，这很想函数内的局部变量，当函数退出时，局部变量也会消失。当 HanldeScope 退出时，他管理的 Handle 也会被回收。同时，V8 也规定：一个局部 Handle 必须属于某个 HandleScope，这就是为什么我们经常看到 V8 内很多函数的第一条语句是 HandleScope，例如下面的函数：
```c++
RUNTIME_FUNCTION(Runtime_InstallBaselineCode) {
  HandleScope scope(isolate);//这是第一条，HandleScope
//省略很多........
  return baseline_code;
}
``` 
不创建 HandleScope，直接使用局部 Handle 会报错，导致 Crash。下面是 HandleScope 的源码：  
```c++
1.  class V8_NODISCARD HandleScope {
2.  public:
3.  //省略..............
4.   private:
5.    Isolate* isolate_;
6.    Address* prev_next_;
7.    Address* prev_limit_;
8.    V8_EXPORT_PRIVATE static Address* Extend(Isolate* isolate);
9.  };
```   
HandleScope 是栈对象，遵循先进后出原则，这就像函数之间的嵌套调用一样，经常会有这样的情况：函数 caller-fun 中调用了函数 callee-fun，在 caller-fun 中有 caller-Scope，在 callee-fun 中有 callee-Scope，那么 callee-Scope 在栈顶，caller-Scope 次之。 创建 HandleScope 的源码如下：  
```c++
HandleScope::HandleScope(Isolate* isolate) {
  HandleScopeData* data = isolate->handle_scope_data();
  isolate_ = isolate;
  prev_next_ = data->next;
  prev_limit_ = data->limit;
  data->level++;
}
```   
prev_next_、preve_limit_ 用于连接 HandleScope，也就是链表，表达了不同 HandleScope 之间的堆栈关系。销毁 HandleScope 操作如下：  
```c++
void HandleScope::CloseScope(Isolate* isolate, Address* prev_next,
                             Address* prev_limit) {
//省略.....                               
  HandleScopeData* current = isolate->handle_scope_data();
  std::swap(current->next, prev_next);
  current->level--;
  Address* limit = prev_next;
  if (current->limit != prev_limit) {
    current->limit = prev_limit;
    limit = prev_limit;
    DeleteExtensions(isolate);
  }
//省略
}
```  
注意看 prev_next、preve_limit_ 的操作与创建 HandleScope 的操作正好相反，实现退栈的目的。  
**技术总结**  
**（1）** V8 的堆对象必须使用 Handle 管理，Handle 是对象拥有者与 GC 之间的桥梁；   
**（2）** V8 局部 Handle 必须由 HandleScope 管理。否则，虽然可以通过编译，但一定会有 Crash；    
**（3）** 局部 Handle 不能跨越函数，例如返回 callee 的结果给 caller，请使用其类的 Handle；   
**（4）** V8 包括：局部、全局、外部、永久型 Handle。

好了，今天到这里。    
**恳请批评指正，你的建议是我进步的动力！**    






