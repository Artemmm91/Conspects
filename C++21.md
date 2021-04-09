## 11.11 std::move_if_noexcept

Раньше в reserve делали `std::uninitialized_copy`, теперь знаем что это необязательно. И раз мы сделали `std::move`, то надо деструктор тех элементов вызвать, потому что они остались в валидном состоянии (но пустые). И также надо все разрушить обратно, если в какой-то момент словили исключение.

```cpp
// in reserve
size_t i = 0;
try {
    for (; i < sz; ++i) {
        AllocTraits::construct(alloc, newarr + i, std::move_if_noexcept(arr[i]));
    }
} catch(...) {
    for (size_t j = 0; j < i; ++j) {
        AllocTraits::destroy(alloc, newarr + i);
    }
    AllocTraits::deallocate(alloc, newarr, n);
}
```

Почему используем `std::move_if_noexcept` а не `std::move`? Потому что могут быть исключения.

```cpp
temaplte <typename T>
std::conditional_t<noexcept(T(std::move(T()))), T&&, const T&> move_if_noexcept(T& x) noexcept {
    return std::move(x);
}
```

Так как `std::move(x)` rvalue, a `T&` это lvalue, поэтому надо добавлять const. Мы принимаем `T&`, так как принимаем только lvalue - нас интересует этот случай, для существующего объекта может std::move пойти не так. Условие такое - потому что нам нужно проверить является ли мув-конструтор от T() noexcept.   
В этой реализации есть проблема, что может не быть move-конструктора, или по умолчанию - и должно быть const T&. Это уже сложно. Например может решаться так:

```cpp
template <typename T>
T f() noexcept;

std::conditional_t<noexcept(T(f<T>())), T&&, const T&> ...
```

Теперь будет работать лучше, потому что не зависит от конструктора по умолчанию. Проблема еще остается, если move-конструктор удален. Пока с этим разбираться не будем. 

# XII. Type deduction

## 12.1 auto keyword

auto стоит писать, хотя бы потому что не будет дублирования сущности, и меньше вероятности ошибки. Можно писать так `for (auto& p: m) {}`.  

auto ведет себя как T в шаблонах, auto тоже отбрасывает ссылки.

Также можно написать `auto && x = y;`, то это считается универсальной ссылкой (второй способ) - здесь x будет lvalue ссылкой. Если написать `auto && x = 5;` то это будет rvalue ссылка. А например `const auto && x = y;` бы не скопилировалось - потому что теперь это не универсальная ссылка, а константная rvalue ссылка, а значит ей нельзя присваивать lvalue.  


```cpp
tempalte <typename T, typename U>
auto& f(const T& x, const U& y) {
    return x < y ? x : y;
}

f('a', 5);
```

Такой вызов не компилитсья, потому что он приводит для сравнения к общему типу - int - а когда char приведут к int сделается временный int - rvalue - значит и весь тернарный - rvalue - ну он не сбиндится к auto&

## 12.2 decltype (declared type)

```cpp
int x = 5;
decltype(x) y = x;
```
Он берёт точный тип того, что написано.

```cpp
int x = 5;
int& r = x;
decltype(r) y = x;
```
И в этом примере, y будет еще одной ссылкой на x.  Это единстенное различие между ссылкой и не ссылкой - где они по разному ведут себя.

```cpp
int&&x = 5;
decltype(x) y = x;
```

Здесь будет СЕ, потому что пытаемся проинициализировать rvalue ссылку lvalue значением.  

```cpp
template <typename Container>
... get_at(Container& container, size_t index) { 
    return container[index];
}

std::vector<int> v(10);
get_at(v, 5) = 1;
```

Возвращаемый тип не может быть auto (так как ссылка уйдет) и auto& (потому что иногда возвращается не ссылка - при vector<bool>). Есть решение:

```cpp
auto get_at(Container& container, size_t index) -> decltype(container[index]) {
    return container[index];
}
```

Это называется явное указание используемого типа. При этом выражение не вычисляется, смотрятся только типы (как и под `noexcept`, `sizeof`)!

Начиная с C++14 работает следующее:

```cpp
decltype(auto) get_at(Container& container, size_t index) {
    return container[index];
}
```

Теперь нет копипасты, и выглядит лучше.

С decltype можно делать все модификаторы типа `*, &, const`.
Например `decltype(throw 1)* y = &x;` у `y` будет тип void*.

Что происходит при decltype(expression)? Если expression lvalue типа T, то будет возвращен тип T&. Если expression это prvalue, то возвращает T. А если это xvalue - то возвращается T&&. 
