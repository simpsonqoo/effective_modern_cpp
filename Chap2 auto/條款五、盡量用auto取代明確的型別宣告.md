主要有三點
1. 強迫一定要初始化
2. 可避免要打出下列的型別名稱
    - 複雜且冗長的型別名稱
    - 只有compier才知道，使用者根本不知道的型別
3. 可以直接保存closure，避免之前使用C++11的std::function所造成不必要的複製

## 強迫一定要初始化
在沒有auto出現時，下列這種為初始化的情形是合法的

e.g.
```cpp
int a;
```

此時a值並不確定，不同的compiler可能會有不同的結果，雖然通常而言int未定義時的初始值都是0，但是誰也沒辦法保證。

## 可避免要打出下列的型別名稱
假設我們想以iterator的方式存取vector中的所有元素，可以向下列方式使用

```cpp
std::vector<int> vec(100);
for (std::vector<int>::iterator it = vec.begin(); it != bev.end(); ++it)
{
    ...
}
```

在沒有auto之前，我們必須明確打出每個物件的型別名，像是這裡的std::vector<int>::iterator，我們必須背起來或是特別去查才找的到，但有了auto後，一切就變得簡單

```cpp
std::vector<int> vec(100);
for (auto it = vec.begin(); it != bev.end(); ++it)
{
    ...
}
```