假設我們有一個函式可以記錄目前的資訊，並將傳入的名字加入global scope中的multiset中

首先，我們可以考慮這樣做
```cpp
std::multiset<std::string> names;

void logAndAdd(const std::string& name)
{
    log();
    ...
    names.emplace(name);
}
```

而以上函式有三種使用方式<br>
e.g.
```cpp
std::string manName("Alex");

logAndAdd(manName);    // version1

logAndAdd(std::string("Alex"));    // version2

logAndAdd("Alex");    // version3
```

- 第一種使用方式沒有問題，就是一般的傳入參考物件的用法，在這裡由於name作為lvalue傳入emplace，所以無論如何都一定會有複製行為。
- 第二種使用方式為傳入rvalue，參數型別為const參考是可以接受rvalue的，但是由於name為lvalue，必須將rvalue複製成lvalue，之後該lvalue又會傳入emplace，產生兩次複製行為。但是由於傳入為rvalue，其實第一次的複製是可以用搬移取代，可以以此增加效能。
- 第三種是使用string literal，實際運作情形如第二種使用方式，不過如果可以的話，emplace是可以直接由string literal建立std::string的，但是在這裡卻會執行不必要的複製行為。

而如果使用forwarding reference，就可以直接傳入引數並不會發生不必要的複製行為

e.g.
```cpp
template <typename T>
void logAndAdd(T&& name)
{
    log();
    ...
    names.emplace(name);
}
```

第一種方式過程不會改變，但第二種則可以使用搬移語意省去第一次複製的成本，而第三種則是可以直接在std::multiset中建立std::string。

以上看似很完美的達成目標，但是實際上呼叫名字的方式可能不止一種，假設有另外一個方式可以藉由index來傳入名字，那我們就必須過載logAndAdd函式

e.g.
```cpp
std::string nameFromIdx(int idx);

void logAndAdd(int idx)
{
    log();
    ...
    names.emplace(nameFromIdx(idx));
}
```

其實如果正常使用下應該不會出現問題，但假設傳入的是unsigned int，就會出現錯誤
e.g.
```cpp
unsigned int i = 1;
logAndAdd(i);    // 一大串error message
```

這是由於相對於使用需要由unsigned int轉型成int的logAndAdd(int idx)，使用forwarding reference能更好的匹配參數，但是這樣就會使得unsigned int直接傳入forwarding reference的版本，造成unsigned int直接傳入emplace中，造成error。

由於forwarding reference幾乎可以接受任何的引數，所以任何需要轉型才能匹配的函式都不可能被呼叫。

本章後面有提到其他過載forwarding reference的情境，但是覺得這個例子已經足夠，如果有需要會再補充後面這一段。

條款27我覺得C++ template中第六章、8.4節都有更詳細的說明，看那裏的章節我覺得會比較清楚，所以將會省略條款27