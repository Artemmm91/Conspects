**The Rule of Three**

Эвристика: если в классе или конструктор копирования, или деструктор, или операцию присваивания пришлось определять (а не автоматически скомпилирует), то надо определять все три метода.  
Можно делать так:  
```cpp
{
    String s = s;
}
```
Объект еще не создан, но он скопирует сам себя (но может фигня случиться)

## 3.4 Member initializer lists in constructor

Бывают поля без инициализатора по умолчанию  
```cpp
class A {
    A() = delete;
}

class String{
    size_t sz;
    char* str = nullptr;
public:
    String(size_t sz, char ch): sz(sz), str(new char[sz]){
        memset(str, ch, sz);
    }
}
```

Таким образом списки инициализации работают без доп памяти и без доп копирования, что является хорошим код-стайлом.  
При этом порядок исполнения в том же порядке должен быть как и порядок инициализации (и в таком, чтобы зависимость была правильной)  
Также делегирующий конструктор нельзя одновременно использовать со списками инициализации  
```cpp
class A{
    const int& a;
public:
    A(const int& a): a(a){}
}

void f(){
    A a(5);
}
```
Это проблема, потому что 5 - rvalue, и ссыдка на него уничтожится, так что после инициализации будет битая ссылка

## 3.5 Arithmetic operators overloading
```cpp
class Complex{
    double re;
    double im;
public:
	Complex& operator+=(const Complex& c){
        re += c.re;
        im = += c.im;
        return *this;
    }
    Complex operator+(const Complex& c){
        Complex sum = *this;
        sum+=c;
        return sum;
    }
};
```
Здесь происходит **return value optimization**. Потому что sum должен копироваться 3 раза: при инициализации, при возвращении, и при присваивании. Но это можно соптимизировать: компилятор кладет объект сразу туда куда нужно, и происходит только одно копирование. 
Но при этом он всегда работает. Так же существует оптимизация (copy ellision или move), когда компилятор видит `cc = c + c;` потому что справа временный объект, который не надо копировать, и он не копирует а сразу присваивает  

Почему надо вначале `+= а` потом `+`? Потому что иначе `+=` будет больше копирований делать, что плохо  
При этом оператор + должен быть вне класса, потому что правая часть обязана быть обхъектом класса, и не может быть например `5.0 + с` (если определить конструктор для 5.0, то можно будет с + 5)  
```cpp
Complex operator+(const Complex& a, const Complex& b){
	Complex sum = a;
	sum += b;
	return sum;
}

Complex::Complex& operator++(){
	return *this += 1;
}

Complex::Complex operator++(int){
	complex copy = *this;
	++*this;
	return copy;
}
```
(префиксный оператор)

## 3.6 Constant methods and [] overloading
```cpp
class String(){
	...
    char& operator[](size_t index){  // метод обычных строк
        return str[index];	
    }
    char operator[](size_t index) const{ // метод константных строк, все поля константные здесь
        return str[index];
    }
    friend std::istream& operator>>(std::istream in, String& s) // функции доступны приватные поля
}
```
Const после функции позволяет перегружать оператор, чтобы работало от константных строк. Везде где только можно надо писать const, чтобы работало  
Если нужно менять поле константного объекта, то перед этим полем в инициализации нужно писать mutable (нужно в счетчиках функций, а также во внешних использованиях)  

## 3.7 Stream input output overloading 

Так как левый операнд - поток, а не объект, то он должен быть определен вне класса, а также он должен быть ссылкой на поток  
```cpp
std::ostream& operator<<(std::ostream& out, const String& s){
    for(int i = 0; i < s.size(); ++i){
        out << s.sz[i];
    }
    return out;
}

std::istream& operator>>(std::istream in, String& s){

}
```
`friend` - особая штука, не надо ее использовать кроме как в вводе
