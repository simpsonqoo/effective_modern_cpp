lambda表達式在C++11後推出，但其實推出lambda表達式並沒有提供新的功能，相同的功能只需要多打幾個字就可以用以前的方法達成。不過因為有了lambda表達式，STL的_if演算法就可以不再只能提供最簡單的判斷，變得可以提供較為複雜的判斷模式。而同樣的也發生在一些改變容器的演算法上(e.g. std::sort、std::nth_element...)。

以下是容易混淆的lambda相關術語
1. lambda表達式(lambda expression)：就只是個表達式而已

e.g.
```cpp
std::vector<int> v{5,1,2,6};
std::sort(v.begin(), v.end(), "[](int n1, int n2){return n1 > n2;}");
```

""內的程式碼就是lambda表達式

2. closure：closure是lambda建立的執行期物件，根據不同的擷取及行為就會產生不同型別的物件，像是上例的closure物件就做為std::sort的第三個引數傳入。

3. closure類別：實體化closure的類別

明確的區分以上三者其實在大多數場合都不重要，大概知道有以上幾個名詞就好。