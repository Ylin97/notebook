# CPP基础

1. 如果想在多个文件中共享`const`对象，必须在变量的定义前加上`extern`关键字。这与共享普通变量不同，不同变量只需在声明的地方加`extern`就行，而`const`对象必须在声明和定义时都加，这是因为普通变量默认是外部链接，而`const`对象默认是内部链接，只在文件内有效。

2. 修饰变量本身的`const`是**顶层`const`**，修饰变量指向或者绑定对象的`const`是**底层`const`**，后者只用于指针和引用。当执行对象的拷贝操作时，顶层`const`可以被忽略，但是底层的`const`则不能被忽略：

   ```cpp
   int i = 2;
   const int ci = 3;    // 顶层const
   int *const p1 = &i;  // 顶层const
   const int *p2 = &i;  // 底层const
   
   int a = ci;  // 正确，ci顶层const被忽略，其值被拷贝一份给了a，修改a不会改变ci的值
   int b = *p1; // 正确，顶层const被忽略
   int c = p2;  // 错误，底层const不能被忽略
   int &r = ci; // 错误，普通int&不能绑定到int常量上
   const int &r2 = i; // 正确，const int&可以绑定到一个普通int上
   ```

3. `constexpr`用于声明一个常量表达式。常量表达式是指能值不会改变并且在编译期就能得到计算结果的表达式，字面值和用常量表达式初始化的`const`对象都是常量表达式。一般来说，如果认定变量是一个常量表达式，那就把它声明成`constexpr`类型，编译器会在编译时对它进行检查。

   一个`constexpr`指针的初始值必须是`nullptr`或者`0`，或者是存储于某个固定地址中的对象（这里的“固定”是指在整个程序运行过程中对象的地址是固定不变的）。必须注意，**在`constexpr`声明中如果定义了一个指针，限定符`constexpr`仅对指针有效，与指针所指向的对象无关**：

   ```cpp
   const int *p = nullptr;     // p是一个指向整型常量的指针
   constexpr int *q = nullptr; // q是一个指向整型的常量指针
   ```

   换句话说，`constexpr`把它所定义的对象置为了顶层`const`。

4. 由于引用是已有变量的别名，真正参与初始化的其实是它所绑定对象的值，而`auto`是根据初始值来推断变量类型，所以当`auto`推断的类型是引用时，编译器会以引用对象的类型作为`auto`的类型。其次，`auto`通常会忽略掉顶层`const`，而只保留底层`const`，但是当设置一个类型为`auto`的引用时，初始值中的顶层`const`属性依然会被保留，只是此时的`const`不再是顶层`const`：

   ```cpp
   const int ci = 7;
   const int *const p = &ci;
   
   auto b = ci;       // b类型为int
   const auto d = ci; // d类型为const int
   auto q = p;        // q类型为const int*
   const auto q2 = p; // q2类型为const int *const
   auto &r = ci;      // r类型为const int，但此时的const是指r所绑定的对象是常量，即转为了底层 const
   auto &r2 = p;      // r2的类型是const int *const, r2绑定的对象是const int *const，即转为了底层const
   ```


5. `decltype`推断类型时，处理顶层`const`的方式与`auto`不同，即如果它推断的对象是一个变量时，`decltype`会返回该变量的完整类型（包括顶层`const`和引用）。也就是说`decltype`推断的是变量本身，而`auto`推断的是变量的值。

6. 当`decltype`使用的表达式不是一个变量，则它返回表达式结果对应的类型。特别地，当表达式的结果能作为赋值语句的左值时，`decltype`会返回一个引用，例如对指针解引用。

   ```cpp
   int i = 42, *p = &i, &r = i;
   decltype(r + 0) b; // 正确，加法的结果是int，因此b是一个（未初始化的）int
   decltype(*p) c;    // 错误，c是int&，必须初始化
   ```

7. 当`decltype`或`auto&`推断的对象是数组时，数组将不会被转换为指针类型。所以，**多维数组使用范围 for 进行遍历时，除了最内层的循环外，其他所有循环的控制变量都应该设为引用类型。**

   ```cpp
   int ia[] = {1, 2, 3};
   int *p = &ia[0];
   
   decltype(ia) ia2 = {4, 5, 6, 7}; 
   ia2 = p; // 错误，不能用整型指针给数组赋值
   ```

   

8. `decltype((variable))`（注意是双层括号）的结果永远是引用，而`decltype(variable)`结果只有当`variable`本身就是一个引用时才是引用。

9. 头文件中不应使用`using`声明，源文件中可使用。

10. 当使用`{}`进行初始化时，编译器会先尝试进行列表初始化，如果`{}`里面的值不能用来列表初始化，则编译器会尝试用它们来构造该类型的对象。

   ```cpp
   vector<string> v1{"hi"};     // 列表初始化：v5有一个元素
   vector<string> v2("hi");     // 错误，不能使用字符串字面值构建vector对象
   vector<string> v3{10};       // v7有10个默认初始化的元素
   vector<string> v4{10, "hi"}; // v8有10个值为"hi"的元素
   ```

11. 对于`vector`，通常定义一个空对象，然后在运行时根据需要动态添加元素比预先分配空间效率更高。只有一个例外，即当其中的元素值都是一样时，预先分配空间效率才高。

12. 标准库类型的下标必须是无符号类型，但是内置的下标运算符无此要求，所以数组下标可以是负数。

    ```cpp
    int ia[] = {1, 2, 3};
    int *p = &ia[2];
    int k = p[-2]; // p[-2]是ia[0]表示的那个元素，实际上它会被转换为 int k = *(p - 2);
    ```

13. 数组名实际上是指向数组第一个元素的指针。对于一维数组，其数组名指向第一个元素，而对于多维数组则指向第一个内层数组。

    ```cpp
    int ia[2][3] = {1, 2, 3, 4, 5, 6};
    
    auto p = ia;  // p指向包含3个元素的数组
    auto q = *p;  // q指向p所指向数组的首元素
    
    /* 
      实际上指针p、q存的地址值是一样的都是ia最内层数组的首元素的地址，但是p和q的性质完全不同，p的类型是int (*)[3]，而q的类型是int*，p+1指向的是ia内层第二个包含3个整型元素的数组，即[4, 5, 6]，而q+1则是指向ia内层第一个包含3个整型元素的数组的第二个元素，即2。
    */
    ```


14. 在C语言中左值是指能够位于赋值语句左侧的对象（如变量），右值则相反。而在C++中左右值区分更为复杂，归纳而言：**当一个对象被用作右值时，用的是对象的值（内容）；当对象被用作左值时，用的是对象的身份（在内存中的位置）。** 对于不同运算符，它们都有一个共同原则：在需要右值的地方可以用左值代替，但不能把右值当成左值（也就是位置）使用。当一个左值被当成右值使用时，实际用的是它的内容（值）。

    使用`decltype`推断类型时，如果它作用的表达式的求值结果是左值，那它将得到一个引用类型。例如`int *p`对p解引用的结果是左值，所以`decltype(*p)`的结果是`int&`。而取地址符`&`的结果是右值，所以`decltype(&p)`的结果是`int**`。

15. 在书写复合表达式时，有以下两条经验：
    - 拿不准的时候最好用括号明确求值的优先级；
    - 如果改变了某个运算对象的值，在表达式的其他地方不要再使用这个运算对象。

16. 由于标准对`位运算`时符号位如何处理没有明确规定，所以位运算符最好只用于处理无符号数。

    <center><b>C语言常用运算符优先级与结合性</b></center>
    
    | 符号 1                                                  | 操作类型         | 结合性   |
    | :------------------------------------------------------ | :--------------- | :------- |
    | `[]` `()` `.` `->` `++` `--`（后缀）                    | 表达式           | 从左到右 |
    | **`sizeof`** `&` `*` `+` `-` `~` `!` `++` `--`（前缀）  | 一元             | 从右到左 |
    | *typecasts*                                             | 一元             | 从右到左 |
    | `*` `/` `%`                                             | 乘法             | 从左到右 |
    | `+` `-`                                                 | 加法             | 从左到右 |
    | `<<` `>>`                                               | 按位移动         | 从左到右 |
    | `<` `>` `<=` `>=`                                       | 关系             | 从左到右 |
    | `==` `!=`                                               | 相等             | 从左到右 |
    | `&`                                                     | 按位“与”         | 从左到右 |
    | `^`                                                     | 按位“异或”       | 从左到右 |
    | `|`                                                     | 按位“与或”       | 从左到右 |
    | `&&`                                                    | 逻辑“与”         | 从左到右 |
    | `||`                                                    | 逻辑“或”         | 从左到右 |
    | `? :`                                                   | 条件表达式       | 从右到左 |
    | `=` `*=` `/=` `%=` `+=` `-=` `<<=` `>>=` `&=` `^=` `|=` | 简单和复合赋值 2 | 从右到左 |
    | `,`                                                     | 顺序计算         | 从左到右 |

    > 完整C++内置运算符、优先级和关联性[见此](./C++内置运算符的优先级与关联性.md)。

17. 对`string`或`vector`等标准库类型的对象执行`sizeof`运算只返回该类型固定部分的大小，不会计算对象中的元素占用了多少空间。

18. C++新式的强制类型转换包含四种（[详情](./C++强制类型转换运算符小结.md)）：

    - `dynamic_cast`：用于多态类的向下转换，安全但有运行时开销。
    
    - `static_cast`：编译时确定的类型（即非多态）转换，高效但没有运行时检查，适合类层次的向上转换和基本类型之间的转换。
    - `const_cast`：用于去除 `const` 、 `volatile`和`__unsigned` 属性，需确保对象实际不是常量。
    
    - `reinterpret_cast`：用于任意类型之间的位模式转换，灵活但极其危险。
    

19. 使用`空语句`时应该加上注释，好让别人知道该语句是有意省略的。

20. 由于`else`总是与离它最近且尚未匹配的`if`匹配（悬垂 else），为了避免不必要的`else`匹配错误，嵌套`if - else`语句不应省略`{}`：

    ```cpp
    if (a > 0)
        if (a > 3)
            cout << "good" << endl;
    else // 与if (a > 3)匹配
        cout << "bad" << endl;  
    
    /*
      输入：a = 2
      输出：bad
    */
    ```


21. 当`数组`作为函数参数进行传递时，真正传递的实际是指向数组首元素的指针，所以对于多维数组，除最外层数组维度可省略外，所有内层数组的维度都不能省略。

    ```cpp
    void func(int arr[2][10]);  // 正确
    void func(int arr[][10]);   // 正确
    void func(int (*arr)[10]);  // 正确
    
    void func(int **arr);       // 语法正确，但将导致func函数内部不能通过下标访问数组元素
    void func(int arr[][]);     // 错误
    void func(int arr[2][]);    // 错误
    ```

22. C++编写可变参数函数的方法主要有三种（[详情](./C++可变参数函数小结.md)）：

    - `initializer_list`：如果参数类型固定且一致，`std::initializer_list` 是一种简单而高效的方法。（**推荐**）
    - `可变参数模板`：如果需要处理类型不同的多个参数，C++11 的可变参数模板提供了更好的灵活性和类型安全。
    - `C风格可变参数`：C风格的可变参数函数适用于与C代码的兼容，但现代C++中不推荐使用。

23. 调用一个返回引用的函数得到左值，其他返回类型得到右值。

    ```cpp
    char &get_val(string &str, string::size_type ix) {
        return str[ix];    // get_val假定索引值有效
    }
    int main() {
        string s("a value");
        get_val(s, 0) = 'A';
        cout << s << endl;  // 输出：A value
    }
    ```

24. 函数可以使用`{}`返回一个值列表，这实际上是使用列表初始化语法构建一个返回类型对应的临时对象。

    ```cpp
    vector<string> process() {
        // ...
        // expand和actual是string对象
        if (expand.empty())
            return {};                    // 返回一个空vector对象
        else if (expand == actual)
            return {"functionX", "okay"}; // 返回列表初始化的vector对象
        else
            return {"functionX", expected, actual};
    }
    ```

25. 对于返回值类型复杂的函数，C++可以通过`尾置返回类型`来简化声明：

    ```cpp
    // 定义一个返回值是指向数组的指针的函数
    int (*func(int i))[10]; 
    // 可以使用“尾置返回类型”简化：
    auto func(int i) -> int(*)[10];
    ```


26. 声明含`默认实参`的函数时需注意，在给定作用域中一个形参只能被赋予一次默认实参，也就是说，函数后续的声明只能为之前那些没有默认值的形参添加默认实参，而且该形参右侧的所有形参必须都有默认值。

27. 不仅仅是`常量表达式`可以作为默认实参的初始值，其他表达式只要它的类型能够转换成形参所需的类型，该表达式就能作为默认实参。这是因为用作默认实参的名字在函数声明所在的作用域内解析（函数外部），而**这些名字的求值过程发生在函数调用时**。

    ```cpp
    sz wd = 80;
    char def = ' ';
    sz ht();
    string screen(sz = ht(), sz = wd, char = der);
    string window = screen();  // 调用screen(ht(), 80, ' ')
    
    void f2() {
        def = '*';
        sz wd = 100;
        window = screen();  // 调用screen(ht(), 80, '*');
    }
    ```

28. `constexpr函数`是指能用于常量表达式的函数，其返回值类型及所有形参类型都是字面值类型，而且函数体中有且仅有一条`return`语句。但是要注意，`constexpr函数`不一定返回常量表达式，此时如果将`constexpr函数`用在需要常量表达式的上下文中时，由编译器负责检查函数的结果是否符合要求，不符合时编译器将发出警告。

    和其他函数不一样，为了能够让编译器随时展开，`内联函数`和`constexpr函数`可以在程序中多次定义。所以，**通常将`内联函数`和`constexpr函数`定义在头文件中**。

29. `assert`宏通常用于**程序调试**时验证那些确实不可能发生的事情，当程序中没有定义`NDEBUG`宏时，`assert`宏会对其作用的表达式进行检查，如果表达式的值为`假`时，`assert`输出信息并终止程序；而如果表达式的值为`真`，则`assert`什么也不做。**关闭`assert`宏功能只需定义`NDEBUG`宏。**此外，C/C++预处理器还定义了4个用于辅助调试的宏和一个`__func__`静态变量：
    - `__FILE__`：存放文件名的字符串字面值；
    - `__LINE__`：存放当前行号的整型字面值；
    - `__TIME__`：存放文件编译时间的字符串字面值；
    - `__DATE__`：存放文件编译日期的字符串字面值；
    - `__func__`：静态数组，存放当前函数的名字。
30. 定义在类内部的函数是隐式声明的`inline`函数。
31. 启用默认构造函数的`= default`修饰符既可以和声明一起出现在类的内部，也可以作为定义出现在类的外部。和其他函数一样，如果`= default`在类内部则该默认构造函数是内联的。

32. `友元`可以声明在类内的任何地方，因为它不是类的成员，不受它所在区域的访问控制级别的约束，但是为了方便阅读，**通常将`友元`集中声明在类的开始或结束前的位置。**需要注意的是，类内的`友元`声明仅仅是声明了访问权限，所以为了使用户能够调用友元函数，必须在类外部再声明一次（此时不再有`friend`了）。通常将`友元`的声明和类本身放在同一个头文件中（类的外部）。

    ```cpp
    struct X {
        friend void f() {
            /* 友元函数可以定义在类的内部 */
        }
        X() { f(); } // 错误：f还没有被声明
        void g();
        void h();
    };
    void X::g() { return f(); } // 错误：f还没有被声明
    void f();                   // 声明那个定义在X中的函数
    void X::h() { return f(); } // 正确：现在f的声明在作用域中了
    ```

    > 虽然许多编译器并未强制限定友元函数必须在使用之前再类的外部声明，但是为了兼容更多的编译器，最好还是再加一个友元函数的外部声明。

33. 由类定义的类型名字和其他成员一样存在访问限制，可以是`public`或`private`。

    ```cpp
    class Screen {
    public:
        typedef std::string::size_type pos;  // pos别名具有public权限
    private:
        pos cursor = 0;
        pos height = 0, width = 0;
        std::string contents;
    };
    ```

34. 声明内联成员函数时，`inline`关键字既可以写在类里面成员函数声明的地方，也可以写在类外部成员函数定义的地方，或者两个地方都写，但是为了方便阅读，最好是将`inline`关键字写在类外部成员函数定义的地方。`inline成员函数`和普通`inline函数`一样，我们同样也应该把它们（与相应的类一起）放在头文件中。

35. `mutable`成员永远不会是`const`，即使它是`const`对象的成员。因此，一个`const成员函数`可以改变一个`mutable`成员的值。

36. 当为成员变量提供类内初始值时，必须以`=`或者`{}`表示。
37. `前向声明`是一种不完全类型，它只能在以下两种情况下使用：
    - 定义指向类型的指针或者引用；
    - 声明（但是不能定义）以这种类型作为参数或者返回类型的函数。

38. 除了能够定义普通友元函数外，还可以定义`友元类`和`友元成员函数`，但是在定义`友元成员函数`时要注意各个部分的声明顺序。

    ```cpp
    class ClassB;  // 前向声明（因为 ClassA 需要在 ClassB 之前知道 ClassB 的存在）
    
    class ClassA {
    public:
        void showClassBPrivateData(const ClassB& b);  // ClassA 的成员函数
    };
    
    class ClassB {
    private:
        int privateData = 42;
    
        // 声明 ClassA 的成员函数为友元
        friend void ClassA::showClassBPrivateData(const ClassB& b);
    };
    
    void ClassA::showClassBPrivateData(const ClassB& b) {
        // 由于是友元函数，可以访问 ClassB 的私有成员
        std::cout << "ClassB's private data: " << b.privateData << std::endl;
    }
    ```

    - 首先定义`ClassA`类，其中声明`showClassBPrivateData`函数，但是不能定义它。在`showClassBPrivateData`使用`ClassB`的成员之前必须先声明`ClassB`；
    - 接下来定义`ClassB`，包括对于`showClassBPrivateData`的友元声明；
    - 最后定义`Clear`，此时它才可以使用`Screen`的成员。

39. 编译器在处理类的定义时，分为两步：

    - 首先，编译成员的声明；
    - 直到类全部可见后才编译函数体。

    正是因为成员函数体直到整个类可见后才会被处理，所以它能使用类中定义的任何名字。

40. **类型名通常定义在类的开始处**，这样就能确保所有使用该类型的成员都出现在类型名的定义之后。

41. `委托构造函数`：指使用它所属类的其他构造函数执行它自己的初始化过程，或者说它把自己的一些（或全部）职责委托给了其他构造函数。在委托构造函数内，成员初始值列表只有一个唯一的入口，就是类名本身。

    ```cpp
    class Person {
    public:
        std::string name;
        int age;
    
        // 构造函数1：带有name和age参数的完全初始化
        Person(const std::string& name, int age) : name(name), age(age) {
            std::cout << "Person(const std::string&, int) called" << std::endl;
        }
    
        // 构造函数2：仅带有name参数，age默认为0
        Person(const std::string& name) : Person(name, 0) {  // 委托给第一个构造函数
            std::cout << "Person(const std::string&) called" << std::endl;
        }
    
        // 构造函数3：无参数，name默认为"Unknown"，age默认为0
        Person() : Person("Unknown") {  // 委托给第二个构造函数
            std::cout << "Person() called" << std::endl;
        }
    };
    ```

42. 在实际编程中，如果定义了其他构造函数，那么最好也提供一个默认构造函数。

43. `转换构造函数`指的就是只接受一个实参的构造函数，它定义了从构造函数参数类型向类类型**隐式转换**的规则。需要注意的是，只准许一步类类型转换：

    ```cpp
    class Book {
    public:
        Book(std::string x) {
            std::cout << "A constructor called with " << x << std::endl;
        }
    };
    
    void func(Book bk) {
        std::cout << "func(Book) called" << std::endl;
    }
    
    int main() {
        string s = "CODE";
        func(s);  // 正确：从string到Book只需一步隐式转换
        
        // func("CODE");  // 错误：这里需要两步转换
        // 1. 从const char* 到string
        // 2. 从string到Book
    }
    ```

44. 当需要阻止隐式转换时，可以在`转换构造函数`前面加上`explicit`关键字。需要注意的是，`explicit`关键字只对一个实参的构造函数有效。另外，当`explicit`关键字声明构造函数时，它将**只能以直接初始化的形式使用**，而且编译器将不会再自动转换过程中使用该构造函数（但是可以显式地强制转换）。

    ```cpp
    class Book {
    public:
        explicit Book(std::string x) {
            std::cout << "A constructor called with " << x << std::endl;
        }
    };
    
    void func(Book bk) {
        std::cout << "func(Book) called" << std::endl;
    }
    
    int main() {
        // func(string("CODE"));  // 错误：explicit修饰的构造函数不能用于隐式转换
        func(static_cast<Book>(string("CODE"))); // 正确：显式强制类型转换
    }
    ```

45. `字面值常量类`允许定义更复杂的数据结构，并将其用于 `constexpr` 表达式中，这使得它们非常适合用于编译期的计算或常量初始化。这在诸如元编程、编译期优化等场景下非常有用。

46. 类的静态成员（[详情](./C++类的静态成员知识小结.md)）：

    - 静态成员（变量和函数）是类级别的，不属于具体的对象。
    - 静态成员变量需要在类外定义，并且可以通过类名访问，无需实例化对象;
    - 静态成员函数只能访问静态成员，不能访问非静态成员。
    - 静态成员通常用于存储类的全局数据或者提供与对象无关的功能。

47.  定义类成员时，当类名出现后，类名之后的内容都属于类的作用域，故可省略类名。

    ```cpp
    class Book {
        typedef std::string::size_type pos;
    public:
        Book(pos _n = 0);
        static double initCnt();
    private:
        pos n;
        string title;
        static double cnt;
    };
    
    double Book::cnt = initCnt();  // initCnt省略了"Book::"
    
    Book::Book(pos _n) : n(_n) {  // pos省略了"Book::"
        cout << "A constructor called with " << _n << std::endl;
    }
    ```

    
