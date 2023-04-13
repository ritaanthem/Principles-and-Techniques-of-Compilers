# Lab3 实验报告

### 小组成员：PB19050946 郑雨霏 PB19050991 陶思成  PB19111659 胡冰


## 实验难点

1. 当`if`语句和`while`语句的`endBB`基本块为空时翻译结果不符合语法规则
2. `if`语句和`while`语句的标号问题
3. `ASTVar`结点处理问题

## 实验设计

1. ### 如何设计全局变量

   > 全局变量的存在意义是将在下层才能取到的信息带回上层，或者将上层的状态信息传递给下层，一般来说有这样的需要就应该创建一个对应的全局变量，在我们的实验中，有以下全局变量：
   >
   > - `int i_val`,`float f_val`在`ASTNum`结点中被赋值，在数组创建时作为数组长度值使用。 
   > - `bool IsFloat`用于标记当前字面值是否为Float型
   > - `int lr`最重要的一个全局变量，指定了当前是在计算和处理左值还是右值。
   > - `bool debug` 用于打开和控制debug输出
   > - `int labelnum`用于全局标签计数
   > - `Function* Curr_Fun`用于保存当前处理的函数，便于使用。
   > - `bool is_return` `bool_if_is_return`,`bool else_is_return` 用于保存当前跳转状态

2. ### 遇到的难点以及解决方案

   - 当`if`语句和`while`语句的`endBB`基本块为空时翻译结果不符合语法规则

     >因为在函数中必须保证`endBB`或者其后继基本块中需要有`return`语句，故可以利用一个全局布尔变量`is_return`来记录语句中是否有`return`语句。
     >
     >- 在`if`情况下，再利用`if_is_return` 和`else_is_return` 来记录`if`语句和`else`语句中是否有`return`语句，如果`if`语句和`else`语句（如果存在`else`语句）中均有`return`语句，则说明无论如何都会`ret`，则可以在`endBB`中也要加入`return`语句来确保语法正确。
     >- 在`while`情况下，在`function declaration`的最后根据`is_return`的值来确定是否在返回，若没有返回值，则根据函数的类型添加`return`语句

     

   - `if`语句和`while`语句的标号问题

     >由于函数中可能出现多个if和while语句，故将需要将每次跳转的label区分开，有以下两种解决方案
     >
     >- 利用`sprintf`语句生成序号不同的`labelname`
     >- 直接讲`labelname`设置为空，翻译时会自动填充如不同的`label`

     

   - `ASTVar`结点处理问题

     >变量要区分是左值还是右值，通过全局枚举变量`lr`来记录该节点是左值还是右值，左值取变量地址，右值取变量的值。
     >
     >通过`node.expression`判断参数是整型/浮点型还是数组类型。在处理右值时需要注意函数调用的`形参`在符号表均为指针类型（`node.expression`均为`nullptr`），故需要在整型/浮点型中需要特别处理形参是数组、指针的情况。
     >
     >在处理`取了下标的数组变量`时，需要考虑数组下标是否合法，以及通过指针类型访问数组的情况。

   - 数组下标报错问题

     >数组下标如果是浮点数，调用`create_fptosi`接口转化为整型，再判断是否是负数，如果是负数需要调用`neg_idx_except()`，通过符号表得到该方法的地址然后使用接口`create_call`调用。

3. ### 编译器处理流程设计

   > 如上文架构分析中所述，编译器处理流程以一颗抽象语法树的根节点为开始结点，自顶向下对这颗抽象语法树进行遍历，一边从这棵树的结点中取出我们需要的关键信息，一边进行调用C++类的接口，生成供`llvm`运行的TAC代码。以下对遍历这颗树的各个结点的流程做简要介绍。
   >
   > 1. `ASTProgram`:这个结点是一切翻译的开始，也是抽象语法树的根结点，在这个结点中的下层结点是`ASTVarDeclaration`和`ASTFunDeclaration`，需要注意的是此处的下层结点指针需要调用`std::dynamic_pointer_cast`函数进行对应转化，进而遍历下层结点。
   > 2. `ASTNum`:这个结点是一切翻译的最底层，也是抽象语法树的最底层，在这个结点中会保存来自词法、语法分析器的变量字面值，这是一切数据处理的根基。由于我们是自顶向下去遍历抽象语法树，很自然的一点想法是，我们需要用全局变量将来自下层的值带到上层，方便使用。在这里我们使用了`result`这样一个变量用于在下层保存值，这样上层调用下层遍历接口之后，就能从`result`中直接取数。
   > 3. `ASTVarDeclaration`:这个结点是一切变量声明的必经之路。所有本地生命声明变量（函数形参除外）都在这里被分配空间并放入符号表中，方便后续调用取值。这里将对各种变量类型进行区分，并为之创建分配不同的空间。
   > 4. `ASTFunDeclaration`:个人认为这个结点是比较复杂的一个结点，他的最终任务是创建一个函数，并放入符号表中。为此，在这个结点中，我们首先需要向下取`params-list`,即函数形参列表，这个列表和函数返回值一起，决定了函数的类型。之后我们创建函数；最为重要的是在这个结点中，我们还要为函数的形参分配本函数内空间，这个也需要对形参的种类做具体讨论。
   > 5. `ASTCompoundStmt`:这个结点是对真正执行代码的分析的开始与结束。每次进入这个结点这里我们需要向符号表堆栈中插入一个新的符号表，这个起到作用域转移的作用。在进入新的作用域之后，我们就继续对本地声明语句和基本语句块进行遍历分析。
   > 6. `ASTExpressionStmt`:这个结点作为一个中继结点，其子结点是一个个表达式，从这里下去，依次遍历。
   > 7. `ASTSelectionStmt`:这个结点用于处理选择语句（即`if-else`语句），在此处我们将生成不同的基本块与对应标签，并加入条件跳转和无条件跳转指令。为了避免嵌套带来的跳转标签冲突，在这里和下面的循环语句中，我们自己指定了标签的内容。
   > 8. `ASTIterationStmt`:这个结点用于处理循环语句（即`while`语句），与上面大的选择语句类似，我们同样需要生成不同的基本块和对应的标签，并加入跳转指令。同样我们在这里自己指定了标签的内容。
   > 9. `ASTReturnStmt`：这个结点用于处理函数的返回语句，根据需要带回返回值。
   > 10. `ASTVar`:这个结点绝对是这次实验中最为复杂的一个结点了，它需要响应上层对变量的调用，在这个结点中查找合法作用域内的符号表，并且返回**正确值**。最困难的，最容易出bug的点也在这里，因为不同情况下我们对同一个变量的引用会需要它不同的返回值，特别是数组变量`a[10]`，不同位置的引用会需要带回完全不同的类型回去。
   > 11. `ASTAssignExpression`:这个结点用于处理赋值语句，由于下层已经完成了取值，这一结点内需要将右值赋值给左值，但更重要的是还需要处理类型转化，以保证进行赋值操作时右侧值的类型与左值的类型一致。
   > 12. `ASTSimpleExpression`:这个结点用于处理关系表达式语句。需要注意的是，在这里我们根据`cmp`指令得到的返回值是`Int1`类型，为了方便求关系表达式的表达式值，我们在此处将所有的返回值手动转换为`Int32`类型，当在选择和循环语句中需要`Int1`l类型的值时，我们再转回`Int1`类型。用这种方法来实现`Int32`类型与`Int1`类型通用转化使用。
   > 13. `ASTAdditiveExpression`:这个结点用于处理可迭代使用的加减计算表达式语句。在这里我们添加了类型的检查与转化，确保每次进行加减法运算时左操作数和右操作数类型一致。
   > 14. `ASTTerm`:这个结点用于处理可迭代使用的乘除计算表达式。这个结点是语法分析的最底层，它的下面的结点是由词法分析器生成的。在这里我们同样添加了类型的检查与转化，以保证进行乘除运算时左操作数和右操作数类型一致。
   > 15. `ASTCall`:这个结点用于处理函数调用语句。主要的功能是根据调用函数的参数列表，先进行实参的类型检查与转化，将所有的实参取值压入函数中，之后进行函数调用与返回值的处理。
   >
   > 

## 实验总结

本次实验通过使用 `LightIR` 框架自动产生 `cminus-f` 语言的`LLVM IR`，需要根据`cminus-f`的语义规则编写`cminusf_builder.cpp`中的16个函数。16个函数较为独立但是要做好函数间的接口，需要一些参数的传递来上层函数得到的信息。下层的信息被封装在`Value*`类型中，故在每个变量处理时要清楚`Value*`中包含的信息。总的来说，需要对`cminus-f`的语义规则全局的把握，以及对遍历语法树过程各种信息传递的把握。

## 实验反馈 

本次实验有一定难度，但实验设计思路清晰，又有前两个lab作为铺垫，有挑战性，可以大大提升对编译的理解。通过对接口的调用、对封装的类型等的处理，也更加理解C++面向对象的特性。

## 组间交流 

**无**
