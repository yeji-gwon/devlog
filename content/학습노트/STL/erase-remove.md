---
tags:
  - stl
  - algorithm
---

#### Erase-Remove 

>[!summary] 한 줄 정의
>컨테이너에서 조건에 맞는 원소들을 지울 때, 살릴 원소를 앞으로 압축 (remove)하고, 쓰레기 구간을 실제로 제거(erase)하는 두 단계를 조합한 STL 관용구


#### 1. 왜 필요한가

`erase(it)`를 순회 중에 반복 호출하면, 매번 뒤 원소들을 한 칸씩 이동시켜야 함 → 지울 원소가 N개면 **O(N²)**
→ 대신 한번만 훑으면서 모아 옮기고, 마지막에 한 번만 잘라 낸다 **O(N)**


#### 2. vector / deque: `remove_if` + `erase(range)`

원소가 메모리상 연속(vector) 혹은 청크(deque) 구조라 **압축(재배치)이 필요**

```cpp
vec.erase(std::remove_if(vec.begin(), vec.end(), pred), vec.end());
```

**remove_if**
- `writeIdx`(살아남을 자리)와 `readIdx`(원본 훑기) 두 포인터로 동작
- 조건에 안 맞는(살릴) 원소만 앞으로 `std::move`
- **반환값**: 살아남은 원소들의 새 논리적 끝을 가리키는 iterator
- **size는 그대로**, 소멸자도 안 부름 → 뒤쪽은 쓰레기 상태로 남음

```
원본: [A, X, B, X, C, X, D]
압축 후: [A, B, C, D, ?, ?, ?]
					 ↑ remove_if의 반환 iterator (newEnd)
```

**erase(newEnd, end)가 하는 일**
- `newEnd ~ 실제 end` 구간의 소멸자를 호출하고 size를 축소
- 이 단계에서 컨테이너 사이즈가 줄어듦

>[!note] 왜 두 단계로 나뉘는가
>`remove_if`는 `<algorithm>` 소속이라 iterator만 다룰 뿐 컨테이너를 모름. size 조작, 소멸자 호출 같은 컨테이너 종속적 책임은 컨테이너의 `erase`에게 위임해서 알고리즘-컨테이너 분리

#### 3. list: 멤버 `remove_if` ``

```cpp
lst.remove_if([](int x){ return x % 2 == 0; });`
```

- 노드 기반 연결 구조라 앞으로 밀기 개념 자체가 불필요
- 앞뒤 노드 포인터만 재연결하면 삭제 완료 → 압축·erase 분리 없이 **멤버 함수가 한 번에 처리**
- 삭제 비용: 원소당 O(1) (포인터 재배치만)


#### 4. map / set / unordered_: 순회 + `erase(iterator)`

```cpp
for(auto it = m.begin(); it != m.end();)
{
	if(조건(*it))
		it = m.erase(it); // 다음 유효 iterator 반환
	else
		++it;
}
```

- 정렬 불변식(map/set) 또는 해시 버킷 구조(unordered_) 때문에 `std::remove_if` 자체가 적용 불가
- 대신 `erase(iterator)`가 노드 하나만 트리/버킷에서 떼어내면 되므로 **O(1)** → vector처럼 이동 비용이 없어서 순회하면서 개별적으로 erase해도 O(N) ~ O(N log N)이면 충분

>[!warning] 주의
>`erase(it)`호출 후 `it`는 무효화됨. 반드시 반환값을 받아서 갱신해야 함.



#### 5. 정리 표

| 컨테이너                         | 방식                           | 삭제 비용             | 이유                                       |
| ---------------------------- | ---------------------------- | ----------------- | ---------------------------------------- |
| `vector` / `deque`           | `remove_if` + `erase(range)` | O(N)              | 메모리 연속성 유지 필요 → 압축 후 일괄 제거               |
| `list`                       | `list.remove_if`             | O(N), 원소당 O(1)    | 노드 재연결만 하면 됨, 이동 불필요                     |
| `map` / `set` / `unordered_` | 순회 + `erase(it)`             | O(N) ~ O(N log N) | 정렬/해시 불변식 때문에 remove_if 불가, 개별 erase는 저렴 |


#### 6. C++20: 'std::erase_if' 자유 함수

컨테이너별로 다른 패턴을 통일한 자유 함수

```cpp
std::erase_if(vec, pred);    // 내부: remove_if + erase
std::erase_if(lst, pred);    // 내부: list::remove_if
std::erase_if(m, pred);      // 내부: 순회 + erase(it)
```

사용자는 컨테이너 종류 신경 안 쓰고 동일하게 호출하며 내부에서 컨테이너 특성에 맞는 최적 전략 선택