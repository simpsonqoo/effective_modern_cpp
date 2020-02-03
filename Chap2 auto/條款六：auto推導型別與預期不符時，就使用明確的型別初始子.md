在某些時候，使用auto初始化推導出來的型別並不符合預期，通常這種狀況是發生在proxy類別的時候。

# proxy簡介
proxy簡單來說就是設計該型別的開發者為了效能的考量，不使用直觀的回傳的型別，而是回傳了一個不同的型別，而為了使用者使用，會另外做出該型別轉換為直觀型別的機制，對於使用者的觀點來看，正常使用都不會發現有該不同型別的產生，所以就好像是被隱形了一樣，以下以std::vector<bool>為例說明。

每個bool物件佔據1byte，但實際上一個bool僅需1個位元就能表示，std::vector<bool>為了空間的考量，compiler在編譯時會將8個bool物件緊縮成一個byte，而每個bool的型別會變成std::vector<bool>::reference。
[參考資料](https://en.cppreference.com/w/cpp/container/vector_bool/reference)

如果不做特別的使用，通常使用者不會發現這個型別的產生
e.g.
```cpp
std::vector<bool> a{true, false, true};

bool b = a[2];  // a[2]是std::vector<bool>::reference，轉型為bool後存為b
```

中間其實有做型別轉型，但使用者如果沒有注意完全不會察覺，也不會影響使用。

# 使用auto在proxy類別上
當使用auto在proxy類別上時，該型別就被推導為proxy的型別，此時在使用時就很有可能會出錯，以上例來說，呼叫a[2]時會先產生一個暫存的指標指向vector，並給予bit的位移量，但是由於c++不允許能夠存取bit的行為，故該指標在指令結束後就會消失，此時該指標就會成為dangling pointer，往後使用b時就很有可能出錯。

還有另外一個使用proxy類別的例子，就是表示式樣板(expression template)，像是有些實作矩陣Matrix類別的library，在做兩個矩陣的相加時，為了效能的優化並不會回傳Matrix物件，而是會回傳類似於Sum<Matrix, Matrix>的型別。

所以，我們要避免這種程式碼
```cpp
auto someVar = "隱形"的proxy類別
```

但是這並不會是叫我們不要使用auto的原因，雖然proxy類別在使用時我們幾乎不會察覺(這也是開發者設計proxy的目的)，不過，通常我們可以在標頭檔中看見該proxy物件，以std::vector<bool>為例
e.g.
```cpp
namespace std{
    class vector<bool, Allocator> {
      public:
        ...
        class reference {...};
        ...
        reference operator[](size_type n);
        ...
    }
}

```
如果我們知道通常vector的[]回傳值通常都是reference，那麼看到以上程式我們就可以知道這是proxy。

但是就算是proxy類別，我們仍然可以使用auto，而且使用auto可以更明確的知道有使用型別轉型。

e.g.
```cpp
std::vector<bool> a{true, false, true};
auto b = static_cast<bool>(a[2]);
```

這種方式本文作者稱為"明確型別初始子"，而且這種明確型別的形式不僅限於proxy類別，在以往會做隱性轉換的初始化使用這種形式能夠更直觀的知道有型別轉換的發生。

e.g.
```cpp
int a = 1.1;    // 發生double -> int 的隱性型別轉換
auto b = static_cast<int>(1.1); // 明確的知道有形別轉換的發生
```