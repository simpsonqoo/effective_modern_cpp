## namespace汙染
以前，大括號內的範圍稱為local scope，任何在範圍內的變數在大括號結束時就會消失。但是這種規則在enum中卻不存在，所以一旦一個名稱已經給某一個enum內，在enum外的scope都還是可以使用該參數，當然，如果要重新宣告這個名稱的物件是不行的。

e.g.
```cpp
enum color = {red, black, blue};

auto while = false; // error，重複宣告變數名while
```

所以enum又被稱為unscoped enum，因為enum的名稱會洩漏到該scope外的地方。而C++11新增了scope enum，可以避免這種狀況

e.g.
```cpp
enum class Color = {red, black, blue};

auto white = false;     // ok，while變數名在這個scope第一次被宣告

Color c = white;        // error
Color c = Color::white  // ok
```

而因為使用scoped enum時會透過enum class宣告，所以又稱為enum class。scoped enum可以避免enum對命名空間汙染的狀況。

## 自動轉型
而使用scoped enum還有另一項優勢，由於unscoped enum會自動轉型成整數型別，再轉型成浮點數型別也是合法的，所以以下的例子是合法的：

e.g.
```cpp
enum Color {red};

Color c = red;

if (c > 10.1)
{
    ...
}
```

上述例子居然將enum內之物件與浮點數做比較。而如果使用scoped enum，就不會有自動轉型的問題，如果真的需要比較，就需要使用強制轉型(static_cast)。

e.g.
```cpp
enum class Color {red};

Color c = Color::red;

// if (c > 10.1)   // error Color::red不能和浮點數比較
// if (static_cast<int>(c) > 10.1)
{
    ...
}
```

## 需要定義
而使用scoped enum還有最後一個好處，可以使用forward-declare，也就是可以只宣告不定義。

```cpp
enum Color;         // error
enum class Color    // ok
```

會發生這種情況是因為enum是由編譯器決定其底層型別(underlying type)。若物件並不多，則就不會使用太多的byte儲存一個物件。(e.g.若只有三個物件，則可能compiler就會使用一個char儲存一個物件。)

而若定義數值較大的物件(大於一個char)，可能就會使用較多的byte型別儲存一個物件

e.g.
```cpp
enum Color
{
    red = 1,
    black = 0xFFFFFFFFFF
};
```

為了速度及記憶體考量，compiler就需要決定一個物件要用多少byte儲存，所以就需要知道enum內的物件有多少個、數值為多少，所以使用unscoped enum就需要宣告時就要定義。除非可以直接指定unscoped enum之物件大小

e.g.
```cpp
enum Color :: std::uint8_t; // ok
```

而一旦有使用新的狀態值，整個系統可能就要重新編譯，這是很多人最不喜歡碰到的事。

```cpp
enum Color
{
    red = 1,
    blue = 200,
    black = 0xFFFFFFFFFF
}
```

而使用scoped enum就不會有這種問題，這是因為scoped enum的底層預設型別是int
e.g.
```cpp
enum class Color;
// 和enum class Color : int; 同義

void doSomething(Color c);
```

而如果Color的變動不會影響doSomething的所有行為，該函式就不需要重新編譯。

## unscoped enum的好處及建議
在某些情況下，使用unscoped enum還是有好處的，假設我們要儲存一個使用者的相關資訊，其欄位如下

```cpp
using UserInfo = 
    std::tuple<
    std::string,    // 姓名
    std::string,    // email
    std::size_t>    // 年齡
```

如果想要讀取某一個使用者的email，可以用以下方式
```cpp
UserInfo user1;

auto mail = std::get<1>(user1);
```

1代表欄位1，但是可讀性較差，每次都必須將欄位1和email聯想在一起，如果不清楚的人，看到這個1是一點感覺都沒有，但如果有先定義每個欄位的名稱並將其做連結，就會使得可讀性上升。

```cpp
enum uInfo
{
    name,
    email,
    age
}

UserInfo user2;

auto mail = std::get<email>(user1);
```

一看就知道是讀取使用者的email欄位。但如果使用scoped enum，舊會顯得較為繁瑣。

```cpp
enum class userInfo
{
    name,
    email,
    age
};

UserInfo user3;

auto mail = std::get<static_cast(std::size_t)(userInfo::email)>(user3);
```

而若是有不同的底層型別，使用同一個std::size_t就顯得不太合適。如果想寫一個function實踐此功能，就必須是個constexpr函式(條款15)。而enum的底層型別可以由type_traits中的std::underlying_type_traits得到。實踐如下

```cpp
template <typename E>
constexpr auto toUType(E enumerator) noexcpet
{
    return static_cast<std::underlying_type_traits_t<E>>(enumerator);
}

auto mail = std::get<toUType(UserInfo::email)>(user3);
```

雖然比unscoped enum繁瑣很多，但是在避免namespace汙染其不必要的行別轉換的優點下，使用scoped enum還是比較好的選擇。