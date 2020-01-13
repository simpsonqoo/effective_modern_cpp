decltype會將給定物件之型別回傳，且幾乎都會符合預期。而不同於樣板推導及auto，使用decltype幾乎都會回傳與原物件完全相同的型別，連前面幾個條款中樣板推導或是auto可能推導不同的型別，使用decltype都會回傳相同的型別。

C++11後推出了trailing return type的語法，函式除了引數之外，也可以推導返回物件的型別。

e.g.
```cpp
template <typename Container, typename Index>
auto ContainerAccess(Container& c, Index& i) -> decltype(c[i])
{
    return c[i];
}
```

而C++14後不必使用trailing return type，可以直接用auto就自動推導返回的型別，但使用的是樣板推導型別。

雖然型別的推導通常不會有問題，但是在使用的時候還是有些地方需要注意一下，通常在使用容器內的元素時，通常我們都會預期可以直接修改容器內的元素值，故通常都是回傳T&。e.g. a[5]=3，我們會預期能將容器a內第五個元素值改為3。但是在樣板型別推導的規則中，若是回傳型別含有參考，會去除參考的部分(參考條款一)，如此若是修改回傳的物件，則無法修改原本想要修改的物件內容。

e.g.
```cpp
#include <iostream>
#include <vector>

template <typename Container, typename Index>
auto ContainerAccess(Container& c, Index& i)
{
    return c[i];
}


int main()
{
    std::vector<int> vec{1};
    int idx = 0;

    ContainerAccess(vec, idx) = 3;  //compiler error，rvalue不能放在等號左側
    std::cout << vec[0] << std::endl;
}
```

為了改善這樣的狀況，型別推導必須使用decltype，故將回傳型別改為decltype(auto)即可，可用在所有auto適用的地方，且使用decltype的規則。

e.g.
```cpp
#include <iostream>
#include <vector>

template <typename Container, typename Index>
decltype(auto) ContainerAccess(Container& c, Index& i)
{
    return c[i];
}


int main()
{
    std::vector<int> vec{1};
    int idx = 0;

    ContainerAccess(vec, idx) = 3;
    std::cout << vec[0] << std::endl;
}
```

output:
```
3
```

這裡是修正參考回傳型別，其實decltype(auto)不僅限於函式回傳型別，一般的物件宣告也可以使用

e.g.
```cpp
Widget w;
Widget& rw = w;         //型別為Widget
auto cp = w;            //型別為Widget
decltype(auto) dw = w;  //型別為Widget&
```

而有時候可能我們不是想要修改容器內容，只是想要取得容器中某個元素的內容，如果是這樣的狀況，理應上來說我們應該是可以接受取用rvalue的容器，以上上例來說，但是如果直接給定c的物件型別為Container&，就無法接受rvalue容器。

e.g.
```cpp
#include <iostream>
#include <vector>

auto content = ContainerAccess(std::vector<int>{3}, idx);
```

要解決這種問題，解決方式為universal reference。

e.g.
```cpp
template <typename Container, typename Index>
decltype(auto) ContainerAccess(Container&& c, Index&& i)
{
    return c[i];
}


int main()
{
    std::vector<int> vec{1};
    int idx = 0;

    ContainerAccess(vec, idx) = 3;
    std::cout << vec[0] << std::endl;
    std::cout << ContainerAccess(std::vector<int>{1}, 0) << std::endl;
}
```

本章接續內容的std::forward就等到條款25再做說明。

最後，需要注意的是當int x = 0時，decltype(x)回傳之型別為int，但是如果為decltype((x))，則是回傳int&。同樣的情形也會發生在函式的回傳上。

e.g.
```cpp
decltype(auto) f1()
{
    int x = 0;
    return x;   //回傳型別為int
}

decltype(auto) f2()
{
    int x = 0;
    return (x); //回傳型別為int&
}
```
