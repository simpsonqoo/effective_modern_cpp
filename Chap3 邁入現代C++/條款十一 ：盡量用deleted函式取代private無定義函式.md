如果想要避免一些會自動生成的函式(e.g. constructor、過載函式)，e.g. STL的istream和ostream都繼承自iostream，而在面對複製行為時，無法知道到底怎樣做會比較好，到底是要連同該值之後未讀取的值都複製，還是只要複製已讀取的值就好？所以，最後的結論是，就不要給使用者使用就好。

# C++11以前作法
在C++11以前通常都是將該函式置於private區域。

e.g.
```cpp
template <class charT, class traits = char_traits<charT>>
class biasic_ios : public ios_base
{
  public:
    ...
  
  private:
    basic_ios(const basic_ios&);
    basic_ios& operator=(const basic_ios&);
};
```

除了置於private無法被使用者呼叫外，因為沒有定義函式，即使被friend函式使用，也會無法被使用(無法在連結期bind)。

# C++11後做法
而C++11後提供了更好的方式實作這樣的功能，也就是delete。

e.g.
```cpp
template <class charT, class traits = char_traits<charT>>
class biasic_ios : public ios_base
{
  public:
    basic_ios(const basic_ios&) = delete;
    basic_ios& operator=(const basic_ios&) = delete;
};
```

而使用delete和置於private是不同的，置於private還能被friend函式使用，但是若使用delete後，只要嘗試呼叫該函式就會無法compile，將錯誤階段提前到compile時期。

而一般慣例是會將delete函式宣告為public，因為當錯誤出現時，才會發現該函式被delete，否則若置於private，compile失敗可能會顯示為呼叫private函式而非呼叫已被delete的函式。

# delete可以但private無法實現的功能
## 所有函式皆適用
private僅限於類別的成員物件才能夠使用，但delete不限於類別成員，所有的函式都可以使用delete，像是如果不希望一些重載情況發生，可以將那些函式設為delete。

e.g.
假設我們想判斷一個數字是否是偶數，那麼傳入的引數應該是整數型別，我們使用bool isEven(int num)來實現，但是由於隱性轉換，即使傳入的引數是double、char...都是合法的，但是判斷這些物件是不是偶數對我們來說沒有意義，所以我們可以使用delete將這些過載視為非法。

e.g.
```cpp
bool isEven(int a)
{
    return (a & 1) == 0;
}

bool isEven(double) = delete;
bool isEven(char) = delete;
bool isEven(bool) = delete;

isEven(10);     // ok
//isEven('a');  // error
//isEven(5.2);  // error
//isEven(true); // error
```

而無論是double或是float，都會使用isEven(float)，因為過載會優先選擇isEven(double)而非isEven(float)。

## 避免產生不想要產生的樣板
若有某幾種型別的樣板不想要被產生，可以使用delete，這也是private所無法達到的功能。

e.g.
若我們在處理指標的template，但是指標型別中有兩個比較特別的特例，分別是void*及char*，前者無法被解參考及遞增遞減，後者則是字串，我們想要避免這兩種template函式被產生。

```cpp
template <typename T>
void processPointer(T* p)
{
    ...
}

void processPointer(void*) = delete;
void processPointer(const void*) = delete;
void processPointer(char*) = delete;
void processPointer(const char*) = delete;

processPointer("hello");    // error
```

不過如果要完善禁止這些型別，還需要禁止vollatile、std::wchar_t...之類的過載，要全都禁止也是蠻麻煩的一件事。

### C++11前不能禁止某些特定樣板
因為規定主樣板和樣板特化必須有相同的存取權限，所以不能讓樣板特化設為private。