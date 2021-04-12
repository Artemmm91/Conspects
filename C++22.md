## 12.2 decltype

```cpp
template <typename T>
class C {
    C() = delete;
};

int main() {
    int x = 0;
    C<decltype(std::move(x))>();
    C<decltype(x++)>();
    C<decltype(++x)>();
}
```
Будет подставлен int&&, int, int& соответственно с правилами decltype.
А что будет если взять `decltype((x))`? Будет подставлено int&, так как скобки делают из этого выражение (которое lvalue), а без них бы просто int подставился. 

## 12.3 structured binding
Since c++17:
```cpp
std::pair<int, std::string> p{3, "abc"};
auto [key, value] = p;
```

Такой вывод работает, если есть массив, если для структуры применима агрегатная инициализация (вектор, с кастомными полями), и также с pair/tuple.

Можно делать еще так:
```cpp
auto& [x, y] = s;
const auto& [x, y] = s;
auto&& [x, y] = std::move(s);
for (auto& [key, value]: unordered_map_object) {}
```

## 12.4 std::common_type implementation

```cpp
template <typename Head, typename... Tail>
struct common_type {
    using type = common_type<Head, typename common_type<Tail...>::type>::type;
};

template <typename T, typename U>
struct common_type<T, U> {
    using type = decltype(true ? T() : U());
};


template <typename... Types>
using common_type_t = typename common_type<Types...>::type;

```

Возвращаемый тип будет наименьшим общим предком, потому что тип тернарного оператора - как раз нимаеньший общий тип. Но работает не идеально. не все случаи обрабатываются, и зависит от порядка еще. Но конструктора по умолчанию не существует. Чтобы исправить надо написать так:

```
template <typename T>
T f();

template <typename T, typename U>
struct common_type<T, U> {
    using type = decltype(true ? f<T>() : f<U>());
};
```

Тут будет все в порядке. Такую функцию сделали в стандарте: declval()
```cpp
template <typename T>
T&& declval() noexcept;
```

decltype: expression -> its type
declval: type -> expression of this type


Почему declval возвращает Т&&? Потому что если тип является неполным (то есть объект нельзя объявить - нет реализации или приватный деструктор), и нельзя написать что функция такой тип возвращает.

```cpp
template <typename T>
class BigClass {
    // dependency on T
};
```
Для такого класса тоже будет проблемно, потому что надо будет инстанцировтаь шаблон, а если возвращать T&& то можно не инстанцировать. Тогда итоговый common type:

```cpp
template <typename T, typename U>
struct common_type<T, U> {
    using type = std::remove_reference_t<decltype(true ? std::declval<T>() : std::declval<U>())>;
};
```

# XIII. Smart pointers.

## 13.1 unique_ptr

```cpp
void f(int x) {
    int* p = new int(5);
    if (x == 0) {
        throw 1;
    }
}
```

Проблема - хотим чтобы память освобождалась. Для этого нужны другие уазатели.

```cpp
template <typename T>
class unique_ptr {
private:
    T* ptr;
public:
    unique_ptr(T* ptr): ptr(ptr) {}
    
    unique_ptr(const unique_ptr&) = delete;
    unique_ptr& operator=(const unique_ptr&) = delete;
    
    unique_ptr(unique_ptr&& another): ptr(another.ptr) noexcept {
        another.ptr = nullptr;
    }
    unique_ptr& operator=(unique_ptr&&) noexcept {
        delete ptr;
        ptr = another.ptr;
        another.ptr = nullptr;
    }
    
    ~unique_ptr() {
        delete ptr;
    }
    
    T& operator*() const {
        return *ptr;
    }
    
    T* operator->() const {
        return ptr;
    }
};


void f(int x) {
    unique_ptr<int> p = new int(5);
    if (x == 0) {
        throw 1;
    }
}
```

