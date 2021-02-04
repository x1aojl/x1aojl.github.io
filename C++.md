### 访问控制
#### class / struct 里成员的默认访问控制
定义在第一个访问说明符之前的成员：class 中默认为 private，struct 中默认为 public。
| <sub>类型</sub> \ <sup>成员访问说明符</sup> | public | protected | private |
| :---- | :----: | :----: | :----: |
| 类的实例 | yes  | no | no |
| 类的友元 | yes | yes | yes |
| 类的派生类 | yes | yes | no |

#### class / struct 继承链里的默认访问控制
如果不写继承方式：class 默认以 private 的方式继承，struct 默认以 public 的方式继承。
| <sub>继承方式</sub> \ <sup>基类成员</sup> | public | protected | private |
| :---- | :---- | :---- | :---- |
| public | public  | protected | 不可访问 |
| protected | protected | protected | 不可访问 |
| private | private | private | 不可访问 |

### [static](https://www.zhihu.com/question/274217344/answer/379696251) 关键字
#### static 修饰内部链接的全局函数 / 变量
表示该函数 / 变量只在所在的编译单元内可见。与之相关的是外部链接声明，使用 extern 修饰。
``` 
static int a = 0; // 内部链接的变量
static void function() { } // 内部链接的函数
```

#### static 修饰局部变量
静态局部变量在程序第一次运行到该声明时做初始化，作用域结束后变量不被销毁，下一次运行时仍保持状态。普通的局部变量称为自动变量。
```
int function()
{
    static int num = 0;
    num++;
    return num;
}
```

#### static 修饰成员函数 / 变量
表示函数 / 变量不绑定至实例。静态变量除了声明外还需要在类之外显示定义。
```
class C
{
    static void function(); // 静态成员函数声明
    static int num; // 静态成员变量声明
};

void C::function() { } // 静态成员函数定义
int C::num = 0; // 静态成员变量定义
```

### [extern](https://docs.microsoft.com/en-us/cpp/cpp/extern-cpp?view=msvc-160) 关键字
#### extern 在非 const 全局函数 / 变量声明中
指定在其他编译单元中定义函数 / 变量。
```
//fileA.cpp
int i = 42; // declaration and definition

//fileB.cpp
extern int i;  // declaration only. same as i in FileA

//fileC.cpp
extern int i;  // declaration only. same as i in FileA

//fileD.cpp
int i = 43; // LNK2005! 'i' already has a definition.
extern int i = 43; // same error (extern is ignored on definitions)
```

#### extern 在 const 变量声明中
指定变量具有外部链接（默认情况下全局常量具有内部链接）。
```
//fileA.cpp
extern const int i = 42; // extern const definition

//fileB.cpp
extern const int i;  // declaration only. same as i in FileA
```

#### extern "C" 函数声明
指定该函数在其他位置定义，并使用C语言调用约定。
```
// Declare printf with C linkage.
extern "C" int printf(const char *fmt, ...);

//  Cause everything in the specified
//  header files to have C linkage.
extern "C" {
    // add your #include statements here
#include <stdio.h>
}

//  Declare the two functions ShowChar
//  and GetChar with C linkage.
extern "C" {
    char ShowChar(char ch);
    char GetChar(void);
}

//  Define the two functions
//  ShowChar and GetChar with C linkage.
extern "C" char ShowChar(char ch) {
    putchar(ch);
    return ch;
}

extern "C" char GetChar(void) {
    char ch;
    ch = getchar();
    return ch;
}

// Declare a global variable, errno, with C linkage.
extern "C" int errno;
```

### inline（内联）函数
| | |
| :---- | :---- |
| 函数在 class body 内定义完成 | 自动成为 inline 候选人。 |
| 函数在 class body 外定义完成 | 需要在函数返回类型前面显示地加上 inline。 |

### virtual function（虚函数）
| non-virtual 函数 | virtual 函数 | pure virtual 函数 |
| :-----| :---- | :---- |
| 你不希望派生类重新定义（override, 重写）它。 | 你希望派生类重新定义（override, 重写）它，且你对它已有默认定义。 | 你希望派生类一定要重新定义（override, 重写）它，你对它没有默认定义。在声明语句的分号之前书写=0 |
| `void function();` | `virtual void function();` | `virtual void function() = 0;` |

### overload override overwrite
| overload 重载 | override 覆盖 | overwrite 重写 |
| :---- | :---- | :---- |
| 同一个类中, 函数名字相同但形参列表不同 | 派生类重写了基类的同名同参 virtual 函数 | 派生类重写了基类同名非 virtual 函数 |

### override final
| | 含义 | 作用域 |
| :---- | :---- | :---- |
| override | 表示派生类应当覆盖基类中的这个虚函数 | 用于派生类的虚函数中 |
| final | 阻止类继承或阻止派生类重写这个虚函数 | 用于类或基类的虚函数中 |

### new expression
```T* p = new T();```</br>
编译器转为：
```
// 分配内存 allocate
void* memory = operator new(sizeof(T)); // -> ::operator new(size_t); -> malloc(size_t);
// 转换类型 cast
p = static_cast<T*>(memory);
// 调用构造函数 invoke ctor
p->T::T();
```

### delete expression
```delete p;```</br>
编译器转为：
```
// 调用析构函数 invoke dtor
p->~T();
// 释放内存 deallocate
operator delete(p); // -> ::operator delete(void*); -> free(void*);
```
