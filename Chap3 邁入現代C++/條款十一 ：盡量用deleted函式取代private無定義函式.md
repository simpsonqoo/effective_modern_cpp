如果想要避免一些會自動生成的函式(e.g. constructor、過載函式)，e.g. STL的istream和ostream都繼承自iostream，而在面對複製行為時，無法知道到底怎樣做會比較好，到底是要連同該值之後未讀取的值都複製，還是只要複製已讀取的值就好？所以，最後的結論是，就不要給使用者使用就好。

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