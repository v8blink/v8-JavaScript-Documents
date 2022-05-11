# 《Chrome V8 源码》48. 弱类型加法的奥秘，"+" 源码分析  
# 0 通知  
**（1）** 《Chrome V8 源码》 不在安全客发表了，请各位关注我的知乎。  
**（2）** V8 源码视频课程即将上线。  
**（3）** 最近一直在忙项目验收，还没结束。文章更新慢了些，大家多多理解，谢谢。  
# 1 介绍  
JavaScript 是弱类型语言，那么它的变量、表达式等在参与运算时，即使类型不正确，也能通过隐式转换来得到正确地类型，这对使用者而言，就好像所有类型都能进行所有运算。本文通过分析 V8 的加法源码，带领大家了解 JavaScript 加法运算的细节，看看 V8 到底是怎么做的。  
# 2 ADD_HANDLER
V8 从字节码开始执行 JavaScript 源码，所以我们从加法的字节码处理程序入手分析，源码如下：  
```c++
IGNITION_HANDLER(Add, InterpreterBinaryOpAssembler) {
  BinaryOpWithFeedback(&BinaryOpAssembler::Generate_AddWithFeedback);
}
```
JavaScript 源码中的所有加法运算都要从上面这个 ADD_HANDLER 开始，它内部还会调用 Builtin::kAdd 等功能。而在TurboFan中，ADD_HANDLER 有可能被优化为 NewConsString 或 StringConcat 等功能，下面跟随代码来看看 V8 是如何把这些功能组织起来的。   
# 3 Generate_AddWithFeedback  
```c++
1.  TNode<Object> BinaryOpAssembler::Generate_AddWithFeedback() {
2.    Label if_lhsisnotsmi(this,
3.                         rhs_known_smi ? Label::kDeferred : Label::kNonDeferred);
4.    Branch(TaggedIsNotSmi(lhs), &if_lhsisnotsmi, &if_lhsissmi);
5.    BIND(&if_lhsissmi);
6.    {
7.      TNode<Smi> lhs_smi = CAST(lhs);
8.      if (!rhs_known_smi) {
9.        Label if_rhsissmi(this), if_rhsisnotsmi(this);
10.        Branch(TaggedIsSmi(rhs), &if_rhsissmi, &if_rhsisnotsmi);
11.        BIND(&if_rhsisnotsmi);
12.        {
13.          TNode<HeapObject> rhs_heap_object = CAST(rhs);
14.          GotoIfNot(IsHeapNumber(rhs_heap_object), &check_rhsisoddball);
15.          var_fadd_lhs = SmiToFloat64(lhs_smi);
16.          var_fadd_rhs = LoadHeapNumberValue(rhs_heap_object);
17.          Goto(&do_fadd);      }
18.        BIND(&if_rhsissmi);    }
19.      {
20.        TNode<Smi> rhs_smi = CAST(rhs);
21.        Label if_overflow(this,
22.                          rhs_known_smi ? Label::kDeferred : Label::kNonDeferred);
23.        TNode<Smi> smi_result = TrySmiAdd(lhs_smi, rhs_smi, &if_overflow);
24.        {
25.          var_type_feedback = SmiConstant(BinaryOperationFeedback::kSignedSmall);
26.          UpdateFeedback(var_type_feedback.value(), maybe_feedback_vector(),
27.                         slot_id, update_feedback_mode);
28.          var_result = smi_result;
29.          Goto(&end);      }
30.        BIND(&if_overflow);
31.        {
32.          var_fadd_lhs = SmiToFloat64(lhs_smi);
33.          var_fadd_rhs = SmiToFloat64(rhs_smi);
34.          Goto(&do_fadd);
35.        }    }  }
36.    BIND(&if_lhsisnotsmi);
37.    {
38.      TNode<HeapObject> lhs_heap_object = CAST(lhs);
39.      GotoIfNot(IsHeapNumber(lhs_heap_object), &if_lhsisnotnumber);
40.      if (!rhs_known_smi) {
41.        Label if_rhsissmi(this), if_rhsisnotsmi(this);
42.        Branch(TaggedIsSmi(rhs), &if_rhsissmi, &if_rhsisnotsmi);
43.        BIND(&if_rhsisnotsmi);
44.        {
45.          TNode<HeapObject> rhs_heap_object = CAST(rhs);
46.          GotoIfNot(IsHeapNumber(rhs_heap_object), &check_rhsisoddball);
47.          var_fadd_lhs = LoadHeapNumberValue(lhs_heap_object);
48.          var_fadd_rhs = LoadHeapNumberValue(rhs_heap_object);
49.          Goto(&do_fadd);
50.        }
51.        BIND(&if_rhsissmi);
52.      }
53.      {
54.        var_fadd_lhs = LoadHeapNumberValue(lhs_heap_object);
55.        var_fadd_rhs = SmiToFloat64(CAST(rhs));
56.        Goto(&do_fadd);
57.      }
58.    }
59.    BIND(&do_fadd);
60.    {
61.      var_type_feedback = SmiConstant(BinaryOperationFeedback::kNumber);
62.      UpdateFeedback(var_type_feedback.value(), maybe_feedback_vector(), slot_id,
63.                     update_feedback_mode);
64.      TNode<Float64T> value =
65.          Float64Add(var_fadd_lhs.value(), var_fadd_rhs.value());
66.      TNode<HeapNumber> result = AllocateHeapNumberWithValue(value);
67.      var_result = result;
68.      Goto(&end);
69.    }
70.    BIND(&if_lhsisnotnumber);
71.    {
72.      TNode<Uint16T> lhs_instance_type = LoadInstanceType(CAST(lhs));
73.      TNode<BoolT> lhs_is_oddball =
74.          InstanceTypeEqual(lhs_instance_type, ODDBALL_TYPE);
75.      Branch(lhs_is_oddball, &if_lhsisoddball, &if_lhsisnotoddball);
76.      BIND(&if_lhsisoddball);
77.      {
78.        GotoIf(TaggedIsSmi(rhs), &call_with_oddball_feedback);
79.        Branch(IsHeapNumber(CAST(rhs)), &call_with_oddball_feedback,
80.               &check_rhsisoddball);
81.      }
82.      BIND(&if_lhsisnotoddball);
83.      {
84.        GotoIf(TaggedIsSmi(rhs), &call_with_any_feedback);
85.        TNode<HeapObject> rhs_heap_object = CAST(rhs);
86.        GotoIf(IsStringInstanceType(lhs_instance_type), &lhs_is_string);
87.        GotoIf(IsBigIntInstanceType(lhs_instance_type), &lhs_is_bigint);
88.        Goto(&call_with_any_feedback);
89.        BIND(&lhs_is_bigint);
90.        Branch(IsBigInt(rhs_heap_object), &bigint, &call_with_any_feedback);
91.        BIND(&lhs_is_string);
92.        {
93.          TNode<Uint16T> rhs_instance_type = LoadInstanceType(rhs_heap_object);
94.          GotoIfNot(IsStringInstanceType(rhs_instance_type),
95.                    &call_with_any_feedback);
96.          var_type_feedback = SmiConstant(BinaryOperationFeedback::kString);
97.          UpdateFeedback(var_type_feedback.value(), maybe_feedback_vector(),
98.                         slot_id, update_feedback_mode);
99.          var_result =
100.              CallBuiltin(Builtin::kStringAdd_CheckNone, context(), lhs, rhs);
101.          Goto(&end);
102.        }  } }
103.    BIND(&check_rhsisoddball);
104.    {
105.      TNode<Uint16T> rhs_instance_type = LoadInstanceType(CAST(rhs));
106.      TNode<BoolT> rhs_is_oddball =
107.          InstanceTypeEqual(rhs_instance_type, ODDBALL_TYPE);
108.      GotoIf(rhs_is_oddball, &call_with_oddball_feedback);
109.      Goto(&call_with_any_feedback);
110.    }
111.    BIND(&bigint);
112.    {
113.      var_result = CallBuiltin(Builtin::kBigIntAddNoThrow, context(), lhs, rhs);
114.      GotoIf(TaggedIsSmi(var_result.value()), &bigint_too_big);
115.      var_type_feedback = SmiConstant(BinaryOperationFeedback::kBigInt);
116.      UpdateFeedback(var_type_feedback.value(), maybe_feedback_vector(), slot_id,
117.                     update_feedback_mode);
118.      Goto(&end);
119.      BIND(&bigint_too_big);
120.      {
121.        UpdateFeedback(SmiConstant(BinaryOperationFeedback::kAny),
122.                       maybe_feedback_vector(), slot_id, update_feedback_mode);
123.        ThrowRangeError(context(), MessageTemplate::kBigIntTooBig);
124.      }  }
125.    BIND(&call_with_oddball_feedback);
126.    {
127.      var_type_feedback = SmiConstant(BinaryOperationFeedback::kNumberOrOddball);
128.      Goto(&call_add_stub);  }
129.    BIND(&call_with_any_feedback);
130.    {
131.      var_type_feedback = SmiConstant(BinaryOperationFeedback::kAny);
132.      Goto(&call_add_stub);  }
133.    BIND(&call_add_stub);
134.    {
135.      UpdateFeedback(var_type_feedback.value(), maybe_feedback_vector(), slot_id,
136.                     update_feedback_mode);
137.      var_result = CallBuiltin(Builtin::kAdd, context(), lhs, rhs);
138.      Goto(&end);  }
139.    BIND(&end);
140.    return var_result.value();
141.  }
```    
**第一部分：左操作数是 Smi**  
第 4 行代码判断左操作数是不是 Smi；如果是 Smi 进入第 5 行代码，否则进入第 36 行代码；**提示** 我们写 JS 程序时，加法算运最常于两个数值的相加，所以最先判断左、右操作数是不是数值。    
第 7 行代码把左操作数转为 Smi；  
第 8-10 行代码判断右操作类型；  
第 12-17 行处理右操作数不是 Smi 的情况；**左、右类型不一致，需要类型转换**    
* 第 13-14 行转换右操作数为 HeapObject，且为 HeapNumber时，将左操作数（第 15 行）转为 HeapNumber，跳转到 do_fadd 标签（第 59-68行）以完成 float 加法；     
  
第 18 行处理右操作数是 Smi 的情况：
* 第 20-23 行完成 Smi 加法；**注意** TrySmiAdd() 求和的同时还负责判断结果是否越界，Smi是 31bit；  
* 第 26 行更新 Feedback，TurboFan优化时使用，下篇文章讲解；   
* 第 30-34 行加法越界时采用fload 加法（第 59 行）；  
  
**第二部分：左操作是 HeapNumber**  

第 38-39 行代码把左操作数转为 HeapObject，并判断是否为 HeapNumber，不是就跳转到第 70 行代码；  
第 40-58 行左操作数是 HeapNumber:  
* 第 44-49 行判断出右操作数是 HeapNumber，跳转到第 59 行以完成 float 加法；
* 第 54-56 行判断出右操作数是 Smi，转换右操作数为 HeapNumber 后跳转到第 59 行；  
  
第 59-67 行完成 float 加法；  

**第三部分：左操作是 HeapObject，例如：BigInt、string**    
第 72-91 行取出左、右操作数的 instance_type 类型标记符，并进一步判断两个操作的类型。  
第 96-100 行左、右操作数均为 String、调用 Builtin::kStringAdd_CheckNone 完成字符串加法；  
第 91-94 行左操作数是 String 且右操作数不是 String 时跳转到 131 行；  
第 131-135 行更新 feedback 为 kAny 之后，调用Builtin::kAdd 完成“字符串” 和 “非字符串” 的加法。  
上述源码中的其它情况请自行分析，下面简单讲解Builtin:kAdd源码  
# 4 Builtin:kAdd  
其源码采用 TQ 编写，源码如下：
```c++
1.  transitioning builtin Add(implicit context: Context)(
2.      leftArg: JSAny, rightArg: JSAny): JSAny {
3.    try {
4.      while (true) {
5.        typeswitch (left) {
6.          case (left: Smi): {
7.  //省略................
8.          }
9.          case (left: HeapNumber): {
10.            typeswitch (right) {
11.  //省略................
12.          }
13.          case (left: BigInt): {
14.  //省略................
15.          }
16.          case (left: String): {
17.            goto StringAddConvertRight(left, right);
18.          }
19.          case (leftReceiver: JSReceiver): {
20.  //省略................
21.          }
22.          case (HeapObject): {
23.  //省略................
24.          }      }    }  }
25.    unreachable;
26.  }
```  
上述代码先判断左操作数的类型、再判断右操作类型，然后进行类型转换并计算结果。第 17 行代给出了左操作数是字符串、右操作数非字符串的加法实现，即 Builtin 方法 StringAddConvertRight，该方法采用 TQ 实现，其源码在 builtins-string.tq中，下篇文章讲解。
# 5 技术总结
**（1）** 加法操作（所有操作）都是先由字节码开始，达到热点条件时进入TurboFan；  
**（2）** Generate_AddWithFeedback 的 “Feedback” 收集收左、右操作数的类型信息，用于TurboFan的投机优化；  
**（3）** Generate_AddWithFeedback 做不了的加法使用 builtin::kAdd 完成，而 kAdd 还会一步做功能细分。

好了，今天到这里，下次见。    
**个人能力有限，有不足与纰漏，欢迎批评指正**  
**微信：qq9123013  备注：v8交流    知乎：https://www.zhihu.com/people/v8blink**  



