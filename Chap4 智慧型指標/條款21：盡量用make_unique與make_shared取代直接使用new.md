C\++11時std::make_unique還尚未推出，推遲到在C\++14才推出，std::make_unique和std::make_shared就是特地為建立std::unique_ptr及std::shared_ptr推出的函式。

建立指標的函式有三個，但是作者著重在較重要的兩個，另外一個std::make_allocate_shared功能和std::make_shared差不多。

而作者強烈建議使用std::make相關函式建立智慧指標原因有下列

1. 建立智慧型指標時型別不需要重複出現物件型別
2. 例外安全
3. 對於std::shared_ptr而言，可以一次直接配置好兩個指標

## 建立智慧型指標時不需要重複出現物件型別
舉幾個例子看就一目瞭然了
e.g.
```cpp
auto upw1 = std::make_unique<Widget>();        // Widget出現一次
std::unique_ptr<Widget> upw2 = new Widget();    // Widget出現兩次

auto spw1 = std::make_shared<Widget>();        // Widget出現一次
std::shared_ptr<Widget> spw2 = new Widget();    // Widget出現兩次
```

由於使用new時預設是使用原始指標初始化，所以必須指明其為智慧型指標，不能使用auto，而std::make相關函式已經很明確地指明其會建立智慧型指標，所以使用auto即可。

## 例外安全
我們使用智慧型指標有一個很重要的目的就是只要使用智慧型指標後，指標會在該scope結束或是產生例外時也能夠自動清除該指標，但是使用new後當例外發生時卻不能保證指標被自動清除，原因是若以new創建智慧型指標，仍會先創建原始指標後再轉型為智慧型指標，但是這兩個動作卻不保證為atomic operation，所以一旦使用new後程式發生例外，則會發生洩漏。

這裡舉一個例子，假設有一個計算優先權的函式int computePriority()，會回傳其優先權值，而processWidget(std::shared_ptr<Widget> spw, int priority)則是會對Widget物件進行些運算。

e.g.
```cpp
process(std::shared_ptr<Widget>(new Widget()), computePriority());
```

這個函式在運作前會做三件事
1. new Widget
2. 將原始指標轉換為std::shared_ptr
3. 計算優先權computePriority

而第二件事和第三件事其先後順序是無法得知的，第一件及第二件事不會發生例外，但是第三件事情有可能，若在計算優先權優先執行且在執行時發生例外，則原始指標就會發生洩漏。

而使用std::make相關函式就不會產生這些問題，在執行優先權計算時，必定已經建立好智慧型指標，而即使優先權計算會發生例外，智慧型指標也會安全的清除該指標。

## 對於std::shared_ptr而言，可以一次直接配置好兩個指標
上面也提到，在以new創建智慧型指標時，會先建立原始指標後在由其為智慧型指標初始化，也因為如此，創建智慧型指標前會先建立一指標，而如果是創建std::shared_ptr，則需要再配置一個指標指向control block，可以提高執行的速度。

而雖然使用std::make相關函式取代new好處是相當明顯的，但是仍然有些狀況需要使用new
1. 自訂destructor
2. 創建智慧型指標時傳入創建物件參數使用std::initializer_list

## 自訂destructor
std::make相關函式沒辦法初始化智慧型指標時接受自訂destructor，所以如果要實現自訂destructor的智慧型指標，就必須使用new

e.g.
```cpp
auto delPtr = [](Widget* pw){};

std::unique_ptr<Widget, decltype(delUniquePtr)> upw(new Widget, delPtr);

std::shared_ptr<Widget> spw(new Widget, delPtr);
```

## 創建智慧型指標時傳入創建物件參數使用std::initializer_list
當創建智慧型指標時，創建物件時需要使用參數使物件初始化，std::make相關函式會對這些參數進行完美轉發，但是若要使用轉發功能，就必須使用小括號語法，大括號初始化是不合法的。

e.g.
```cpp
auto up1 = std::make_unique<std::vector<int>>(3, 2);.
//auto up2 = std::make_unique<std::vector<int>>{3, 2};    // error

auto sp1 = std::make_shared<std::vector<int>>(3, 2);.
//auto sp2 = std::make_shared<std::vector<int>>{3, 2};    // error
```

在這種情況，就只能建立出元素值皆相同的std::vector，但是若我們想要創建指向內容為{3, 2}的std::vector，使用std::make相關函式是沒辦法直接做到的

e.g.
```cpp
std::unique_ptr<std::vector<int>> up(new std::vector<int>{3, 2});    // ok
std::shared_ptr<std::vector<int>> sp(new std::vector<int>{3, 2});    // ok
```

但是還是有迂迴的做法，雖然無法直接藉由大括號作初始化，但是可以先創建出std::initializer_list後以其為智慧型指標初始化。

e.g.
```cpp
auto initlist = {3, 2};
auto up = std::make_unique<std::vector<int>>(initlist);    // ok
auto sp = std::make_shared<std::vector<int>>(initlist);    // ok
```

而作者還列出了兩個比較冷門的需要使用new的情況，但因為不常碰到，這裡就先省略了。