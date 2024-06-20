# 24/06/11 - FlipBook Component(2) - Component 구성원 제작과 Animation 출력

상위 항목: Week29 (Week29%208c950cb34a664d9ab3834144e1984a6f.md)
주차: 0011_Week20~29

> 1교시 녹음본
- [https://clovanote.naver.com/s/uPxZtyqPJYHbXiRpRFGb5jS](https://clovanote.naver.com/s/uPxZtyqPJYHbXiRpRFGb5jS)
2교시 녹음본
- [https://clovanote.naver.com/s/Wcj8QLzKMd7PEfwGDjKotS](https://clovanote.naver.com/s/Wcj8QLzKMd7PEfwGDjKotS)
> 

## FlipBook과 Sprite를 제작해보자

### Sprite을 제작하고, 이를 모아 FlipBook을 제작해보자

하나의 FlipBook을 만들기 위한 구성원이 될 Sprite를 한번 제작해보자

Asset Manager에서, Sprite를 제작하는 함수를 작성할 것이다

**Sprite 제작 함수 CreateEngineSprite 작성하기**

```cpp
class CAssetMgr
	: public CSingleton<CAssetMgr>
{
	SINGLE(CAssetMgr);
	// ...
	
private:
	// ...
	void CreateEngineSprite();
	// ...
};
```

```cpp
// ...
void CreateEngineSprite()
{
	// 사용할 Atlas Texture를 Load한 후,
	Ptr<CTexture> pAtlasTex = Load<CTexture>(L"texture\\link_32.bmp", L"texture\\link_32.bmp");
	
	// 사용할 만큼의 Sprite를 제작한다 -> 10개 제작
	Ptr<CSprite> pSprite = nullptr;
	
	for (int i = 0; i < 10; ++i)
	{
		wchar_t szKey[50] = {};
		swprintf_s(szKey, 50, L"Link_Move_Down_&d", i);
		
		pSprite = new CSprite;
		pSprite->Create(pAtlasTex, Vec2(0.f, 520.f), Vec2(120.f, 130.f));
		
		// 생성한 Sprite를 새로운 에셋으로 추가
		AddAsset(szKey, pSprite);
	}
}
```

→ 이렇게 생산한 10장의 Sprite를, 하나의 FlipBook으로 제작해보자 (같은 함수에 구현)

 

**FlipBook 제작 함수를 CreateEngineSprite에 이어서 작성**

```cpp
void CreateEngineSprite()
{
	// Sprite 생성
	// ...
	
	// FlipBook 생성
	Ptr<CFlipBook> pFlipBook = nullptr;
	pFlipBook = new CFlipBook;
	
	for(int i = 0; i < 10; ++i)
	{
		wchar_t szKey[50] = {};
		swprintf_s(szKey, 50, L"Link_Move_Down_&d", i);
		
		// 생성했던 10개의 Sprite를 FlipBook에 추가
		pFlipBook->AddSprite(FindAsset<CSprite>(szKey));		
	}
	
	// 생성한 FlipBook을 새로운 에셋으로 추가
	AddAsset(L"Link_MoveDown", pFlipBook);
}
```

방금 사용된 FlipBook에 생성한 Sprite를 추가하는 `AddSprite`를, FlipBook에서 구현해주자

```cpp
class CFlipBook :
	public CAsset
{
	// ...
	void AddSprite(Ptr<CSprite> _Sprite) { m_vecSprite.push_back(_Sprite); }
}

```

<aside>
🗒️ [기존 코드 수정 사항]

AddAsset 시에 본인의 경로를 설정해야 했음

```cpp
template<typename T>
void CAssetMgr::AddAsset(const wstring& _Key, Ptr<T> _Asset)
{
	// ...
	// Asset 추가 시 경로 설정 추가
	_Asset->SetKey(_Key);
	// ...
}
```

</aside>

### FlipBook Component (Animator)에서 필요한 기능

WinAPI 때에는 한 Object당 한 Animation 객체를 갖고 있었고, 해당 Animation 객체에서 현재 재생중인 Frame 등…을 관리하고 있었다

그러나 지금은 공유된 자원인 **Asset**을 통해 Animation(FlipBook)을 가져오고, **FlipBook Component**가 Frame Index 등 출력과 관련된 정보를 가지고 있는 형태! 

그렇다면 **FlipBook 입장**에서 생각해보면,

- 본인을 사용하고 있는 FlipBook Component가 많을 수도 있다
- 해당 FlipBook에 대한 정보는 본인이 들고 있지만, FlipBook Component가 관리하게 된다
    
    → 따라서 여러 FlipBook Component가 서로 다른 Frame이나 FPS 등을 출력할 수 있다
    
- 본인을 호출시키는 FlipBook Component의 `FinalTick`을 통해 출력이 이루어진다

이제 **FlipBook Component 입장**에서 생각해보면,

종합적으로 FlipBook을 관리할 수 있어야 한다!

따라서 FlipBook Component에게 필요한 기능을 정리하자면…

1️⃣ FlipBook 등록

2️⃣ 등록된 FlipBook를 검색

3️⃣ 지정된 FlipBook을 Play

**FlipBook Component에게 필요한 기능 구현**

```cpp
class CFlipBookComponent :
	public CComponent
{
	// ...
	
public:
	void AddFlipBook(Ptr<CFlipBook> _FlipBook);          // 1️⃣
	Ptr<CFlipBook> FindFlipBook(const wstring& _Key);    // 2️⃣
	void Play(const wstring& _FlipBookName);             // 3️⃣
	
// ...
}
```

1️⃣ **FlipBook 등록**

```cpp
void CFlipBookComponent::AddFlipBook(Ptr<CFlipBook> _Flipbook)
{
	m_vecFlipBook.push_back(_Flipbook);
}
```

**2️⃣ 등록된 FlipBook를 검색**

```cpp
Ptr<CFlipBook> CFindFlipBook::FindFlipBook(const wstring& _Key)
{
	for(size_t i = 0; i < m_vecFlipBook.size(); ++i)
	{
		if (m_vecFlipBook[i]->GetKey() == _Key)
			return m_vecFlipBook[i];
	}
	return nullptr;
}
```

3️⃣ **지정된 FlipBook을 Play**

```cpp
void CFlipBookComponent::Play(int _FliBookIdx, float _FPS, bool _Repeat)
{
	m_CurFlipBook = m_vecFlipBook[_FliBookIdx];

	if (nullptr == m_CurFlipBook)
	{
		return;
	}
	
	m_CurFrmIdx = 0;
	m_AccTime = 0.f;
	m_FPS = _FPS;
	m_Repeat = _Repeat;
}
```

그럼 이제 FinalTick을 통해 생성한 FlipBook을 출력할 수 있다

**FlipBook Component의 FinalTick**

```cpp
void CFlipBookComponent::FinalTick()
{
	// 현재 FlipBook의 재생이 종료되었는지 먼저 확인
	// 반복 재생이 설정되어 있는지도 확인
	if (m_Finish)
	{
		if (m_Repeat == false)
			return;
			
		Reset();
	}
	
	// 현재 FlipBook이 지정되어있다면 재생
	if (m_CurFlipBook != nullptr)
	{
		float MaxTime = 1.f / FPS;
		m_AccTime += DT;
		
		if (MaxTime < m_AccTime)
		{
			m_AccTime -= MaxTime;
			++m_CurFrmIdx;
			
			if (m_CurFlipBook->GetMaxFrameCount() <= m_CurFrmIdx)
			{
				--m_CurFrmIdx;
				m_Finish = true;
			}
		}
		m_CurFrmSprite = m_CurFlipBook->GetSprite(m_CurFrmIdx);
	}
}
```

반복 재생되면서, 재생이 종료된 FlipBook이 있다면 Reset을 통해 다시 재생될 수 있도록 설정해준다

**초기화 함수 Reset 작성**

```cpp
void Reset()
{
	m_CurFrmIdx = 0;
	m_AccTime = 0.f;
	m_Finish = false;
}
```

위에서 FlipBook을 통해 호출한 함수들도 구현해주자

- 갖고있는 Sprite 중 Index를 통해 원하는 Sprite를 Get해오는 함수
    
    → `GetSprite`
    
- FlipBook의 최대 프레임을 Get해오는 함수
    
    → `GetMaxFrameCount`
    

가 필요하다

**FlipBook의 Get 함수 작성**

```cpp
class CFlipBook
	: public CAsset
{
	// ...
public:
	int GetMaxFrameCount() { return (int)m_vecSprite.size(); }
	Ptr<CSprite> GetSprite(int _Idx) { return m_vecSprite[_Idx]; }
};
```

## FlipBook Component의 출력을 위한 Rendering 설정

### MeshRender Component는 이제 FlipBook Component를 고려한다

현재 Rendering을 담당하는 컴포넌트 → MeshRender Component

이제 MeshRender Component는 렌더링할 때에, FlipBook Component를 고려하게 된다

FlipBook Component가 어떤 FlipBook을 지정하고 있다면, 해당 FlipBook에서 현재 재생중인 Sprite을 같이 출력해야 한다

**이제 MeshRender의 Render에서는,  FlipBook Component의 여부를 확인한다**

```cpp
void CMeshRender::Render()
{
	// ...
	
	if (FlipBookComponent())
	{
		Ptr<CSprite> pCurSprite = FlipComponent()->GetCurSprite();		
		// ...
	}
	else
	{
		// ...
	}
}

```

→ 이제 Sprite가 렌더링될 수 있도록 바인딩해줘야 하는데, 이때 필요한 정보들을 생각해보자

### Sprite 정보의 바인딩을 위한 설계

렌더링을 위해 사용할 Sprite의 정보들을 멤버로 선언해 줄 것이다!

- `m_Atlas` : 해당 Sprite가 사용하는 Atlas Texture
- `m_LeftTopUV` : 좌상단 위치 (Position) → UV 좌표계 사용
- `m_SliceUV` : Slice할 사이즈 (Size) → UV 좌표계 사용

그리고 이와 함께 Set, Get할 수 있는 함수들도 작성해주자

**CSprite 클래스 멤버 작성**

```cpp
class CSprite :
	public CAsset
{
private:
	Ptr<CTexture>     m_Atlas;
	Vec2              m_LeftTopUV;
	Vec2              m_SliceUV;
	
public:
	Ptr<CTexture> GetAtlasTexture() { return m_Atlas; }
	Vec2 GetLeftTopUV() { return m_LeftTop; }
	Vec2 GetSliceUV() { return m_Slice; }
	
	void SetBackground();
	void SetLeftTopUV();
	void SetSliceUV()
}
```

이 정보들을 바인딩해서 레지스터를 통해 GPU로 넘겨주어야 하니, Material의 파라미터를 빌려서 사용하자

```cpp
cbuffer MATERIAL : register(b1)
{
	int g_int_0;            // -> FlipBook을 사용하고 있는지 / 사용하지 않는지에 대한 파라미터
	
	// ...
	
	float2 g_vec2_0;        // LeftTopUV 파라미터
	float2 g_vec2_1;        // SliceUV 파라미터
}
```

그리고 Atlas Texture를 전달하기 위한, Atlas 전용 Texture Register 선언해주자

```cpp
// ...
Texture2D g_AtlasTex : register(t10);  
```

현재 Sprite가 가리키고 있는 Atlas Texture를 해당 레지스터에 바인딩

```cpp
pCurSprite->GetAtlasTexture()->Binding(10);
GetMaterial()->SetScalarParam(VEC2_0, pCurSprite->GetLeftTopUV());
GetMaterial()->SetScalarParam(VEC2_1, pCurSprite->GetSliceUV());
GetMaterial()->SetScalarParam(INT_0, 1);     // FlipBook을 사용하고 있음을 나타내는 파라미터
```

그럼 픽셀 쉐이더는, FlipBook의 사용여부에 따라 참조하는 레지스터가 달라지게 된다

- 이때 FlipBook의 사용 여부는 `g_int_0` 을 통해 전달되었다 → 1이 담기면 FlipBook 사용, 아니면 0

```cpp
float4 PS_Std2D(VTX_OUT in) : SV_Target
{
	// ...
	
	// FlipBook을 사용한다
	if(g_int_0)
	{
		// Sprite를 참조할 위치를 비율로 환산한 값
		float2 
	
		vColor = g_AtlasTex.Sample(g_sam_1, );       // 샘플링 진행 (포인터 
		g_vec2_0;
		g_vec1_0;
	}
}
```

입력으로 들어온 UV를 그대로 사용할 수는 없지만, 우리가 사용할 Sprite 안에서 어디를 참조해야 하는지를 대략적으로 가리키고 있다 

→ 즉, 입력으로 들어온 UV는 Sprite를 참조할 위치를 비율로 환산한 값으로 볼 수 있다

그런데 우리가 아는 것은 Slice Size! 즉, 가로 / 세로 길이

입력으로 들어온 UV와 우리가 알고 있는 Size를 곱하면 좌상단을 기준으로 해당 지점까지 그려야하는 거리가 나오기 때문에…

즉, 최종 UV는 LeftTop + (입력UV * SliceSize)인 것이다

이제 한 Object에 FlipBook Component를 연결하고, 사용해보자

```cpp
pObject->FlipBookComponent()->AddFlipBook(CAssetMgr::GetInst()->FindAsset<CFlipBook>(L"Link_Down));
pObject->FlipBookComponent()->Play(0, 10, true);
```