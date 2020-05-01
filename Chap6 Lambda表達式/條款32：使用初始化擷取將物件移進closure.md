此條款只要使用C\++14以上的版本都會很簡單，若是C\++11就需要用比較迂迴的方法，篇幅佔了一半左右，但因為也不是個好方法，這裡不會提C\++11的方法。

如果我們需要將暫存物件傳入closure，或是想要將物件搬移進closure，就需要使用這個方法，因為之前的條款lambda只能接受已定義的物件，像是如果要傳1這個物件進入closure，在之前只能用這個方法。

e.g.
```cpp
int x = 1;
auto lambda = [x](){};

// auto lambda = [1](){};    // fail
```

甚至有些型別不支援複製(e.g. std::unique_ptr)，之前的方法都無法將這些物件傳入closure。

但是C++14後提供初始化擷取(init capture)使傳入任何種類的物件都成為可能。

假設我們想要搬移一個std::unique_ptr物件進入closrue

e.g.
```cpp
auto pw = std::make_unique<int>(0);

auto func = [pwInClosure = std::move(pw)](){...};
```

[]中等號兩端的可視範圍是不一樣的，pw的可視範圍為lambda定義式本身，而pwInClosure則是創建一個物件，在closure中都可以使用該物件。

而當然，可以直接初始化後就傳入closure

e.g.
```cpp
auto func = [pw = std::make_unique<Widget>()](){...};
```

由於初始化擷取比起之前的擷取更具有一般性，所以初始化擷取又稱為通用lambda擷取(generalized lambda capture)。