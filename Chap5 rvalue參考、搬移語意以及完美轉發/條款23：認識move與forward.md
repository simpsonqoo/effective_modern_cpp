其實，std::move並沒有做搬移動作，std::forward並沒有做轉發動作，實際上真的甚麼事情都沒有做。兩者皆為轉型函式，std::move一定會將物件轉型為rvalue，而std::forward則是"有可能"會將物件轉型。

在此之前，需要注意的是所有函式的參數都是lvalue，無一例外，就算是以rvalue reference參數也是一樣
e.g.
```cpp
void foo(int&& i){}    // i為lvalue
```

# std::move

只要看一下std::move的實作就可以知道真的甚麼事都沒做了，以下是簡略的std:move的實作

```cpp
template <typename T>
decltype(auto) move(T&& param)
{
    using ReturnType = remove_reference_t<T>&&;
    return static_cast<ReturnType>(param);
}
```

所以std::move並沒有做搬移，其只有做轉型而已。而由於只有rvalue才能做搬移，所以使用std::move後的物件就可以做搬移。

但是不見得所有的rvalue都能做搬移，像是const rvalue就無法做搬移，以下就是一個例子

e.g.
```cpp
const std::string s1{"abc"};    // s1內容未被搜刮，為複製行為
std::string s2{"def"};          // s2內容被搜刮，為搬移行為
auto s3(std::move(s1));
auto s4(std::move(s2));   
```

只要加上const語意，其物件內容就無法被修改，所以就無法使用搬移行為，進而轉為複製行為。以下是一般STL或是物件的特殊函式

e.g.
```cpp
class string
{
  public:
    string(const string& rhs);    // copy constructor
    string(string&& rhs);    // move constructor
};
```

copy constructor由於不會更改argument的內容，所以copy constructor使用const語意較為恰當。s1由於帶有const語意，s3在初始化時就會呼叫較為接近的copy constructor。

所以如果要使用搬移行為，被搬移的物件就不能是const，否則本來要發生搬移行為處會轉為複製行為。

# std::forward
相較於std::move是強制轉型，std::forward則是"有條件"的轉型。std::forward常用於將函式的參數型別相同的傳遞給另一個函式。

```cpp
void g(int& a);
void g(int const& a);
void g(int&& a);

template <typename T>
void f(T&& val)
{
    g(std::forward<T>(val));
}

Widget w;
f(w);    // 呼叫第一個g函式
f(std::move(w));    // 呼叫第三個g函式 
```

雖然val的型別仍然是lvalue，但是std::forward藉由T和val，偵測到"引數是rvalue，參數是lvalue"的情形時，會將參數轉為rvalue。條款28會對其實作做更詳細的說明。

書中後半段在描述std::forward也可以替代std::move，可以替代但不建議的理由，這裡就直接省略，結論就是由於std::forward還需要給定template argument，當template argument型別錯誤時就會有錯誤的執行方式，所以在必要時才使用std::forward，若確定要轉型為rvalue，建議使用std::move。