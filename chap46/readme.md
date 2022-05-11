# 《Chrome V8 源码》 BigInt的疑惑
# 1 背景 
本文内容来自朋友的提问，测试用例如下：
```c++
1.  var a = 4611686018427387904
2.  console.log(a.toString()) // => 4611686018427388000
3.  a = 4611686018427387905
4.  a.toString() // => 4611686018427388000
5.  console.log(BigInt(a).toString()) // => 4611686018427387904
```
上述代码中，我们认为第 2 行的执行结果是“4611686018427387904”，真实结果为什么是“4611686018427388000”？？   
第 3 行的重新赋值似乎没有起作用？？   
第 5 行的执行结果为什么是“4611686018427387904”？？  
答案：  
**（1）** 第 1 行代码，V8 使用 HeapNumber 变量表示 4611686018427387904，值是：4.61169e+18。可以看到，这种表示方法舍弃了一些精度；  
**（2）** 第 1 行代码，V8 创建 HeapNumber 变量时会把原始值保存到特定位置（kValueOffset），以备有需要时使用；  
**（3）** 第 3 行代码，重新赋值为 4611686018427387905 时超过了 HeapNumber 的表示范围（精度），V8 认为它和 4611686018427387904 是同一数据，不再创建新变量；  
**（4）** 第 3 行代码，执行了“重新赋值”操作，操作数是变量 4.61169e+18，这个变量是由 4611686018427387904 生成的；        
**（5）** 第 5 行代码，BigInt 需要使用 a 的原始值，它的原始值正是 4611686018427387904，所以才有这样的输出结果。     
下面给出答案的详细解释：
# 2 字节码与寄存器  
下面回答第一个疑问：第 2 行的结果为什么是“4611686018427388000”？？
给出测试用例的字节码。  
```c++
1.   00000234CF862176 @    0 : 13 00             LdaConstant [0]
2.   00000234CF862178 @    2 : c3                Star1
3.   00000234CF862179 @    3 : 19 fe f8          Mov <closure>, r2
4.   00000234CF86217C @    6 : 65 54 01 f9 02    CallRuntime [DeclareGlobals], r1-r2
5.   00000234CF862181 @   11 : 13 01             LdaConstant [1]
6.   00000234CF862183 @   13 : 23 02 00          StaGlobal [2], [0]
7.   00000234CF862186 @   16 : 21 03 02          LdaGlobal [3], [2]
8.   00000234CF862189 @   19 : c2                Star2
9.   00000234CF86218A @   20 : 2d f8 04 04       LdaNamedProperty r2, [4], [4]
10.   00000234CF86218E @   24 : c3                Star1
11.   00000234CF86218F @   25 : 21 02 06          LdaGlobal [2], [6]
12.   00000234CF862192 @   28 : c0                Star4
13.   00000234CF862193 @   29 : 2d f6 05 08       LdaNamedProperty r4, [5], [8]
14.   00000234CF862197 @   33 : c1                Star3
15.   00000234CF862198 @   34 : 5d f7 f6 0a       CallProperty0 r3, r4, [10]
16.   00000234CF86219C @   38 : c1                Star3
17.   00000234CF86219D @   39 : 5e f9 f8 f7 0c    CallProperty1 r1, r2, r3, [12]
18.   00000234CF8621A2 @   44 : 13 01             LdaConstant [1]
19.   00000234CF8621A4 @   46 : 23 02 00          StaGlobal [2], [0]
20.   00000234CF8621A7 @   49 : 21 03 02          LdaGlobal [3], [2]
21.   00000234CF8621AA @   52 : c2                Star2
22.   00000234CF8621AB @   53 : 2d f8 04 04       LdaNamedProperty r2, [4], [4]
23.   00000234CF8621AF @   57 : c3                Star1
24.   00000234CF8621B0 @   58 : 21 02 06          LdaGlobal [2], [6]
25.   00000234CF8621B3 @   61 : c0                Star4
26.   00000234CF8621B4 @   62 : 2d f6 05 08       LdaNamedProperty r4, [5], [8]
27.   00000234CF8621B8 @   66 : c1                Star3
28.   00000234CF8621B9 @   67 : 5d f7 f6 0e       CallProperty0 r3, r4, [14]
29.   00000234CF8621BD @   71 : c1                Star3
30.   00000234CF8621BE @   72 : 5e f9 f8 f7 10    CallProperty1 r1, r2, r3, [16]
31.   00000234CF8621C3 @   77 : 21 03 02          LdaGlobal [3], [2]
32.   00000234CF8621C6 @   80 : c2                Star2
33.   00000234CF8621C7 @   81 : 2d f8 04 04       LdaNamedProperty r2, [4], [4]
34.   00000234CF8621CB @   85 : c3                Star1
35.   00000234CF8621CC @   86 : 21 06 12          LdaGlobal [6], [18]
36.   00000234CF8621CF @   89 : c0                Star4
37.   00000234CF8621D0 @   90 : 21 02 06          LdaGlobal [2], [6]
38.   00000234CF8621D3 @   93 : bf                Star5
39.   00000234CF8621D4 @   94 : 62 f6 f5 14       CallUndefinedReceiver1 r4, r5, [20]
40.   00000234CF8621D8 @   98 : c0                Star4
41.   00000234CF8621D9 @   99 : 2d f6 05 16       LdaNamedProperty r4, [5], [22]
42.   00000234CF8621DD @  103 : c1                Star3
43.   00000234CF8621DE @  104 : 5d f7 f6 18       CallProperty0 r3, r4, [24]
44.   00000234CF8621E2 @  108 : c1                Star3
45.   00000234CF8621E3 @  109 : 5e f9 f8 f7 1a    CallProperty1 r1, r2, r3, [26]
46.   00000234CF8621E8 @  114 : c4                Star0
47.   00000234CF8621E9 @  115 : a9                Return
48.  Constant pool (size = 7)
49.  00000234CF8620E9: [FixedArray] in OldSpace
50.   - map: 0x018a0a0812c1 <Map>
51.   - length: 7
52.   0: 0x0234cf8620d1 <FixedArray[1]>
53.   1: 0x0234cf862131 <HeapNumber 4.61169e+18>
54.   2: 0x02a3fc4a2559 <String[1]: #a>
55.   3: 0x015b3bfc1af9 <String[7]: #console>
56.   4: 0x015b3bfc1b79 <String[3]: #log>
57.   5: 0x018a0a086889 <String[8]: #toString>
58.   6: 0x018a0a084721 <String[6]: #BigInt>
```  
上述代码中，第 48-58 行是常量池。第 5 行代码 LdaConstant 加载常量池中的第 1 项数据（第 53 行的HeapNumber）并存储到累加寄存器；第 6 行代码 StaGlobal 使用常量池的第 2 项数据做为变量名，把累加寄存器的值保存为全局变量。  
第 5、6 两行共同实现了测试用例中第 1 行变量的定义。其中第 53 行代码常量 4.61169e+18 是科学计数法表示的变量 a，还原之后是 “4611686018427388000”。  
**第一题答案是：** 测试用例第 2 行（a.toString()）的结果是 “4611686018427388000” ，这是因为 V8 编译测试用例时使用 HeapNumber 表示数据，而测试用例中的数据超过了 HeapNumber 的精度，所以 HeapNumber 舍弃了一些精度。    
下面回答第二个问题：第 3 行的重新赋值似乎没有起作用？？    
第 18-19 行代码实现测试用例中第 3 行代码，即 变量 a 的重新赋值（a = 4611686018427387905）。前面说了 V8 的HeapNumber的精度，V8认为 “4611686018427387905” 和 “4611686018427387904” 是同一个值，所以在常量池中只保存了一份（HeapNumber 4.61169e+18），**我们得到下面的结论：**  
**（1）** V8 编译测试用例的第 1 行代码时，根据 4611686018427387904 生成了常量 4.61169e+18 并保存在常量池中；  
**（2）** 编译 第 3 行代码时，检测到常量池中已经有数据了，无需新生成，只需重新执行“重新赋值”操作。  
**第二题答案是：** “重新赋值”执行了，使用的值是 4.61169e+18，这个值是 4611686018427387904 生成的。  
# 3 BigInt 源码分析  
下面回答第三个问题：测试用例第 5 行的结果为什么是“4611686018427387904”？？  
先来介绍 HeapNumber 数据结构，源码如下：  
```c++
// The HeapNumber class describes heap allocated numbers that cannot be
// represented in a Smi (small integer).
class HeapNumber
    : public TorqueGeneratedHeapNumber<HeapNumber, PrimitiveHeapObject> {
        //........省略..........
    }
//分隔线............
template<class D, class P>
double TorqueGeneratedHeapNumber<D, P>::value() const {
  double value;
  value = this->template ReadField<double>(kValueOffset);
  return value;
}
//分隔线................
// src/objects/heap-number.tq?l=8&c=20
template<class D, class P>
void TorqueGeneratedHeapNumber<D, P>::set_value(double value, RelaxedStoreTag) {
  this->template WriteField<double>(kValueOffset, value);
}
```
上述代码中，HeapNumber用于表示超过 SMI 范围的数据，见代码中的注释。value()、set_value() 方法用于读取、设置原始值到kValueOffset 位置。
在测试用例中，第 1 行代码，4611686018427387904 超过了 SMI 范围，V8 为其生成 HeapNumber 变量 a，并赋值 4.61169e+18，同时使用 set_value 把 4611686018427387904 存到 a 变量的 kValueOffset 位置; 
第 3 行代码，为 a 重新赋值时检测到已经存在 HeapNumber 变量了，无需再生成新变量；  
测试代码第 5 行 BigInt(a) 使用 value() 从 kValueOffset 位置读取原值并生成符合操作系统要求（32/64 bit）的 BigInt 值。  
**第三题答案是：** HeapNumber 变量 a 由 4611686018427387904 生成，所以第 5 行的输出结果是  4611686018427387904。  
下面给出 BigInt 的源码：  
```c++
BUILTIN(BigIntConstructor) {
//省略................
  if (value->IsNumber()) {
    RETURN_RESULT_OR_FAILURE(isolate, BigInt::FromNumber(isolate, value));
  } else {
    RETURN_RESULT_OR_FAILURE(isolate, BigInt::FromObject(isolate, value));
  }
}
```
上述代码中调用 BigInt::FromNumber 方法，其源码如下：
```c++
MaybeHandle<BigInt> BigInt::FromNumber(Isolate* isolate,
                                       Handle<Object> number) {
//省略..........
  double value = HeapNumber::cast(*number).value();
//省略..........
  return MutableBigInt::NewFromDouble(isolate, value);
}
```
上述代码中 HeapNumber::cast(*number).value() 的用于从 HeapNumber 变量中读取原始值并保存为 double 变量，在本文的测试用例中，这个原始值是 4611686018427387904，所以测试用例的第 5 行结果是：4611686018427387904。
至此，分析完毕。  
# 4 附录  
下面给出字节码的执行过程，供大家自行分析。    
```c++
 -> 000001CDADCA2176 @    0 : 13 00             LdaConstant [0]
      [ accumulator <- 0x01cdadca20d1 <FixedArray[1]> ]
 -> 000001CDADCA2178 @    2 : c3                Star1
      [ accumulator -> 0x01cdadca20d1 <FixedArray[1]> ]
 -> 000001CDADCA2179 @    3 : 19 fe f8          Mov <closure>, r2
      [   <closure> -> 0x01cdadca2219 <JSFunction (sfi = 000001CDADCA2051)> ]
      [          r2 <- 0x01cdadca2219 <JSFunction (sfi = 000001CDADCA2051)> ]
 -> 000001CDADCA217C @    6 : 65 54 01 f9 02    CallRuntime [DeclareGlobals], r1-r2
      [          r1 -> 0x01cdadca20d1 <FixedArray[1]> ]
      [          r2 -> 0x01cdadca2219 <JSFunction (sfi = 000001CDADCA2051)> ]
      [ accumulator <- 0x02e7aa9815b9 <undefined> ]
 -> 000001CDADCA2181 @   11 : 13 01             LdaConstant [1]
      [ accumulator <- 0x01cdadca2131 <HeapNumber 4.61169e+18> ]
 -> 000001CDADCA2183 @   13 : 23 02 00          StaGlobal [2], [0]
      [ accumulator -> 0x01cdadca2131 <HeapNumber 4.61169e+18> ]
      [ accumulator <- 0x01cdadca2131 <HeapNumber 4.61169e+18> ]
 -> 000001CDADCA2186 @   16 : 21 03 02          LdaGlobal [3], [2]
      [ accumulator <- 0x01cdadc886b9 <console map = 00000199F1B81F71> ]
 -> 000001CDADCA2189 @   19 : c2                Star2
      [ accumulator -> 0x01cdadc886b9 <console map = 00000199F1B81F71> ]
 -> 000001CDADCA218A @   20 : 2d f8 04 04       LdaNamedProperty r2, [4], [4]
      [          r2 -> 0x01cdadc886b9 <console map = 00000199F1B81F71> ]
      [ accumulator <- 0x01cdadc88849 <JSFunction log (sfi = 000003D1D2A93899)> ]
 -> 000001CDADCA218E @   24 : c3                Star1
      [ accumulator -> 0x01cdadc88849 <JSFunction log (sfi = 000003D1D2A93899)> ]
 -> 000001CDADCA218F @   25 : 21 02 06          LdaGlobal [2], [6]
      [ accumulator <- 0x01cdadca2131 <HeapNumber 4.61169e+18> ]
 -> 000001CDADCA2192 @   28 : c0                Star4
      [ accumulator -> 0x01cdadca2131 <HeapNumber 4.61169e+18> ]
 -> 000001CDADCA2193 @   29 : 2d f6 05 08       LdaNamedProperty r4, [5], [8]
      [          r4 -> 0x01cdadca2131 <HeapNumber 4.61169e+18> ]
      [ accumulator <- 0x01cdadc90a41 <JSFunction toString (sfi = 000003D1D2A98A99)> ]
 -> 000001CDADCA2197 @   33 : c1                Star3
      [ accumulator -> 0x01cdadc90a41 <JSFunction toString (sfi = 000003D1D2A98A99)> ]
 -> 000001CDADCA2198 @   34 : 5d f7 f6 0a       CallProperty0 r3, r4, [10]
      [          r3 -> 0x01cdadc90a41 <JSFunction toString (sfi = 000003D1D2A98A99)> ]
      [          r4 -> 0x01cdadca2131 <HeapNumber 4.61169e+18> ]
 -> 000001CDADCA219C @   38 : c1                Star3
      [ accumulator -> 0x01cdadca2309 <String[19]: "4611686018427388000"> ]
 -> 000001CDADCA219D @   39 : 5e f9 f8 f7 0c    CallProperty1 r1, r2, r3, [12]
      [          r1 -> 0x01cdadc88849 <JSFunction log (sfi = 000003D1D2A93899)> ]
      [          r2 -> 0x01cdadc886b9 <console map = 00000199F1B81F71> ]
      [          r3 -> 0x01cdadca2309 <String[19]: "4611686018427388000"> ]

 -> 000001CDADCA21A2 @   44 : 13 01             LdaConstant [1]
      [ accumulator <- 0x01cdadca2131 <HeapNumber 4.61169e+18> ]
 -> 000001CDADCA21A4 @   46 : 23 02 00          StaGlobal [2], [0]
      [ accumulator -> 0x01cdadca2131 <HeapNumber 4.61169e+18> ]
      [ accumulator <- 0x01cdadca2131 <HeapNumber 4.61169e+18> ]
 -> 000001CDADCA21A7 @   49 : 21 03 02          LdaGlobal [3], [2]
      [ accumulator <- 0x01cdadc886b9 <console map = 00000199F1B81F71> ]
 -> 000001CDADCA21AA @   52 : c2                Star2
      [ accumulator -> 0x01cdadc886b9 <console map = 00000199F1B81F71> ]
 -> 000001CDADCA21AB @   53 : 2d f8 04 04       LdaNamedProperty r2, [4], [4]
      [          r2 -> 0x01cdadc886b9 <console map = 00000199F1B81F71> ]
      [ accumulator <- 0x01cdadc88849 <JSFunction log (sfi = 000003D1D2A93899)> ]
 -> 000001CDADCA21AF @   57 : c3                Star1
      [ accumulator -> 0x01cdadc88849 <JSFunction log (sfi = 000003D1D2A93899)> ]
 -> 000001CDADCA21B0 @   58 : 21 02 06          LdaGlobal [2], [6]
      [ accumulator <- 0x01cdadca2131 <HeapNumber 4.61169e+18> ]
 -> 000001CDADCA21B3 @   61 : c0                Star4
      [ accumulator -> 0x01cdadca2131 <HeapNumber 4.61169e+18> ]
 -> 000001CDADCA21B4 @   62 : 2d f6 05 08       LdaNamedProperty r4, [5], [8]
      [          r4 -> 0x01cdadca2131 <HeapNumber 4.61169e+18> ]
      [ accumulator <- 0x01cdadc90a41 <JSFunction toString (sfi = 000003D1D2A98A99)> ]
 -> 000001CDADCA21B8 @   66 : c1                Star3
      [ accumulator -> 0x01cdadc90a41 <JSFunction toString (sfi = 000003D1D2A98A99)> ]
 -> 000001CDADCA21B9 @   67 : 5d f7 f6 0e       CallProperty0 r3, r4, [14]
      [          r3 -> 0x01cdadc90a41 <JSFunction toString (sfi = 000003D1D2A98A99)> ]
      [          r4 -> 0x01cdadca2131 <HeapNumber 4.61169e+18> ]
 -> 000001CDADCA21BD @   71 : c1                Star3
      [ accumulator -> 0x01cdadca2309 <String[19]: "4611686018427388000"> ]
 -> 000001CDADCA21BE @   72 : 5e f9 f8 f7 10    CallProperty1 r1, r2, r3, [16]
      [          r1 -> 0x01cdadc88849 <JSFunction log (sfi = 000003D1D2A93899)> ]
      [          r2 -> 0x01cdadc886b9 <console map = 00000199F1B81F71> ]
      [          r3 -> 0x01cdadca2309 <String[19]: "4611686018427388000"> ]

 -> 000001CDADCA21C3 @   77 : 21 03 02          LdaGlobal [3], [2]
      [ accumulator <- 0x01cdadc886b9 <console map = 00000199F1B81F71> ]
 -> 000001CDADCA21C6 @   80 : c2                Star2
      [ accumulator -> 0x01cdadc886b9 <console map = 00000199F1B81F71> ]
 -> 000001CDADCA21C7 @   81 : 2d f8 04 04       LdaNamedProperty r2, [4], [4]
      [          r2 -> 0x01cdadc886b9 <console map = 00000199F1B81F71> ]
      [ accumulator <- 0x01cdadc88849 <JSFunction log (sfi = 000003D1D2A93899)> ]
 -> 000001CDADCA21CB @   85 : c3                Star1
      [ accumulator -> 0x01cdadc88849 <JSFunction log (sfi = 000003D1D2A93899)> ]
 -> 000001CDADCA21CC @   86 : 21 06 12          LdaGlobal [6], [18]
      [ accumulator <- 0x01cdadc87429 <JSFunction BigInt (sfi = 000003D1D2A93189)> ]
 -> 000001CDADCA21CF @   89 : c0                Star4
      [ accumulator -> 0x01cdadc87429 <JSFunction BigInt (sfi = 000003D1D2A93189)> ]
 -> 000001CDADCA21D0 @   90 : 21 02 06          LdaGlobal [2], [6]
      [ accumulator <- 0x01cdadca2131 <HeapNumber 4.61169e+18> ]
 -> 000001CDADCA21D3 @   93 : bf                Star5
      [ accumulator -> 0x01cdadca2131 <HeapNumber 4.61169e+18> ]
 -> 000001CDADCA21D4 @   94 : 62 f6 f5 14       CallUndefinedReceiver1 r4, r5, [20]
      [          r4 -> 0x01cdadc87429 <JSFunction BigInt (sfi = 000003D1D2A93189)> ]
      [          r5 -> 0x01cdadca2131 <HeapNumber 4.61169e+18> ]
 -> 000001CDADCA21D8 @   98 : c0                Star4
      [ accumulator -> 0x020bc06cfde9 <BigInt 4611686018427387904> ]
 -> 000001CDADCA21D9 @   99 : 2d f6 05 16       LdaNamedProperty r4, [5], [22]
      [          r4 -> 0x020bc06cfde9 <BigInt 4611686018427387904> ]
      [ accumulator <- 0x01cdadc87661 <JSFunction toString (sfi = 000003D1D2A93209)> ]
 -> 000001CDADCA21DD @  103 : c1                Star3
      [ accumulator -> 0x01cdadc87661 <JSFunction toString (sfi = 000003D1D2A93209)> ]
 -> 000001CDADCA21DE @  104 : 5d f7 f6 18       CallProperty0 r3, r4, [24]
      [          r3 -> 0x01cdadc87661 <JSFunction toString (sfi = 000003D1D2A93209)> ]
      [          r4 -> 0x020bc06cfde9 <BigInt 4611686018427387904> ]
 -> 000001CDADCA21E2 @  108 : c1                Star3
      [ accumulator -> 0x020bc06cfe01 <String[19]: "4611686018427387904"> ]
 -> 000001CDADCA21E3 @  109 : 5e f9 f8 f7 1a    CallProperty1 r1, r2, r3, [26]
      [          r1 -> 0x01cdadc88849 <JSFunction log (sfi = 000003D1D2A93899)> ]
      [          r2 -> 0x01cdadc886b9 <console map = 00000199F1B81F71> ]
      [          r3 -> 0x020bc06cfe01 <String[19]: "4611686018427387904"> ]

 -> 000001CDADCA21E8 @  114 : c4                Star0
      [ accumulator -> 0x02e7aa9815b9 <undefined> ]
 -> 000001CDADCA21E9 @  115 : a9                Return
      [ accumulator -> 0x02e7aa9815b9 <undefined> ]
```   
好了，今天到这里，下次见。    
**个人能力有限，有不足与纰漏，欢迎批评指正**  
**微信：qq9123013  备注：v8交流    邮箱：v8blink@outlook.com**  
