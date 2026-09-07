---
tags:
  - refactoring
  - Overcooked2
---
Overcooked2 모작에서 재료, 조리 도구, 조리대처럼 플레이어가 상호작용하는 환경 오브젝트를 담당했다.

당시에는 오브젝트마다 할 수 있는 행동이 다르다는 점에 주목했다. 물건을 들 수 있는 오브젝트는 `ICarry`, 물건을 올릴 수 있는 오브젝트는 `IPlace`, 썰기나 조리처럼 시간이 필요한 행동은 `IProcess` 계열 인터페이스로 나눴다.

```
class CChopStation
    : public CInteract
    , public IPlace
    , public IChop
{
    // ...
};

class CPot
    : public CInteract
    , public IPlace
    , public ICook
{
    // ...
};
```

공통 기능은 `CInteract`에 두고, 오브젝트마다 필요한 역할을 인터페이스로 추가하는 방식이었다. 당시에는 이렇게 하면 오브젝트의 기능을 역할에 따라 나눌 수 있다고 생각했다.

프로젝트가 끝난 뒤 코드를 다시 볼 때는 기준을 조금 바꿨다.

> 새로운 재료나 조리대가 하나 추가된다면 어디를 수정해야 할까?

기능이 정상적으로 동작하는지만 볼 때는 잘 보이지 않았던 문제가 이 질문을 기준으로 코드를 따라가자 보이기 시작했다.

### 당시 구조

```
CInteract
├─ 재료
│  ├─ ICarry
│  └─ IState를 이용한 조리 상태 관리
│
└─ 조리대
   ├─ IPlace
   └─ IProcess
      ├─ IChop
      ├─ ICook
      ├─ IWash
      └─ IExtinguish
```

예를 들어 도마는 물건을 올릴 수 있고 썰기 작업을 할 수 있으므로 `IPlace`, `IChop`을 구현한다. 냄비는 `IPlace`, `ICook`을 구현한다.

도마인지 냄비인지보다 **무엇을 할 수 있는지**를 기준으로 나누려고 한 것이다.

문제는 인터페이스를 나눈 뒤 실제로 사용하는 코드가 그 의도를 끝까지 유지하지 못했다는 점이었다.

### 역할을 나눴는데 다시 타입을 확인하고 있었다

플레이어가 조리대와 상호작용하는 코드에서는 다음과 같이 구체 인터페이스를 확인한다.

```
if (dynamic_cast<IChop*>(m_pCursorStation))
{
    m_pIChop = dynamic_cast<IChop*>(m_pCursorStation);

    if (m_pIChop->Enter_Process())
    {
        // ...
    }
}
else if (dynamic_cast<IWash*>(m_pCursorStation))
{
    m_pIWash = dynamic_cast<IWash*>(m_pCursorStation);

    if (m_pIWash->Enter_Process())
    {
        // ...
    }
}
```

`IChop`, `IWash`로 역할을 나눠놓았지만 정작 플레이어는 현재 조리대가 어떤 역할을 가지고 있는지 하나씩 확인하고 있었다.

이 구조에서는 새로운 처리 방식이 추가될 때 조리대만 추가해서 끝나지 않는다. 해당 처리를 플레이어가 사용해야 한다면 플레이어의 분기도 같이 확인해야 한다. 비슷한 형태의 코드가 `CRealPlayer`, `CFakePlayer`에 있다면 변경 지점도 늘어난다.

당시에는 인터페이스를 만들었다는 사실 자체에 집중했던 것 같다. 다시 보니 중요한 건 인터페이스의 개수가 아니라 **새 기능이 추가됐을 때 그 사실을 누가 알아야 하는가**였다.

### 분리한 인터페이스끼리도 서로를 확인했다

`IPlace`의 `Set_Empty()`에도 비슷한 코드가 있었다.

```
virtual void Set_Empty()
{
    m_bFull = false;
    m_pPlacedItem = nullptr;

    if (dynamic_cast<IProcess*>(this))
        dynamic_cast<IProcess*>(this)->Set_Progress(0.f);
}
```

`IPlace`는 물건을 올리고 내리는 역할이고 `IProcess`는 시간 기반 처리를 담당한다. 겉으로는 서로 다른 역할로 나눴지만 실제로는 `IPlace`가 자신에게 `IProcess`가 있는지 확인한 뒤 진행도를 초기화한다.

인터페이스를 나눴다고 해서 역할 사이의 의존까지 자동으로 사라지는 것은 아니었다.

### 재료도 종류가 늘수록 기존 코드의 분기가 늘었다

재료는 `INGREDIENT_TYPE`과 조리 상태를 가지고 있고, 상태 변화는 State로 처리했다.

```
if (CIngredient::TOMATO == eType ||
    CIngredient::LETTUCE == eType ||
    CIngredient::CUCUMBER == eType ||
    CIngredient::FISH == eType ||
    CIngredient::SHRIMP == eType)
{
    pIngredient->ChangeState(new IDoneState());
}
```

State를 분리했지만 State 내부에서는 다시 재료 종류를 확인한다.

새로운 재료가 추가되면 그 재료가 어떤 상태 전이를 거치는지에 따라 기존 State의 분기를 수정해야 한다. 종류별 판단이 다른 곳에도 있다면 재료 하나를 추가하기 위해 확인해야 할 위치가 같이 늘어난다.

여기서 `enum`이나 State를 사용한 것 자체가 문제라고 생각하지는 않는다. 문제는 **재료의 종류가 추가될 때 그 변경이 여러 기존 코드로 퍼질 수 있다는 것**이었다.

### 처음 생각한 개선 방향 - 컴포지션

처음 회고했을 때는 이 문제를 보고 상속으로 조합한 역할을 객체에서 떼어내는 방향을 생각했다.

기존에는 조리대의 능력을 다음처럼 표현했다.

```
CChopStation
├─ CInteract
├─ IPlace
└─ IChop

CPot
├─ CInteract
├─ IPlace
└─ ICook
```

이를 필요한 기능을 가진 컴포넌트를 조합하는 구조로 바꾸는 것이다.

```
CChopStation
├─ PlaceComponent
└─ ChopProcess

CPot
├─ PlaceComponent
└─ CookProcess

CSinkStation
├─ PlaceComponent
└─ WashProcess
```

조리대의 종류에 기능을 고정하기보다 필요한 기능을 조합한다. `Place`와 `Process`도 서로의 존재를 `dynamic_cast`로 확인하기보다 명시적으로 연결할 수 있다.

재료 역시 상태 처리와 종류별 규칙을 한 클래스 계층에 계속 추가하기보다 변하는 부분을 별도의 구성 요소나 데이터로 분리하면 새 재료가 기존 코드에 미치는 영향을 줄일 수 있다고 생각했다.

### 1차 개선안

```
기존

오브젝트의 정체성
      +
인터페이스 상속 조합
      ↓
호출부에서 dynamic_cast로 역할 확인
      ↓
새 역할 추가 시 기존 분기 수정


1차 개선안

오브젝트
├─ 공통 기능
└─ 필요한 기능 컴포넌트
      ↓
공통 접근 방식으로 기능 사용
      ↓
필요한 기능을 조합
```

당시 구현에서는 역할을 인터페이스로 나누는 것까지 생각했다면, 프로젝트 이후에는 변화할 가능성이 있는 기능을 상속 계층에서 떼어내 조합하는 방향을 개선안으로 잡았다.

새로운 종류나 기능을 추가할 때 기존 코드 여러 곳의 타입 분기를 찾아다니는 일을 줄이고 싶었기 때문이다.

그런데 여기까지 정리한 뒤 한 가지 의문이 생겼다.

> 정말 문제의 원인은 상속이었을까?  
> 컴포지션으로 바꾸면 지금 발견한 문제가 해결될까?

코드에는 이미 `IProcess`라는 공통 인터페이스가 있다. 그렇다면 구조 전체를 바꾸기 전에 이 인터페이스를 제대로 사용하지 못한 것이 먼저 해결해야 할 문제일 수도 있다.

재료도 처음에는 기능을 컴포넌트로 분리하면 된다고 생각했지만, 실제 재료 클래스들의 차이가 행동인지 단순한 데이터인지부터 확인할 필요가 있었다.

그래서 이 개선안을 정답으로 두지 않고 실제 코드를 더 넓게 비교해보기로 했다.

### 다음 글
[[개선안을 다시 검증하다 - Overcooked2 조리 시스템 다시 설계하기 2]]