而還有一種指標的需求是既需要std::shared_ptr的行為，但卻又不共享其指向資源的所有權，其只有在確定需要物件資源的時候才會去確認其物件狀況，需要的時後再轉成std::shared_ptr共享資源。而這正是std::weak_ptr的用途。

std::weak_ptr不能解參考，不能測試是否為nullptr，std::weak_ptr並不是一個獨立的智慧型指標，其功能為std::shared_ptr的擴充。

當std::weak_ptr以std::shared_ptr初始化時，兩者均指向相同的物件，但std::weak_ptr並不會增加標的物件的參考計數。

一旦所有的shared_ptr都被清除，則物件也會隨之被清除，此時weak_ptr成為懸置指標，稱為過期(expired)，可用其member function: expired()檢驗。

e.g.
```cpp
auto sp(std::make_shared<Widget>());

std::weak_ptr wp(sp);

sp = nullptr;

wp.expired();
```

而指標的一個重點就在於可以存取指向的物件，std::weak_ptr並沒辦法解參考，但是其能夠為std::shared_ptr初始化，但這過程中必須保證是atomic operation，因為若檢查完weak_ptr指向的物件未過期，但是在初始化std::shared_ptr前該物件已經被清除，則建立的std::shared_ptr就成為未定義行為。
有兩種方式可以由std::weak_ptr安全的建立std::shared_ptr
1. lock()，保證檢查過期狀態及為std::shared_ptr初始化在一個atomic operation完成，而若weak_ptr過期，則shared_ptr為nullptr<br>
e.g.
```cpp
auto sp1 = wp.lock();    // 等同於std::shared_ptr sp1(wp.lock());
```
2. 直接藉由std::weak_ptr初始化std::shared_ptr，一樣是一個atomic operation可完成，但若weak_ptr過期，則會拋出例外

"只作為檢查是否過期，無法得知物件行為"這件事情看似沒甚麼用途，這裡舉幾個有機會使用到的時機

1. 存取資料庫、檔案庫中其中一筆資料，輸入為ID，輸出為該筆資料的指標
形式為
```cpp
std::unique_ptr<const Widget> loadWidget(WidgetId id);
```
假設資料庫、檔案庫非常龐大，若每次藉由ID尋找檔案都得從頭找起，速度太慢。如果cache能夠將最近幾次使用的檔案存起來，之後在找尋這些檔案時就會快速許多。不過，存在快取時我們不應該干涉該檔案的運作及生命週期，應該只有在需要的時候檢查其狀態，若該檔案是可以被讀取的，則存取，若否，在根據使用方式決定其行為，所以std::weak_ptr較std::unique_ptr是更好的選擇。<br>
e.g.
```cpp
std::shared_ptr<const Widget> fastLoadWidget(WidgetID id)
{
    static std::unoreder_map<WidgetID, std::weak_ptr<Widget>> cache;
    
    auto objPtr = cache[id].lock();    // 若物件不在cache中，則objPtr為nullptr
    
    if (objPtr != nullptr)
    {
        objPtr = loadWidget(id);    // 重頭找尋檔案方式
        cache[id] = objPtr;
    }
    
    return objPtr;
}
```

2. observer模式
observer模式主要由subject(會改變狀態的物件)及observer(物件改變時需要通知的對象)所構成。對於subject來說，其在發送通知前不需要知道observer的任何資訊也不需要控制其週期，只需要在改變自身狀態時，檢查observer物件是否存在，若存在則發送通知。
所以合理的設計是創在一個subject指向observer的std::weak_ptr，在通知observer之前先得知該指標是否為懸置狀態。

3. 避免發生循環std::shared_ptr
假設有A、B、C三個Widget，而A、C皆有std::shared_ptr指向B，此時，若需要有個指標由B指向A，使用哪種指標會比較好呢？
![](https://i.imgur.com/ZKovXDD.png)

1. 原始指標：當然是最差的選擇，因為就算A被清除，仍然有指標由B指向A，此時就會產生一個懸置指標
2. std::shared_ptr：會產生A指向B而B又指向A的share_ptr循環，若C先被清除後，除了A、B本身之外，再也沒有別的指標能夠找到這兩個物件，照理來說，應該要清除這兩個物件，但是卻因為彼此只向對方，導致參考計數皆為1，無法被清除，形成memory leak，又稱為資源洩漏，也就是明明就已經沒有在使用該memory，但卻因為被佔有而無法使用。
3. std::weak_ptr：能避免懸置指標的問題，因為當A被清除時，std::weak_ptr為過期狀態，使用前會先作檢查，無法直接解參考。而也不會有循環shared_ptr的問題，因為若A有shared_ptr作為標的，當不再有shared_ptr指向A時，還是會照樣清除A，因為weak_ptr並不會增加參考計數。

不過第三種狀況其實並不常見，像是常使用指標的樹狀結構，是不可能會形成cycle的，所有的子代節點生命週期絕對不可能長過父節點，當有這種階級關係時，使用std::unique_ptr就已經很足夠了。

最後，在效率方面，std::weak_ptr和std::shared_ptr是差不多的，其都含有兩個指標，一個指向物件，一個指向control block，且一樣必須要一個atomic operation內完成建構、解構、賦值等操作。而std::weak_ptr也會有自己的參考計數，只是和std::shared_ptr不相同，會有單獨給std::weak_ptr使用的參考計數，條款21會對其有詳細解釋。