## 11.8 Perfect forwardung problem

```cpp
template <typename... Args>
void emplace_back(Args&&... args) {
    if (cap == sz) {
        reserve(2 * sz);
    }
    AllocTraits::construct(alloc, arr + sz, std::move(args)...);
    ++sz;
}
```

Проблема в том, что в такой реализации проблема - нам хочется чтобы rvalue делались через std::move, а lvalue мы не хотим мувать.   
Для этого есть следующая функция: `std::forward<Args>(args)....`. Если у Args тип с одним `&`, то это lvalue и forward не будет кастовать, а если без `&`, то это было rvalue, и нужно делать мув.

```cpp
template <typename T>
T&& forward(std::remove_reference_t<T>& x) {
    return static_cast<T&&>(x);
}
```
Возвращение универсальной ссылки будет работать как нам надо - либо навесится && либо победит &, так что получим что надо. А что по принимаемому значению?  

case of lvalue: T == type&, typeof(X) == type&, T&& == type&  
case of rvalue: T == type, typeof(X) == type&, T&& == type&&  

В конструкторе AllocTraits тоже нужно делать forward. А также в push_back таким же образом можно избавиться от копипасты. 

## 11.9 Definition and properties of xvalue

rvalue бывает двух типов: `xvalue`, `prvalue`  
А есть еще категория glvalue: `lvalue` и `xvalue`

`prvalue` - pure rvalue  
`xvalue` - expired value  
`glavlue` - generalized lvalue

**xvalue**:  
1. cast to rvalue-ref  
2. function call with rvalue-ref return type   
3. (?: ,) - результаты если xvalue где надо  
4. `.`, `->`, `[]`, `f(X).a`  

В чем отличие этих двух типов? `prvalue` - это сущность, которая может не иметь памяти даже, проходящее выражение, а `xvalue` - полноценный объект в памяти, но по свойствам оно rvalue  

Есть конверсия из lvalue в prvalue (считывание значения), а есть конверсия из prvalue в xvalue - temporary materialization.

`C c = C();` - copy ellision, объект будет создан только для одной части.  
Но: `int a = C().a;` - здесь должно быть приведение к xvalue - потому что происходит обращение к полю и нам надо создавать объект.  

`glvalue` - полимофрен: динамический типо может не совпадать со статическим.

```cpp
struct Base {
    virtual ~Base() {}
};

struct Derived : public Base {
};

int main() {
    Base b = Derived();
    
}

```
Справа не временная материализация и prvalue - значит собственная часть Derived не создается. Если он видит prvalue - значит он может не делать любых вещей связанных с полиморфизмом (но для любой операции потребующей поля и методы придется уже создавать и делать временну материализацию). 

## 11.10 Reference qualifiers

```cpp

struct Data {
    Data(const std::String& s): data(s) {}
    
    std::string getData() const {
        return data;
    }
    
    std::string. getData() {
        return data;
    }
    
private:
    std::string data;
};
```

Если написать по обычному - то будет даваться копия данных. Но что если иногда когда есть `f(Data&& x)` - то можно было забирать данные а не копировать. При этом мы умеем перегружать написав const после фукнции - в завиисмости от константности объекта.  

Такая же вешь есть и для rvalue/lvalue - по умолчанию метод можно как для rvalue так и для lvalue сделать. А если написать `f() (const) & {}`, то функция будет работать только для lvalue значений. То есть теперь решения по левому операнду принимается.   

То есть если написать `std::string getData() & {...}; std::move(x).getData();`, то будет CE, а если `const &` написать, то не будет CE, потому что можно биндить константную lvalue-ref к rvalue. 

```cpp
std::string getData() & {
    return data;
}
    
std::string getData() && {
    return std::move(data);
}
```

