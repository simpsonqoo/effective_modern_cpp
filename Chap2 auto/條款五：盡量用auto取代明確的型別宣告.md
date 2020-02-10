主要有三點
1. 強迫一定要初始化
2. 可避免要打出下列的型別名稱
    - 複雜且冗長的型別名稱
    - 只有compier才知道，使用者根本不知道的型別
3. 可以直接保存closure，避免之前使用C++11的std::function所造成不必要的複製

## 強迫一定要初始化
在沒有auto出現時，下列這種為初始化的情形是合法的

e.g.
```cpp
int a;
```

此時a值並不確定，不同的compiler可能會有不同的結果，雖然通常而言int未定義時的初始值都是0，但是誰也沒辦法保證。

## 可避免要打出下列的型別名稱
假設我們想以iterator的方式存取vector中的所有元素，可以向下列方式使用

```cpp
std::vector<int> vec(100);
for (std::vector<int>::iterator it = vec.begin(); it != bev.end(); ++it)
{
    ...
}
```

在沒有auto之前，我們必須明確打出每個物件的型別名，像是這裡的std::vector<int>::iterator，如果是更複雜的巢狀類別，我們必須知道每一層class的名稱，這實在太麻煩了，我們必須背起來或是特別去查才找的到，但有了auto後，一切就變得簡單

```cpp
std::vector<int> vec(100);
for (auto it = vec.begin(); it != bev.end(); ++it)
{
    ...
}
```

而甚至有些型別是只有compiler才知道的，例如closure(之後的條款會提到)。

e.g.
```cpp
auto derefUpLess = 
[](const std::unique_ptr<Widget>& p1,
    const std::unique_ptr<Widget>& p2)
    {
        return *p1 < *p2;
    }
```

而C++14後甚至連closure內的參數型態都可以設定為auto

e.g.
```cpp
auto derefUpLess = 
[](const auto& p1,
    const auto& p2)
    {
        return *p1 < *p2;
    }
```

雖然C++11 STL中的std::function也可以用來表示closure變數，std::function是一個可以表示任何可呼叫物件的函式，如同建立函式指標時需要明確表示其指向的函式型別，std::function的物件也需要在建立時說明函式的型別，以下範例為表示一個函式型別

e.g.
```cpp
bool(const std::unique_ptr<Widget>& p1, 
    const std::unique_ptr<Widget>& p2)
```

若要建立一個std::function的物件：

e.g.
```cpp
std::function(bool(const std::unique_ptr<Widget>& p1, 
    const std::unique_ptr<Widget>& p2)) func;
```

所以如果要如同前面auto所表示的closure物件，C++11的方式為

e.g.
```cpp
std::function(bool(const std::unique_ptr<Widget>& p1, 
const std::unique_ptr<Widget>& p2)) func = 
    [](const std::unique_ptr<Widget>& p1, 
        const std::unique_ptr<Widget>& p2))
    {
        return *p1 < *p2;
    }
```

實在太麻煩及繁瑣了，就跟如果要建立一個std::vector，使用以下方式建立是一樣的感覺

e.g.
```cpp
std::vector vec = std::vector<int>{1, 2, 3}; //建立 [1, 2, 3]的vector
```

除了簡潔之外，auto表示closure物件的實現方式也與std::function不同，由於auto一定是使用和欲表示的closure物件有相同的型別，所以實際使用的記憶體也會和closure相同。但是使用std::function時，會額外建立std::function的template函式，對於其內所有的template parameter都會額外再耗用記憶體空間，而且實現std::function時限制不能使用inline，會造成許多額外的函式呼叫，使得其建立closure物件時幾乎一定會比使用auto來的慢。無論記憶體或是速度的考量，幾乎沒有使用std::function而不使用auto的理由。

而除了避免未初始化、簡潔的變數宣告及直接保存closure物件外，另外一個好處是該書作者認為的type shortcut的問題。

例如int可能在不同的環境有不同的位元數，各種不同container的size_t也是，有時我們可能無法保證在定義時不會發生位元數不同而需要轉換的問題。

e.g.
```cpp
std::vector<int> vec;
...
unsigned int size = vec.size();
```

上例左方的型別是unsigned int，右方是std::vector<int>::size_t，在不同的環境中，可能等號兩側的位元數會因為不同而需要做轉換而發生問題，例如在一般的32位元環境中unsigned int及std::vector<int>::size_t皆為32 bits，但是在64位元的環境中unsigned int為32 bits而std::vector<int>::size_t為64 bits，進而導致本來在32 bits的環境中可以正常執行的程式，在64 bits中卻會發生這種沒人想去特別解決的問題。

如果使用auto就不會有轉換方面的問題，一定保證不會發生轉換的問題。

e.g.
```cpp
auto size = vec.size();
```

除了以上優點之外，有些時候非得要宣告正確的型別才得以以我們比較想要、較有效率的方式執行。以一個要經過所有map中的key和value值的for loop為例：

e.g.
```cpp
std::unordered_map<int, int> m;
...
for (std::pair<int, int>& elem : m)
{
    ...
}
```

如果用以上的方式去經過所有map中的key和value，會造成額外的複製成本。由於std::unordered_map其key值都是const，由於std::pair中其第一個參數不為const int，故compiler會嘗試將std::pair<const int, int>的物件轉型為std::pair<int, int>，故會先建立std::pair<int, int>的暫存物件並將elem連結到該暫存物件，再將各個m的元素複製到暫存物件中，當loop結束時，暫存物件隨之消失。
但是我們寫出這個loop時，是希望不要有額外的記憶體配置和複製行為的。而如果使用auto，一切將變得簡單，不需要知道各個物件底下的實作細節

e.g.
```cpp
for (auto& elem : m)
{
    ...
}
```

造成以上兩個例子問題的根源都是型別轉換，如果在定義時明確指定物件型別，就有可能會造成自動轉型，而這些自動轉型可能會造成我們不想要的額外成本。一旦使用auto後，這些隱藏的自動轉型行為都將消失，既簡單又省事。

這裡還要在重複提醒一次的是，auto推導的物件型別可能和預期的不符(條款二、條款六)。當推導型別不符合預期的狀況發生時，需要做適時的修正。

雖然使用auto可能會使得開發人員對於物件的型別較不清楚，但是通常我們在開發時只需要知道物件大概的型別就好，例如我們只要知道是int相關的型別就好，是int、unsigned int、size_t...其實對我們來說不是很重要的事。通常只要知道物件是容器、數值或是指標...之類的就足夠了，所以auto通常都是可以被接受的選擇，但要不要選擇使用auto仍然是可考慮的。

而最後，開發程式最惱人的就是相依的物件卻沒有給予其相依的關係，一旦我們使用auto後，物件型別就會自動推導，當往後我們要對相依的物件改變其型別時，我們就不需要修改每個物件的型別，compiler會在每次編譯時都更新其型別。