只要看到物件使用ravlue參考做連結，就代表其一定可以被搬移

e.g.
```cpp
class Widget
{
  public:
    Widget(Widget&& w);    // w必定可以被搬移
};
```

若需要將rvalue參考函式參數傳遞給下一個函式，就需要使用std::move

e.g.
```cpp
class Widget
{
  public:
    Widget(Widget&& w):
      name(std::move(w.name)), p(std::move(w.p)){}
    ...
};
```

而若是forwarding reference，則可以使用std::forward來傳遞可搬移的物件

e.g.
```cpp
class Widget
{
  public:
    template <typename T>
    Widget(T&& n):
    {
        name = std::forward<T>(n);
    }
};
```

條款23有提到盡量不要用std::forward代替本來std::move使用的時機，因為std::forward需要再加上template argument，會多一個程式出錯的可能。但是，反過來說情況會更糟糕，一定要避免原本使用std::forward的時機使用std::move

因為使用std::forward的物件並不一定為rvalue，若將std::move取代std::forward，可能會將原本的lvalue做搬移，導致原有物件成為未知

e.g.
```cpp
class Widget
{
  public:
    template <typename T>
    void setName(T&& n)
    {
        name = std::move(n);
    }
  private:
    std::string name;
}

std::string widgetName("lable1");

Widget w;
w.setName(widgetName);

...    // widgetName成為未知
```

而或許可以使用條款24使用的方法，分別過載const lvalue及rvalue

e.g.
```cpp
class Widget
{
  public:
    template <typename T>
    void setName(const std::string& n)
    {
        name = n;
    }
    
    void setName(std::string&& n)
    {
        name = std::move(n);
    }
  private:
    std::string name;
}
```

# forwarding reference v.s. 固定型別<br>
1. 效能
而如果直接將參數設為std::string(也就是固定參數的型別)，比起forwarding reference可能會多幾次型別轉換

e.g.
```cpp
w.setName("lable1");
```
- forwarding reference<br>
T&&為string literal，在做賦值時才會轉成std::string
- std::string<br>
先將string literal轉為std::string n(暫存物件)，做std::string的搬移行為後再將該暫存物件解構

所以使用forwarding reference可以有更少的型別轉換，有更好的效能

2. 擴縮性(scalibility)<br>
但是如果參數量很大，如果每個參數都要分別使用過載const lvalue及rvalue，那過載函數就需要宣告很多。而且如果是不固定數量的參數(pack parameter)，是不可能事前將所有的過載版本事先宣告好

e.g.
```cpp
template <typename T, typename... Args>
std::unique_ptr make_unique(Args&&... args); // 無法將所有的參數做過載
```

函式內將參數傳遞給其他函式時，通常使用std::forward

不過如果該參數需要在函式內反覆使用，在前幾步時就以std::forward或std::move傳遞可能使得該參數被搬移，導致之後無法使用這些參數，所以通常會在最後一次使用時才使用std::move或std::forward。

e.g.
```cpp
template <typename T>
void setText(T&& t)
{
    reverseText(t);
    modifyText(t);
    ... 
    appendText(std::forwrad<T>(t));
}
```

# 回傳值為傳值
## 回傳函式參數(轉發)
### rvalue reference
當回傳值為傳值時，函式參數為rvalue reference時以std::move回傳，為forwarding reference時以std::forward回傳，可以避免不必要的複製行為

以矩陣加法為例
e.g.
```cpp
Matrix operator+(Matrix&& m1, const Matrix m2)
{
    m1 += m2;
    return std::move(m1);
}
```

若不以std::move回傳
e.g.
```cpp
Matrix operator+(Matrix&& m1, const Matrix m2)
{
    m1 += m2;
    return m1;    // 將m1回傳會產生複製行為
}
```

由於m1是lvalue，使用傳值時會產生複製行為。但是若Matrix支援搬移行為，則使用std::move的矩陣加法並不會產生複製成本。

### forwarding reference
對於forwarding reference也是一樣
e.g.
```cpp
template <typename T>
void reduceAndCopy(T&& w)
{
    w.reduce();
    return std::forward<T>(w);
}
```

# 回傳值為傳值，且不為參數(為local物件)
當回傳值不為函式的參數，為local物件時，就不適用於以上的規則，這是因為C++標準有特別為這種情況做優化，稱為傳回值最佳化(return value optimization(RVO))。

e.g.
```cpp
Widget makeWidget()
{
    Widget w;
    ...
    return w;
}
```

但是RVO要符合下列兩個條件才會啟動最佳化
1. local物件型別與回傳型別相同
2. 回傳的是local物件本身

RVD會直接local物件記憶體位置配置給回傳值，避免複製行為發生。

而若是使用std::move，因為回傳值並不是local物件，而是w的reference，並無法符合第二點要求，所以使用std::move並無法觸動RVO

e.g.
```cpp
Widget makeWidget()
{
    Widget w;
    ...
    return std::move(w);
}
```

而可能某些狀況難以將local物件記憶體位置配置回傳值，可能這時候覺得使用std::move才比較保險，但事實上並不是。就算符合RVO但是無法將記憶體配置給回傳值，RVO會退而求其次使用std::move搬移，也就等同於上一個範例。

所以，若是符合RVO的兩個條件，其他的就交給compiler吧！別自己自作主張，否則就只是阻礙compiler為程式做優化而已。