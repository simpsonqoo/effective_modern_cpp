通常來說，數值和指標應該是兩個不同的概念，但是很多時候，NULL和0都可以代表0或是空指標。但是nullptr明確定義是指標，而且其型別是所有型別之空指標。

# 對於使用者而言
## 不會發生ambigous
對於0而言，compiler會優先希望其為數值，真的不行才會勉為其難視為指標，但是NULL則是都可以。

e.g.
```cpp
void f(int a);
void f(bool a);
void f(void* a);

f(0);       // 呼叫f(int)
f(NULL);    // 發生歧義，無法判斷為f(int)、f(bool)或是f(void*)
```

不過，通常使用者使用NULL時都是希望使用f(void*)，所以通常，設計者常被要求不能同時提供int和void*的過載，就是為了避免使用者使用NULL。但是如果使用者能使用nullptr，就不會有這種情況發生。

```cpp
f(nullptr); //呼叫 f(void*)
```

## 明確得知行為
當使用auto時，常常無法直接得知物件型別，而如果發生在NULL上時，更無法知道設計者設計時是期望為0或是空指標。

e.g.
```cpp
auto result = doSomething(參數);

if (result == 0)     // 期望result是0或是空指標?
...
```

但如果使用nullptr，就可以很明確知道設計者的期望

e.g.
```cpp
auto result = doSomething(參數);

if (result == nullptr)     // 期望result是空指標
...
```

# 對於設計者而言
## 可以避免非預期狀況發生
假設設計者期望傳入的是指標型別，不允許其他像是0、NULL之類的物件傳入的話，可以先使用一個template function將該型別固定(int)，在使用該物件，compiler就不會將int轉為指標。由於0或是NULL在template推導時其最佳型別都不會是指標，故使用0或是NULL做為參數都會發生錯誤。

e.g.
```cpp
void f(void*);

template <typename T>
void g(T t)
{
    f(t);
}

int main()
{
    g(nullptr); // ok
    g(0);       // compiler error
    g(NULL);    // compiler error
}
```

g(0)錯誤為
```
error: invalid conversion from 'int' to 'void*'
```

g(NULL)錯誤為
```
error: invalid conversion from 'long long int' to 'void*'
```

至此我們也可以知道，0或是NULL都不是表示空指標的最佳表示法，其都還需要做型別轉換後才會是空指標，故使用nullptr來表示空指標是最佳解。