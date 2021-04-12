## 14.2 std::variant and its implementation

```cpp
#include <variant>

int main() {
    std::variant<int, double, std::string> v = 1;
    std::cout << std::get<int>(v); // 1
    std::cout << std::get<double>(); // exception
    v = "abc";
    std::cout << std::get<std::string>(v);  // "abc"
    v = 5.0;
    std::cout << std::holds_alternative<double>(v); // true
    std::cout << v.index(); // 1
}

```

```cpp

template <size_t N, typename T, typename... Tail>
struct get_index_by_type {
    static const size_t value N;
};

template <size_t N, typename T, typename Head, typename... Tail>
struct get_index_by_type {
    static const size_t value = std::is_same_v<T, Head> ? N:
            get_index_by_type<N + 1, T, Tails...>::value;
    
};

template <typename... Types>
class variant;

template <typename T, typename... Types>
struct VariantAlternative {
    // CRTP
    using Derived = variant<Types...>;
    
    VariantAlternative(const T& value) {
        static_cast<Derived*>(this).storage.put<sizeof...(Types)>(value);
    }
    
    VariantAlternative(T&. value) {
        static_cast<Derived*>(this).storage.put<sizeof...(Types)>(std::move(value));
    }
    
    void destroy() {
        auto this_ptr = static_cast<Derived*>(this);
        if (get_index_by_type<N, T, Types...?::value == this_ptr->current) {
            // this_ptr->storage.destroy 
        }
    }
    // определения надо вынести вниз, после variant
};


template <typename... Types>
class variant: private VariantAlternative<Types, Types...>... {
private:
    template <typename... TTypes>
    union VariadicUnion {};

    temaplte <typename Head, typename... Tails>
    union VariadicUnion<Head, Tail...> {
        Head head;
        VariadicUnion<Tail...> tail;
        
        tempalte <size_t N, typename T>
        void put(const T& value) {
            if constexpr (N == 0) {
                new (&head) T(value);
            } else {
                tail.put<N - 1>(value);
            }
        }
    };
    
    VariadicUnion<Types...> storage;
    size_t current = 0;
public:
    using VariantAlternative<Types, Types...>::VariantAlternative...;
    
    /*
    template <typename T>
    variant(const T& value) {
        // static_assert(T is one of types)
        current = get_index_by_type<0, T, Types...>::value;
        storage.put<get_index_by_type<0, T, Types...>::value>(value);
    }
    */

    size_t index() const {
        return current;
    }
    
    template <typename T>
    bool holds_alternative() const {
        return current == get_index_by_type<0, T, Types...>::value;
    }
    
    ~variant {
        (VariantAlternative<Types, Types...>::destroy(), ...);
    }
};
```
Конструктор можно сделать либо через шаблонный конструктор с проверками. Либо можно отнаследвоаться от такой штуки `VariantAlternative<Types, Types...>...` - то есть это N родителей с типами <T_i, Types>. При этом сам VariantAlternative - тоже является variant. И когда мы пишем using... то мы подключаем конустркторы родителя а именно конструкторы VariantAlternative и от нужных типов как раз.


Деструктор тоже можно сделать в классе "родителя". 

## 14.3 std::launder

```cpp
struct S {
    const int x;
};

union U {
    S s;
    float f;
    U() {}
};

int main() {
    U u;
    
    new(&u.s) S{1};
    std::cout << u.s.x << "\n"; // 1
    new(&u.s) S{2};
    std::cout << u.s.x << "\n"; // может быть 1
}
```

Тут может быть 1, потому что он видит что в структуре S имеет типо const int, поэтому он может быть захардкожен, потому что использование placement_new это на самом деле UB, мы с памятью все-таки имеем дело. То есть у компилятора могли быть знания о предыдущем объекте, а placement_new может не обновить его - он просто в памяти создает.  

Такая проблема как раз может возникнуть в variant. И для этого ввели новую функцию - launder.

В примере нужно поменять placement_new:
```cpp
struct S {
    const int x;
};

int main() {
    S* s = new S{1};
    std::cout << s->x;
    new(s) S{2};
    std::cout << std::launder(s)->x;
}
```
Суть в том что когда так пишешь, это явно говорит компилятору забыть все что компилятор знает про этот указатель. 

## 14.4 std::any

```cpp
#include <any>

int main() {
    std::any a = 5;
    std::cout << std::any_cast<int>(a); // 5
    std::cout << std::any_cast<double>(a); // Exception
    
    a.type(); // std::type_info
    std::cout << a.type().name();
    
    a = "abcde";
    // std::any_cast<std::string>(a);
    std::any_cast<const char*>(a);
}
```

Как же any можно реализовать?

```cpp
class any {
private:
    void* storage;
    
    struct Base {
        virtual Base* get_copy();
        virtual ~Base() {}
    };
    
    template <typename T>
    struct Derived: public Base {
        T value;
        
        Derived(const T& value): value(value) {}
        
        Base* get_copy() {
            return new Derived<T>(value);
        }
    };
    
    Base* storage = nullptr;
public:
    template <typename U>
    any(const U& value): storage(new Derived<U>(value)) {}
    
    ~any() {
        delete storage;
    }
    
    any(const any& a): storage(a.storage->get_copy()) {}
    
    tempalte <typename U>
    any& operator=(const U& value) {
        delete storage;
        storage = new Derived<U>(value);
    }
};
```

Проблема возникает в деструкторе - потмоу что надо разрушить тип предыдущий, но не понятно как достать его тип. 
Благодаря такому полиморфизму мы можем подменять один тип на другой, и все будет корректно вызываться. Такой приём называется идиома Type erasure.

## 14.5 shared_ptr deleter

С этим знанием вернемся к shared_ptr

```cpp
private:
    struct DeleterBase {
    virtual void operator()(T*);
        virtual ~DeleterBase() {}
    };
    
    template <typename U>
    struct DeleterDerived: public DeleterBase {
        U deleter;
        DeleterDerived(const U& deleter):deleter(deleter) {}
        void operator()(T* ptr) override {
            deleter(ptr);
        }
    };
    
    DeleterBase* deleter = nullptr;
public:
    ~shared_ptr() {
            --*counter;
            
    }
```
У shared_ptr может быть свой deleter - в случаях если мы хотим не удалять а чтото еще сделать например. А Аллокатор нужен на свои нужды, выделять указатели на контрольные блоки. 


## 14.6 std::optional

Реализация несложная - опять делаем storage лежит чтото или нет и bool. При конструировании - сделать new и не забыть launder.






