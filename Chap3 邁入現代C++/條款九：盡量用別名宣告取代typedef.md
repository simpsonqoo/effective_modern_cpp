在C++11之前，如果要反覆使用一個型別名稱很複雜的型別，常常會使用typedef，而在C++11之後，另外提供了一個別名宣告的方式希望能代替typedef。

e.g.
```cpp
// 使用typedef
typedef std::vector<int>::iterator vecIt;

// 使用別名宣告
using vecIt = std::vector<int>::iterator;
```

而別名宣告有個typedef所沒有的功能，那就是可以適用template。

e.g.
```cpp
template <typename T>
using vecIt = std::vector<T>::iterator;

vecIt<int> it;
```

但如果使用typedef，因為template不適用於typedef，就必須做一個template class後在class內建立一個型別。

e.g.
```cpp
template <typename T>
struct vecIt
{
    typedef std::vector<T>::iterator type;
};

vecIt<int>::type it;
```

而除了使用該型別需要另外刻一個class，使用時還要加上::type外，如果在另外一個class內會和該行別有相依的樣板參數T的話，就需要在定義前加上typename

e.g.
```cpp
template <typename T>
class Widget
{
    typename vecIt<int>::type it;
};
```

這實在太麻煩了，我們通常希望能夠使用的方式應該會是像使用別名宣告這樣，而使用別名宣告則不需要加上typename

e.g.
```cpp
template <typename T>
using vecIt = std::vector<T>::iterator;

template <typename T>
class Widget
{
    vecIt<int> it;
};
```