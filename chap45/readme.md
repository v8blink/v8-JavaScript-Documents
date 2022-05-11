# 《Chrome V8 源码》45. JavaScript API 源码分析（1）  
![avatar](../v8.png)   
# 1 介绍
substring、getDate、catch 等是常用的 JavaScript API。接下来的几篇文章将从整体上对 JavaScript API 的设计思想、源码和关键函数进行讲解，并能通过例子来分析 JavaScript 在 V8 中的初始化、运行方式，以及它与解释器、编译器、字节码之间的关系。  
# JavaScript API 的初始化  
在 V8 中，JavaScript API（以下简称：API）的初始化由 IniitializeGlobal() 方法负责，该方法在创建 snapshot 时被调用以完成所有 API的初始化，通过调试 mksnapshot 解决方案（VS 2019）可以看到该函数的运行过程，源码如下：  
```c++
1.  void Genesis::InitializeGlobal(Handle<JSGlobalObject> global_object,
2.                              Handle<JSFunction> empty_function) {
3.  Handle<JSFunction> array_prototype_to_string_fun;
4.    // 省略...............
5.   SimpleInstallFunction(isolate_, array_function, "isArray",
6.                         Builtin::kArrayIsArray, 1, true);
7.   SimpleInstallFunction(isolate_, array_function, "from", Builtin::kArrayFrom,
8.                         1, false);
9.   SimpleInstallFunction(isolate_, array_function, "of", Builtin::kArrayOf, 0,
10.                          false);
11.    JSObject::AddProperty(isolate_, proto, factory->constructor_string(),
12.                          array_function, DONT_ENUM);
13.    SimpleInstallFunction(isolate_, proto, "concat",
14.                          Builtin::kArrayPrototypeConcat, 1, false);
15.    SimpleInstallFunction(isolate_, proto, "copyWithin",
16.                          Builtin::kArrayPrototypeCopyWithin, 2, false);
17.    SimpleInstallFunction(isolate_, proto, "reverse",
18.                          Builtin::kArrayPrototypeReverse, 0, false);
19.    SimpleInstallFunction(isolate_, proto, "shift",
20.                          Builtin::kArrayPrototypeShift, 0, false);
21.    SimpleInstallFunction(isolate_, proto, "unshift",
22.                          Builtin::kArrayPrototypeUnshift, 1, false);
23.    SimpleInstallFunction(isolate_, proto, "slice",
24.                          Builtin::kArrayPrototypeSlice, 2, false);
25.    // 省略...............
26.  }}
```  
通过上述代码可以看到 SimpleInstallFunction() 每执行一次安装一个 API 到 isolate_ 中。以第 13 行为例，参数 "concat" 是字符串，参数 Builtin::kArrayPrototypeConcat 是枚举值，SimpleInstallFunction() 为二者建立了对应关系，当我们在 JavaScript 源码中使用 array.concat 方法时，就是使用对应的 Builtin 方法。下面讲解 SimpleInstallFunction 如何为"concat" 和 Builtin::kArrayPrototypeConcat 建立对应关系，源码如下：   
```c++
1.  V8_NOINLINE Handle<JSFunction> SimpleInstallFunction(
2.      Isolate* isolate, Handle<JSObject> base, const char* name, Builtin call,
3.      int len, bool adapt, PropertyAttributes attrs = DONT_ENUM) {
4.    Handle<String> internalized_name =
5.        isolate->factory()->InternalizeUtf8String(name);
6.    Handle<JSFunction> fun =
7.        SimpleCreateFunction(isolate, internalized_name, call, len, adapt);
8.    JSObject::AddProperty(isolate, base, internalized_name, fun, attrs);
9.    return fun;
10.  }
```  
上述代码中，第 4 行创建 V8 内部字符串 internalized_name，它的值是 "concat"；  
第 6 行创建 JSFcunction 方法 fun，把 internalized_name 填充到 fun 内部的 SharedFunction 中，该 JSFunction 的功能是 Builtin::kArrayPrototypeConcat。  
第 8 行把 fun 填充进 JSOBject 的属性中。  
下面讲解 InternalizeUtf8String() 方法，源码如下：  
```c++
1.  Handle<String> Factory::InternalizeUtf8String(
2.      const base::Vector<const char>& string) {
3.    base::Vector<const uint8_t> utf8_data =
4.        base::Vector<const uint8_t>::cast(string);
5.    Utf8Decoder decoder(utf8_data);
6.    if (decoder.is_ascii()) return InternalizeString(utf8_data);
7.    if (decoder.is_one_byte()) {
8.      std::unique_ptr<uint8_t[]> buffer(new uint8_t[decoder.utf16_length()]);
9.      decoder.Decode(buffer.get(), utf8_data);
10.      return InternalizeString(
11.          base::Vector<const uint8_t>(buffer.get(), decoder.utf16_length()));
12.    }
13.    std::unique_ptr<uint16_t[]> buffer(new uint16_t[decoder.utf16_length()]);
14.    decoder.Decode(buffer.get(), utf8_data);
15.    return InternalizeString(
16.        base::Vector<const base::uc16>(buffer.get(), decoder.utf16_length()));
17.  }
```  
上述代码创建 V8 内部字符串，该字符串的类型是InternalzieString，它与 ConsString、OneByteString 等类型的区别是：ConsString 等是 JavaScript 字符串在 V8 中的不同实现，InternalzieString 被用于表达 V8 的基础组件，正如我们现在所说的 "concat"，它是内部字符串，它用于表达一个 SharedFuncion。
上述代码判断字符串 "concat" 的类型是 ASCII、one_byte 或是 two_byte，创建相应的内部符串，并使用 StringTable 缓存以备后面复用。
下面讲解 SimpleCreateFunction() 方法，源码如下：
```c++
1.  V8_NOINLINE Handle<JSFunction> SimpleCreateFunction(Isolate* isolate,
2.                                                      Handle<String> name,
3.                                                      Builtin call, int len,
4.                                                      bool adapt) {
5.    name = String::Flatten(isolate, name, AllocationType::kOld);
6.    Handle<JSFunction> fun =
7.        CreateFunctionForBuiltinWithoutPrototype(isolate, name, call);
8.    JSObject::MakePrototypesFast(fun, kStartAtReceiver, isolate);
9.    fun->shared().set_native(true);
10.    if (adapt) {
11.      fun->shared().set_internal_formal_parameter_count(JSParameterCount(len));
12.    } else {
13.      fun->shared().DontAdaptArguments();
14.    }
15.    fun->shared().set_length(len);
16.    return fun;
17.  }
```  
上述代码中，参数 name、len、adapt 在 InitializeGlobal() 中规定好了。
第 5 行使用 Flatten 创建简单字符串，本文中的 "concat" 已经是简单字符串；
第 6 行创建 JSFunction，此时的 JSFuncion 还没有被安装到 JSObject 上； 
第 10~15 行创建 Builtin call 的参数，这些参数设置在 SharedFunction 中。  
CreateFunctionForBuiltinWithoutPrototype() 方法使用 NewSharedFunctionInfo() 创建 SharedFunction，NewSharedFunctionInfo()源码如下：  
```c++
1.  Handle<SharedFunctionInfo> FactoryBase<Impl>::NewSharedFunctionInfo(
2.      MaybeHandle<String> maybe_name, MaybeHandle<HeapObject> maybe_function_data,
3.      Builtin builtin, FunctionKind kind) {
4.    Handle<SharedFunctionInfo> shared = NewSharedFunctionInfo();
5.    DisallowGarbageCollection no_gc;
6.    SharedFunctionInfo raw = *shared;
7.    Handle<String> shared_name;
8.    bool has_shared_name = maybe_name.ToHandle(&shared_name);
9.    if (has_shared_name) {
10.      DCHECK(shared_name->IsFlat());
11.      raw.set_name_or_scope_info(*shared_name, kReleaseStore);
12.    } else {
13.      DCHECK_EQ(raw.name_or_scope_info(kAcquireLoad),
14.                SharedFunctionInfo::kNoSharedNameSentinel);
15.    }
16.    Handle<HeapObject> function_data;
17.    if (maybe_function_data.ToHandle(&function_data)) {
18.      DCHECK(!Builtins::IsBuiltinId(builtin));
19.      DCHECK_IMPLIES(function_data->IsCode(),
20.                     !Code::cast(*function_data).is_builtin());
21.      raw.set_function_data(*function_data, kReleaseStore);
22.    } else if (Builtins::IsBuiltinId(builtin)) {
23.      raw.set_builtin_id(builtin);
24.    } else {
25.      DCHECK(raw.HasBuiltinId());
26.      DCHECK_EQ(Builtin::kIllegal, raw.builtin_id());
27.    }
28.    raw.CalculateConstructAsBuiltin();
29.    raw.set_kind(kind);
30.    return shared;
31.  } 
```  
上述代码中，第 2 行参数 maybe_name 是 'concat'，参数 builtin 是 Builtin::kArrayPrototypeConcat；
第 4 行创建 SharedFunctionInfo shared；
第 7-15 行把 'concat' 填充进 shared；
第 17-29 行验证 Builtin::kArrayPrototypeConcat 的值是否正确，并将其设置到 shared 中。
下面讲解 MakePrototypesFast() 方法，源码如下：  
```c++
1.  void JSObject::MakePrototypesFast(Handle<Object> receiver,
2.                                    WhereToStart where_to_start,
3.                                    Isolate* isolate) {
4.    if (!receiver->IsJSReceiver()) return;
5.    for (PrototypeIterator iter(isolate, Handle<JSReceiver>::cast(receiver),
6.                                where_to_start);
7.         !iter.IsAtEnd(); iter.Advance()) {
8.      Handle<Object> current = PrototypeIterator::GetCurrent(iter);
9.      if (!current->IsJSObject()) return;
10.      Handle<JSObject> current_obj = Handle<JSObject>::cast(current);
11.      Map current_map = current_obj->map();
12.      if (current_map.is_prototype_map()) {
13.        // If the map is already marked as should be fast, we're done. Its
14.        // prototypes will have been marked already as well.
15.        if (current_map.should_be_fast_prototype_map()) return;
16.        Handle<Map> map(current_map, isolate);
17.        Map::SetShouldBeFastPrototypeMap(map, true, isolate);
18.        JSObject::OptimizeAsPrototype(current_obj);
19.      }
20.    }
21.  }
```  
上述代码通过循环迭代的方式查找相应的 prototype 并设置好 map。图 1 给出了此时的调用堆栈。  
![avatar]  
# 3.JavaScript API 的使用方法  
上面从 V8 源码的角度讲解了从 Builtin::kArrayPrototypeConcat 到 JSFuncion 的创建。下面从 JavaScript 源码的角度讲解 Builtin::kArrayPrototypeConcat 的使用方法。测试代码如下：    
```c++
1.  1.  var a=[1,2,3];
2.  2.  var b=[4,5,6];
3.  var c= a.concat(b);
4.  console.log(c);
5.  //分隔线..........................
6.  Bytecode Age: 0
7.  //省略...........................
8.   00000159B6561FCA @   28 : c1                Star2
9.   00000159B6561FCB @   29 : 2d f8 05 08       LdaNamedProperty r2, [5], [8]
10.   00000159B6561FCF @   33 : c2                Star1
11.   00000159B6561FD0 @   34 : 21 04 0a          LdaGlobal [4], [10]
12.   00000159B6561FD3 @   37 : c0                Star3
13.   00000159B6561FD4 @   38 : 5d f9 f8 f7 0c    CallProperty1 r1, r2, r3, [12]
14.   00000159B6561FD9 @   43 : 23 06 0e          StaGlobal [6], [14]
15.   00000159B6561FDC @   46 : 21 07 10          LdaGlobal [7], [16]
16.   00000159B6561FDF @   49 : c1                Star2
17.   00000159B6561FE0 @   50 : 2d f8 08 12       LdaNamedProperty r2, [8], [18]
18.   00000159B6561FE4 @   54 : c2                Star1
19.   00000159B6561FE5 @   55 : 21 06 14          LdaGlobal [6], [20]
20.   00000159B6561FE8 @   58 : c0                Star3
21.   00000159B6561FE9 @   59 : 5d f9 f8 f7 16    CallProperty1 r1, r2, r3, [22]
22.   00000159B6561FEE @   64 : c3                Star0
23.   00000159B6561FEF @   65 : a8                Return
```  
上述代码中，第 9 行代码加载数组的属性 concat；在本例中，字节码 LdaNamedProperty 的作用是通过字符串 'concat' 加载对应的 JSFunction 方法。  
第 13 行代码调用该函数 JSFunction。
字节码由 JavaScript 源码编译并生成，'concat' 在字节码中保存为常量，该常量是 JavaScript 源码到 JSFuncion 的唯一联系。
后续的几篇文章将会讲解 JavaScript API 的调用过程。   
**技术总结**  
**（1）** JavaScript API 以 Builtins 形式存在 V8中；
**（2）** 在 V8 中使用 SharedFuncion 保存 API，并保存在 JSObject 属性中；
**（3）** JavaScript 源码中使用的 API 在字节码中被保存为常量字符串，在使用之前利用 LdaNamedProperty 加载相应的 JSFunction。
好了，今天到这里，下次见。    
**个人能力有限，有不足与纰漏，欢迎批评指正**  
**微信：qq9123013  备注：v8交流    邮箱：v8blink@outlook.com**  