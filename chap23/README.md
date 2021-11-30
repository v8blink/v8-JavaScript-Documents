# 《Chrome V8源码》23.编译4：数据结构、重要的自动机  
![avatar](../v8.png)   
# 1 摘要  
本篇是编译链专题的第四篇，梳理V8编译期间涉及到的数据结构和自动机，并解释其作用。  
# 2 Parse_Info  
```c++
1.  // A container for the inputs, configuration options, and outputs of parsing.
2.  class V8_EXPORT_PRIVATE ParseInfo {
3.  public:
4.  //省略................
5.  AstValueFactory* ast_value_factory() const {
6.    DCHECK(ast_value_factory_.get());
7.    return ast_value_factory_.get();
8.  }
9.  const AstRawString* function_name() const { return function_name_; }
10.  void set_function_name(const AstRawString* function_name) {
11.    function_name_ = function_name;
12.  }
13.  FunctionLiteral* literal() const { return literal_; }
14.  void set_literal(FunctionLiteral* literal) { literal_ = literal; }
15.  private:
16.   //------------- Inputs to parsing and scope analysis -----------------------
17.   const UnoptimizedCompileFlags flags_;
18.   UnoptimizedCompileState* state_;
19.   std::unique_ptr<Zone> zone_;
20.     v8::Extension* extension_;
21.     DeclarationScope* script_scope_;
22.     uintptr_t stack_limit_;
23.     int parameters_end_pos_;
24.     int max_function_literal_id_;
25.     //----------- Inputs+Outputs of parsing and scope analysis -----------------
26.     std::unique_ptr<Utf16CharacterStream> character_stream_;
27.     std::unique_ptr<ConsumedPreparseData> consumed_preparse_data_;
28.     std::unique_ptr<AstValueFactory> ast_value_factory_;
29.     const AstRawString* function_name_;
30.     RuntimeCallStats* runtime_call_stats_;
31.     SourceRangeMap* source_range_map_;  // Used when block coverage is enabled.
32.     //----------- Output of parsing and scope analysis ------------------------
33.     FunctionLiteral* literal_;
34.     bool allow_eval_cache_ : 1;
35.   #if V8_ENABLE_WEBASSEMBLY
36.     bool contains_asm_module_ : 1;
37.   #endif  // V8_ENABLE_WEBASSEMBLY
38.     LanguageMode language_mode_ : 1;
39.   };
```  
把JavaScript源码（v8::internal::source）封装成Parse_Info。上述第5行代码是AST的工厂方法；13行代码是AST成员的获取方法；17行代码（UnoptimizedCompileFlags flags_）作用是优化标记；33行代码存储生成的AST。AST是最重要的成员，在学习这部分源码时只要盯住这个成员就可以了，其它成员和方法可以忽略。  
# 3 AST树  
```c++
1.  class FunctionLiteral final : public Expression {
2.   public:
3.    template <typename IsolateT>//省略很多代码...............
4.    MaybeHandle<String> GetName(IsolateT* isolate) const {  }
5.    const AstConsString* raw_name() const { return raw_name_; }
6.    DeclarationScope* scope() const { return scope_; }
7.    ZonePtrList<Statement>* body() { return &body_; }
8.    bool is_anonymous_expression() const {
9.      return syntax_kind() == FunctionSyntaxKind::kAnonymousExpression;
10.    }
11.    V8_EXPORT_PRIVATE LanguageMode language_mode() const;
12.    void add_expected_properties(int number_properties) {}
13.    std::unique_ptr<char[]> GetDebugName() const;
14.    Handle<String> GetInferredName(Isolate* isolate) {  }
15.    Handle<String> GetInferredName(LocalIsolate* isolate) const {}
16.    const AstConsString* raw_inferred_name() { return raw_inferred_name_; }
17.    FunctionSyntaxKind syntax_kind() const {}
18.    FunctionKind kind() const;
19.    bool IsAnonymousFunctionDefinition() const {  }
20.    int suspend_count() { return suspend_count_; }
21.    int function_literal_id() const { return function_literal_id_; }
22.    void set_function_literal_id(int function_literal_id) {  }
23.   private:
24.    const AstConsString* raw_name_;
25.    DeclarationScope* scope_;
26.    ZonePtrList<Statement> body_;
27.    AstConsString* raw_inferred_name_;
28.    Handle<String> inferred_name_;
29.    ProducedPreparseData* produced_preparse_data_;
30.  };
```   
以函数为单位生成的AST树保存在`FunctionLiteral`类的根节点中。通过`ZonePtrList<Statement> body_`成员（26行代码）可以遍历树中的每个节点。具体遍历方法可参考bytecode生成时的相关操作。通过代码4~15行可以设置函数的语言模式（strict sloppy）等相关信息。18行代码是函数的类型，定义如下： 
```c++
1.  enum FunctionKind : uint8_t {
2.    // BEGIN constructable functions
3.    kNormalFunction,kModule,kAsyncModule,kBaseConstructor,kDefaultBaseConstructor,
4.    kDefaultDerivedConstructor,
5.    kDerivedConstructor,
6.  //省略很多..............
7.    kLastFunctionKind = kClassStaticInitializerFunction,
8.  };
```   
上述代码中的类型和JavaScript语言教材中提到的函数类型并非是一一对应的，这是因为在V8实现中对JavaScript语言的函数类型划分比较详细。  
# 4 AST树节点
```c++
1.  #define DECLARATION_NODE_LIST(V) \
2.    V(VariableDeclaration)         \
3.    V(FunctionDeclaration)
4.  #define ITERATION_NODE_LIST(V) \
5.    V(DoWhileStatement)          \
6.    V(WhileStatement)            \
7.    V(ForStatement)              \
8.    V(ForInStatement)            \
9.    V(ForOfStatement)
10.  #define BREAKABLE_NODE_LIST(V) \
11.    V(Block)                     \
12.    V(SwitchStatement)
13.  #define STATEMENT_NODE_LIST(V)       \
14.    ITERATION_NODE_LIST(V)             \
15.    BREAKABLE_NODE_LIST(V)             \
16.    V(ExpressionStatement)             \
17.    V(EmptyStatement)                  \
18.    V(SloppyBlockFunctionStatement)    \
19.    V(IfStatement)                     \
20.    V(InitializeClassStaticElementsStatement)//省略很多
21.  #define LITERAL_NODE_LIST(V) \
22.    V(RegExpLiteral)           \
23.    V(ObjectLiteral)           \
24.    V(ArrayLiteral)
25.  #define EXPRESSION_NODE_LIST(V) \
26.    LITERAL_NODE_LIST(V)          \
27.    V(Assignment)                 \
28.    V(Await)                      \
29.    V(BinaryOperation)            \
30.    V(NaryOperation)              \
31.    V(Call)                       \
32.    V(YieldStar)//省略很多
33.  #define FAILURE_NODE_LIST(V) V(FailureExpression)
34.  #define AST_NODE_LIST(V)                        \
35.    DECLARATION_NODE_LIST(V)                      \
36.    STATEMENT_NODE_LIST(V)                        \
37.    EXPRESSION_NODE_LIST(V)
```   
AST树节点由宏模板实现，通过34行代码可知节点共分为三个子类型，分别是DECLARATION、STATEMENT和EXPRESSION，在构建语法树节点时EXPRESSION可能会有左、右之分（左EXPRESSION和右EXPRESSION）。  
```c++
class AstNode: public ZoneObject {
 public:
#define DECLARE_TYPE_ENUM(type) k##type,
  enum NodeType : uint8_t {
    AST_NODE_LIST(DECLARE_TYPE_ENUM) /* , */
    FAILURE_NODE_LIST(DECLARE_TYPE_ENUM)
  };
#undef DECLARE_TYPE_ENUM
//省略很多..................
}
```  
上述代码中，树节点类型由枚举成员`NodeType`表示，在构建语法的自动机中`NodeType`是判断条件。展开`FAILURE_NODE_LIST`和`AST_NODE_LIST`后可以看到所有类型的树节点。
# 5 Token宏模板  
```c++
1.  #define TOKEN_LIST(T, K)                                           \
2.    T(TEMPLATE_SPAN, nullptr, 0)                                     \
3.    T(TEMPLATE_TAIL, nullptr, 0)                                     \
4.    /* BEGIN Property */                                             \
5.    T(PERIOD, ".", 0)                                                \
6.    T(LBRACK, "[", 0)                                                \
7.    /* END Property */                                               \
8.    /* END Member */                                                 \
9.    T(QUESTION_PERIOD, "?.", 0)                                      \
10.    T(LPAREN, "(", 0)                                                \
11.    /* END PropertyOrCall */                                         \
12.    T(RPAREN, ")", 0)                                                \
13.    T(RBRACK, "]", 0)                                                \
14.    T(LBRACE, "{", 0)                                                \
15.    T(COLON, ":", 0)                                                 \
16.    T(ELLIPSIS, "...", 0)                                            \
17.    T(CONDITIONAL, "?", 3)                                           \
18.  //省略很多..................
```  
上述第5行代码表明Token的类型是`PERIOD`,对应JavaScript源码中的语法符号"."；10，12行代码表明Token的类型是左右括号；Token的其他类型还有“+，-”等操作，样例代码中已省略，请读者自行分析。以10行代码T(LPAREN, "(", 0)为例，第三个参数0是优先级，优先级的具体用法把TOKEN_LIST展开就明白了。
一宏多用是V8的代码风格，在宏定义时尽可能全面定义所有的参数，而使用时可以只用部分参数。例如下面的样例中只使用了`TOKEN_LIST`宏的第一个参数`name`，代码如下：
```c++
#define T(name, string, precedence) #name,
const char* const Token::name_[NUM_TOKENS] = {TOKEN_LIST(T, T)};
#undef T
```   
# 6 两个自动机  
词法分析和语法分析的核心功能都是由自动机实现的，源码如下：
```c++
1.  V8_INLINE Token::Value Scanner::ScanSingleToken() {
2.   Token::Value token;
3.   do {
4.     next().location.beg_pos = source_pos();
5.     if (V8_LIKELY(static_cast<unsigned>(c0_) <= kMaxAscii)) {
6.       token = one_char_tokens[c0_];
7.       switch (token) {
8.         case Token::LPAREN:
9.         case Token::RPAREN:
10.          case Token::LBRACE:
11.          case Token::RBRACE:
12.          case Token::LBRACK:
13.          case Token::RBRACK:
14.          case Token::COLON:
15.          case Token::SEMICOLON:
16.          case Token::COMMA:
17.          case Token::BIT_NOT:
18.          case Token::ILLEGAL:
19.            // One character tokens.
20.            return Select(token);
21.          case Token::CONDITIONAL:
22.            // ? ?. ?? ??=
23.            Advance();
24.            if (c0_ == '.') {
25.              Advance();
26.              if (!IsDecimalDigit(c0_)) return Token::QUESTION_PERIOD;
27.              PushBack('.');
28.            } else if (c0_ == '?') {
29.              return Select('=', Token::ASSIGN_NULLISH, Token::NULLISH);
30.            }
31.            return Token::CONDITIONAL;
32.  //省略很多
33.          case Token::WHITESPACE:
34.            token = SkipWhiteSpace();
35.            continue;
36.          case Token::NUMBER:
37.            return ScanNumber(false);
38.          case Token::IDENTIFIER:
39.            return ScanIdentifierOrKeyword();
40.          default:
41.            UNREACHABLE();
42.        }
43.      }
44.      if (IsIdentifierStart(c0_) ||
45.          (CombineSurrogatePair() && IsIdentifierStart(c0_))) {
46.        return ScanIdentifierOrKeyword();
47.      }
48.    } while (token == Token::WHITESPACE);
49.    return token;
50.  }
```
`ScanSingleToken()`扫描一个Token，8~40行代码中每个`case`代表一种Token类型。第6行代码`one_char_tokens`数组的作用是判断字符的Token类型，数组采用宏模板定义，详见第四篇文章。
```c++
1.  ParserBase<Impl>::ParseStatementListItem() {
2.    switch (peek()) {
3.      case Token::FUNCTION:
4.        return ParseHoistableDeclaration(nullptr, false);
5.      case Token::CLASS:
6.        Consume(Token::CLASS);
7.        return ParseClassDeclaration(nullptr, false);
8.      case Token::VAR:
9.      case Token::CONST:
10.        return ParseVariableStatement(kStatementListItem, nullptr);
11.      case Token::LET:
12.        if (IsNextLetKeyword()) {
13.          return ParseVariableStatement(kStatementListItem, nullptr);
14.        }
15.        break;
16.      case Token::ASYNC:
17.        if (PeekAhead() == Token::FUNCTION &&
18.            !scanner()->HasLineTerminatorAfterNext()) {
19.          Consume(Token::ASYNC);
20.          return ParseAsyncFunctionDeclaration(nullptr, false);
21.        }
22.        break;
23.      default:
24.        break;
25.    }
26.    return ParseStatement(nullptr, nullptr, kAllowLabelledFunctionStatement);
27.  }
```  
`ParseStatementListItem()`生成AST树节点的自动机，在词法分析和语法分析阶段自动机会被频繁使用。在V8源码中，跟踪上述代码的执行可以看到完整的执行流程。  
**要点总结**  
**（1）** 上述内容包括了V8编译的主要数据结构，看懂这些就能明白V8的编译原理；
**（2）** 宏模板是V8的编码风格，一宏多用的情况随处可见，简单的宏展开可以用纸笔完成，复杂的宏可以利用编译器输出预处理文件展开。  

好了，今天到这里，下次见。   

**恳请读者批评指正、提出宝贵意见**  
**微信：qq9123013  备注：v8交流    邮箱：v8blink@outlook.com**   
本文由灰豆原创发布  
转载出处： https://www.anquanke.com/post/id/259279   
安全客 - 有思想的安全新媒体   
