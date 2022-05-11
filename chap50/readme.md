# 《Chrome V8 源码》50.JS 数值转换的内幕，Number() 源码分析
# 1 背景   
本文内容来自网友的提问：“Number 是按什么逻辑去执行那些转换规则的？就如同Equal是如何执行比较是否相等的规则”。  
另外，也有网友问为什么 var a = 1n 中 a 的类型是 BigInt，本文最后给出答案。    
# 2 Number() 介绍  
V8 内部对象分为 SMI 和 HeapObject 两大基本类型，V8 使用 tagged pointer 判读一个对象是 SMI（small int）或 HeapObject。SMI 用于存储 31 bit 的整数，故称为小整型，而String、Array 等其它数据则是 HeapObject的子类。
SMI、HeapNumber、BigInt，用于表示数值，这类数据在做 Number() 转时，V8 按约定规则取出相应数据并返回结果即可。  
Number() 代码逻辑复杂，充斥着大量的条件判断，不可能逐条说明，本文通过字符串到数值的转换过程来分析 Number() 的工作流程，希望能起到抛砖引玉的作用。纸太短、码太长，写不完啊... ...  
# 3 Number() 源码分析  
测试用例如下：  
```c++
a="123";
Number(a);
```  
上述代码中，a 是字符串，Number() 的结果类型是 Smi。Number() 由 Builtin::NumberConstructor 实现，源码如下：  
```c++
1.  // ES #sec-number-constructor
2.  transitioning javascript builtin
3.  NumberConstructor(
4.      js-implicit context: NativeContext, receiver: JSAny, newTarget: JSAny,
5.      target: JSFunction)(...arguments): JSAny {
6.    // 1. If no arguments were passed to this function invocation, let n be +0.
7.    let n: Number = 0;
8.    if (arguments.length > 0) {
9.      const value = arguments[0];
10.      n = ToNumber(value, BigIntHandling::kConvertToNumber);
11.  	}
12.  //.........省略余下代码............
13.  }
```
上述代码采用 V8 官方自创的 TQ 语言编写。  
第 3 行代码 NumberConstructor 包括 4 个隐式参数（context、receiver、newTarget、target）和一个显示参数 arguments。  
结合本文用例，第 9 行代码 value 为变量 a，其值是字符串 "123"；    
第 10 行 ToNumber 将 a 转换为数字，ToNumber 内部调用 ToNumberOrNumberic() 完成最终的转换，ToNumberOrNumberic() 源码如下：  
```c++
1.  ToNumberOrNumeric(...省略参数...) {
2.    GotoIfNot(TaggedIsSmi(input), &not_smi);
3.    TNode<Smi> input_smi = CAST(input);
4.    var_result = input_smi;
5.    Goto(&end);
6.    BIND(&not_smi);
7.     {........//省略部分代码........
8.       Label not_heap_number(this, Label::kDeferred);
9.       TNode<HeapObject> input_ho = CAST(input);
10.       GotoIfNot(IsHeapNumber(input_ho), &not_heap_number);
11.       TNode<HeapNumber> input_hn = CAST(input_ho);
12.       var_result = input_hn;
13.       Goto(&end);
14.       BIND(&not_heap_number);
15.       {
16.         if (mode == Object::Conversion::kToNumeric) {
17.           Label not_bigint(this);
18.           GotoIfNot(IsBigInt(input_ho), &not_bigint);
19.           {
20.             var_result = CAST(input_ho);
21.             Goto(&end);
22.           }
23.           BIND(&not_bigint);
24.         }
25.         var_result = NonNumberToNumberOrNumeric(context(), input_ho, mode,
26.                                                 bigint_handling);
27.         Goto(&end);
28.       }
29.     }
30.     BIND(&end);
31.     return var_result.value();
32.   }
```  
**第一种情况：input  是 Smi**  
第 2 行代码判断 input 是不是 Smi，结果为真；  
第 3~4 行代码把 input 转为 Smi 并返回；  
**第二种情况：input 是 HeapNumber**  
第 2 行代码判断结果为假，跳转到第 6 行代码；  
第 8-10 行代码判断 ipnut 为 HeapNumber；  
第 11 行代码把 input 转为 HeapNumber 并返回；  
**第三种情况：input 是 BigInt**  
第 10 行代码结果为假，跳转到第 14 行代码；  
第 16-18 行判断 input 为 BigInt；  
第 20 行代码转换 input 为 BigInt 并返回；  
**第四种情况：以上都不是，这里也是我们关注的重点**  
第 18 行代码结果为假，跳转到第 25 行代码，进入下面的函数  
```c++
1.  TNode<Numeric> CodeStubAssembler::NonNumberToNumberOrNumeric(
2.   TNode<Context> context, TNode<HeapObject> input, Object::Conversion mode,
3.   BigIntHandling bigint_handling) {
4.  //..省略..........
5.  BIND(&if_inputisnotreceiver);
6.  {
7.     TryPlainPrimitiveNonNumberToNumber(var_input.value(), &var_result_number,
8.                                        &not_plain_primitive);
9.     Goto(&end);
10.   }
11.  //...省略..........
12.   BIND(&end);
13.   return var_result.value();
14.  }
```   
在进入上述函数之前已经确定参数 input 为 HeapObject 类型。  
结合本文测试用例，a 是字符串，属于基本类型，执行第 7 行代码 TryPlainPrimitiveNonNumberToNumber，该函数中调用 StringToNumber() 完成最终转换，StrintToNumber() 源码如下：  
```c++
1.  TNode<Number> CodeStubAssembler::StringToNumber(TNode<String> input) {
2.    Label runtime(this, Label::kDeferred);
3.    Label end(this);
4.    TVARIABLE(Number, var_result);
5.    TNode<Uint32T> raw_hash_field = LoadNameRawHashField(input);
6.    GotoIf(IsSetWord32(raw_hash_field, Name::kDoesNotContainCachedArrayIndexMask),
7.           &runtime);
8.    var_result = SmiTag(Signed(
9.        DecodeWordFromWord32<String::ArrayIndexValueBits>(raw_hash_field)));
10.    Goto(&end);
11.    BIND(&runtime);
12.    {
13.      var_result =
14.          CAST(CallRuntime(Runtime::kStringToNumber, NoContextConstant(), input));
15.      Goto(&end);
16.    }
17.    BIND(&end);
18.    return var_result.value();
19.  }
```    
上述代码第 4-10 行取出 hash，尝试利用 hash 转换为数字，如果第 6 行代码转换失败则调用第 14 行代码 runtime_StringToNumber 完成转换。**注意**hash 是在创建字符串生成的，属于字符串的内部成员。runtime_StringToNumber 方法请自行分析。   
# 4 1n 为什么是 BigInt   
答：在 Parser 时期完成 BigInt 定义。
```c++
1.  a=123n;
2.  console.log(typeof(a));
3.  ==字节码================
4.  Bytecode Age: 0
5.           00000192BD821F36 @    0 : 13 00             LdaConstant [0]
6.           00000192BD821F38 @    2 : 23 01 00          StaGlobal [1], [0]
7.  //省略.....................
8.  Constant pool (size = 4)
9.  00000192BD821EB9: [FixedArray] in OldSpace
10.   - map: 0x0340ecf012c1 <Map>
11.   - length: 4
12.             0: 0x0192bd821ee9 <BigInt 123>
13.             1: 0x0362461193c9 <String[1]: #a>
14.             2: 0x007b646818d1 <String[7]: #console>
15.             3: 0x007b64681931 <String[3]: #log>
16.  Handler Table (size = 0)
17.  Source Position Table (size = 0)
```  
见上述的 3-15 代码，字节码在 Parser 阶段生成，常量池的第 1 项数据（第 12 行）说明 123 是 BigInt，所以证明了它在 Parser 阶段完成定义。无论什么类型的数据，只要它是常量，在 Parser 时期都可以确定其类型，这是编译的基本功能。     

好了，今天到这里，下次见。    
**个人能力有限，有不足与纰漏，欢迎批评指正**  
**微信：qq9123013  备注：v8交流    知乎：https://www.zhihu.com/people/v8blink**  






