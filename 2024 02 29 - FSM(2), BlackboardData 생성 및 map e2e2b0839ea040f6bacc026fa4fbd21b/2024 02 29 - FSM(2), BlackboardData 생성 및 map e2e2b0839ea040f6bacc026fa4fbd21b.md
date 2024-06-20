# 2024/02/29 - FSM(2), BlackboardData 생성 및 map 활용, State별 기능 구현

태그: C++, WinAPI, 중급
날짜: 2024/02/29
상위 항목: Week14 (Week14%202545b77261d946f984f92ede76a23ddb.md)
주차: 0010_Week10~19

## BlackboardData를 추가하고, 가져오기

### BlackboardData를 생성하고, map에 추가해보자

FSM에서 BlackboardData를 생성하고 map에 insert하는 SetBlackboardData 함수를 작성

```cpp
class CFSM
	: public CComponent
{
	// ...
public:
		void SetBlackboardData(const wstring& _DataKey, DATA_TYPE _Type, void* _pData);
}
```

```cpp
void CFSM::SetBlackboardData(const wstring& _DataKey, DATA_TYPE _Type, void* _pData)
{
	tBlackboardData data = { _Type, _pData };
	m_mapData.insert(make_pair(_DataKey, data);
}
```

객체의 begin에서 해당 객체의 FSM에 BlackboardData를 추가함으로써 정보를 넣어놓는다

```cpp
CMonster::begin()
{
	m_FSM->SetBlackboard(L"DetectRange", DATA_TYPE::FLOAT, &m_DetectRange);
}
```

### BlackboardData를 Key값을 통해 저장한 내용을 가져오자

BlackboardData는 구조체로, DATA_TYPE의 `Type`과 void*의 `pData`으로 구성되어 있다.

따라서 우리가 실질적인 데이터를 저장해놓은 pData는 void* 타입이기 때문에, 정보를 가져올 때마다 캐스팅을 해서 가져와야 한다

→ 따라서 함수 템플릿을 활용해서 구현!

```cpp
class CFSM
	: public CComponent
{
	// ...
public:
	template<typename T>
	T& GetBlackboardData(const wstring& _DataKey);
}
```

```cpp
template<typename T>
inline T CFSM::GetBlackboardData(const wstring& _DataKey)
{
	map<wstring, tBlackboardData>::iterator iter = m_mapData.find(_DataKey);
	
	assert(iter != m_mapData.end());

	// 내가 지정한 T가 int였다면 true를 반환, 아니라면 false를 반환
	if constexpr (std::is_same_v<int, T>)
	{
		assert(DATA_TYPE::INT == iter->second.Type);
		return *((T*)iter->second.pData);
	}

	// 내가 지정한 T가 float였다면 true를 반환, 아니라면 false를 반환
	if constexpr (std::is_same_v<float, T>)
	{
		assert(DATA_TYPE::FLOAT == iter->second.Type);
		return *((T*)iter->second.pData);
	}

	// 내가 지정한 T가 Vec2였다면 true를 반환, 아니라면 false를 반환
	if constexpr (std::is_same_v<Vec2, T>)
	{
		assert(DATA_TYPE::VEC2 == iter->second.Type);
		return *((T*)iter->second.pData);
	}

	// 내가 지정한 T가 CObj였다면 true를 반환, 아니라면 false를 반환
	if constexpr (std::is_same_v<CObj, T>)
	{
		assert(DATA_TYPE::OBJECT == iter->second.Type);
		return *((T*)iter->second.pData);
	}
} 

```

- `constexpr` 키워드를 활용할 수 있다
    
    → `if constexpr` : 런타임이 아닌 컴파일 시간에 조건 체크를 진행하고 거짓에 해당하는 분기문은 컴파일 과정에서 생략해버린다. 실행 파일의 크기가 감소하고, 불필요한 코드 실행이 방지되는 조건부 컴파일 기능이다.
    

### BlackboardData의 활용 방법

만약 FindObjectByName이라는 함수를 통해, Monster의 대상이 되는 Player 객체를 찾으려고 한다고 가정해보자! 그럼 Monster는 자신의 탐지 범위 내에 Player가 위치하는지를 확인하기 위해서 매 tick마다 계속해서 Player 객체를 찾아야 하는데, 이럴 경우 프레임 드랍이 심하기 때문에…

차라리 Monster의 begin에서, Object 타입으로 자신의 타겟이 되는 Player 객체를 BlackboardData로 저장해놓는다면, 쉽게 Player 객체의 위치를 비교할 수 있을 것이다

먼저, LevelMgr 단에서 작성

```cpp
class CLevelMgr
{
	// ...
public:
	CObj*  FindObjectByName(const wstring& _strName);
	// ...	
}
```

```cpp
CObj* CLevelMgr::FindObjectByName(const wstring& _strName)
{
	// 각 Level단에 있는 FindObject 함수 호출
	return m_pCurrentLevel->FindObjectByName(_strName);
}
```

- 참고로 여기서 호출하는 Level단의 FindObjectByName 함수는 이렇게 작성했었다
    
    ```cpp
    CObj* CLevel::FindObjectByName(LAYER_TYPE _Type, const wstring& _Name)
    {
    	for (size_t i = 0; i < m_arrObj[(UINT)_Type].size(); ++i)
    	{
    		if (_Name == m_arrObj[(UINT)_Type][i]->GetName())
    		{
    			return m_arrObj[(UINT)_Type][i];
    		}
    	}
    
    	return nullptr;
    }
    ```
    

Monster의 begin에서, 본인과 본인의 타겟인 Player를 Object 타입으로 Blackboard에 추가한다

```cpp
void CMonster::begin()
{
	m_FSM->SetBlackboardData(L"DetectRange", DATA_TYPE::FLOAT, &m_DetectRange);
	m_FSM->SetBlackboardData(L"Speed", DATA_TYPE::FLOAT, &m_Speed);
	m_FSM->SetBlackboardData(L"Self", DATA_TYPE::OBJECT, this);
	
	CObj* pPlayer = CLevelMgr::GetInst()->FindObjectByName(L"Player");
	m_FSM->SetBlackboardData(L"Target", DATA_TYPE::OBJECT, pPlayer);

	m_FSM->ChangeState(L"Idle");       // Idle State로 시작한다는 뜻
}
```

## State의 기능 구현

위에서 추가한 Player의 객체에 대한 BlackboardData를 활용해, Monster가 Idle State일 경우 Player의 위치를 계속해서 탐지할 수 있도록 기능을 구현해주자

`Idle` State의 FinalTick에서는 **플레이어를 탐지**한다

```cpp
void CIdleState::FinalTick()
{
	// 근처(탐지 범위)에 플레이어가 있는지 탐지하기 위해 가져온 BlackboardData들
	float Range = GetBlackboardData<float>(L"DetectRange");
	CObj* pSelf = GetBlackboardData<CObj*>(L"Self");
	CObj* pPlayer = GetBlackboardData<CObj*>(L"Target");
	
	// 몬스터의 탐지 범위를 시각화하기
	DrawDebugCircle(PEN_TYPE::PEN_GREEN, pSelf->GetPos(), Vec2(Range * 2.f, Range * 2.f), 0);
	
	// 플레이어가 감지되면, Trace 상태로 변경하기
	if (pPlayer->GetPos().GetDistance(pSelf->GetPos()) < Range)
	{
			GetFSM()->ChangeState(L"Trace");
	}
}
```

`Trace` State의 FinalTick에서는 **플레이어를 추적**한다

```cpp
void CTraceState::FinalTick()
{
	float Range = GetBlackboardData<float>(L"DetectRange");
	float Speed = GetBlackboardData<float>(L"Speed");
	CObj* pSelf = GetBlackboardData<CObj*>(L"Self");
	CObj* pPlayer = GetBlackboardData<CObj*>(L"Target");
	
	// 플레이어를 추적한다
	Vec2 vDir = pPlayer->GetPos() - pSelf->GetPos();       // 플레이어를 향한 방향 계산
	if (!vDir.IsZero())
	{
		vDir.Normalize();                                     // 단위벡터로 방향 설정
		Vec2 vPos = pSelf->GetPos() + (vDir * Speed * DT);    // 해당 방향으로 이동
		pSelf->SetPos(vPos);
	}
}
```