Pimpl Idiom是創造一個指向實作的class，取代在宣告class時就需要定義member class。

[好文共賞](https://goodspeedlee.blogspot.com/2016/01/c-pimpl.html)

這裡簡單講一下網站裡說明Pimpl Idiom的好處，假設下圖中A.cpp、B.cpp、C.cpp、D.cpp皆會include A.hpp，那每當A.hpp中有變動時，在編譯時因為A.hpp比四個cpp檔都還要新，所以有include這個A.hpp的cpp都需要重新編譯一次，若有include A.hpp的cpp很多，會造成每次修改一次A.hpp重新編譯時要花費非常多的時間。Pimpl Idiom就是希望將容易修改的部分獨立於A.hpp之外，每次編譯時因為沒有修改A.hpp，就不會重新為A.hpp做編譯。<br>
![](https://i.imgur.com/NFfhMYS.png)


e.g.
```cpp
// Widget1.hpp
#include <string>
#include "Gadget.hpp"

class Widget
{
  public:
    Widget();
  Private:
    std::string name;
    Gadget g1, g2;
};
```
![](https://i.imgur.com/NKyf9Pl.png)


如果Gadget物件是很常被更動的，那麼在每次更動Gadget時每個Widget都需要重新編譯，如果能改成這樣![](https://i.imgur.com/uS64ZNQ.png)在真正該Widget實作時才會看的到Gadget，減少編譯時間。

e.g.
```cpp
// Widget1.hpp

class Widget
{
  public:
    Widget();
    ...
    
  private:
    struct Impl;
    std::unique<Impl> pImpl;
};

//Widget1.cpp
#include <string>
#include "Gadget.hpp"

struct Widget::Impl
{
    std::stirng name;
    Gadget g1, g2;
}

Widget() : pImpl(std::make_unique<Impl>(){}

// main.cpp
#include "Widget.hpp"

int main()
{
    Widget w;    // compiler error
}
```

error message
```
error: invalid application of 'sizeof' to incomplete type 'Widget::Impl'
  static_assert(sizeof(_Tp)>0,
                ^~~~~~~~~~~
test.cpp:142:1: error: no declaration matches 'int Widget::widget()'
 Widget::widget(): pImpl(std::make_unique<Impl>()){}
```

使用std::unique_ptr很方便的就是不需要在destructor中定義Impl的delete行為，但需要注意的是widget.hpp的Impl屬於不完全型別，且std::unique_ptr指向該不完全型別。

雖然看起來沒甚麼問題，但在創建Widget時卻會出現compiler error，這是由於std::unique_ptr的預設解構子所產生的問題
std::unique_ptr的解構子會呼叫assert以確保指向的物件並不是不完全型別。

由於該問題是發生在解構時，所以僅需要在std::unique_ptr呼叫解構子前，Impl就已經是個完全型別即可，而上例中std::unique_ptr只有在Widget呼叫解構子時才會自動為智慧型指標std::unique_ptr做解構動作，所以僅需要在看到Widget的解構子前，Impl已是完整型別即可

e.g.
```cpp
// Widget1.hpp
#include <memory>

class Widget
{
  public:
    Widget();
    ~Widget();    // 因為之後要安排解構子和Impl定義的順序，所以需要宣告解構子
    ...
    
  private:
    struct Impl;
    std::unique_ptr<Impl> pImpl;
};

// Widget1.cpp
#include <string>
#include "widget.hpp"
#include "Gadget.hpp"

struct Widget::Impl
{
    std::string name;
    Gadget g1, g2;
};

Widget::Widget() : pImpl(std::make_unique<Impl>()){}
Widget::~Widget() = default;

// main.cpp
#include "widget.hpp"

int main()
{
    Widget w;
}
```

error messate
```
error: invalid application of 'sizeof' to incomplete type 'Widget::Impl'
  static_assert(sizeof(_Tp)>0,
```

而由於自行定義了destructor，所以move constructor及move assignment就不會自動被生成，需要自行定義，但如果只是單純的宣告即定義，使用時會發生compiler error。這是因為在move操作時，若例外發生會啟動刪除行為，而刪除行為需要知道Impl的完整定義，由於Impl是未完整型別，故會發生compiler error。

e.g.
```cpp
// Widget1.hpp
#include <memory>

class Widget
{
  public:
    Widget();
    ~Widget();    // 因為之後要安排解構子和Impl定義的順序，所以需要宣告解構子
    Widget(Widget&&) = default;
    Widget& operator=(Widget&&) = default;
    ...
    
  private:
    struct Impl;
    std::unique_ptr<Impl> pImpl;
};

// Widget1.cpp
#include <string>
#include "widget.hpp"
#include "Gadget.hpp"

struct Widget::Impl
{
    std::string name;
    Gadget g1, g2;
};

Widget::Widget() : pImpl(std::make_unique<Impl>()){}
Widget::~Widget() = default;

// main.cpp
#include "widget.hpp"

int main()
{
    Widget w1;
    Widget w2(std::move(w1));    // compiler error
    w1 = std::move(w2);          // compiler error
}
```

而解決方法一樣先宣告後在定義destructor處先定義move操作相關函式即可

e.g.
```cpp
// widget1.hpp
#include <memory>

class Widget
{
  public:
    Widget();
    ~Widget();    // 因為之後要安排解構子和Impl定義的順序，所以需要宣告解構子
    Widget(Widget&&);
    Widget& operator=(Widget&&);
    
  private:
    struct Impl;
    std::unique_ptr<Impl> pImpl;
};

// widget1.cpp
#include <string>
#include "widget.hpp"
#include "Gadget.hpp"

struct Widget::Impl
{
    std::string name;
    Gadget g1, g2;
};

Widget::Widget() : pImpl(std::make_unique<Impl>()){}
Widget& Widget::operator=(Widget&&) = default;
Widget::Widget(Widget&& rhs) = default;
Widget::~Widget() = default;

// main.cpp
#include "widget.hpp"

int main()
{
    Widget w;
    Widget w1(std::move(w));
    w = std::move(w1);
}
```

constructor、destructor即move操作都提到了，就剩下copy操作，由於std::unique_ptr並不支援copy操作，所以若要支援copy操作則需要自行定義。而copy操作和move操作類似，都必須在destructor定義前先定義好copy操作函式。以下是完整的程式

e.g.
```cpp
// widget1.hpp
#include <memory>

class Widget
{
  public:
    Widget();
    ~Widget();    // 因為之後要安排解構子和Impl定義的順序，所以需要宣告解構子
    Widget(Widget&&);
    Widget& operator=(Widget&&);
    Widget(const Widget&);
    Widget& operator=(const Widget&);
    
  private:
    struct Impl;
    std::unique_ptr<Impl> pImpl;
};

// widget1.cpp
#include <string>
#include "widget.hpp"
#include "Gadget.hpp"

struct Widget::Impl
{
    std::string name;
    Gadget g1, g2;
};

Widget::Widget() : pImpl(std::make_unique<Impl>()){}
Widget& Widget::operator=(Widget&&) = default;
Widget::Widget(const Widget& rhs): pImpl(std::make_unique<Impl>(*rhs.pImpl)){}
Widget& Widget::operator=(const Widget& rhs)
{
    pImpl = std::make_unique<Impl>(*rhs.pImpl);
    return *this;
}
Widget::Widget(Widget&& rhs) = default;
Widget::~Widget() = default;

// main.cpp
#include "widget.hpp"

int main()
{
    Widget w;
    Widget w1(std::move(w));
    w = std::move(w1);
    Widget w2(w);
    w = w2;
}
```

由於使用std::unique_ptr可以擁有單一所有權，所以是最應該優先被考慮使用的，而若真的需要使用std::shared_ptr，由於實作std::shared_ptr時並不會有像std::unique_ptr產生destructor的問題，所以可以直接使用自動生成的特殊函式

e.g.
```cpp
// widget2.hpp
#include <memory>

class Widget
{
  public:
    Widget();
    
  private:
    struct Impl;
    std::shared_ptr<Impl> pImpl;
};

// widget2.cpp
#include <string>
#include "widget.hpp"
#include "Gadget.hpp"

struct Widget::Impl
{
    std::string name;
    Gadget g1, g2;
};

Widget::Widget() : pImpl(std::make_unique<Impl>()){}

// main.cpp
#include "widget.hpp"

int main()
{
    Widget w;
    Widget w1(std::move(w));
    w = std::move(w1);
    Widget w2(w);
    w = w2;
}
```