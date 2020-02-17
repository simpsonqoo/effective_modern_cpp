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
class AddType
{
    typename vecIt<int>::type it;
};
```

這實在太麻煩了，我們通常希望能夠使用的方式應該會是像使用別名宣告這樣，而使用別名宣告則不需要加上typename及::type結尾

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

而在class宣告式內若template paramter有相依關係時，就必須加上typename。這是必要的，由於class宣告式內除了可以宣告型別外(e.g. typedef)，也可以宣告一般的物件(e.g. int)，所以需要有個方法可以區分二者，故規定在型別宣告需要加上typename。別以為這不重要，以下提供一個如果不使用typename就無法區別的例子。

```cpp
// AddType的特化
template <>
class AddType<int>
{
    int type;   // ...，AddType如果T是int，type就會是一個int物件
}
```

如此T若是其他非int型別，則AddType::type為std::vector<int>::iterator，是一種型別，而T若是int，則AddType::type是一個int物件！這是兩個完全不一樣的東西，是不是一個型別需然要根據T是甚麼而定，所以才需要在宣告型別時前面加上typename。

所以使用typedef除了需要多加上class宣告外，定義型別時還需要加上typename，種種因素使得使用別名宣告比typedef還要好。

而type_traits內有許多針對型別方面的物件，在C++11之前應該都是使用typedef方式宣告型別名稱，所以都需要加上::type，而在C++14之後，就可以使用_t代替::type

e.g.
```cpp
std::remove_const<T>::type  // C++11
std::remove_const_t<T>      // C++14以後

std::remove_reference<T>::type  // C++11
std::remove_reference_t<T>      // C++14以後
```

而兩者的轉換其實也很容易
```cpp
namespace
{
    template <typename T>
    using remove_const_t<T> = remove_const<T>::type;
}
```

而除了class宣告式內之外，所有使用class內部型別成員且和template paramter有相依關係的地方也都必須要加上typename。

e.g.
```cpp
template <typename T>
typename std::remove_cv<T>::type RemoveCv(T object)
{
    return object;
}
```

但如果使用std::remove_cv_t，就不需要加上typename，因為別名宣告的關係，compiler會直接知道std::remove_cv_t是型別

```cpp
template <typename T>
std::remove_cv_t<T> RemoveCv(T object)
{
    return object;
}
```