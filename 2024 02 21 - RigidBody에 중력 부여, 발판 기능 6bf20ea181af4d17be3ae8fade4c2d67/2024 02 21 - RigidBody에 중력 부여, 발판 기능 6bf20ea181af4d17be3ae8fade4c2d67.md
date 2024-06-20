# 2024/02/21 - RigidBody에 중력 부여, 발판 기능의 Platform 만들기

태그: C++, WinAPI, 중급
날짜: 2024/02/21
상위 항목: Week13 (Week13%20853476a83b5b4f59bfdc32a469058068.md)
주차: 0010_Week10~19

## RigidBody에 중력 부여하기

지금은 탑 뷰 형태의 게임이다 (like 포켓몬처럼!)

그런데 만약 횡스크롤 게임이라면 player가 어떻게 움직일지 생각해보자

- 좌, 우로 움직이는 기능밖에 없을 것이고..
- 상, 하로 직접 컨트롤 할 순 없고, Jump를 이용해 상, 하를 조절해야 할 것이다

따라서 이를 구현하기 위해, 기존의 코드를 수정해 횡 스크롤 형식의 컨트롤을 구현해보자!

### SpaceBar를 통해 Jump 구현하기

기존에 있던 SpaceBar의 상호작용은 ‘미사일 발사’였다!

→ 이를 나중에 재사용할 수 있도록 함수로 빼버리자 (`Shoot`)

그리고 대신, `Jump`라는 함수를 통해 RigidBody를 활용해 Jump기능을 수행할 수 있도록 작성해주자

```cpp
class CPlayer
	: public CObj
{
private:
	// ...

public:
	// ...

private:
	void Shoot();
	void Jump();
}
```

```cpp
void Cplayer::Shoot()
{
	CMissile* pMissile = new CGuidedMissile;
	pMissile->SetName(L"Missile");

	Vec2 vMissilePos = GetPos();
	vMissilePos.y -= GetScale().y / 2.f;

	pMissile->SetPos(vMissilePos);
	pMissile->SetScale(Vec2(20.f, 20.f));

	SpawnObject(CLevelMgr::GetInst()->GetCurrentLevel(), 
							LAYER_TYPE::PLAYER_MISSILE, 
							pMissile);

	LOG(LOG_TYPE::DBG_WARNING, L"미사일 발사");
}
```

```cpp
void CPlayer::tick()
{
	// ...
	if (KEY_TAP(KEY::SPACE))
	{
		Jump();
	}
	// ...
}

void CPlayer::Jump()
{
	CRigidBody::Jump();	
}
```

```cpp
void CRigidBody::Jump()
{
	// 점프 기능 구현
}
```

### Jump 기능을 위한 중력의 적용 및 설정

**중력의 적용**

- `*중력 가속도*` 멤버 변수 추가
- `*중력에 의해서 증가하는 속도*` 멤버 변수 추가
    
    → 중력에 의해 붙는 속도는 기본 최대 속도 제한이 붙으면 안되므로 별도로 연산!
    
- `*중력으로 인한 속도의 최대 제한*`을 따로 정의 → 멤버 변수 추가
- `중력 적용 유무` 멤버 변수 추가

---

**Jump의 적용**

- `Jump되는 Speed`에 대한 멤버 변수 추가

---

```cpp
class CRigidBody
	: public CComponent
{
private:
	Vec2   m_Velocity               // 속도 (방향, 크기)
	float  m_Mass                   // 질량
	Vec2   m_Force                  // 힘

	float m_GravityAccel;           // 중력 가속도
	Vec2  m_VelocityByGravity;      // 중력에 의해서 증가하는 속도
	
	float m_MaxGravitySpeed;        // 중력으로 인한 속도의 최대 제한

	bool m_UseGravity;              // 중력에 대한 on/off
	
	float m_JumpSpeed;              // 점프 속력
	// ...

public:
	// 추가한 멤버 변수로 인한 멤버 함수 역시 추가
	void SetMaxGravitySpeed(float _Speed) { m_MaxGravitySpeed = _Speed; }
	void SetGravityVelocity(Vec2 _Velocity) { m_VelocityByGravity = _Velocity; }
	void SetJumpSpeed(float _Speed) { m_JumpSpeed = _Speed; }

	Vec2 GetGravityVelocity() { return m_VelocityByGravity; }

	void UseGravity(bool _Use)
	{
		m_UseGravity = _Use;
		if(!m_UserGravity)
			m_VelocityByGravity = Vec2(0.f, 0.f);    // 중력 off
	}
	void Jump();
	// ...
}
```

```cpp
CRigidBody::CRigidBody()
	: // ...
		// 추가한 멤버변수들 초기화 작업 필요!
		m_MaxGravitySpeed(500.f),
		m_GravityAccel(980.f),
		m_UseGravity(false),
		m_JumpSpeed(400.f)
{
}

void CRigidBody::finaltick()
{
	// ...
	// 중력이 On일때만 중력을 받는 속도를 연산!
	if (m_UseGravity)
	{
		// 중력 가속도를 이용해 m_VelocityByGravity 계산
		m_VelocityByGravity += Vec2(0.f, 1.f) * m_GravityAccel * DT;  

		// 최대 속도 보정
		if ( m_MaxGravitySpeed != 0.f && m_MaxGravitySpeed < m_VelocityByGravity.Length())
		{
			m_VelocityByGravity.Normalize();
			m_VelocityByGravity *= m_MaxGravitySpeed;
		}	
	}          

	// 최종 속도
	Vec2 vFinalVelocity = m_Velocity + m_VelocityByGravity;

	// 현재 속도에 따른 이동 (속도 = 거리 / 시간)
	vObjPos += vFinalVelocity * DT;
	GetOwner()->SetPos(vObjPos);
}
```

- 생성자를 통해 초기화한 중력 가속도를 이용해 `m_VelocityByGravity`를 계산해 속도를 누적한다
- 누적되는 속도가 최대 속도를 넘지않도록 보정 작업을 한다
- “최종적인 속도 = 기본 속도 + 중력에 의한 속도” 를 연산한다
- 현재 속도에 따른 이동거리를 계산한 후, 해당 위치를 업데이트한다

**중력 관련 설정은 중력을 적용시킬 RigidBody 컴포넌트의 주인인 Player에서!**

```cpp
CPlayer::CPlayer()
{
	// ...
	// 중력 관련 설정
	m_RigidBody->SetMaxGravitySpeed(1500.f);
	m_RigidBody->SetJumpSpeed(800.f);
}
```

**Jump 기능 구현**

```cpp
void CRigidBody::Jump()
{
	m_VelocityByGravity += Vec2(0.f, -1.f) * m_JumpSpeed;
}
```

***→** 방향*(Jump는 위쪽이니까 `Vec2(0.f, -1.f)`) * *속력*인 *Jump된 속도*가 중력의 영향을 받는 속도에 합산되는 방식으로 구현

### 게임적 연출을 위한 속도의 고정량 세팅

위 방법처럼 가속도를 통해 물리적으로 연산하는 방법도 있지만,

좀 더 캐주얼적인 게임만의 자연스러움을 연출하기 위해 고정된 값으로 속도를 계산할 수도 있다

- `AddVelocity()` : 속도의 고정량을 더해주는 함수
- `SetVelocity()` : 속도를 일정값으로 세팅하는 함수

```cpp
void AddVelocity(Vec2 _Velocity)          // 속도의 고정량을 더해주는 함수
{
	m_Velocity += _Velocity; 
} 

void SetVelocity(Vec2 _Velocity)          // 속도를 일정 값으로 세팅하는 함수
{
	m_Velocity = _Velocity;
}
```

→ 이 두 함수의 호출을 통해 외부에서 속도를 더해주거나 일정량으로 지정할 수 있다

아니면, 아예 현재 받는 중력과 상관없이 고정 속도로 점프하도록 세팅할 수도 있는데,

**즉시 점프 구현**

```cpp
void CPlayer::Jump()
{
	// 중력을 즉시 0으로 설정
	m_RigidBody->SetGravityVelocity(Vec2(0.f, 0.f));

	// 그 상태로 Jump 호출
	m_RigidBody->Jump();
}
```

또는, **더블 점프 시 가속도를 더 붙게** 하고 싶다면

```cpp
void CPlayer::Jump()
{
	// 현재 중력이 있다면 0으로 세팅
	Vec2 vGV = m_RigidBody->GetGravityVelocity();
	if (0.f < vGV.y)
	{
		m_RigidBody->SetGravityVelocity(Vec2(0.f, 0.f));
	}
	
	m_RigidBody->Jump();
}
```

## 발판 기능의 Platform 오브젝트 만들기

<aside>
📁 03. Game > 05. Object > Platform 생성

</aside>

발판이자, 땅(Ground)의 기능을 하는 Plaform은 Player나 다른 Object와의 충돌체 검사를 통해서 그 위에 서있거나, 부딪혀 접근하지 못하는 등의 기능을 한다

따라서

- Collider Component를 가짐
- Begin-On-End Overlap 함수를 오버라이딩함

### Platform 클래스의 틀

```cpp
class CPlatform
	: public CObj
{
private:
	CCollider* m_Collider;

public:
	virtual void tick();
	virtual void BeginOverlap(CCollider* _OwnCollider, 
														CObj* _OtherObj, 
														CCollider* _OtherCollider) override;
	virtual void OnOverlap(CCollider* _OwnCollider, 
												 CObj* _OtherObj, 
												 CCollider* _OtherCollider) override;
	virtual void EndOverlap(CCollider* _OwnCollider, 
													CObj* _OtherObj,
													CCollider* _OtherCollider) override;

	CLONE(CPlatform)
public:
	CPlatform();
	~CPlatform();
};
```

```cpp
CPlatform::CPlatform()
{
	// Platform의 크기 설정
	SetScale(Vec2(500.f, 150.f));

	// 충돌체 컴포넌트 추가(Platform 크기만큼의)
	m_Collider = (Collider*)AddComponent(new CCollider);
	m_Collider->SetScale(GetScale());
}

CPlatform::~CPlatform()
{
}

void CPlatform::tick()
{
}

void CPlatform::BeginOverlap(CCollider* _OwnCollider, CObj* _OtherObj, CCollider* _OtherCollider)
{
	// 충돌이 시작될 때 상호작용
}

void CPlatform::OnOverlap(CCollider* _OwnCollider, CObj* _OtherObj, CCollider* _OtherCollider)
{
	// 충돌 중일 때 상호작용
}

void CPlatform::EndOverlap(CCollider* _OwnCollider, CObj* _OtherObj, CCollider* _OtherCollider)
{
	// 충돌이 종료될 때 상호작용
}
```

### 현재 Level에 Platform을 생성해 추가하자!

레벨 매니저를 통해, 현재 Level에 Platform Object를 생성하고 현재 레벨에 해당 오브젝트를 추가한다. 또, Player와의 레벨 충돌 검사가 이루어줄 수 있도록 CollisionCheck도 설정하기!

```cpp
void CLevelMgr::init()
{
	// ...
	// 플랫폼 생성
	pObject = new CPlatform;
	pObject->SetName(L"Platform");
	pObject->SetPos(Vec2(640.f, 700.f));
	m_pCurrentLevel->AddObject(LAYER::PLATFORM, pObject);

	// ...
	// 레벨 충돌도 설정 (플레이어 - 플랫폼 간 충돌 체크 시행)
	CCollisionMgr::GetInst()->CollisionCheck(LAYER_TYPE::PLAYER, 
																					 LAYER_TYPE::PLATFORM);	
}
```

### Platform 위에서의 중력 설정

Object가 Platform위에 올라와 있는 상태에서 계속해서 중력 가속도가 붙는다면 Object의 위치가 안정적이지 못하고 계속해서 중력이 누적될 것이다. 

따라서, RigidBody의 구현에서 Platform 위에 Object가 서있는 상태를 `m_Ground`으로 명명하고, 해당 상태에서는 중력에 의한 속도 증가 연산이 이루어지지 않도록 해야할 것!

**RigidBody 변경**

- `m_Ground` 멤버 변수 추가 : 현재 RigidBody가 Platform위에 있는 상태인지?
- `SetGround` 멤버 함수 추가 : m_Ground에 따라 중력에 의한 속도 설정
- `IsGround` 멤버 함수 추가 : m_Ground의 get함수

```cpp
class CRigidBody
	: public CComponent
{
private:
	// ...
	bool m_Ground;       // 땅(Platform) 위에 서있는지 체크

public:
	void SetGround(bool _Ground)
	{
		m_Ground = _Ground;

		if (m_Ground)
		{
			// m_Ground가 true, 즉 flatform 위인 경우 중력에 의한 속도를 0으로 설정
			m_VelocityByGravity = Vec2(0.f, 0.f);
		}
	}

	bool IsGround() { return m_Ground; }
}
```

- Jump시 `m_Ground`는 false로 변경
- 공중에서 (`m_Ground`가 false인 상태에서) 좌,우로 움직이는 힘이 적용될 경우에 대해 설정 → 공중일 때는 땅 위에서의 1/2로 적용
- 공중에서는 마찰력으로 인한 속도의 상쇄를 땅 위에서의 1/5로 적용

→ 여기서 적용되는 힘이나 마찰 계수는 변수화하는 것이 좋겠지!

```cpp
void CRigidBody::Jump()
{
	//...
	// 점프 당시에는 Ground의 상태가 false가 되면서 중력 적용 X
	m_Ground = false;
}

void CRigidBody::finaltick()
{
	// ...
	// 중력을 사용할 때, 공중에서 힘이 적용된 경우
	if (m_UseGravity && !m_Ground)
	{
		m_Velocity += vAccel * DT * 0.5f;
	}
	else
	{
		m_Velocity += vAccel * DT * 1.f;
	}

	// 적용되는 힘이 없을 때 마찰에 의해서 속도를 줄이는 경우
	// 중력을 사용하고, 공중 상태인 경우 마찰을 더 적게 적용한다
	if (m_UseGravity && !m_Ground)
	{
		// 마찰을 더 적게 적용한다
		Speed -= m_Friction * DT * 0.2f;
	}
	else
	{
		Speed -= m_Friction * DT;
	}
}
```

### 그렇다면 m_Ground는 어디서 true / false로 변경되어야 할까?

→ Player와 platform이 충돌된 상태여야 m_Ground가 될 수 있으므로, 충돌 체크로 스위치!

```cpp
// 충돌이 시작될 때
void CPlatform::BeginOverlap(CCollider* _OwnCollider, CObj* _OtherObj, CCollider* _OtherCollider)
{
	if (_OtherObj->GetName() == L"Player")
	{
		// Player의 RigidBody를 가져와 m_Ground를 true로 변경
		CRigidBody* pRB = _OtherObj->GetComponent<CRigidBody>();
		pRB->SetGround(true);
	}
}

// 충돌이 종료될 때
void CPlatform::EndOverlap(CCollider* _OwnCollider, CObj* _OtherObj, CCollider* _OtherCollider)
{
	if (_OtherObj->GetName() == L"Player")
	{
		// Player의 RigidBody를 가져와 m_Ground를 false로 변경
		CRigidBody* pRB = _OtherObj->GetComponent<CRigidBody>();
		pRB->SetGround(false);
	}
}
```

### Player를 Platform 위에 잘 올리는(?) 방법에 대한 아이디어

현재 진행 상황에서는 Player가 어느 방향에서 Platform에 접근을 하더라도 충돌체크가 되면서 m_Ground 판정으로 인해 중력을 잃게 된다!

따라서 끼임이 발생하거나.. 이상한 위치에서 땅을 밟았다고 인식하는 문제점이 발생한다

> 따라서 Player가 Platform을 향해 온 진입 각도에 따라 충돌체의 적용 여부를 설정하는 아이디어로 해당 문제점을 해결해볼 것이다!
> 

그럼 진입 각도는 어떻게 알 수 있을까? 🤔

→ Player의 이전 위치와 현재 위치에 따른 **y값**의 차이로!

이전 위치가 어디였냐에 따라서 y값에 차이에 따른 진입 각도가 달라지기 때문에, y 좌표를 기준으로 

→ y 좌표를 기준으로, 

- `CurrentPos의 y좌표 < PrevPos의 y좌표`의 결과가 true일 경우 m_Ground가 가능(Platform 위에 올라가기)하지만 해당 결과가 false일 경우에는 불가능하다!
- 머릿속으로 떠올리기 어렵다면 그림으로 단순히 생각해보기…

![Untitled](2024%2002%2021%20-%20RigidBody%E1%84%8B%E1%85%A6%20%E1%84%8C%E1%85%AE%E1%86%BC%E1%84%85%E1%85%A7%E1%86%A8%20%E1%84%87%E1%85%AE%E1%84%8B%E1%85%A7,%20%E1%84%87%E1%85%A1%E1%86%AF%E1%84%91%E1%85%A1%E1%86%AB%20%E1%84%80%E1%85%B5%E1%84%82%E1%85%B3%E1%86%BC%206bf20ea181af4d17be3ae8fade4c2d67/Untitled.png)

따라서,

**이전 좌표를 기록하자**

- 모든 위치변경 명령이 일어나기 전, 현재 Pos를 이전 Pos로 저장해놓아야 하므로 CObj의 tick에서 해당 작업을 수행했다
- 단, 모든 Obj들의 자식 클래스는 tick을 오버라이딩해 사용하므로, 꼭 자식 클래스의 tick에서 `Obj의 tick`을 호출해주어야 한다! (번거롭지만 현재 찾은 방법이 이거라..)

```cpp
class CObj
	: public CEntity
{
private:
	Vec2      m_Pos;      // 현재 위치
  Vec2      m_PrevPos;  // 이전 프레임에서의 위치
// ...
}
```

```cpp
void CObj::tick()
{
	m_PrevPos = m_Pos();
}
```

```cpp
void CPlayer::tick()
{
	CObj::tick();
	// ...
}
```