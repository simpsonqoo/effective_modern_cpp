C\++14後引入了generic lambda，可以讓參數式使用auto表示其型別，如此幾乎甚麼函式都可以用lambda表達式表現了，但還記得很特別的一種函式嗎？轉發函式，將引數完美轉發給其他函式的函式，如果我們試著實作轉發函式，大概會像這個樣子

e.g.
```cpp
auto func = [](auto&& param){std::forward<???>(param)};
```

來看看原本的轉發函式
```cpp
template <typename T>
void f(T&& param)
{
    g(std::forward<T>(param));
}
```

但是在lambda表達式中，我們沒辦法像函式那般有template parameter T可以使用，因為在lambda表達式中使用的是auto。

我們可能會想使用decltype(param)來代替T，但謹慎起見我們要檢查一下其是否真的適用於轉發函式。

在forwarding reference中我們提到lvalue會轉換成lvalue reference，rvalue會轉換成rvalue，auto&& param的型別為reference，lvalue reference使用decltype後轉發沒有問題，跟之前提到的forwarding reference狀況相同(兩個lvalue reference)

e.g.
```cpp
Widget& && forward(Widget&& param)
{
    return static_cast<Widget & &&>(param);
}
```

根據reference collasping，至少含有一個lvalue reference會合併成一個lvalue reference。

但是若是rvalue reference狀況就有些不一樣了，由於forwarding reference是將rvalue轉成rvalue，並不存在參考，可是這裡的auto&&卻會將rvalue轉成rvalue reference，forward函式實作如下

e.g.
```cpp
Widget&& && forward(Widget&& && param)
{
    return static_cast<Widget&& &&>(param);
}
```

根據forwarding collasping，兩個rvalue reference會合併成一個rvalue reference，雖然跟forwarding reference狀況有些不同，但是結果來說都是一樣的，都是轉發rvalue reference給其他函式。

所以保證了lvalue及rvalue行為都沒問題後，我們就可以安心的使用decltype(param)了

e.g.
```cpp
auto func = [](auto&& param){return g(std::forward<decltype(param)>(param));};
```

如果想要更一般化，使其可以接受任意數量的參數，一樣可以使用variadic方式

e.g.
```cpp
auto func = [](auto...&& params){return g(std::forward<decltype(params)>(params)...);}
```