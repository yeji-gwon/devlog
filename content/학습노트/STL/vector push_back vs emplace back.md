---
tags:
  - stl
  - vector
---
#### | 요약

**push_back** = 객체를 만들어서 넣음 → lvalue는 복사, rvalue는 이동

**emplace_back** = 생성자 인자를 전달 → Perfect Forwarding → vector 내부에서 직접 생성

#### | push_back

이미 만들어진 객체를 vector에 추가

```
std::vector<A> v;

A a(10);
v.push_back(a);            // 복사
v.push_back(std::move(a)); // 이동
v.push_back(A(10));        // 임시 객체 → 이동
```

`push_back`은 **lvalue / rvalue**를 받을 수 있도록 오버로딩 되어 있음

```
void push_back(const T& value); // lvalue → 복사 생성
void push_back(T&& value);      // rvalue → 이동 생성
```

`T&&`로 받은 매개변수는 이름이 붙으면 lvalue가 되므로, 내부에서 `std::move`등을 사용해 이동 시킴

#### | emplace_back

생성자 인자를 받아 vector 내부에서 객체를 직접 생성

```
v.emplace_back(10, 20);
```

→ vector 내부에서 `A(10, 20);`을 직접 생성

`emplace_back`은 **Perfect Forwarding**을 사용해서 전달 받은 인자의 value category를 유지함

```
template<class... Args>
void emplace_back(Args&&... args)
{
	// ...
	T(std::forward<Args>(args)...);
}
```

생성자 인자 → Perfect Forwarding → vector 내부에서 객체 직접 생성


#### | 언제 emplace_back을 쓰면 좋은가?

객체를 새로 만들어 vector에 넣는 상황에서

```
// 객체를 먼저 생성
v.push_back(A(10, 20));

// 생성자 인자를 바로 전달
v.emplace_back(10, 20);
```

`emplace_back`은 **임시 객체를 먼저 만들 필요 없이 vector 내부에서 직접 생성**할 수 있음

특히 생성자 인자가 여러 개이거나 객체 생성 비용이 큰 경우 유용함

`unique_ptr`를 `vector`에 담을 때는 보통 `push_back(std::make_unique<T>())`를 사용하면 됨 (`unique_ptr`의 이동은 포인터 소유권만 넘기는 아주 가벼운 연산이기 때문에) [[unique_ptr]]
