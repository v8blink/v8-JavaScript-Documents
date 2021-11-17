# 《Chrome V8原理讲解》20.编译链1：语法分析，被遗忘的细节  
![avatar](../v8.png)   
# 1 摘要
第三、四、五三篇文章对V8编译流程的主要功能做了介绍，在基础之上，接下来的几篇文章是编译专题，讲解V8编译链，从读取Javascript源码文件开始，到字节码的生成，详细说明编译过程和技术细节。编译专题的知识点包括：生成Token、生成AST、生成常量池、生成Bytecode和Sharedfunction。本文讲编译的准备工作，Javascirpt源码的读取与转码（章节2）；语法分析的准备工作（章节3）。   
# 2 读取Javascript源码  
测试源码如下：  
```c++
function ignition(s) {
    this.slogan=s;
	this.start=function(){eval('console.log(this.slogan);')}
}
worker = new ignition("here we go!");
worker.start();
```
Javascript源码先转成V8的内部字符串，内部字符串编译后生成Sharedfunction，Sharedfunction绑定Context等信息后生成JSfunction后交给执行单元。从读取Javascript源码讲起，源码如下：  
```c++
1.  bool SourceGroup::Execute(Isolate* isolate) {
2.  //............省略很多..................
3.      // Use all other arguments as names of files to load and run.
4.      HandleScope handle_scope(isolate);
5.      Local<String> file_name =
6.          String::NewFromUtf8(isolate, arg, NewStringType::kNormal)
7.              .ToLocalChecked();
8.      Local<String> source = ReadFile(isolate, arg);
9.      if (source.IsEmpty()) {
10.        printf("Error reading '%s'\n", arg);
11.        base::OS::ExitProcess(1);
12.      }
13.      Shell::set_script_executed();
14.      if (!Shell::ExecuteString(isolate, source, file_name, Shell::kNoPrintResult,
15.                                Shell::kReportExceptions,
16.                                Shell::kProcessMessageQueue)) {
17.        success = false;
18.        break;
19.      }
20.    }
21.    return success;
22.  }
```  
我用d8做讲解，d8方便加载Javascript源码，不需要重复造轮子。代码5行`file_name`的值是test.js；代码8行读取文件内容，`ReadFile()`代码如下：  
```c++
1.  Local<String> Shell::ReadFile(Isolate* isolate, const char* name) {
2.  //只保留最重要的部分...............................
3.    char* chars = static_cast<char*>(file->memory());
4.    Local<String> result;
5.    if (i::FLAG_use_external_strings && i::String::IsAscii(chars, size)) {
6.      String::ExternalOneByteStringResource* resource =
7.          new ExternalOwningOneByteStringResource(std::move(file));
8.      result = String::NewExternalOneByte(isolate, resource).ToLocalChecked();
9.    } else {
10.      result = String::NewFromUtf8(isolate, chars, NewStringType::kNormal, size)
11.                   .ToLocalChecked();
12.    }
13.    return result;
14.  }
```  
代码3行读取文件内容；代码5行判断`i::FLAG_use_external_strings`和ASCII字符，代码10行返回UTF8编码的Javascript源码。  
进入`bool SourceGroup::Execute()`代码14行，源码如下：  
```c++
1.  bool Shell::ExecuteString(Isolate* isolate, Local<String> source,
2.                      Local<Value> name, PrintResult print_result,
3.                      ReportExceptions report_exceptions,
4.                      ProcessMessageQueue process_message_queue) {
5.  	//省略很多............................
6.  bool success = true;
7.  {
8.    if (options.compile_options == ScriptCompiler::kConsumeCodeCache) {
9.  	//省略很多............................
10.       } else if (options.stress_background_compile) {
11.  	//省略很多............................
12.       } else {
13.         ScriptCompiler::Source script_source(source, origin);
14.         maybe_script = ScriptCompiler::Compile(context, &script_source,
15.                                                options.compile_options);
16.       }
17.       Local<Script> script;
18.       if (!maybe_script.ToLocal(&script)) {
19.         // Print errors that happened during compilation.
20.         if (report_exceptions) ReportException(isolate, &try_catch);
21.         return false;
22.       }
23.       if (options.code_cache_options ==
24.           ShellOptions::CodeCacheOptions::kProduceCache) {
25.  	//省略很多............................
26.       }
27.       maybe_result = script->Run(realm);//这是代码执行.....................
28.  }
29.  }
```   
省略了不执行的代码，代码13行，把Javascript源码封装成`ScriptCompiler::Source`；代码14行，ScriptCompiler::Compile是编译入口，开始进入编译阶段。  
# 3 语法分析器初始化
编译的第一阶段是词法分析，生成Token字；第二阶段是语法分析，生成语法树；V8的编译工具链中，先启动语法分析器，它读取Token字失败时启动词法分析器工作，按照这一流程，我们先讲解语法分析器的初始化。  
`ScriptCompiler::Compile()`方法内部调用`CompileUnboundInternal()`方法，源码如下：  
```c++
1.  MaybeLocal<UnboundScript> ScriptCompiler::CompileUnboundInternal(
2.      Isolate* v8_isolate, Source* source, CompileOptions options,
3.      NoCacheReason no_cache_reason) {
4.  //省略很多................
5.    i::Handle<i::String> str = Utils::OpenHandle(*(source->source_string));
6.    i::Handle<i::SharedFunctionInfo> result;
7.    i::Compiler::ScriptDetails script_details = GetScriptDetails(
8.        isolate, source->resource_name, source->resource_line_offset,
9.        source->resource_column_offset, source->source_map_url,
10.        source->host_defined_options);
11.    i::MaybeHandle<i::SharedFunctionInfo> maybe_function_info =
12.        i::Compiler::GetSharedFunctionInfoForScript(
13.            isolate, str, script_details, source->resource_options, nullptr,
14.            script_data, options, no_cache_reason, i::NOT_NATIVES_CODE);
15.    if (options == kConsumeCodeCache) {
16.      source->cached_data->rejected = script_data->rejected();
17.    }
18.    delete script_data;
19.    has_pending_exception = !maybe_function_info.ToHandle(&result);
20.    RETURN_ON_FAILED_EXECUTION(UnboundScript);
21.    RETURN_ESCAPED(ToApiHandle<UnboundScript>(result));
22.  }
```  
“Bind”（绑定）是V8中使用的语术，作用是绑定上下文（context）。“Unbound”是没有绑定上下文的函数，即Sharedfunction，类似DLL函数，使用之前要配置相关信息。代码7行，`GetScriptDetails()`是计算行、列偏移量等信息；代11行`Sharedfunction()`，从编译缓存中读取Sharedfunction，缓存缺失时启动编译器，编译源码生成并返回Sharedfunction，源码如下：  
```c++
1.  MaybeHandle<SharedFunctionInfo> Compiler::GetSharedFunctionInfoForScript(
2.      Isolate* isolate, Handle<String> source,
3.      const Compiler::ScriptDetails& script_details,
4.   .................) {
5.  //省略很多.........................
6.  		{
7.      maybe_result = compilation_cache->LookupScript(
8.          source, script_details.name_obj, script_details.line_offset,
9.          script_details.column_offset, origin_options, isolate->native_context(),
10.          language_mode);
11.    }
12.    if (maybe_result.is_null()) {
13.      ParseInfo parse_info(isolate);
14.      // No cache entry found compile the script.
15.      NewScript(isolate, &parse_info, source, script_details, origin_options,
16.                natives);
17.      // Compile the function and add it to the isolate cache.
18.      if (origin_options.IsModule()) parse_info.set_module();
19.      parse_info.set_extension(extension);
20.      parse_info.set_eager(compile_options == ScriptCompiler::kEagerCompile);
21.      parse_info.set_language_mode(
22.          stricter_language_mode(parse_info.language_mode(), language_mode));
23.      maybe_result = CompileToplevel(&parse_info, isolate, &is_compiled_scope);
24.      Handle<SharedFunctionInfo> result;
25.      if (extension == nullptr && maybe_result.ToHandle(&result)) {
26.        DCHECK(is_compiled_scope.is_compiled());
27.        compilation_cache->PutScript(source, isolate->native_context(),
28.                                     language_mode, result);
29.      } else if (maybe_result.is_null() && natives != EXTENSION_CODE) {
30.        isolate->ReportPendingMessages();
31.      }
32.    }
33.    return maybe_result;
34.  }
```
代码7行查询`compilation_cache`上篇文章讲过，初次查询结果为空。代码13行创建ParseInfo实例，为语法分析器（Parser）做准备工作。代码15行初始化Parser_info,源码如下：  
```c++
1.  Handle<Script> NewScript(Isolate* isolate, ParseInfo* parse_info,
2.                           Handle<String> source,
3.                           Compiler::ScriptDetails script_details,
4.                           ScriptOriginOptions origin_options,
5.                           NativesFlag natives) {
6.    Handle<Script> script =
7.        parse_info->CreateScript(isolate, source, origin_options, natives);
8.    Handle<Object> script_name;
9.    if (script_details.name_obj.ToHandle(&script_name)) {
10.      script->set_name(*script_name);
11.      script->set_line_offset(script_details.line_offset);
12.      script->set_column_offset(script_details.column_offset);
13.    }
14.    Handle<Object> source_map_url;
15.    if (script_details.source_map_url.ToHandle(&source_map_url)) {
16.      script->set_source_mapping_url(*source_map_url);
17.    }
18.    Handle<FixedArray> host_defined_options;
19.    if (script_details.host_defined_options.ToHandle(&host_defined_options)) {
20.      script->set_host_defined_options(*host_defined_options);
21.    }
22.    return script;
23.  }
```  
代码6~12行，把源码封装到Parser_info中，设置行、例偏移量信息。  
回到`Compiler::GetSharedFunctionInfoForScript()`，代码23行，进入`CompileToplevel()`，源码如下：  
```c++
1.  MaybeHandle<SharedFunctionInfo> CompileToplevel(
2.      ParseInfo* parse_info, Isolate* isolate,
3.      IsCompiledScope* is_compiled_scope) {
4.  //省略很多.........................
5.    if (parse_info->literal() == nullptr &&
6.        !parsing::ParseProgram(parse_info, isolate)) {
7.      return MaybeHandle<SharedFunctionInfo>();
8.    }
9.  //省略很多.........................
10.    MaybeHandle<SharedFunctionInfo> shared_info =
11.        GenerateUnoptimizedCodeForToplevel(
12.            isolate, parse_info, isolate->allocator(), is_compiled_scope);
13.    if (shared_info.is_null()) {
14.      FailWithPendingException(isolate, parse_info,
15.                               Compiler::ClearExceptionFlag::KEEP_EXCEPTION);
16.      return MaybeHandle<SharedFunctionInfo>();
17.    }
18.    FinalizeScriptCompilation(isolate, parse_info);
19.    return shared_info;
20.  }
```  
代码5行`literal()`判断抽象语法树是否存在，首次执行为空，所以进入代码6行，开始语法分析，源码如下：  
```c++
1.  bool ParseProgram(ParseInfo* info, Isolate* isolate,
2.                    ReportErrorsAndStatisticsMode mode) {
3.  //省略代码..............................
4.    Parser parser(info);
5.    FunctionLiteral* result = nullptr;
6.    result = parser.ParseProgram(isolate, info);
7.    info->set_literal(result);
8.    if (result) {
9.      info->set_language_mode(info->literal()->language_mode());
10.      if (info->is_eval()) {
11.        info->set_allow_eval_cache(parser.allow_eval_cache());
12.      }
13.    }
14.    if (mode == ReportErrorsAndStatisticsMode::kYes) {
15.  //省略代码..............................
16.    }
17.    return (result != nullptr);
18.  }
```  
代码4行，使用Parse_info信息创建Parser实例，源码如下：  
```c++
1.  Parser::Parser(ParseInfo* info)
2.      : ParserBase<Parser>(info->zone(), &scanner_, info->stack_limit(),
3.                           info->extension(), info->GetOrCreateAstValueFactory(),
4.                           info->pending_error_handler(),
5.                           info->runtime_call_stats(), info->logger(),
6.                           info->script().is_null() ? -1 : info->script()->id(),
7.                           info->is_module(), true),
8.        info_(info),
9.        scanner_(info->character_stream(), info->is_module()),
10.        preparser_zone_(info->zone()->allocator(), ZONE_NAME),
11.        reusable_preparser_(nullptr),
12.        mode_(PARSE_EAGERLY),  // Lazy mode must be set explicitly.
13.        source_range_map_(info->source_range_map()),
14.        target_stack_(nullptr),
15.        total_preparse_skipped_(0),
16.        consumed_preparse_data_(info->consumed_preparse_data()),
17.        preparse_data_buffer_(),
18.        parameters_end_pos_(info->parameters_end_pos()) {
19.    bool can_compile_lazily = info->allow_lazy_compile() && !info->is_eager();
20.    set_default_eager_compile_hint(can_compile_lazily
21.                                       ? FunctionLiteral::kShouldLazyCompile
22.                                       : FunctionLiteral::kShouldEagerCompile);
23.    allow_lazy_ = info->allow_lazy_compile() && info->allow_lazy_parsing() &&
24.                  info->extension() == nullptr && can_compile_lazily;
25.    set_allow_natives(info->allow_natives_syntax());
26.    set_allow_harmony_dynamic_import(info->allow_harmony_dynamic_import());
27.    set_allow_harmony_import_meta(info->allow_harmony_import_meta());
28.    set_allow_harmony_nullish(info->allow_harmony_nullish());
29.    set_allow_harmony_optional_chaining(info->allow_harmony_optional_chaining());
30.    set_allow_harmony_private_methods(info->allow_harmony_private_methods());
31.    for (int feature = 0; feature < v8::Isolate::kUseCounterFeatureCount;
32.         ++feature) {
33.      use_counts_[feature] = 0;
34.    }
35.  }
```  
代码8-18行，从Parser_Info中获取信息；代码19-23行是lazy compile开关，`allow_lazy_`是最终结果；代码25是否支持natives语法，也就是Javascript源码中是否允许使用以`%`开头的命令；代码26~30行是否支持私有方法等等。至此，语法分析器初始化工作完毕。    
创建`Paser`实例后，返回`bool ParseProgram()`，代码6行，进行语法分析，期间还需要创建扫描器，下次讲解。  
**技术总结**  
**（1）** Javascript源码进入V8后需要转码；  
**（2）** Javascript源码在V8内的表示是`Source`类,全称是`v8::internal::source`；  
**（3）** 先查编译缓存，缓存缺失时启动编译；  
**（4）** 语法分析器先启动，Token缺失时启动词法分析器。  
好了，今天到这里，下次见。   

**恳请读者批评指正、提出宝贵意见**  
**微信：qq9123013  备注：v8交流    邮箱：v8blink@outlook.com**

