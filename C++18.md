## 10.8 Scoped allocators

```cpp
int main() {
    std::vector<std::string, MyAllocator> v;
}
```

Таким образом вектор выделятся будет своим аллокатором, а строки стандартным. Если хотим свой аллокатор и для элементов вектора:

```cpp
int main() {
    using MyVector = std::vector<int, MyAllocator<int>>;
  
    std::vector<MyVector, MyAllocator<MyVector>> v;
    v.push_back(MyVector(1));
}
```
------


```cpp
int main() {
    using MyVector = std::vector<int, MyAllocator<int>>;
    
    MyAllocator<MyVector> outer_alloc;
    MyAllocator<int> inner_alloc(outer_alloc);
    
    std::vector<MyVector, MyAllocator<MyVector>> v(inner_alloc);
    
    v.push_back(MyVector(1));
}
```

А теперь у нас есть контейнер из контейнеров. Но мы хотим чтобы элементы внутреннего контейнера создавалася внешним аллокаторо - чтобы например в тот же пулл их клали.  
Так как написано не получится - придется каждый раз передавать нужный аллокатор.  
Для этого нужен класс `scpoed_allocator_adaptor`


```cpp
#include <scoped_allocator>

int main() {
    using MyVector = std::vector<int, MyAllocator<int>>;
    
    MyAllocator<MyVector> outer_alloc;
    //MyAllocator<int> inner_alloc(outer_alloc);
    
    std::Vector<MyVector, std::scpoed_allocator_adaptor<
        MyAllocator<MyVector>, MyAllocator<int>>> v(outer_alloc);
    
    std::vector<MyVector, MyAllocator<MyVector>> v(inner_alloc);
    
    v.push_back(MyVector(1));
}
```
В scoped_allocator передаем вначале внегний аллокатор, потом внутренний. И создаем вектор от внешнего аллокатора. Как только кладем в вектор элементы - будет сконструирован аллокатором, который будет копией внешнего - то есть из внутреннего создаст внешний.  
У scoped_allocator есть функция construct - 

```cpp
temaplte<typename OuterAlloc, typename... InnerAlloc>
struct scoped_allocator_adaptor: public OuterAlloc {
private:
    scoped_allocator_adaptor<InnerAlloc..., inner_alloc;
public:
    template <typename T, typename... Args>
    void constructor(T* ptr, const Args&... args) {
        if (/* elemets of type T can be created with instance of InnerAlloc */) {
            new(ptr) T(Args..., inner_aloc);
        } else {
            new(ptr) T(args...);
        }
    }
};
```
Здесь inner_alloc - аллокатор предназначенный для внутреннего контейнера, но который стал копией внешнего аллокатора на каком-то этапе. 

В C++17 придумали - polymorphic_allocator - которому не нужен шаблонный тип, он общий для всех (в прицнипе так было бы лучше). А аллокатор создается из memory_resource (напрмиер пул памяти). А дальше он уже сам решает какой тип нужен.

## 10.9 Alignments

Alignment - минимальная степень двойки, такая что только с адресов кратных этой степени двойки можно класть объект этого типа - выравнивание.

Выравнивание структуруы - выравнивание наибольшого поля 

```cpp
struct S {
    int x;
    double y;
    int z;
}

int main() {
    std::cout << sizeof(S);
}
```
Будет 24 байта: 4 на инт, 4 на выравнивание дабла, 8 на дабл, 4 на инт и 4 на выравнивание структуры (чтобы заканчивалась делящимся на 8).  

`std::cout << alignof(S);` - как раз минимальная степень двойки, чтобы положить. Здесь 8 (потому что дабл).

Можно написать:

```cpp
struct alignas(8) S {
    int x;
    int y;
};
```

Проблема может быть такая: когда в аллокаторе вызывается `new(count * sizeof(T))` может быть не учтено выравнивание. Правда malloc умная функция и выделит с учетом выравнивания для стнадартных типо (обычно это 8 байт, но мб и больше) - `std::max_align_t`. 

Но проблема может быть если мы собственноручно сделали alignas больше чем максимальный в стандарте, и тогда будет UB.  

Есть `aligned_alloc` - в который выравнивание: `aligned_alloc(alignof(S), n)` - в операторе new. 

Это бывает нужно если есть мощный комп и за одну процессорную инструкцию можно сделать действие - за раз 32байтное число, или сделать операцию сразу с 4 8-байтными числами. Или наоборот для char можно ходить выравнивание 1. 

# XI. Move-semantics and rvalue-references.

## 11.1 Introduction, problem cases

```cpp
template <typename T>
void swap(T& x, T& y) {
    T t = x;
    x = y;
    y = t;
}

int main() {
    std::vector<std::string> v;
    
    v.push_back(std::string("abcd"));
}

```

Что плохого в этом swap? Будет 3 копирования, и если типы большие, то это очень долго будет.  

Или во втором примере происходит 2 создания: при создании и при push_back мы будем создавать копию. Как можно было бы решить пробему? Метод `emplace_back()`, который рабоатет без коипрований.

```
tempalte <typename... Args>
void emplace_back(const Args&... args) {
    ...
    new (arr+sz) T(args...);
    ...
}
```

В случае вектора от строк это было бы лучше - string создавался бы 1 раз из строки, а не 2. 

А если напсиать так:

```cpp
int main() {
    std::vector<std::vector<std::string>> v;
    v.emplace_back(3, "abc");
}
```

Было бы 4 копирования: один раз создаем string и 3 раза в векторе создадим.  
В общем-то проблема осталась, но ушла на уровень ниже. 

А еще при реаллоцировании в векторе постоянно просиходит `new (newarr) std::string(arr[0]);` Мы каждый раз обращаемся к new, а надо всего лишь поменять указатели. То есть мы и new вызываем так и коипруем.

```cpp
std::string f() {
    //...
    return std::string("abc");
}

int main() {
    std::vector<std::string> v;
    
    v.push_back(f());
}

```
Хоть и RVO поможет и сэкономит 1 копирование, но в пуше все равно будет еще одно копирование. 

## 11.2 std::move function

```cpp
template <typename T>
void swap(T& x, T& y) {
    T t = std::move(x);
    x = std::move(y);
    y = std::move(t);
}

int main() {
    std::vector<std::string> v;
    
    v.push_back(std::move(f())); // since c++11 this is already correct
}

```

Добавляем функцию `move` и указатель переложится на другую память. 

## 11.3 Rvalue references, move operations

```cpp
class String {
private:
    char* arr;
    size_t sz;
    size_t cap;
    
public:
    String(const String& s); // конструктор копирования
    
    String(String&& s):: arr(s.arr), s(s.sz), cap(s.cap) { 
        s.arr = nullptr;
        s.sz = 0;
        s.scap = 0;
    }
    
    String& operator=(String&& s) {
        String news(std::move(s));
        swap(news);
    }
};
```
`String&& s` - пока думаем что это часть синтаксиса (называется rvalue reference).  

Таким образом конструктор перемещения выглядит так. То есть мы указатель переподвесили, и убрали из второго. При этом обнулять другие переменные необязательно, но лучше мб так, чтобы строка оставалась валидной. При этом в полях `s(s.sz)` конструктор копирования - для тяжелых полей можно `std::move` писать.  

Функция `std::move` - способ попасть в констурктор перемещения, а если его нет, то просто конструктор копирования. То есть для полной работы нужен и конструктор перемещения.  

Также кроме конструктора есть перемещающий оператор присваивания.

Дефолтный перемещающий конструктор просто перемещает (`std::move` по всем полям), оператор присваивания тоже. Здесь это не подходило бы, потому что надо еще обнулять в строках. Но если есть объект из других объектов, то подошло бы. 

Если реализован нетандартный конструктор копирования, то компилятор не будет генерировать дефолтный, надо явно попросить. И аналогично наоборот. Но если явно не определять и то и то, то он будет использовать оба дефолтные.


```cpp
void push_back(const T& value) {
    //...
    new (arr+sz) T(value);
    //...
}

void push_back(T&& value) {
    //...
    new (arr+sz) T(std::move(value));
    //...
}
```

Теперь пуш работает лучше. Но могут быть еще проблемы в аллокаторах.  

Было Rule of Three - теперь правило Rule of Five - добавилось move-конструктор и move-оператор присваивания. 


Пока остается проблема с emplace_back - потому что может быть некоторые аргументы которые надо скопировать, а некоторые надо сделать move: `std::move(args...)`.  По этой же причине не выразишь два пуша через emplace_back (потому что в одном случае надо делать мув, а в дургом нет). 

