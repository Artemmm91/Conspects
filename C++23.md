## 13.2 shared_ptr

Можно копировать их, и они все будут указывать на один и тот же объект. И вызывается деструктор, когда никто на объект не указывает.

```cpp
template <typename T>
class shared_ptr {
private:
    T* ptr = nullptr;
    size_t* counter = nullptr;
public:
    shared_ptr(T* ptr): ptr(ptr), counter(new size_t(1)) {}
    
    shared_ptr(const shared_ptr& other):
        ptr(other.ptr), count(other.counter) {
        ++(*counter);
    }
    
    shared_ptr(shared_ptr&& other):
        ptr(other.ptr), counter(other.counter) {
        other.ptr = nullptr;
        other.counter = nullptr;
    }
    
    // operator equality, same
    
    T& operator*() const {
        return *ptr;
    }
    
    T* operator->() const {
        return ptr;
    }
    
    size_t use_count() const {
        return *counter;
    }
    
    ~shared_ptr() {
        if (*counter > 1) {
            --(*counter);
            return;
        }
        delete ptr;
        delete counter;
    }
};

int main() {

}
```

В первом приближении так. Но это плохая реализация. 

## 13.3 make_shared

```cpp
template <typename T>
class shared_ptr {
private:
    template <typename U, typename... Args>
    friend shared_ptr<U> make_shared(Args&&... args);
    
    struct make_shared_t;
    
    template <typename U>
    struct ControlBlock {
        size_t counter;
        U object;
    }
    
    ControlBlock<T>*cptr = nullptr;
    
    template <typename... Args>
    shared_ptr(make_shared_t, ControlBlock<T>* storage, Args&&... args):
        cptr(storage) {}
}

temaplte <typename T, typename... Args>
shared_ptr<T> make_shared(Args&&... args) {
    auto ptr = new ControlBlock<T>{1, T(std::forward<Args>(args)...)}
    return shared_ptr<T>(shared_ptr::make_shared_t(), ptr);
}

int main() {
    auto sp = make_shared<S>(5, "abc"); // S - some struct
}
```

Преимущества этой функции - не надо дублировать название при создании, теперь new в программе сосвем писать не надо, а также из-за того что укзаатели лежат рядом, обращение к памяти эффективней.

-----

```cpp
temaplte <typename T, typename... Args>
unique_ptr<T> make_unique(Args&&... args) {
    return unique_ptr<T>(new T(std::forward<Args>(args)...));
}

int f(int x) {
    // ... 
    return result;
}

void g(const unique_ptr<int>& ptr, int x) {
    // ...
}

int main() {
    g(unique_ptr<int>(new int(5)), f(1));
    g(make_unique<int>(5), f(1));
}
```

Что может потйи не так? Порядок вызовов функций не гарантирован, поэтому если f бросает исключения, а в это же время исполняется создание unique_ptr, то delete никто не вызовет и будет утечка памяти.
Поэтому надо делать make_unique. 

## 13.4 weak_ptr

Какая есть проблема? Если есть зацикленная цепочка объектов, то при уничтожении последнего shared_ptr на него, они не будут уничтожаться, потому что видят что нужны кому-то еще. И получается, что у нас нет доступа к объектам которые не удаляются.

Сделали weak_ptr - указатель, который не влияет на то будут ли hsared_ptr уничтожать объект. Можно у него спросить сколько shared_ptr укзаывается, и жив ли объект. Но уничтожение weak_ptr ни на что не вляиет.

Это решает проблему циклических зависимостей.


```cpp
// weak_ptr - friend shared_ptr
// считаем что shared_ptr на контрольных блоках написан

template <typename T>
class weak_ptr {
    ControlBlock<T>* cptr = nullptr;
public:
    weak_ptr(const shared_ptr<T>& ptr):
        cptr(ptr.cptr) {}
    
    bool expired() const {
       // return cptr->counter > 0; // проблема потому что вначале
    }
    
    shared_ptr<T> lock() const {
        
    }
};
```

```cpp
template <typename T>
class shared_ptr {
private:
    template <typename U>
    friend weak_ptr<U>;
    
    template <typename U>
    struct ControlBlock {
        size_t shared_count;
        size_t weak_count;
        U object;
    }
public:
    ~shared_ptr() {
        --cptr->shared_count;
    
        if (cptr->shared_count > 0) {
            return;
        }
        if (cptr->weak_count == 0) {
            delete cptr;
            return
        }
        
        cptr->object.~T();
    }
};

template <typename T>
class weak_ptr {
public:
    ~weak_ptr() {
        --cptr->weak_count;
        if (cptr->shared_count == 0 && cptr->weak_count == 0) {
            // deallocate
        }
    }

};
```

Надо сделать по итогу так: потому что weak_ptr не должен удалять объект, но должен освобождать память. То есть ControlBlock живет до того момента пока вообще есть хоть какой-то укзаатель.


## 13.5 allocate_shared

shared_ptr нужен ControlBlock - он выделяет память, поэтому ему нужно уметь выделять память на нестандратном аллокаторе. 

```cpp
template <typename T, typename... Args>
shared_ptr<T> make_shared(Args&&... args) {
    return allocate_shared(std::allocator<T>(), std::forward<Args>(args)...);
}

template <typename T, typename Alloc, typename... Args>
shared_ptr<T> allocate_shared(Alloc& alloc, Args&&... args) {
    ...
}
```

То есть allocate_shared должен аллоцировать память. При этом нужно в контрольном блоке нужно хранить указатель на аллокатор, чтобы потом освобождать его. Потом надо доставать его, сделать rebind, и освобождать память. Тут есть некоторые проблемы, к которым мы потом вернемся.


## 13.6 enable_shared_from_this. CRTP

CRTP - Curiously Recursive Template Pattern.
Хотим сделать shared_ptr на себя (структуру), но тут проблема - что если два раза сделать от this - то будут два shared ptr от одного укзаателя, что неправильно (тк будет UB при удалении)

```cpp
struct S: public enable_shared_from_this<S> {
    int x = 0;
    std::string str;
    
    S(int x, const std::string& str): x(x), str(str) {}
    
    shared_ptr<S> get_ptr() const {
        return shared_from_this();
    }
}
```

Наследование может показать невозможным, но по факту рекурсии там нет, просто происходит подстановка. И там есть метод shared_from_this.

```cpp
template <typename T>
class enable_shared_from_this {
private:
    weak_ptr<T> wptr;
    
    template <typename U>
    friend class shared_ptr;
public:
    shared_ptr<T> shared_from_this() {
        return wptr.lock();
    }
};
```

Но нужно получать shared_ptr, как это сделать? Надо поменять конструктор:

```cpp
shared_ptr(T* ptr):
        ptr(ptr), counter(new size_t(1)) {
    if constexpr (std::is_base_of_v<enable_shared_from_this<T>, T>) {
        ptr->wptr = /*...*/; // конструктор от контрольного блока
    }
}
```
Полностью реализуем позже. Когда будет сделан shared_ptr, то тогда wptr будет инициализирован.

# XIV. Vocabulary types and type erasure idiom.

pair, tuple, variant, any, optional

optional - в нем либо лежит T либо ничего не лежит
Например используется в функциях, которые могут ничего не возвращать, либо сделать поле в классе - которое либо есть либо нет

varinat - позволяет хранить чтото из списка - динамически меняется что в нем лежит

any - можно класть что угодно

## 14.1 Unions

```cpp
struct S {
    int i;
    double;
    std::string str;
}

union U {
    int i;
    int ii;
    double d;
    std::string str;
};
```

В Union может лежать чтото одно из этого, и размер его - максимум из размеров полей.

```cpp
int main() {
    U u;
    u.i = 5;
    std::cout << u.ii << '\n'; // 5
}
```
А здесь будет сегфолт, потому что конструктора не было
```cpp
int main() {
    U u;
    u.str = "abc"; // SF
    new (&u.str) std::string("abc");
    std::cout << u.str.size();
    u.str[2] = 'd';
    std::cout << u.str << '\n';
    u.d = 5.0;
    std::cout << u.i << '\n'; // но это UB
}
```

Есть понятие - active Union member - то куда последняя запись была, только к нему обращение не UB

Проблема работы с union - что все объекты надо создавать самим и уничтожать самим - строки надо явно уничтожать например. Union не могут наследоваться, но могут быть шаблонными.   

Но в C++17 сделали адекватную замену - variant. 

## 14.2 satd::variant and its implementaton

Как сделать tuple и union?

```cpp
tempalte <Head, Tail...>
class tuple {
    Head head;
    tuple<Tail...> tail;
}

template <Head, Tail...>
class optional {
    temaplte <Head, Tail...>
    union VarUnion {
        Head head;
        VarUnion<Tail...> tail;
    };
    
    VarUnion<Head, Tail...> storage;
}
```






