對於一般變數而言，如果能夠設為const，會鼓勵盡量使用const，因為如果我們之後不小心修改了一個本來不應該可以被修改的變數，compiler會提醒我們。但是，在C++11以前指標卻不是這樣被鼓勵的，指向const的指標應該要設為const_iterator，但是在C++11以前卻會發生一些問題，因為C++11以前沒有支援像是cbegin(), cend()之類的成員函式(e.g. std::vector([cppreference](https://en.cppreference.com/w/cpp/container/vector/begin)))，針對iterator的操作也僅限於iterator而沒有包括const_iterator(e.g. insert)，所以如果需要用到const_iterator，就需要呼叫begin()或是end()後轉型成const_iterator，或是使用像是insert之類的函式時，需要將const_iterator轉型成iterator。

e.g.
```cpp
using IterT = std::vector<int>::iterator; 
    using ConstIterT = std::vector<int>::const_iterator;

    ConstIterT ci = std::find(static_cast<ConstIterT>(vec.begin()),
            static_cast<ConstIterT>(vec.end()), 2);     // 到目前還ok
    
    vec.insert(static_cast<IterT>(ci), 5);  // error，無法由const_iterator轉型為iterator
```

由const_iterator轉型成iterator是很困難的，沒辦法用一般正規的static_cast做轉型，所以在C++11以前通常建議大家不要使用const_iterator。

但在C++11後，隨著container都支援cbegin(),cend()，大部分的成員函式、STL algorithm都支援const_iterator後，就可以建議能使用const_iterator就使用const_iterator。

e.g.
```cpp
std::vector<int> vec{1, 2, 3};

auto it = std::find(vec.cbegin(), vec.end(), 2);
vec.insert(it, 5);  // ok
```

本條款後面有C++11沒有支援好的部分，這在C++14後都已經獲得改善，所以就不多花篇幅贅述。