## UE4 断言（搬运）

原文 https://zhuanlan.zhihu.com/p/64462945

Source

    Runtime/Core/Public/Misc/AssertionMacros.h

宏

在读断言代码之前先需要注意两个宏, "DO_CHECK" & "DO_GUARD_SLOW"

    在Runtime\Core\Public\Misc\Build.h读到以下默认信息: 

DO_CHECK=1 只在打包时跟随USE_CHECKS_IN_SHIPPING的设置




DO_GUARD_SLOW=0 只在debug模式为1

	三种类型的断言  
	1. 停止执行 (check族)
		表达式为false即立刻中断，引擎崩溃  
	2. Debug(DO_GUARD_SLOW=1)时停止执行 (checkSlow族)
		仅在Debug模式下崩溃
	3. 不停止执行 (ensure族)  
		不会中断 但会生成callstack报告供debugensure会返回bool型结果


```C++
// -----------类型1--------------
// DO_CHECK=1才执行exp
// 人话: SHIPPING版本可能不执行exp
check(exp);

// DO_CHECK=1时和check一样 但DO_CHECK=0时还是会执行exp
// 人话: 和check一样，但保证执行exp
verify(exp);

// 人话: 和check, verify完全一样。 后面加的是断言失败可以print出来的调试信息
checkf(exp, ...);
verifyf(exp, ...);

// 人话: 这特么根本不是断言
// 实现是 do { exp; } while ( false );
// 建议别用
checkCode(exp);

// 没有参数 用来标记不应该运行到的代码 只DO_CHECK=1有效
// 人话: 放到运行到就代表出错的地方，比如switch的default
checkNoEntry();

// 被用来防止重入/防止递归 用它来防止一个函数被调用了第二次 只DO_CHECK=1有效
// 人话: 放到不想进入第二次/不想让它递归的函数开头
checkNoReentry(); 
checkNoRecursion();

// 用来防止想要被子类重写的虚函数在父类被调用
// 人话: 建议用纯虚函数代替
unimplemented();

// -----------类型2--------------
// 只在DO_GUARD_SLOW=1时运行这些函数 功能和上面对应的完全一致
checkSlow(exp);
checkfSlow(exp);
verifySlow(exp);

// -----------类型3--------------
// DO_CHECK=1时exp为false则会打印堆栈 把exp布尔化一下返回 
// 人话: ensure可以帮你生成堆栈信息来调试
ensure(exp);

// 这个API官方4.9版本文档还有 但4.20版本代码中已经删去了
// 人话: 别用
ensureMsg(exp, msg);

// 人话: 在ensure基础上加上自定义msg供调试
ensureMsgf(exp, msg, ...);
```

