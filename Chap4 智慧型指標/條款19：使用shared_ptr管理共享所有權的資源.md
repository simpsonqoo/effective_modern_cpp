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

