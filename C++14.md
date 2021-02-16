## 7.3) Exceptions and copying

```cpp
struct S {
	S() {}
    S(const S&) {
		std::cout << "copy\n";
	}
	~S() {
		std::cout << "destruct\n";
	}
};

void f() {
	S s;
	throw s;
}

int main() {
	try {
		f();
	} catch (S s) {
		std::cout << "caught\n";
	}
}
```

Нас интересует сколько будет копирований и разрушений. Выведется `copy destruct copy catch destruct destruct`. 
Собственно throw и catch копируют, но можно catch принимать по ссылке (`copy destruct catch destruct`).

Как работает try-catch? catch может быть несколько

```cpp
void g() {
	try {
		f();
	} catch (S& s) {
		std::cout << "caught\n";
		throw s;
	}
}

int main() {
	try {
		g();
	} catch (S& s) {
		std::cout << "caught_in_main\n";
		throw s;
	}
}
```
В этом случае будет `copy destruct caught copy destruct caught_in_main destruct`, что в общем-то логично.

В `catch` можно написать просто `throw;` - это значит что на этом уровне обработали свою часть и дальше эта же ошибка идет наверх.
При таком синтаксисе копия не будет создаваться.

----------------------------
## 7.4) Exceptions and type conversions

```cpp
void f() {
	throw 1;
}

int main() {
	try {
		f();
	} catch (double x) {
		std::cout << "caught_double\n";
	} catch (long long x) {
		std::cout << "caught_lon_long\n";
	} catch (char x) {
		std::cout << "caught_char\n";
	} catch (unsigned int x) {
		std::cout << "caught_uint\n";
	}
}
```
В try-catch должно быть полное соответствие типов, конверсий не будет.
Единственное приведение типов в исключении - можно ловить базовый класс, если был кинут его потомок.

```cpp
struct Base {

};

struct Derived: public Base {

};

void f() {
	throw Derived();
}

int main(){
	try {
		f();
	} catch (Base& b){
		std::cout << "caught Base\n";
	}
}
```
Момент - он не делает копию, хотя Derived() - rvalue, которое по неконстантной ссылке не передается, 
но происходит copy elision - зачем ему создавать временно объект и потом копировать, если срзу можно присвоить.

Если в catch еще написать `throw b`, то будет копия Base, а предыдущий наслденик уничтожится. 
А если написать `thow;`  то тот же наследник летит дальше

```cpp
	try {
		f();
	} catch (Base b){
		std::cout << "caught Base\n";
		throw;
	} catch (Derived& d){
		std::cout << "caught Derived\n";
	}
```

Такой код бесполезный, потому что throw; хоть и отправляет дальше Derived& но catch должен ловить на уровень выше. 
Catch работают подряд, а не по принципу перегрузки.

---------------

## 7.5) Exceptions in constructor

```cpp
struct S {
	int a = 0;
	int* p;
	
	s() {
		p = new int(5);
		throw 1;
	}
	
	~S() {
		delete p;
	}
};

void f() {
	S s;
}

int main() {
	try {
		f();
	} catch (...) {
		std::cout << "caught\n";
	}
}
```

Объект s недостроился, деструктор не вызовешь, потому что конструктор недоделался, и поэтому про этот объект как бы все забудут и будет утечка памяти.
При этом деструктор полей вызовется, потому что были созданы, но не вызвется деструктор самого объекта. 
Но вот int* которое нужно delete он не освободит (вот и утечка), потому что числа протсо снимаются со стека.
`-fsanitize==adress` - отлавливает утечки памяти.

Такую проблему с указателями решили классом `std::shared_ptr<int> p(new int(5));`.
Деструктор класса будет вызван, и в нем уже очиститься память. 
Это паттерн RAII = resource acquisition is initialization	

------------------

## 7.6) Exceptions in destructors

```cpp
struct S {
	S() {
		std::cout << "S\n";
	}
	
	~S() noexecept(false) {
		std::cout << "~S\n";
		throw 2;
	}
};

void f() {
	S s;
	throw 1;
}

int main() {
	try {
		f();
	} catch (...) {
		std::cout << "caught_in_main\n";
	}
}
```

Чтобы из деструктора можно было кинуть исключение нужно вызвать `noexcept(false)`
В чем проблема кроме утечки памяти? Не может быть два исключения одновременно.
Уже создано исключение 1, и хотим уничтожить локальные объекты чтобы выйти из фукнции до try-catch и тут вызывается еще одно исключение - terminate.

```cpp
~S() noexecept(false) {
	std::cout << "~S\n";
	if(!std::uncaught_exception()) {
		throw 2;
	}
}
```

Если нет необратанных исключений, то функция вызовется. Но это плохой стайл, и так не надо делать. 











