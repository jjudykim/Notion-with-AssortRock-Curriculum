# 24/05/31 - Debug Render 제작(2), 충돌체 Collider 제작

상위 항목: Week27 (Week27%205552e993c4f84e00b508591b9983697f.md)
주차: 0011_Week20~29

> 1교시 녹음본 -
[https://clovanote.naver.com/s/QWc7vL9HKmQwfcXtG5fTFHS](https://clovanote.naver.com/s/QWc7vL9HKmQwfcXtG5fTFHS)

2교시 녹음본 -
> 

정점은 4개지만 결국 6번의 인덱스를 호출하고있는데,
점 3개씩 끊어서 삼각형으로 보고있기 때문이다.

레스터라이저는 그 내부의 픽셀을 확인해서 PS를 호출한다.

그래픽쉐이더의 m_Topology 를 현재는 트라이앵글을 사용하고있는데, 이것도 여러종류가있음

Line Strip 은 정점의 순서대로 라인만 그려지는 Topology 이고,
Line list는 징검다리처럼 끊어지는 Topology,
한번은 점선, 한번은 실선인것, 이런 여러가지 Topology가 있다.

이중에 Line Strip 을 써보자. 디버그렌더하기위해서!

그래픽쉐이더에 SetTopolgy 하나 만들고

에셋이닛에서 셋토폴로지에서 LINESTRIP으로 설정하자

에셋이닛에서
디버그용으로 조금 변형된 RECT MESH를 만들어보자 (인덱스를 다르게)

0-1-2-3-0 하면 됨

우리가 원하는 line 모양을 따기 위해선

인덱스 버퍼에서 저장한 정점들을 0-1-2-3-0의 순서로 그려져야 한다

Mesh를 Debug 용으로 하나 생성

Circle Mesh의 경우?

Circle Mesh Debug

```cpp
for(size_t i = 1; i < vecVtx.size(); ++i)
{
	vecIdx.push_back(i);
}

pMesh = new CMesh;
pMesh->Create(vecVtx.data(), (UINT)
```

```cpp
void CRenderMgr::RenderDebugShape()
{
	list<tDebugShapeInfo>::iterator iter = m_DebugShapeList.begin();
	
	for (; iter != m_DebugShapeList.end();)
	{
	
	
		switch (DEBUG_SHAPE)
		{
		case DEBUG_SHAPE::RECT:
			CAssetMgr::GetInst()->FindAsset<CMesh>(L"RectMesh_Debug");
			break;
		case DEBUG_SHAPE::CIRCLE:
			CAssetMgr::GetInst()->FindAsset<CMesh>(L"CircleMesh_Debug");
			break;
		// ...
		}
		
		// 디버그 렌더링 전용 재질 가져오기
		pMtrl = CAssetmGr::GetInst()->FindAsset<CMaterial>(L"DebugShapeMtrl");
	}
		
	// 위치 세팅
	
	
	// 재질 세팅
	m_DebugObject->MeshRender()->GetMaterial()->SetScalarParam(VEC4_0, (*iter).vColor);
	
	// 렌더링		
	m_DebugObject->MeshRender()->Render();
	
	
	// 수명이 다한 디버그를 삭제
		(*iter).Age += DT;
		if ((*iter).LifeTime < (*iter).Age)
		{
			iter = m_DebugShapeList.erase(iter);
		}
		else
		{
			++iter;
		}
	}
	

}
```

디버그 렌더링 전용 게임 오브젝트를 생성

```cpp
void CRenderMgr::Init()
{
	m_DebugObject = new CGameObject;
	m_DebugObejct->AddComponent(new CTransform);
	m_DebugObject->AddComponent(new CMeshRender);
	
	// 재질 세팅도 해주기
	m_DebugObject->MeshRender()->SetMaterial(CAssetMgr::GetInst()->FindAsset<CMesh>(L"CircleMesh_Debug);
}
```

나중에 꼭 소멸하는거 잊지말기!

```cpp

```

```cpp
if (KEY_TAP(KEY::SPACE))
{
	DrawDebugRect(Transform()->GetWorldMat(), Vec4(1.f, 0.4f, 0.6f, 1.f), 3.f, true);
}
```

Draw Debug Circle

```cpp
void DrawDebugCircle(Vec3 _Pos, float _Radius, Vec4 _Color, float _Life, bool _DepthTes);
```

```cpp
void DrawDebugCircle(Vec3 _Pos, float _Radius, Vec4 _Color, float _Life, bool _DepthTes)
{
	tDebugShapeInfo Info = {};
	
	Info.Shape = DEBUG_SHAPE::CIRCLE;
	Info.vPos = _Pos;
	Info.vScale = Vec3(_Radius * 2.f, _Radius * 2.f, 1.f);
	Info.vRot = Vec3(0.f, 0.f, 0.f);
	
}
```

Depth Test, 깊이 판정 여부에 따라서 쉐이더의 깊이 판정 방식을 결정

```cpp
if ((*iter).DepthTest)
{
	m_Debug->MeshRender()->GetMaterial()->GetShader()->SetDSType(DS_TYPE::LESS);
}
else
{

}
```

그럼 Debug Render가 생성됐으니까 자기 위치의 충돌체 모양을 Debug Render로 시각화할 수 잇겟다아~~

## 충돌체 Collider

---

<aside>
📁 02. Collider2D > `CCollider.h`,  `CCollider.cpp` 제작!

</aside>

```cpp
class CCollider2D :
	public CComponent
{
private:
	Vec3 m_Offset;
	Vec3 m_Scale;
	Vec3 m_FinalPos;
	
	int m_OverlapCount;
	
public:
	void SetOffset(Vec3 _Offset) { m_Offset = _Offset; }
	void SetScale(Vec3 _Scale) { m_Scale = _Scale; }
	
	Vec3 GetOffset() { return m_Offset; }
	Vec3 GetScale() { return m_Scale; }
	Vec3 GetFinalPos() { return m_FinalPos; }
	
	int GetOverlapCount() { return m_OverlapCount; }
	
public:
	virtual void BeginOverlap(CCollider2D* _Other);
	virtual void Overlap(CCollider2D* _Other);
	virtual void Overlap(CCollider2D* _Other);
	
public:
	virtual void FinalTick() override;

	CLONE(CCollider2D);
	CCollider2D;
	~CCollider2D();
}
```

한 오브젝트가 여러개의 충돌체를 갖고 싶다면 어떻게 해야할까?

현재 구조 → 한 오브젝트가 어떤 종류의 컴포넌트는 하나밖에 지닐 수 없음

계층 구조로 구현

→ 오브젝트의 계층 구조를 설계해보자

```cpp
class CGameObject :
	public CEntity
{
private:
	CComponent* m_arrCom[(UINT)COMPONENT_TYPE::END];
	CRenderComponent* m_RenderCom;
	vector<CScript*>  m_vecScript;
	
	CGameObject*         m_Parent;
	vector<CGameObject*> m_
}
```

업데이트가 부모→자식 순으로 되어야 하는데..

지금 구조상은 Level > Layer 단으로 구분해놨는데, 그럼 Update 순서가 뒤죽박죽일 수 있다

그럼 어떻게 하느냐?

Layer는 부모만 받게 된다. 가장 부모 오브젝트들만!

그럼 자식 오브젝트들은? → 해당 부모인 GameObject들이 들고 있으면서 부모의 Tick이 들어갈 때 부모가 소유하고 있는 자식 Object들의 Tick이 수행되게끔 구현

```cpp
void CGameObject::Tick()
{
	for (UINT i = 0; i < m_vecScript.
}
```

```cpp
void CGameObject::FinalTick()
{
	for
}
```

그럼 자식 GameObejct들은 해당 Layer인 것은 어떻게 알까?

```cpp
class CGameObject:
	public 
{
	
	int 		
}
```

Layer도 부모 자식 상관없는 모든 오브젝트들을 가리키는 vector를 또 하나 구현

```cpp
vector<CGameObject*> m_Parents; // 해당 레이어 소속 오브젝트 중에서 최상위 부모 오브젝트만 상시 관리
vector<CGameObject*> m_Objects; // 해당 레이어 소속 오브젝트 중에서 부모 자식 상관없는 모든 오브젝트 -> 매 프레임마다 등록받는 구조로
```

디폴트 인덱스 값은 -1로 설정 → 어느 레이어 소속도 아니라는 뜻 (Level 안에 있지 않은 상태)

```cpp
CGameObject::CGameObject()
	: m_
{
}
```

그럼 Layer의 AddObject는… 변경되어야 한다?

```cpp
void CLayer::AddObject(CGameObject* _Object. bool _bChildMove)
{
	// 1. 오브젝트가 다른 레이어 소속인 경우
	if (_Object->GetLayerIdx() != -1)
	{
		// 기존에 소속되어 있던 레이어에서 해당 오브젝트를 삭제,
		// 현재 레이어에 해당 오브젝트를 추가
	}
	
	// 2. 부모 오브젝트가 있는 경우
	if (_Object->GetParent())
	{
		_Object->m_LayerIdx = m_LayerIdx;
	}
	// 2-1. 부모 오브젝트가 없는 경우
	else
	{
		m_Parents.push_back(_Object);
	}
	
	m_Parents.push_back(_Object);
}
```

- 경우 1) 오브젝트가 기존에 다른 레이어 소속이었던 경우
- 경우 2) 오브젝트가 부모 오브젝트가 있는 경우 O / X
- 경우 3) 오브젝트가 자식 오브젝트(들)이 있는 경우 O / X

필요한 함수들도 더 추가해 헤더파일을 완성해보자면

```cpp
// ...
CGameObject* GetParent() { return m_Parent; }
int GetLayerIdx() { return m_LayerIdx; }

// ...
friend class CLevel;
friend class C
```