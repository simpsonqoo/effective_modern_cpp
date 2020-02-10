&emsp;&emsp;auto使用時和樣板推導幾乎是一樣的，雖然乍似是不太一樣的動作，但其實是幾乎一樣的。

e.g.
```cpp
auto a = 10;
```

&emsp;&emsp;實際上auto及對應template中的T，會根據a之值(10)去推導其型別(int)。
&emsp;&emsp;若含有型別說明符時(e.g. const, &, ...)，則推導和樣板推導是一樣的。

e.g.
```cpp
auto a = 10;        // 推導a之型別為int
auto& b = a;        // 推導b之型別為int&
const auto c = a;   // 推導c之型別為const int
const auto& d = a;  // 推導d之型別為const int&
```

而一樣的也會有三種狀況：1. 參考或是指標，非universal reference、2. universal reference、3. 非參考或是指標

e.g.
```cpp
auto a = 10;        // 狀況三
const auto& b = a;  // 狀況一
const auto&& c = a; // 狀況二，c為lvalue參考
auto&& d = 20;      // 狀況二，c為int&&
```

而特殊狀況(陣列、函式)也是一樣的
e.g.
```cpp
int a[2] = {1, 2};

auto b = a;     // a之型別為int*
auto& c = a;    // a之型別為int (&)[13]

void some_func();

auto d = some_func;     // d之型別為void (*)()
auto& e = some_func;    // e之型別為void (&)()
```

唯一和樣板推導不同的地方在於大括號的初始化。C++11後增加了大括號的初始化(list initilaizer)，但是型別是std::initializer_list<T>，並不是小括號的T

note1: 在C++17後，若指定型別，則使用大括號初始化，無論使用copy constuctor或是direct constructor其型別都是T，非std::initializer_list<T>
[https://en.cppreference.com/w/cpp/language/list_initialization](https://en.cppreference.com/w/cpp/language/list_initialization)

e.g.
```cpp
int a = 10;     // a型別為int
int a(10);      // a型別為int
int a{10};      // a型別為int
int a = {10};   // a型別為int
```

在C++17後，若使用auto推導型別，若使用大括號初始化，使用copy constructor，型別為std::initializer<T>，而direct constructor，會將大括號初始化退化至一般直接初始化(小括號初始化)。

e.g.
```cpp
auto x1 = {3};      // x1 is std::initializer_list<int>
auto x2{1, 2};      // error: not a single element
auto x3{3};         // x3 is int
auto x4 = {1, 2}    // x4 is std::initializer_list<int>
```
[https://en.cppreference.com/w/cpp/language/template_argument_deduction#Other_contexts](https://en.cppreference.com/w/cpp/language/template_argument_deduction#Other_contexts)

故若使用copy constructor做大括號初始化，會試著對其內容推導型別，若型別不同，則會推導失敗

e.g.
```cpp
auto a = {1, 2, 3.5}; // error，1及3.5推導出的型別不同，1及2為int，3.5為double
```

auto中的copy constructor會域設為std::initializer_list<T>，但是樣板推導並不會，這是auto和樣板推導唯一不同之處

e.g.
```cpp
template <typename T>
void f(T param)

f({1, 2});  // error，無法推導{1, 2}
```

但如果直接給定ParaType為std::initilizer_list<T>，則樣板型別就能推導出T

e.g.
```cpp
template <typename T>
void f(std::initilizer<T> param);

f({1, 2});  // pass
```

做個小總結，除了auto的copy constructor外，auto的其餘規則和樣板推導並沒有差別。

最後，在C++14中的函式回傳形式允許使用auto，但是其推導規則是使用樣板推導的規則

e.g.
```cpp
auto f()
{
    return {1, 2};  // error
}
```
