C++物件導向程式設計以類別、繼承與virtual function為核心，而其中一個概念便是衍生類別(derived class)實作的virtual function會覆寫(override)。而很驚訝的是，在我們想要使用覆寫功能時，很多時候我們根本沒有正確的使用覆寫函式，反而呼叫了基底函式。

# Override例子
e.g.
```cpp
class Base
{
  public:
    virtual void doWork();  // 基底
    ...
};

class Derived: public Base
{
  public:
    virtual void doWord();  // 覆寫doWork
    ...
}

std::unique_ptr<Base> p = std::make_unique<Derived>();  // 建立指向衍生類別的基底類別指標

...

p -> doWork;    // 實際呼叫衍生類別的doWork
```

# 成功覆寫條件
而成功覆寫必須滿足以下所有條件
- 基底class function必須是virtual
- 基底與衍生function名稱必須完全相同
- 基底與衍cuntion參數型別必須完全相同
- 基底與衍生function的const狀態必須相同
- 基底與衍生function傳回物件型別與例外規格必須完全相同

而C++11後又增加了一條限制
- function的參考限定符必須完全相同

# 參考限定符
若是我們希望傳入的引數只能接受lvalue參數，可能實作如下

```cpp
void doSomething(Widget& w);

Widget w1;
doSomething(w1);            // ok
// doSomething(Widget());   // error
```

如果我們希望傳入的引數只能接受rvalue參數，可能實作如下
```cpp
void doSomething(Widget&& w);

Widget w1;
//doSomething(w1);          // error
doSomething(Widget());      // ok
```

而如果我們想要限制的是"使用"該class function的class(也就是*this)為lvalue或是rvalue，就可以使用參考限定符，使用方式與function後的const一模一樣

e.g.

```cpp
class Widget
{
  public:
    Widget w;
    std::vector<double>& lvalueGetData() &
    {
        return data;
    }

    std::vector<double> rvalueGetData() &&
    {
        return std::move(data);
    }

  private:
    std::vector<double> data;
};

Widget w;
w.lvalueGetData();
//w.rvalueGetData();            // error
//Widget().lvalueGetData();     // error
Widget().rvalueGetData();    
```

使用std::move()是為了避免複製成本，條款二十三會對其做詳細說明。

# Override
根據覆寫成功的條件，可以知道只要有些微的不同就會使得覆寫失敗，呼叫錯誤的base calss function，所以我們會希望在覆寫失敗時，compiler能夠告訴我們覆寫失敗的訊息，這也就是識別詞override的用意。

以下例子可以體會一下有多容易出錯

e.g.
```cpp
class Base
{
  public:
    virtual void vf1() const;
    virtual void vf2(int x);
    virtual void vf3() &;
    void vf4() const;
};

class Derived : public Base
{
  public:
    virtual void vf1();
    virtual void vf2(unsigned int x);
    virtual void vf3() &&;
    void vf4() const;
};
```

以上四種都無法成功覆寫，就不一個一個列出哪裡不一樣了，但是我使用的compiler卻完全沒有任何警告訊息。

就是因為容易出錯，如果衍生類別函式要做覆寫動作，在其後加上override後如果無法覆寫base class function，就會產生compiler error

e.g.
```
為vf1加上override

...
class Derived : public Base
{
    ...
    virtual void vf1() override;
};
```

compile後會出現錯誤訊息
```
$ error: 'virtual void Derived::vf1()' marked 'override', but does not override
     virtual void vf1() override;
```

另外一提的是，final也是C++11後提供的識別符，意義和override相反，可以確保該function不會被覆寫。