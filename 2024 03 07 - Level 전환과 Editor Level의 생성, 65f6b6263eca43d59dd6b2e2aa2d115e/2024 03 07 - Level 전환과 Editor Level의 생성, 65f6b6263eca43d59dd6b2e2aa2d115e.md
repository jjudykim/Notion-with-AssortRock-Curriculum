# 2024/03/07 - Level 전환과 Editor Level의 생성, TileMap에 대한 아이디어

태그: C++, WinAPI, 중급
날짜: 2024/03/07
상위 항목: Week15 (Week15%204207aa697c6145e0959981e524666a76.md)
주차: 0010_Week10~19

## Level 전환하기

### Level에 대한 기준 정의하기

Level 전환에 대해 구현하기 전 고민해볼 것은, 내 게임에서는 Level에 대한 단위를 어떻게 나눌지에 대한 것이다

게임 스테이지 전체를 한 레벨로 구성할 수도 있고, 지역 단위로 레벨을 구성할 수도 있고… 게임에 맞게 본인의 선택에 따라서 구성하면 된다

예를 들어보면…

- 슈퍼마리오 → 1-1의 스테이지가 한 레벨
- 스타듀밸리 → 한 지역이 한 레벨 (광산이나 사막.. 같은 경우에는 다른 레벨)

따라서 내가 구현해야 하는 게임의 Level 단위는 어떤 것이 적절할지 잘 고민해보기

### Level의 전환

Level에 대한 전환은, 어떻게 보면 하나의 State Machine처럼 생각할 수 있다

즉, 한 State에서 다른 State로 전환되는 과정과 비슷하게 Level에서 다른 Level로 전환되는 과정에 대해서 고민해보면 된다!

따라서 State가 변경되는 것처럼,

- Level을 진입할 때 → `Enter()`
- Level을 빠져 나올 때 → `Exit()`

와 같이 함수를 작성해 Level의 전환 과정에서 이루어져야 할 작업들에 대해 고민하고 작성해보자

CLevel에서 `Enter()`과 `Exit()`의 **추상화** → to 순수 가상함수로..

```cpp
class CLevel
	: public CEntity
{
public:
	// ...
	virtual void Enter() = 0;        // 레벨이 전환되고 처음 초기화 작업
	virtual void Exit() = 0;         // 레벨이 끝날 때 할 일
	// ...
}

```

- `Enter()`에서는 Level이 전환된 후 처음으로 초기화되는 작업들을,
- `Exit()`에서는 Level이 전환되기 전 종료 시에 해야하는 작업들을 작성해주자

이를 CLevel_Stage01 을 대상으로 구체화해보자!

```cpp
class CLevel_Stage01
	: public CLevel
{
	// ...
public:
	// ...
	virtual void Enter() override;
	virtual void Exit() override;
	// ...
}
```

### Level 전환의 흐름, Level 전환의 주체는? → LevelMgr

직접적으로 현재 Level을 변경할 수 있는 권한을 갖는 것은 `LevelMgr`이다!

따라서 ChangeLevel이라는 Level 변경 함수를 작성하고, 이를 private으로 설정하자

private으로 설정한 이유는, **해당 함수를 TaskMgr를 통해서만 호출**하려 하기 때문이다

- 다른 곳에서 함부로 호출될 수 없도록 해당 함수는 private 영역에 선언,
- 이 함수를 TaskMgr에서 호출할 수 있도록 LevelMgr에서 `freind class taskMgr;` 추가

```cpp
class CLevelMgr
{
	SINGLE(CLevelMgr)
	// ...
private:
	void ChangeLevel(LEVEL_TYPE _NextLevelType);
	// ...
	
	friend class CTaskMgr;
}
```

```cpp
void CLevelMgr::ChangeLevel(LEVEL_TYPE _NextLevelType)
{
	if (m_arrLevel[(UINT)_NextLevelType] == m_pCurrentLevel)
	{
		LOG(LOG_TYPE::DBG_ERROR, L"현재 레벨과 변경하려는 레벨이 동일합니다.");
		return;
	}
	
	// 기존 Level에서 Exit
	if (m_pCurrentLevel)
	{
		m_pCurrentLevel->Exit();
	}
	
	// 새로운 Level로 포인터의 주소값을 교체
	m_pCurrentLevel = m_arrLevel[(UINT)_NextLevelType];
	assert(m_pCurrentLevel);
	
	// 변경된 새로운 레벨로 진입(Enter) 후 시작(begin)
	m_pCurrentLevel->Enter();
	m_pCurrentLevel->begin();
}
```

그리고 `ChangeLevel`이라는 전역 함수를 별도로 만들어, 

어느 시점에서든 원하는 때에 Level 전환을 호출해 TaskMgr를 통해 Level 전환이라는 Task를 추가해주고, 해당 Task가 실행될 때 CLevelMgr의 ChangeLevel 함수가 호출될 수 있도록 로직을 구현해보자!

---

즉, 정리하자면

원하는 시점에서 `::ChangeLevel()` 호출

→ TaskMgr의 Task목록에 Level 전환에 대한 Task 추가

→ 해당 Task의 tick이 실행될 때 `CLevelMgr::ChangeLevel()` 호출

의 흐름인 것!

---

따라서 `func.h` / `func.cpp`에 전역함수 `ChangeLevel`을 추가해준다

```cpp
// ...
// ==================
// Task 관련 함수 
// ==================
// ...
void ChangeLevel(LEVEL_TYPE _NextLevelType);
```

```cpp
void ChangeLevel(LEVEL_TYPE _NextLevelType)
{
	tTask task = {};
	task.Type = TASK_TYPE::CHANGE_LEVEL;
	task.Param1 = (DWORD_PTR)_NextLevelType;
	
	// Level 전환에 대한 Task 추가
	CTaskMgr::GetInst()->AddTask(task);
}
```

그럼, 이렇게 생성된 Level 전환에 대한 Task는 **TaskMgr에서 처리**될 것이다

```cpp
void CTaskMgr::ExecuteTask()
{
	// 레벨 전환이 이루어졌는지에 대한 static bool 변수
	static bool bLevelChanged = false;
	bLevelChanged = false;
	
	for (size_t i = 0; i < m_vecTask.size(); ++i)
	{
		// ...
		case TASK_TYPE::CHANGE_LEVEL:
		{
			// 같은 레벨에서 레벨 전환을 연달아 하려는 경우 예외 처리
			assert(!bLevelChanged));
			bLevelChanged = true;
			
			LEVEL_TYPE NextType = (LEVEL_TYPE)m_vecTask[i].Param1;
			CLevelMgr::GetInst()->ChangeLevel(NextType);
			break;
		}
	}
}
```

- 그런데 여기서 고려해야할 점이, 한 Task 목록에서 Level 전환에 대한 요구가 연달아 들어올 경우에는 충돌이 발생할 수 있다
    
    따라서 `bLevelChanged`라는 Static 변수를 활용!
    
    →  ExecuteTask 함수를 통해 이미 CHANGE_LEVEL의 Task가 실행된 경우에는 해당 bool이 true로 변경되어, 이미 true인 경우에는 assert를 통해 예외 처리
    

```cpp
void CLevelMgr::ChangeLevel(LEVEL_TYPE _NextLevelType)
{
	// 현재 레벨과 변경하려는 레벨이 같은 경우 예외 처리
	if (m_arrLevel[(UINT)_NextLevelType] == m_pCurrentLevel)
	{
		LOG(LOG_TYPE::DBG_ERROR, L"현재 레벨과 변경하려는 레벨이 동일합니다.");
		return;
	}
	
	if (m_pCurrentLevel)
	{
		m_pCurrentLevel->Exit();
	}
	e
}
```

### Level의 Enter와 Exit의 생성으로 인한 코드 정리

여태까지 LevelMgr의 init에서 구현되었던 부분들 (즉, Level_Stage01의 오브젝트, 컴포넌트 설정..)은 모두 ****Level_Stage01의 Enter에서 구현되어야 한다!

→ 즉, 모든 Object들은 ***Level 단위***로 구성되어야 한다

1. **CLevel_Stage01의 Enter() 작성하기**

```cpp
void CLevel_Stage01::Enter()
{
	// 레벨에 물체 추가하기
	CObj* pObject = new CPlayer;
	pObject->SetName(L"Player");
	pObject->SetPos(640.f, 384.f);
	pObject->SetScale(100.f, 100.f);               
	AddObject(LAYER_TYPE::PLAYER, pObject);
	// 만약 다음 프레임부터 설정되길 원한다면..?
	//-> SpawnObject(this, LAYER_TYPE::PLAYER, pObject);

	// 레벨 충돌 설정하기
	// 여태 저장되어 있었던 CollisionCheck에 대한 정보가 있을 수 있으니 한번 초기화해주자!
	CCollisionMgr::GetInst()->CollisionCheckClear();
	CCollisionMgr::GetInst()->CollisionCheck(LAYER_TYPE::PLAYER, LAYER_TYPE::MONSTER);
	// ...
}

void CLevel_Stage01::Exit()
{
	// 레벨에 있는 모든 오브젝트를 삭제한다.
	// -> Level 단에서 모든 오브젝트를 삭제하는 함수를 정의할 필요가 생겼다
}
```

1. **CLevel의 Object 삭제 기능 함수 추가하기**
- CLevel을 상속받는 Level 단위의 클래스들만 호출할 수 있도록 protected로 작성

```cpp
class CLevel
	: public CEntity
{
private:
	//...
public:
	// ...

protected:
	// 해당 레벨에 존재하는 모든 오브젝트들을 삭제
	void DeleteAllObejcts();

	// 해당 레벨의 특정 레이어에 존재하는 오브젝트만 삭제
	void DeleteObjects(LAYER_TYPE _LayerType);
}
```

```cpp
void CLevel::DeleteAllObejcts()
{
	for (UINT i = 0; i < (UINT)LAYER_TYPE::END; ++i)
	{
		DeleteObjects((LAYER_TYPE)i);
	}
}

void CLevel::DeleteObjects(LAYER_TYPE _LayerType)
{
	// 현재 Level의 해당 Layer에 담겨있는 객체 vector를 단일 vector로 옮겨
	vector<CObj*>& vecObejcts = m_arrObj[(UINT)_LayerType];

	// 해당 vector의 내용물 모두 메모리 해제!
	for (size_t i = 0; i < vecObjects.size(); ++i)
	{
		delete vecObjects[i];
	}

	vecObjects.clear();
}
```

### Level이 전환된 후 중간에 Object가 Spawn 된다면?

LevelMgr에서 레벨 전환이 이루어질때, 변경된 새로운 레벨의 Enter 후 해당 Level의 begin 작업이 수행되는데, 이를 통해서 각 Object는 override한 자신의 begin을 수행한다!

```cpp
void CLevelMgr::ChangeLevel(LEVEL_TYPE _NextLevelType)
{
	// ...
	// 변경된 새로운 레벨로 Enter한다
	m_pCurrentLevel->Enter();      // 진입
	m_pCurrentLevel->begin();      // 레벨이 시작
}
```

ex) Player의 begin

```cpp
void CPlayer::begin()
{
	// CallBack 설정
	// ...
	// Delegate 설정
	// ...
}
```

그런데 이때, Enter를 통해 생성된 Object가 아닌, 중간에 Object가 `SpawnObject`를 통해 투입된다면

해당 Object 입장에서는 *Level에 스폰된 시점부터가 Level의 시작*인 개념이다!

따라서 `SpawnObject`가 이루어질 당시에 해당 오브젝트의 begin 작업이 수행되어야 한다

```cpp
void CTaskMgr::ExecuteTask()
{
	// ...
	for (size_t i = 0; i < m_vecTask.size(), ++i)
	{
		case TASK_TYPE::SPAWN_OBJECT:
		{
			CLevel* pSpawnLevel = (CLevel*)m_vecTask[i].Param1;
			LAYER_TYPE Layer = (LAYER_TYPE)m_vecTask[i].Param2;
			CObj* pObj = (CObj*)m_vecTask[i].Param3;

			if (CLevelMgr::GetInst()->GetCurrentLevel() != pSpawnLevel)
			{
				delete pObj;
			}
			pSpawnLevel->AddObject(Layer, pObj);
			pObj->begin();                  // 추가된 Object는 바로 begin이 될 수 있도록
		}
	}
}
```

## Editor Level 제작하기

### Editor Level에서 이루어져야 할 작업?

Editor Level→ 스테이지에서 *필요한 정보들을 제작하고 불러오는 작업*을 위한 장치로

게임 개발 과정을 단순화하고, 시각적으로 게임의 요소들을 조작할 수 있게 해주는 도구(Tool)!

ex) Object들의 초기 위치값을 설정해주거나, 파일 입출력으로 Object에 대한 설치 정보를 저장

<aside>
✍️ **Editor Level을 통해 일반적으로 수행할 수 있는 작업들…**

1. 레벨 디자인 및 구성
2. 오브젝트 배치 및 조정
3. 속성 및 행동 정의
4. 이벤트 및 트리거 설정
5. 시각효과 조정
6. 사운드 및 음악 관리
7. 플레이 테스트 및 디버깅
8. 데이터 관리 및 최적화
9. UI/UX 디자인

</aside>

그러나! Editor는 시각적으로 게임의 요소들을 확인하기 위함이며, 즉, 예상도를 작성해보는 작업에 불과하다. 실질적인 의의는 게임 데이터를 제작하고 저장해 개발의 효율성을 증가하기 위함이다!

### Editor Level의 추가

**CLevel_Editor의 기본적인 틀 작성**

```cpp
class CLevel_Editor
	: public CLevel
{
public:
	virtual void begin() override;
	virtual void tick() override;
	
	virtual void Enter() override;
	virtual void Exit() override;
	
public:
	CLevel_Editor();
	~CLevel_Editor();
}
```

이렇게 생성한 레벨을 CLevelMgr에도 추가해주자!

```cpp
void CLevelMgr::init()
{
	m_arrLevel[(UINT)LEVEL_TYPE::EDITOR] = new CLevel_Editor;
	m_arrLevel[(UINT)LEVEL_TYPE::STAGE_01] = new CLevel_Stage01;
	
	// 초기 레벨을 에디터 레벨로 지정해주기
	::ChangeLevel(LEVEL_TYPE::EDITOR);
}
```

## Tile

이렇게 생성한 Editor에서 첫번째로 관리해볼 것은 Map을 구성하는 Tile들에 대한 정보다!

Tile 기반의 게임들을 보면, 한 칸 한 칸의 Tile들의 이미지가 모여 하나의 Map을 구성하는 것을 볼 수 있다

![Untitled](2024%2003%2007%20-%20Level%20%E1%84%8C%E1%85%A5%E1%86%AB%E1%84%92%E1%85%AA%E1%86%AB%E1%84%80%E1%85%AA%20Editor%20Level%E1%84%8B%E1%85%B4%20%E1%84%89%E1%85%A2%E1%86%BC%E1%84%89%E1%85%A5%E1%86%BC,%2065f6b6263eca43d59dd6b2e2aa2d115e/Untitled.png)

요런식으로..?

따라서 한 칸 한 칸의 Tile을 Object로 한 번 제작해보자!

**Object CTile의 기본적인 틀 작성**

```cpp
class CTile
	: public CObj
{
	virutal void begin() override;
	virutal void tick() override;
	virutal void render() override;

public:
	CLONE(CTile);

public:
	CTile();
	~CTile();
}
```

**Tile의 크기 설정** 

→ 64pixel로!

매크로를 통해 정의해서 나중에 변경이 용이하도록 작성하자

```cpp
#define TILE_SIZE 64
```

```cpp
CTile::CTile()
{
	SetScale(Vec2(TILE_SIZE, TILE_SIZE));       // Tile Object 하나는 64x64의 크기를 가짐
}
```

### Tile 객체를 Level에서 생성하고, 이를 Render해보자

Editor Level에서 Tile Object를 생성하고 추가해보자

```cpp
void CLevel_Editor::Enter()
{
	CObj* pTile = new CTile;
	AddObject(LAYER_TYPE::TILE, pTile);
}
```

그리고 해당 Level이 render될 때,

추가한 Tile 역시 render될 수 있도록 CTile에서 render를 구현해주자

```cpp
void CTile::render()
{
	Vec2 vRenderPos = GetRenderPos();
	Vec2 vScale = GetScale();

	// 비어있는 초록색 테두리의 사각형이 출력
	USE_PEN(DC, PEN_TYPE::PEN_GREEN);
	USE_BRUSH(DC, BRUSH_TYPE::BRUSH_HOLLOW);
	Rectangle(DC, (int)vRenderPos.x, (int)vRenderPos.y
					    , (int)vRenderPos.x + (int)vScale.x, (int)vRenderPos.y + (int)vScale.y);
}
```

- 여기서, 보통 다른 Animation을 가진 Object들은 해당 객체의 중심점을 Pos의 기준으로 잡았기 때문에 시작 위치를 renderPos - 이미지 크기 / 2 를 해주는 것이 일반적이었다!
- 그러나 Tile은 좌상단을 기준으로 Pos를 계산하기 때문에 render시에도 시작 위치를 그대로 좌상단을 기준으로 한 Pos에, 해당 이미지의 크기만큼을 그려준다!

→ 즉, ***다른 Object들은 Pos를 해당 객체의 중심점을 기준으로 하고, Tile은 좌상단을 기준으로 한다***

### 현재 Tile 설계의 문제점

한 칸 한 칸의 Tile을 Object로 제작하면, 한 Map을 구성하는 Tile은 Level에 따라서 수개 ~ 수만개가 될 수 있는데, 이를 모두 Object를 제작할 경우 tick이나 finaltick같은 Object 고유의 작업들에 영향을 줄 수 있기 때문에 프로그램이 무거워지고 최적화가 어려워진다!

그럼 어떻게 개선할 수 있을까?

→ Tile Object 하나는 타일이 조합된 하나의 큰 Map이라고 생각하고, Tile 각자에 대한 내용은 Component로 관리하자! 

해당 Component가 Tile 정보에 대한 행렬을 갖고 이에 대한 Owner인 Tile이 관리하도록?

이에 대한 설계는 다음 시간부터…. 😽