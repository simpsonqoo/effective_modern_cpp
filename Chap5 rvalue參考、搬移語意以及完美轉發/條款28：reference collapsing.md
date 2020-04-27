條款23中介紹std::forward時有提到，std::forward主要適用在將函式的參數轉發給另一個函式時使用。由於函數的參數即使被宣告為rvalue，在函式中的陳述式時都還是會轉換為lvalue

e.g.
```cpp
template <typename T>
void f(T&& param)
{
    if constexpr(std::is_lvalue_reference_v<decltype((param))>)
        std::cout << "param is lvalue" << std::endl;
    
    if constexpr(std::is_rvalue_reference_v<decltype((param))>)
        std::cout << "param is rvalue" << std::endl;
}

int main()
{
    int a = 1;
    f(a);
    f(1);
}
```

output
```
param is lvalue
param is lvalue
```

所以若是想轉發參數，就必須使用std::forward，將rvalue完美轉發至另一個函式。而std::forward會根據推導的T型別決定是否要轉換為rvalue，而這種根據rvalue或是lvalue會有不同的處理的狀況，在條款23時並沒有詳細的說明適用的時機，是因為還沒提到std::forward，本章會詳細說明其流程。

事實上，根據lvalue或是rvalue而有不同處理方法的地方就只有在forwarding reference的情況而已，以下簡單舉個forwading reference的例子

e.g.
```cpp
template <typename T>
void foo(T&& param);
```

而T的推導也很簡單(注意，是T而不是T&&)，只要引數為lvalue，則T會推導為lvalue reference，但是若引數為rvalue，則T會推導為rvalue(注意，沒有reference)。

e.g.
```cpp
Widget w;
foo(w);    // T為Widget&(lvalue reference)

foo(Widget());    // T為Widget(rvalue)
```

而要繼續之前，先說明一下，一般而言參考的參考對於compiler來說是不合法的。
e.g.
```
int x;
int& & rx = x; 
```

雖然forwarding reference也會出現這樣的情形，但是compiler會對其做特別處理
e.g.
```cpp
templaet <typename T>
void foo(T&& w);

Widget w;
foo(w);
```

其中foo(w)時void foo(T&& w)會實體化變成void foo(Widget& && w)。

而compiler會對某些發生參考的參考做特別處理，將lvalue傳入forwarding reference就是一個例持。而這件事情就叫做reference collapsing。而規則如下
1. 若兩個參考中至少一個為lvalue reference，則兩個參考可視為一個lvalue reference
2. 若兩個參考皆為rvalue reference，則兩個參考可視為一個rvalue reference

而上述的例子foo(Widget& && w)為一個lvalue reference及一個rvalue reference，所以可以視為一個lvalue reference(等同於void foo(T& w))。

以上是reference collapsing發生在樣板實體化的狀況，事實上reference collapsing會發生在四種狀況
1. 樣板實體化
2. auto
3. 別名宣告
4. decltype

# 樣板實體化
除了上例樣板實體化的例子外，上面提到要詳細說明的std::forward，以下是簡略的std::forward實作

e.g.
```cpp
template <typename T>
T&& forward(remove_reference_t<T>& param)
{
    return static_cast<T&&>(param);
}
```

而std::forward會被這樣使用
```cpp
template <typename T>
void foo(T&& w)
{
    someFunc(std::forward<T>(w));
    ...
}
```

分成兩個狀況來說明為何std::forward可以達到完美轉發
1. 引數為lvalue，則T被推導為Widget&，則在forward中
```cpp
static_cast<T&&>(param)
```

實體化為
```cpp
static_cast<Widget&>(param);
// 由static_cast<Widget& &&>(param): 轉換而來
```

所以std::forward回傳型別為lvalue reference

2. 引數為rvalue，則T被推導為Widget，則在forward中
```cpp
static_cast<T&&>(param)
```
會被實體化為
```cpp
static_cast<Widget>(param)
```

所以std::forward回傳型別為rvalue reference。(此情況不會出現reference collapsing)

# auto
e.g.
由於auto的型別推導規則和樣板相同，所以一樣的狀況也會發生reference collapsing

e.g.
```cpp
Widget w;
auto&& w1 = w;
```

```cpp
auto&& w1 = w;
```
會被推導為
```cpp
Widget& w1 = w;
// 由Widget& && w1 = w; 推導而來
```

至於rvalue的case則和樣板相同，不會發生reference collapsing

e.g.
```cpp
auto&& w2 = Widget();

// 推導為 Widget&& w2 = Widget();
```

至此，我們可以說完全掌握了forwarding reference，其實可以說forwarding reference並不是新的reference，其只是有特別規則的reference而已
1. lvalue及rvalue會推導出不同型別，T為lvalue時會推導為T&，rvalue時會推導為T&&
2. 發生reference collapsing

至於第三種及第四種，其實只要跟型別宣告有關的都有可能牽扯到reference collapsing
# 別名宣告
以下例子都只是舉其中一種參考的參考做範例，實際上& &、& &&、&& &及&& &&都是合法的
e.g.
```cpp
template <typename T>
class Widget
{
    using T&& = rvalueToT;
    using T& = lvalueToT;
};

Widget<int&> o;
```

由於int&為lvalue reference，故T被推導為int&(T& &&推導為T&)，rvalueToT型別為int&，而lvalueToT根據推導型別也為int&(T& &推導為T&)。

# decltype
```cpp
int&& x1 = 1;

decltype(x1)&& x2 = 2;
```

由於x1為rvalue reference，故decltype(x1)&&推導為
```cpp
int&& x2 = 2;
// 由int&& && x2 = 2 推導而來
```