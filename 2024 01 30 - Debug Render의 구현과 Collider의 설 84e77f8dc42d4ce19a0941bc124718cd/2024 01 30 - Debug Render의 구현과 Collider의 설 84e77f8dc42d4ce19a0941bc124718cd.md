# 2024/01/30 - Debug Render의 구현과 Collider의 설계

태그: C++, WinAPI, 중급
날짜: 2024/01/30
상위 항목: Week10 (Week10%206c3606b897d340fcb90a1dfe2956c91a.md)
주차: 0010_Week10~19

### 현재 프로젝트의 컴포넌트 구체화

1. Collider (충돌체)
    
    <aside>
    📁 *03. Game\06. Component*
    → `CCollider.h`, `CCollider.cpp` 생성
    
    </aside>
    
2. Animator (애니메이션 동작)
    
    <aside>
    📁 *03. Game\06. Component*
    → `CAnimator.h`, `CAnimator.cpp` 생성
    
    </aside>
    
3. Rigidbody (강체, 물리 엔진의 힘과 속도의 영향을 받는 물체)
    
    <aside>
    📁 *03. Game\06. Component*
    → `CRigidbody.h`, `CRigidbody.cpp` 생성
    
    </aside>
    
4. FSM (상태 기반 패턴)
    
    <aside>
    📁 *03. Game\06. Component*
    → `CFSM.h`, `CFSM.cpp` 생성
    
    </aside>
    

Component에 대한 관리는 CObj에서 이루어진다!

```cpp
class CObj :
    public CEntity
{
private:
    Vec2                m_Pos;      // 위치
    Vec2                m_Scale;    // 크기
    vector<CComponent*> m_vecCom;   // 보유 컴포넌트들
// ...
}
```

따라서 CObj의 소멸자에서 해당 컴포넌트들을 메모리에서 해제하는 작업을 꼭 해줘야 한다!

```cpp
CObj::~CObj()
{
	Safe_Del_Vec(m_vecCom);
}
```

### AddComponent, GetComponent 구현

1. **AddComponent 구현하기**

모든 Obj 객체들이 원하는 컴포넌트를 선택해 장착할테니, 

부모 클래스인 CObj에서 원하는 Component를 add할 수 있도록 `AddComponent`를 구현

먼저 그 전에, Component를 add하고 get하는 과정은 Obj 단에서 이루어지므로,

CComponent에서 멤버 변수로 *“자신을 소유하고 있는 Obj 객체의 주소”*를 가리키는 포인터 변수인 m_Owner를 가져야 한다

```cpp
#pragma once
#include "CEntity.h"
#include "CObj.h"

class CComponent :
    public CEntity
{
private:
    CObj*       m_Owner;    // 컴포넌트를 소유하고 있는 오브젝트

public:
    virtual void finaltick() = 0;
    virtual CComponent* Clone() = 0;

public:
    CComponent();
    ~CComponent();

    friend class CObj;    // Obj 객체가 private 영역에 접근할 수 있도록 freind 선언
};
```

이제 생성한 Component를 추가하는 AddComponent를 구현해보자

```cpp
void AddComponent(CComponent* _Component);
```

```cpp
void CObj::AddComponent(CComponent* _Component)
{
	m_vecCom.push_back(_Component);
	_Component->m_Owner = this;
}
```

이때 AddComponent의 반환타입을 `CComponent*`로 하고, 

현재 push_back한 Component의 주소를 반환해주면, 

Component를 생성함과 동시에 그 주소를 저장할 수 있으니 그렇게 수정하도록 하자!

```cpp
CComponent* AddComponent(CComponent* _Component)
{
	m_vecCom.push_back(_Component);
	_Component->m_Owner = this;

	return _Component;
}
```

1. **GetComponent** 구현하기

여러 컴포넌트 객체들을 CComponent 타입의 vector로 관리하고 있으므로,

원하는 타입의 객체가 해당 vector에 포함되어 있는지 확인 후 캐스팅해 가져오는 과정이 필요하다

```cpp
template<typename T>
T* GetComponent()
{
	for (size_t i = 0; i < m_vecCom.size(); ++i)
  {
	  T* pComponent = dynamic_cast<T*>(m_vecCom[i]);
    if(pComponent) { return pComponent; }
   }
    return nullptr;
}
```

이제 실질적인 컴포넌트 객체는 해당 컴포넌트가 필요한 오브젝트에서 생성해줄 것이다

```cpp
#pragma once
#include "CObj.h"

class CCollider;

class CPlayer :
    public CObj
{
private:
    float       m_Speed;

    CCollider*  m_Collider;
//...
};
```

```cpp
// ...
#include "CCollider.h"

CPlayer::CPlayer()
	: m_Speed(500.f)
{
	// Player의 컴포넌트 설정
	m_Collider = (CCollider*)AddComponent(new CColider);

	// Component 생성 후에는 GetComponent를 통해 호출도 가능!
	m_Collider = GetComponent<CCollider>();
}
```

## Debug Render의 구현

우리가 만든 충돌체(Collider)가 실제 게임 플레이에서는 보이지는 않지만, 개발 과정에서는 우리가 원하는 위치에서 잘 작동하고 있는지 확인할 필요가 있을 것이다!

이를 위해 디버깅을 위한 렌더링 기능을 구현해서 우리가 원하는 위치에서 작업이 실행되고 있는지 시각적으로 확인해보자

→ Debug Render 매니저를 구현하기!

### Debug Render의 모양과 정보를 담기 위한 사전 작업

이를 위해서 간편한 기능 구현을 위한 enum과 struct를 정의하자

1. `enum class DBG_SHAPE` : 디버그 렌더링 시 사용될 도형의 모양을 선택할 enum class

```cpp
enum class DBG_SHAPE
{
	CIRCLE,
	RECT,
};
```

1. `struct tDbgRenderInfo` : 디버그 렌더링에 사용될 정보를 담을 struct

```cpp
struct tDbgRenderInfo
{
	DBG_SHAPE	  Shape;             // 사용될 도형의 모양
	Vec2		    Position;          // 위치
	Vec2		    Scale;             // 크기
	PEN_TYPE	  Color;             // 색상

	float		    Duration;          // 생명 주기
	float		    Age;               // 현재 나이 (생성되고 지난 시간)
};
```

### Debug Render Manager의 구현

```cpp
#pragma once

class CDbgRenderMgr
{
	SINGLE(CDbgRenderMgr)

private:
	list<tDbgRenderInfo>	m_RenderList;
	
public:
	void AddDbgRenderInfo(const tDbgRenderInfo& _info)
	{
		m_RenderList.push_back(_info);
	}

public:
	void tick();
	void render();
};
```

- `SINGLE(CDbgRenderMgr)`
    
    : 매니저 클래스이므로 싱글톤 패턴으로 단일 객체 생성
    
- `list<tDbgRenderInfo> m_RenderList;`
    
    : struct ***tDbgRenderInfo***를 통해 각각의 디버그 렌더링 정보를 담은 원소들을 관리하고 이를 통해 화면에 그릴 수 있는 list
    

- `void AddDbgRenderInfo(const tDbgRenderInfo& _info)`
    
    : 인자를 통해 받은 tDbgRenderInfo 원소를 list에 push_back하는 함수 구현
    

- `void tick()`, `void render()`
    
    : DbgRenderMgr에서 구현된 tick과 render는, *CEngine의 progress에서* 각각의 매니저들이 tick이 호출되는 시점과 render가 호출되는 시점에서 호출된다
    

```cpp
#include "pch.h"
#include "CDbgRenderMgr.h"

#include "CEngine.h"
#include "CTimeMgr.h"
#include "CKeyMgr.h"

CDbgRender::CDbgRender()
{}

CDbgRender::~CDbgRender()
{}

void CDbgRender::tick()
{}

void CDbgRender::render()
{
	list<tDbgRenderInfo>::iterator iter = m_RenderList.begin();

	for (; iter != m_RenderList.end(); )
	{
		// 펜 및 브러쉬 설정
		USE_BRUSH(DC, BRUSH_HOLLOW);
		CSelectObj SelectPen(DC, CEngine::GetInst()->GetPen(iter->Color));

		// DBG 가 Rect 면 사각형을 그린다.
		if (DBG_SHAPE::RECT == iter->Shape)
		{
			Rectangle(DC
				, (int)(iter->Position.x - iter->Scale.x / 2.f)
				, (int)(iter->Position.y - iter->Scale.y / 2.f)
				, (int)(iter->Position.x + iter->Scale.x / 2.f)
				, (int)(iter->Position.y + iter->Scale.y / 2.f));
		}
		 
		// DBG_SHAPE 가 Circle 이면 원을 그린다.
		else if(DBG_SHAPE::CIRCLE == iter->Shape)
		{
			Ellipse(DC
				, (int)(iter->Position.x - iter->Scale.x / 2.f)
				, (int)(iter->Position.y - iter->Scale.y / 2.f)
				, (int)(iter->Position.x + iter->Scale.x / 2.f)
				, (int)(iter->Position.y + iter->Scale.y / 2.f));
		}

		// ⭐ 해당 디버그 렌더 정보가 수명을 다하면 리스트에서 제거한다.⭐
		(*iter).Age += DT;
		if (iter->Duration < iter->Age)
		{
			iter = m_RenderList.erase(iter);
		}
		else
		{
			++iter;
		}
	}
}
```

- 디버그 렌더 정보를 리스트에서 제거하는 부분
    - `(*iter).Age += DT`
        
        : 현재 가리키고 있는 디버그 렌더의 나이를 DT를 더함으로써 증가시킴
        
    - `if (iter->Duration < iter->Age)`
        
        : 현재 나이가 생명주기보다 많아졌을 경우 해당 iter가 가리키는 디버그 렌더를 erase
        

### Debug Render가 Debug 모드에서만 실행이 되도록 설정해보자

→ Release 버전에서는 debug render 자체가 실행이 되지 않도록 설정하는 방법

아이디어 1)

```cpp
#ifdef _DEBUG
	// CdbgRenderMgr의 구현

#endif
```

:  시작과 끝에 `#ifdef _DEBUG` ~ `#endif` 를 연결함으로써 DEBUG 모드가 아니라면 해당 코드 자체가 컴파일 되지 않도록 한다 (아예 무시된다! 없는 코드 취급)

아이디어 2)

```cpp
bool					m_bRender;             // 디버그 렌더링 on/off 스위치!
```

: `CDbgRenderMgr`의 멤버 변수로 디버그 렌더링의 on/off 스위치의 역할을 수행하는 bool 타입의 변수 `m_bRender`를 만들어주어, 특정 버튼의 입력을 통해 해당 스위치를 on/off 할 수 있도록 구현해보자

```cpp
void tick()
{
	if(KEY_TAP(KEY::_0))
	{
		m_bRender ? m_bRender = false : m_bRender = true;
	}
}
```

그럼 m_bRender가 false 일 때에는

사각형 / 원 등 디버그 렌더를 그리는 작업 자체가 수행되지 않으면 되므로

```cpp
void CDbgRender::render()
{
	// ...

		// DBG 가 Rect 면 사각형을 그린다.
		if (m_bRender && DBG_SHAPE::RECT == iter->Shape)
		{
			Rectangle(DC
				, (int)(iter->Position.x - iter->Scale.x / 2.f)
				, (int)(iter->Position.y - iter->Scale.y / 2.f)
				, (int)(iter->Position.x + iter->Scale.x / 2.f)
				, (int)(iter->Position.y + iter->Scale.y / 2.f));
		}
		 
		// DBG_SHAPE 가 Circle 이면 원을 그린다.
		else if(m_bRender && DBG_SHAPE::CIRCLE == iter->Shape)
		{
			Ellipse(DC
				, (int)(iter->Position.x - iter->Scale.x / 2.f)
				, (int)(iter->Position.y - iter->Scale.y / 2.f)
				, (int)(iter->Position.x + iter->Scale.x / 2.f)
				, (int)(iter->Position.y + iter->Scale.y / 2.f));
		}
	// ...
}
```

→ 이런식으로 제약을 걸어줌으로써 디버그 렌더러가 표시되지 않도록 설정해준다

그럼 디버그 렌더를 확인해보기 위해서, 

player가 spacebar를 눌렀을 경우 (공격할 경우) player에 대한 원 모양의 디버그 렌더가 생성되고 출력되게끔 작성해보자

```cpp
if (KEY_TAP(SPACE))
{
	// ...
	tDbgRenderInfo info{};
	info.Shape = DBG_SHAPE::CIRCLE;
	info.Color = PEN_TYPE::PEN_GREEN;
	info.Position = GetPos();
	info.Scale = Vec2(500.f, 500.f);
	info.Duration = 1.f;
	info.Age = 0.f;

	CDbgRender::GetInst()->AddDbgRenderInfo(info);
}
```

이 과정을 간편하게 함수로 뺀다면…

```cpp
void DrawDebugRect(PEN_TYPE _Type, Vec2 _Pos, Vec2 _Scale, float _Time);
void DrawDebugCircle(PEN_TYPE _Type, Vec2 _Pos, Vec2 _Scale, float _Time);
```

```cpp
#include "pch.h"
#include "CDbgRender.h"

void DrawDebugRect(PEN_TYPE _Type, Vec2 _Pos, Vec2 _Scale, float _Time)
{
	tDbgRenderInfo info{};
	info.Shape     = DBG_SHAPE::RECT;
	info.Color     = _Type;
	info.Position  = _Pos;
	info.Scale     = _Scale;
	info.Duration  = _Time;
	info.Age       = 0.f;

	CDbgRender::GetInst()->AddDbgRenderInfo(info);
}

void DrawDebugCircle(PEN_TYPE _Type, Vec2 _Pos, Vec2 _Scale, float _Time)
{
	tDbgRenderInfo info{};
	info.Shape     = DBG_SHAPE::CIRCLE;
	info.Color     = _Type;
	info.Position  = _Pos;
	info.Scale     = _Scale;
	info.Duration  = _Time;
	info.Age       = 0.f;

	CDbgRender::GetInst()->AddDbgRenderInfo(info);
}
```

이렇게 DebugRender를 각 도형별로 생성하는 과정을 함수로 빼서 구현할 경우 간략한 작성이 가능해진다

```cpp
void CPlayer::tick()
{
	// ...
	DrawDebugRect(PEN_TYPE::PEN_GREEN, GetPos(), Vec2(500.f, 500.f), 3.f);
}
```

## 충돌체 Collider의 설계

Collider는 Owner가 되는 Obj(소유 오브젝트)가 충돌 감지가 될 수 있도록 막을 하나 만드는거라고 생각하면 된다!

따라서 필요한 정보들, 즉 멤버 변수들은

- 소유 오브젝트로부터 상대적인 좌표 (Offset) → Set 함수 필요 ⭕
    
    : Offset은 원하는 충돌체의 위치를 조절하기 위해서 현재 Owner로 갖고 있는 Obj의 위치를 기준으로 충돌체의 상대적인 위치를 지정하는 것이다!
    
- 크기 → Set 함수 필요 ⭕
- 최종적인 좌표 (소유 오브젝트의 위치 + 오프셋으로, 화면에 위치할 최종 좌표)
    
    → Set 함수가 있으면 안된다! ❌ 내부적으로 계산해서 결정되어야 하는 값
    

따라서 정리해보면

```cpp
#pragma once

#include "CComponent.h"

class CCollider :
    public CComponent
{
private:
	Vec2 m_OffsetPos;
	Vec2 m_Scale;

	Vec2 m_FinalPos;

public:
	void SetOffsetPos(Vec2 _Offset) { m_OffsetPos = _Offset; }
	void SetScale(Vec2 _Scale) { m_Scale = _Scale; }

public:
    virtual void finaltick() override;
    virtual CCollider* Clone() { return new CCollider; }

public:
    CCollider();
    ~CCollider();
};
```

### finaltick의 역할

`tick`에서는 오브젝트가 매 프레임마다 해야할 작업을 구현했었다

> `finaltick`에서는,
> 
> 
> ***오브젝트가 소유한 컴포넌트가 매 프레임마다 해야할 작업을 구현함***으로써, 
> 
> 오브젝트에 의해 그의 부가적인 장치들인 *컴포넌트*가 오브젝트의 변화에 따라 맞춰질 수 있도록 update하는 작업의 개념으로 finaltick을 사용하자
> 

그렇담 CObj의 finaltick에서, (`CObj.finaltick()`)

보유 컴포넌트들의 vector인 m_vecCom 원소들의 finaltick을 호출하게끔 한다!

```cpp
void CObj::finaltick()
{
	for (size_t i = 0; i < m_vecCom.size(); ++i)
	{
		m_vecCom[i]->finaltick();
	}
}
```

→ 이때, m_vecCom 자체는 CComponent이지만 이를 상속하는 자식 클래스들 (Collider, Animator, Rigidbody …) 에서 오버라이딩 했으므로 각각의 클래스에서 구현한 finaltick이 호출된다 (`CCollider.finaltick()`, `CAnimator.finaltick()`, …)

그렇다면 이제부터 finaltick은 CObj에서 관리되는 `m_vecCom`을 통해 직접적으로 구체화된 Component에 구현된 finaltick을 호출하므로.. CObj를 상속받는 자식클래스들은 finaltick을 구현할 일이 없어진다. (컴포넌트의 관리자가 아니니까)

 

따라서, CEntity부터 가상함수로 선언되어왔던 `finaltick`을 *final 키워드*를 통해 CObj의 자식클래스부터는 finaltick을 오버라이딩할 수 없음 명시한다

```cpp
class CObj :
    public CEntity
{
	// ...
public:
	virtual void finaltick() final;

	// ...
};
```

→ 이제부터 CPlayer, CMonster, CMissile 등.. 각각의 구체화된 Obj에서는 finaltick을 오버라이딩할 수 없다

### 그렇다면 Collider의 finaltick에서는 어떤 작업이 이루어져야 할까?

Collider는 render를 통해 충돌체 디버그 렌더를 통해 그리기 직전, 즉 finaltick 단계에서 Collider의 최종적인 좌표를 결정해야 한다 → `m_FinalPos`

즉, 여기서 말하는 최종적인 좌표는 Collider가 본인의 주인이 되는 Obj의 위치(Obj의 pos)와 Collider가 이를 기준으로 가지는 상대적인 위치 (`m_OffsetPos`)가 더해진 형태인데…

해당 계산이 이루어지려면 Collider의 주인이 되는 Obj(`m_Ownder`)의 위치 값, pos를 가져와야 하므로 CComponent 단에서 해당 컴포넌트를 소유하고 있는 오브젝트를 반환하는 Get 함수를 구현해준다

 

**GetOwner 구현**

```cpp
#pragma once
#include "CEntity.h"

#include "CObj.h"

class CComponent :
    public CEntity
{
	// ...
public:
	CObj* GetOwner() { return m_Owner; }
	// ...
};
```

그렇다면, m_OffsetPos를 구하는 작업을 작성해보자

```cpp
void CCollider::finaltick()
{
	m_FinalPos = GetOwner()->GetPos() + m_OffsetPos;
}
```

→ 그런데 여기서, `GetOwner()->GetPos()`를 통해 반환되는 값의 타입은 **Vec2**,

`m_OffsetPos`의 타입도 **Vec2**이므로, 

Vec2의 연산자 오버로딩을 구현해 해당 연산이 이뤄질 수 있도록 하자

**Vec2의 연산자 오버라이딩 구현**

```cpp
#pragma once

struct Vec2
{
public:
	float x;
	float y;

public:
	// Vec2에 숫자 하나를 더함으로써 x와 y의 좌표 모두 연산하는 작업 수행
	// ex) Vec2 pos = {0, 0}, pos + 3 일 경우 {3, 3} 으로 변화
	Vec2 operator +(float f) { return Vec2(x + f, y + f); }
	Vec2 operator -(float f) { return Vec2(x - f, y - f); }
	Vec2 operator *(float f) { return Vec2(x * f, y * f); }
	// 나누기(/) 연산은 0으로 나누면 안되니까 assert 처리
	Vec2 operator /(float f) 
	{ 
		assert(f); 
		return Vec2(x / f, y / f); 
	}

	// Vec2를 인자로 받음으로써 x와 y의 좌표 모두 연산하는 작업 수행
	// ex) Vec2 pos = {0, 0}, Vec2 pos2 = {0, 0,} pos +  일 경우 {3, 3} 으로 변화
	Vec2 operator + (Vec2 _Other) { return Vec2(x + _Other.x, y + _Other.y); }
	Vec2 operator - (Vec2 _Other) { return Vec2(x - _Other.x, y - _Other.y); }
	Vec2 operator * (Vec2 _Other) { return Vec2(x * _Other.x, y * _Other.y); }
	Vec2 operator / (Vec2 _Other)
	{ 
		assert(!(0.f == _Other.x || 0.f == _Other.y)); 
		return Vec2(x / _Other.x, y / _Other.y); 
	}
```

→ 이렇게 연산자 오버로딩을 해주면 finaltick의 연산이 손쉽게 가능해진다

### Collider의 출력 타이밍을 상시로 변경하기

CPlayer에서 구현했던 space bar를 통해 출력하는 디버그 렌더를 확인하면

```cpp
DrawDebugRect(PEN_TYPE::PEN_GREEN, GetPos(), Vec2(500.f, 500.f), 3.f);
```

→ 생명주기가 3 임을 알 수 있다

그런데 매 tick마다 DrawDebug가 호출된다면 3초 동안 쌓이게 되는 양이 많아지므로, Collider를 상시 출력하면서 생명주기를 짧게 해 매 순간 출력되고 소멸되는 실시간 반영의 방식으로 변경해보자

```cpp
void CCollider::finaltick()
{
	m_FinalPos = GetOwner()->GetPos() + m_OffsetPos;

	DrawDebugRect(PEN_TYPE::PEN_GREEN, m_FinalPos, m_Scale, 0.f);
}
```

매 CCollider의 finaltick에서 Debug Render를 생성함과 동시에 생명주기를 0으로 만들어, 실시간으로 Collider가 그려짐과 동시에 소멸될 수 있도록 설정한다

### Collider끼리의 충돌 연산 기능을 구현해보자

Player와의 충돌을 검사할, Monster 객체를 생성하자

```cpp
#pragma once
#include "CObj.h"

class CMonster :
    public CObj
{
private:
    CCollider*  m_Collider;       // 소유하고 있는 Collider 멤버 변수

public:
    virtual void tick() override;
    virtual CMonster* Clone() { return new CMonster(*this); }
public:
    CMonster();
    ~CMonster();
};
```

```cpp
#include "pch.h"
#include "CMonster.h"
#include "CCollider.h"

CMonster::CMonster()
{
	// AddComponent를 통해 새로운 Collider를 추가해주고 이 주소를 받아서 멤버 변수로//
	m_Collider = (CCollider*)AddComponent(new CCollider);
	m_Collider->SetScale(Vec2(120.f, 120.f));
}

CMonster::~CMonster()
{
}

void CMonster::tick()
{

}
```

LevelMgr에서, 현재 Level에 포함될 Monster 객체를 추가해줘야 한다

```cpp
void CLevelMgr::init()
{
	// 모든 레벨 생성
	// ...

	// 현재 레벨 지정
	
	// ...

	// 레벨에 물체 추가하기
	CObj* pObject = new CPlayer;
	pObject->SetPos(640.f, 384.f);
	pObject->SetScale(100.f, 100.f);
	m_pCurrentLevel->AddObject(pObject);		

	pObject = new CMonster;
	pObject->SetPos(800.f, 200.f);
	pObject->SetScale(100.f, 100.f);
	m_pCurrentLevel->AddObject(pObject);
}
```

tick()에서 진행되어야 할 작업은 다음시간부터 😤