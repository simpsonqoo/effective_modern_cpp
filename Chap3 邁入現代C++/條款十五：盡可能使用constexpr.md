# constexpr在不同情境上就會有不同意思
在C++11後提供了constexpr的語法，在物件上及在函式中卻表達不同的意義，以下說明其不同之處

# constexpr物件
constexpr基本上就是const物件，但是其數值必須在編譯期就得知，在某些情況下，物件必須在編譯期得知才能使用，例如陣列大小、整數樣板參數、enum數值...等，所以若是要使用這些物件，就必須是constexpr物件。

e.g.
```cpp
int i1 = 5;

constexpr auto i2 = i1;     // error，編譯期無法得知i1數值
std::array<int, i1> arr1;   // error，同上

constexpr int i3 = 10;      // ok
std::array<int, i3> arr2;   // ok

const int i4 = i1;          // ok
std::array<int, i4> arr3;   // error，編譯期無法得知i4數值
```

# constexpr函式
constexpr函式狀況會和constexpr物件有些不同，constexpr函式並不保證一定會在編譯期執行完畢，若無法在編譯期執行完畢，就會在執行期執行。換而言之，就是不強求函式在編譯期執行完畢。

但是constexpr函式之引數及回傳型別有一個限制，就是其型別只能是literal type，基本上除了void外其他的內建型別都可以作為constexpr函式引數或是回傳值(C++11的限制，C++14後則放寬)，但其他像是STL內的任何物件都不能是constexpr的引數型別。

所以如果將constexpr函式回傳值使用在需要在編譯期完成的狀況，如果constexpr函式無法在執行期執行完畢，就會產生compiler error。

e.g.
```cpp
//e.g.1
constexpr int addOne(int num)
{
    std::cout << std::endl;
    return num + 1;
}

constexpr auto num1 = addOne(1); // compiler error
std::array<int, addOne(5)> arr1;    // compiler error

// e.g.2
constexpr int addTwo(int num)
{
    return num + 2;
}

constexpr auto num2 = addOne(1); // ok
std::array<int, addTwo(5)> arr2;    // ok
```

以下實作一個分數的階乘函式

```cpp
constexpr int factorial(int a)
{
    ...
}
```

此函式有一個引數，回傳$a!$。在C++11時，constexpr函式內僅能接受一個指令，而在C++14後就放寬限制，就可以用已下實現factorial函式。

```cpp
constexpr int factorial(int a) noexcept
{
    int result = 1;
    for (int i = 1; i <= a; ++i)
        result *= i;
    return result;
}
```

如果上述函式引數a可以在編譯期決定，則factorial函式就能在編譯期執行完畢並回傳結果，若否，就是在執行期才能決定回傳值。

而constexpr也可以用在constructor之類的函式上，使得使用者自訂物件就可以在編譯期初始化、取值、賦值...。

```cpp
class Point
{
  public:
    constexpr Point() noexcept : x(0.0), y(0.0){}
    constexpr Point(double xVal, double yVal) noexcept
    : x(xVal), y(yVal){}

    constexpr double xValue() const noexcept { return x; }
    constexpr double yValue() const noexcept { return y; }

    constexpr void setX(double newX) noexcept {x = newX;} // C++14後合法
    constexpr void setY(double newY) noexcept {y = newY;} // C++14後合法

  private:
    double x, y;
};
```

所以只要初始化參數能在編譯期決定數值，Point物件就能在編譯期初始化完成

e.g.
```cpp
constexpr
Point midpoint(const Point& p1, const Point& p2)
{
    return { (p1.xValue() + p2.xValue()) / 2, 
            (p1.yValue() + p2.yValue()) / 2};
}
```

所以以往有些只能在執行期執行的事情，就可以使其在編譯期完成。

而對於會修改Point物件的setX及setY，雖然constexpr成員函式應該也是無法修改的，但是C++14後放寬了這條限制，。

e.g.
```cpp
constexpr
Point reflection(const Point& p) noexcept
{
    Point result;

    result.setX(-p.xValue());
    result.setY(-p.yValue());

    return result;
}

reflection(p1); // ok
```

需要注意的是，
```cpp
Point result;
```
不能改為
```cpp
constexpr Point result;
```