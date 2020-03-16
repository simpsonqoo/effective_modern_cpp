若一個物件被多個指標共同擁有，那就不能使用std::unique_ptr，C++11後為此推出std::shared_ptr，std::shared_ptr可讓多個智慧型指標共同擁有一個物件，並在最後一個指標不再使用後清除該物件。

# 實際操作行為
而std::shared_ptr是透過資源的參考計數來判斷是否還有指標使用該物件，該計數顯示目前有多少指標指向該物件，std::shared_ptr的建構子(除了move constructor外)都會增加參考計數，而複製賦值操作子則會同時減少和增加參考計數(e.g. sp1 = sp2，原有sp1指向的物件其參考計數-1，而sp2指向的物件其參考計數+1)。而若使用std::move相關函式，則會將原有指標設成null，再指向新的std::unique_ptr，若是move constructor，則參考計數不會改變，而若是move assignment，原有物件參考計數-1，新指向物件參考計數不變。

## 參考計數
參考計數的存在會使得std::shared_ptr相較std::unique_ptr使用較多資源及效能

- std::shared_ptr是std::unique_ptr的兩倍大，因為實際上會使用兩個指標，一個指向物件，一個會指向控制區塊(control block)
- 參考計數採動態配置
- 必須以atomic方式增加減少參考計數，不能被平行化程式影響其值的正確性

# destructor
和std::unique_ptr一樣，std::shared_ptr也有自己預設的destructor，而一樣的也可以使用自行定義的desturctor，但是std::shared_ptr為了要接納多個指標，故放寬了destructor的限制，desturctor型別不會被認為是建立std::shared_ptr的template參數。

```cpp
auto delSharePtr = [](int* pw)
{
    delete pw;
    std::cout << "delete a share pointer" << std::endl;
};

std::unique_ptr<int, decltype(delSharePtr)> upw(new int(1), delSharePtr);

std::shared_ptr<int> spw(new int(1), delSharePtr);
```

而因為destructor並不是std::shared_ptr的一部分，所以使用不同destructor的std::shared_ptr可以互相賦值，也可以放入同一個容器內

e.g.
```cpp
auto delSharePtr1 = [](int* pw)
{
    delete pw;
    std::cout << "destructor version1" << std::endl;
};

auto delSharePtr2 = [](int* pw)
{
    delete pw;
    std::cout << "destructor version2" << std::endl;
};

std::shared_ptr<int> sp1(new int(1), delSharePtr1);
std::shared_ptr<int> sp2(new int(2), delSharePtr2);
sp1 = sp2;

std::vector<std::shared_ptr<int>> vec{sp1, sp2};
```

而std::shared_ptr還有一點和std::unique_ptr不同的是，無論使用甚麼destructor，都不會影響其大小。這是因為其使用control block來儲存std::shared_ptr相關的訊息，這並不在std::shared_ptr的物件內，而是儲存在heap中，std::shared_ptr的第二個指標指向該control block。control block內包含參考計數、自行定義destructor、heap相關的allocator...。

![](https://i.imgur.com/U9xuf3t.png)

理想而言，std::shared_ptr的control block應該是由指向物件的第一個std::shared_ptr所建立，但是std::shared_ptr無從得知其他std::shared_ptr的訊息，所以不太可能知道自己是否是該物件的第一個std::shared_ptr，所以會根據以下規則決定是否要建立control block

- std::make_shared一定會建立control block
由於std::make_shared就是建立新的物件作為指標標的的，所以一定會建立control block
- 從std::unique_ptr轉換為std::shared_ptr時一定會建立control block
由於std::unique_ptr並沒有control block，而且轉換為std::shared_ptr時必定只有該指標指向該物件，所以一定會建立control block
- 由原始指標建立std::shared_ptr時會建立control block

而最後一項規則是最危險、最容易出錯的規則，由於每由原始指標建立std::shared_ptr就會建立新的control block，很有可能會以一個原始指標建立多個std::shared_ptr時建立多個control block，程式的行為上會認為兩個std::shared_ptr指向兩個不同的物件，但事實上這兩個std::shared_ptr都指向相同的物件

e.g.
```cpp
auto p = new Widget;

std::shared_ptr<Widget> sp1(p);

std::shared_ptr<Widget> sp2(p);    // 當sp2要執行destructor時，產生未定義行為
```

在建立std::shared_ptr時都沒有問題，但是當清除sp1後，p指向的物件已被清除，此時再清除sp2時會發現sp2指向的物件已經無法讀寫，造成未定義行為。

所以，我們不能輕易地將原始指標傳入std::shared_ptr內，我們可以一律使用std::make_shared，但是如此就無法使用自訂的destructor，如果我們需要使用自己定義的destructor，直接在建立原始指標的同時就傳入std::shared_ptr constructor，不要給該原始指標有可能會被重複利用的機會。

e.g.
```cpp
auto shared_ptr<Widget> sp3(new Widget, delWidget);    // delWidget為自定義的destructor
```

如果要再建立std::shared_ptr，就可以直接使用sp3傳入constructor，就能避免原始指標會發生的錯誤。

不過，有個我們很常使用但又是原始指標，那就是this，this在很多時候我們無法避免不去使用他，但是若將this傳入std::shared_ptr的construtor，就會產生新的control block。

e.g.
```cpp
class Widget
{
  public:
    ...
    void process();
    ...
};

std::vector<std::shared_ptr<Widget>> processWidgets;

void Widget::process()
{
    ...
    processWidgets.emplace_back(this);
    ...
}
```

以上例子是Widget建立一個function，裡面有一個行為是將自己的this指標push進vector中，但是如此便會在建立一個新的control block，當其他地方又建立一個指向該Widget的std::shared_ptr時，就會產生理論上應該是兩個使用同一個control block的std::shared_ptr，但實際上卻各自有自己的control block，其後會產生未定義行為。

所以為了避免這種情況，STL便又提供特別的API專門應付這種情況，便是std::enable_shared_from_this，會特別安全的處理this而不會產生新的control block，裡面的機制有些複雜，但如果只是要使用他，知道其行為即可，使用前須要將Widget物件設定為繼承std::enable_shared_from_this<Widget>的物件，使用上如下

```cpp
class Widget : public std::enable_shared_from_this<Widget>
{
  public:
    ...
    void process();
    ...
};

std::vector<std::shared_ptr<Widget>> processWidgets;

void Widget::process()
{
    ...
    processWidgets.emplace_back(std::shared_from_this());
    ...
}
```