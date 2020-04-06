通常若要將型別T宣告為rvalue reference，會宣告為T&&，但是在許多場合T&&並不代表rvalue reference

e.g.
```cpp
void f(Widget&& param);    // param為rvalue reference

Widget&& w1 = Widget();    // w1為rvalue reference

auto&& w2 = w1;    // w2不為rvalue reference

template <typename T>
void f(std::vector<T>&& param);    // param為rvalue reference

template <typename T>
void f(T&& param);    // param不為rvalue reference
```

表示T&&時可能會有兩種意思
1. rvalue reference
2. lvalue reference或是rvalue reference，也可以同時與const或是volatile連結

第二種表示法目前稱為forwarding reference。

而有兩種方式可以使用forwarding reference
1. function template的參數<br>
e.g.
```cpp
template <typename T>
void f(T&& param);
```
2. auto<br>
e.g.
```cpp
auto&& x = y;
```

# function template的參數
由於forwarding reference代表可以和lvalue reference連結，也能和rvalue reference連結，所以應該是在用到型別推導的時候才會需要使用，如果&&出現的地方並不需要做型別推導，則&&必定表示為rvalue reference

e.g.
```cpp
void f(Widget&& w);    // w為rvalue referencce
Widget&& w1 = w2;    // w1為rvalue reference
```

而由於forwarding reference是參考型別，所以必須要為其做初始化，初始化的物件就決定forwarding reference最終的型別是lvalue reference或是rvalue reference。

e.g.
```cpp
tempalte <typename T>
void f(T&& param);

Widget w;
f(w);    // param為lvalue reference
f(std::move(w));    // param為rvalue reference
```

不過即使需要推導，也不一定&&就是forwarding reference，使用時必須為T&&時才會是forwarding reference。

e.g.
```cpp
template <typename T>
void f1(std::vector<T>&& param);    // param為rvalue reference

template <typename T>
void f2(const T&& param);    // param為rvalue reference
```

## forwarding reference的適用時機看std::vector中push_back及emplace_back的差異
而型別推導若是存在於class中，只要宣告為T&&的物件並不需要做推導，則T&&即為rvalue reference，這裡以std::vector中的push_back為例

e.g.
```cpp
template <typename T>
class vector
{
  public:
    void push_back(T&&);

};
```

由於使用std::vector時，須給定其template argument<br>
e.g.
```cpp
std::vector<Widget> v;
```

所以push_back中的T&&並不需要做推導，其已經實體化為<br>
```cpp
void push_back(Widget&&);
```

所以push_back的參數型別為rvalue reference。


而相對的，使用emplace_back就需要做型別推導<br>
e.g.
```cpp
template <typename T, class Allocator = allocator<T>>
class vector
{
  public:
    template <class... Args>
    void emplace_back(Args&&... args);
};
```

此處emplace_back就需要做引數型別的推導，故args型別為forwarding reference的參數包。


# auto
由於auto也會進行型別推導，所以也是可以使用forwarding reference的時機之一。常用的地方在lambda expression中的參數欄<br>
e.g.
```cpp
auto f = [](auto&& v){...};
```