---
tags:
  - cpp
  - 스마트포인터
  - unique_ptr
---
#### | unique_ptr란

- 특정 객체에 **유일한 소유권, Ownership**을 부여하는 스마트 포인터
- `unique_ptr`가 소멸될 때, 소유하고 있던 객체도 함께 소멸
- `unique_ptr`는 유일한 소유권을 가져야 하기 때문에 **복사 생성자, 복사 대입 연산자가 삭제**
- 소유권 이전은 가능 → **이동 생성자, 이동 대입 연산자** 사용 (`std::move`로 rvalue로 캐스팅 필요)
- `std::move`로 소유권을 이전하면 기존 `unique_ptr`는 **유효하지만 nullptr 상태**가 되므로, 소유권 이전 이후 기존 `unique_ptr`를 역참조하지 않도록 주의


#### | unique_ptr를 함수 인자로 전달하기

함수가 **객체에 접근만 하는지, unique_ptr 자체를 조작하는지, 소유권을 가져가는지**에 따라 전달 방법 구분

>**객체에 접근만 하는 경우**
>소유권을 가져갈 필요 없이 객체만 사용한다면 `get()`을 통해 객체의 포인터를 전달
```
void Foo(A* ptr)
{
	ptr->some();
}

{
	Foo(ptr.get());
}
```
- `get()`은 **소유권을 이전하지 않고**, 관리 중인 객체의 주소만 반환

>**unique_ptr 자체를 조작하는 경우**
>함수에서 `reset()`, 새로운 객체 대입 등 `unique_ptr` 자체의 상태를 변경해야 한다면 `unique_ptr&`로 전달
```
void Reset(std::unique_ptr<A>& ptr)
{
	ptr.reset();
}

std::unique_ptr<A> ptr = std::make_unique<A>();
Reset(ptr);

// ptr = nullptr
```

>**소유권을 함수에 넘기는 경우**
>함수가 객체의 소유권을 가져가야 한다면 `std::move`를 사용해서 `unique_ptr`를 전달
```
void Foo(std::unique_ptr<A> ptr)
{
	//ptr이 소유권을 가짐
}

std::unique_ptr<A> ptr = std::make_unique<A>();
Foo(std::move(ptr)); // 이후 기존 ptr은 nullptr가 됨
```


#### | make_unique

```
std::unique_ptr<Foo> ptr1(new Foo(3, 3));
auto ptr2 = std::make_unique<Foo>(3,3));
```
- `make_unique`를 사용하면 객체 생성과 `unique_ptr`의 소유권 설정을 한 번에 처리 가능
- 일반적으로 `new`를 직접 사용하는 것보다 `make_unique` 사용을 권장


#### | std::vector와 사용할 때 주의점

[[vector push_back vs emplace back]]
`unique_ptr`를 원소로 가지는 `vector`를 사용할 때, `unique_ptr`는 **복사 생성자가 없다고 이동만 가능하다**는 점을 주의해야 함

**push_back**
- `push_back`으로 넣을 경우, rvalue를 넘기도록 주의 (lvalue를 넘기면 컴파일 에러 발생)
- `push_back`은 lvalue / rvalue를 받을 수 있도록 오버로딩되어 있음
```
std::vector<std::unique_ptr<A>> v;
std::unique_ptr<A> pa(new A(1));

// v.push_back(pa);            // X 복사 불가능
v.push_back(std::move(pa));    // O 소유권 이동
v.push_back(make_unique<A>()); // O 소유권 이동
```
- lvalue를 넘기면 복사를 시도하기 때문에 컴파일 에러 발생
- rvalue를 넘기면 이동 생성자를 통해 소유권 이전

**emplace_back**
- `emplace_back` 함수를 이용하면, `vector` 안에서 `unique_ptr`를 직접 **생성**하면서 집어 넣어 
- 전달된 인자는 **완벽한 전달 perfect forwarding**을 통해 원래의 lvalue/rvalue 성격을 유지하면서 `unique_ptr<A>`의 생성자에 전달됨
```
std::vector<std::unique_ptr<A>> v;

v.emplace_back(new A());
```

**push_back + make_unique** 사용 
```
v.push_back(std::make_unique<A>());
```
- 왜냐하면 `unique_ptr`의 이동 비용이 매우 작아서 `emplace_back`의 이점이 거의 없어서
