## 1.4) Definitions and declarations

**Scope** - область видимости - «часть программы где переменная видна/актуальна» (actual scope)  
Есть также потенциальная область видимости (potential scope)  
```cpp
int x = 1; //глобальная переменная, видная отовсюду
int main() {
	int x = 3; // локальная переменная (local variable) 
	{ 
	   int x = 5; // более» локальная переменная, 
                  // фигурная скобка открывает scope
             cout << x;
	}
	for (int x = 0; x < 10; ++x) { // свой новый «х» будет в форе
		cout << x;
	}			
          cout << x;                 
}
```
Локальный `x` затмевает глобальный, и чтобы обратиться к глобальному нужно `cout << ::x;`

Potential scope для x - весь main, actual scope - main без вложения
```cpp
namespace N { // новая область видимости
	int x = ;
}
```
чтобы обратиться к переменной внутри `cout << N::x;`  
Namespace добавляют удобства, чтобы разбивать переменные и методы на блоки  
`-Werror` - параметр, при котором любые warnings становятся errors  
`-Wall` - параметр, при котором компилятор предупреждает обо всем  
`-Wextra` - еще больше предупреждений  
`qualified-id`   - когда добавляем двоеточие `N::x;`
`unqualified-id`  - просто пишем имя `x;`

## 1.5) Basic types and supported operations

С++ — статически типизированный язык, типы надо объявлять сразу  
Статически - в compile-time происходит  
Динамически - в run-time происходит  

**Целочисленные типы**:  
`int` 4 байта [-2^31;2^31 - 1] max_int = 2’147’483’648  
`short [int]` 2 байта max: 32767 = 2^15 - 1  
`long [int]` 4 байта  
`long long [int]` 8 байт  
`int32_t` 4 байта - точно будет столько памяти (в отличие от int)  
`int64_t` 8 байт  
`char` - символы, но по сути ASCII коды (1 байт, может быть беззнаковый)  
`unsigned int` - беззнаковый тип (диапозон начинается с 0)  
`size_t` - беззнаковый большой тип  

Операции:   `+ - * / % ++ — < > <= >= == != & | ^ ~`  

Если переполнять знаковый тип - то это UB, а если переполнять беззнаковый, то это будет операция по модулю (UB не будет)  
Почему? Скорее всего, потому что знаковый тип не специфицирован, и компилятор может по-разному его представлять  

**Типы с плавающей точкой**:  
`float` 4 байт  
`double` 8 байт (основной)  
`long double` 10(16) байт  
Скорее всего double устроен так:  
1 бит - sign  
11 бит - exponent - знаковое число в какую степень возводить  
52 бит - mantissa (b0, b1, ..., b51)  

`(-1)^sign * 1.b0 ... b51 * 2^exp`  
Значения близкие к 0 можно хранить с большой степенью, а большие числа наоборот  

**Булевский тип**:
`bool` 1 байт (чтобы был разный адрес)
Операции: `&& || ! == !=`

**Implicit casts** (неявное приведение) - например с операциями char или bool  
Например может быть проблема в `(i < v.size() - 1)`, потому что если вектор пустой, то будет проблема (тк беззнаковый int)  
**Integer promotion** - типы меньше инта могут приводиться к инту  

**Литеральный суффиксы**:  
`1u;` (unsigned)  
`0.2f;` (float)  
`1ll;` (long long)  

`0x____;` 16-ричная система  
`0_____;` 8-ричная система  

**Контейнеры**:  
1. *String*  
Можно объявлять `«abc»`, 
Операции: `[i], +(конкатенация), +=(приписывание), size(), length(), substr(n, k), find(str), rfind(str)`  

2. *Vector*
```cpp
vector<int> v = {1, 2, 3, 4};
            v(10); // будут 10 нулей
            v(10, 5);
v.push_back(x);
v.pop_back(); // Если вектор пуст то UB
v[i]; // Если индекс неправильный то UB
v.resize(n, 5);
v.size(); 
```
3. *Map*  
`map<string, int>` - сортированный словарь по ключу, отображение, хранит пары значений (ключ, значение)  
`m[k] = v;` перезаписывается ключ k или создается  
`m.insert({k, v});` вставляет ключ k если не было  
`m.erase(k); `	  
`m.count(k);` возвращает 0 или 1  
Базовое значение 0 (`m[1];` <- если не было 1 то будет 0)  
Может быть unordered  
4) *Set*  
`set<int>`  
Также но без обращений к элементу, и хранит сами элементы

## 1.6) Expressions and operators

Все инструкции(statements) - объявление(declarations), выражения(expressions), controls

*Операторы арифметические*: `+ - * / % & | ^ << >> ~`  
*Логические операции* `&& || !` работают по сокращенному правилу (вначале левая часть и только если надо правая часть)

Выражения раскладываются на выражения и операторы, и компилятор смотрит на выражение, разбивает его на меньшие по своим правилам, вплоть до самой низкого уровня.

Например `x || y && z`. Компилятор по своим правилам вначале раскладывает по || а потом уже по &&.  
При этом если несколько операторов на одном уровне, то смотрится левая или правая ассоциативность

*Операторы присваивания*:  
`= += -= ;= /= %= ^= |= &= <<= >>=`

*Инкремент (и декремент)*:
`x++;` суффиксный 
`++x;` префиксный - этот быстрее, тк копию не надо создавать

Оператор `sizeof` - сколько байт памяти занимает элемент такого типа  
`sizeof(++x);` - это 4, но значение не изменится, потому что sizeof смотрит только сколько памяти займет.   
А `sizeof` от вектора это сумма размеров полей класса(примерно, с учетом пэддинга)  

*Тернарный оператор*	`x ? y : z`
Если `x` то `y`, иначе `z`. Для присвоения можно писать
a = (x ? y:z);

*Оператор «,»*:
Смотрит левую часть, потом смотрит правую часть, и возвращает правую часть

*lvalue* и *rvalue*  
`a = b;` Неформально: lvalue - то чему можно присваивать, а rvalue - то чему нельзя присваивать  
`a` - lvalue  
`a + b` - rvalue  
Инкремент как и присваивание требует lvalue, при этом `++a` - lvalue, а `а++` rvalue.

`(x ? ++y : z++) = a;`  
На этапе компиляции нельзя сказать это lvalue или rvalue так как ++y lvalue, а z++ rvalue, поэтому будет CE  
Оператор «,» возвращает такое value какая была правая часть.  
```cpp
int x = 0;
++x = x++;
cout << x;
```

Правила **sequenced-before**:  
1. В пост-инкрементах подстановка значения до side effects  
2. в пре-инкрементах side effects до подстановки  

Но в С++17 вначале происходит вся правая часть(со значением и присваиванием), потом вся левая часть и только потом присваивание.  
А него до было бы UB.