# 2024/02/27 - Camera Effect 구조 개선, PNG 파일 로딩, Clone 추상화 (1)

태그: C++, WinAPI, 중급
날짜: 2024/02/27
상위 항목: Week14 (Week14%202545b77261d946f984f92ede76a23ddb.md)
주차: 0010_Week10~19

## Camera Effect 구조 개선

### 현재 Camera Effect 구조의 한계

현재 설계된 구조는 SetCameraEffect 를 통해서 카메라 효과를 설정하면, 해당 정보를 기반으로 tick()에서 카메라 효과에 대한 연산이 시행된 후 render되는 흐름을 갖고 있다! 

따라서 현재는 두 가지 효과를 연속으로 실행할 수가 없다!

→ SetCameraEffect 함수를 통해 카메라 이펙트 값을 설정하기 때문에, SetCameraEffect를 여러번 호출하더라도 마지막으로 호출되어 설정한 정보를 기반으로 렌더링 된다

그렇다면, 만약 여러개의 카메라 이펙트를 수행하고 싶다면?

→ 카메라 이펙트를 진행하기 위한 정보를 여러 개 들고 있어야겠지!

> 따라서 카메라 이펙트에 대한 정보를 담을 수 있도록 **구조체**를 만들고,
해당 구조체 타입의 **리스트**를 통해 카메라 이펙트들을 관리해보자!
> 

### Camera Effect에 대한 정보를 구조체로 관리하자

**카메라 이펙트 정보 구조체 생성 + 해당 구조체를 담는 리스트 멤버 변수 추가**

```cpp
struct CAM_EFFECT_INFO
{
	CAM_EFFECT  effect;
	float duration;
	float time;
	float alpha;
};

// ...

class CCamera
{
	SINGLE(CCamera)
private:
	Vec2                   m_LookAt;
	Vec2                   m_Diff;
	float                  m_CamSpeed;
	list<CAM_EFFECT_INFO>  m_EffectList;
// ...
}
```

이제 effect를 세팅하는 과정도 해당 구조체를 기준으로 작성해준다!

```cpp
void CCamera::SetCameraEffect(CAM_EFFECT _Effect, float _Duration)
{
	CAM_EFFECT_INFO info = {};

	info.Effect = _Effect;
	info.Duration = _Duration;
	info.Time = 0.f;
	info.Alpha = 0.f;

	m_EffectList.push_back(info);      // EffectList에 해당 효과 등록
}
```

> 이렇게 List에 등록된 카메라 효과를 어떻게 활용할 수 있을까?
> 
- List에 등록된 순서대로, front()에 등록되어 있는 카메라 효과부터 tick에서 CameraEffect 함수를 통해 해당 효과의 생애주기 동안 연산해 render된다
- 현재 연산이 진행되던 카메라 효과의 생애주기가 종료되면 해당 효과를 List에서 pop_front를 통해 삭제한다
- 다음으로 등록되어 있던 카메라 효과가 List의 front가 되므로, 다음 효과가 tick을 통해 연산이 되고 render된다
- 이런 작용이 List에 담긴 효과의 개수만큼 반복되며 연쇄적으로 카메라 효과를 연출할 수 있게 되는 것이다!

**front()에 등록되어 있는 카메라 효과부터 연산 진행 + 생애주기가 끝났다면 pop_front()**

```cpp
void CCamera::CameraEffect()
{
	while (true)
	{
		if (m_EffectList.empty())
			return;

		CAM_EFFECT_INFO& info = m_EffectList.front();
		info.Time += DT;

		if (info.Duration < info.Time) m_EffectList.pop_front;
		else break;
	}
	
	CAM_EFFECT_INFO& info = m_EffectList.front();

	if (info.effect == CAM_EFFECT::FADE_IN)
	{
		info.Alpha = (1.f - (info.TIme / info.Duration)) * 255.f;
	}
	else if (info.effect == CAM_EFFECT::FADE_OUT)
	{
		info.Alpha = (info.Time / info.Duration) * 255.f;
	}
}
```

**이를 기반으로 render**

```cpp
void CCamera::render()
{
	if (m_EffectList.empty())
		return;

	CAM_EFFECT_INFO& info = m_EffectList.front();

	BLENDFUNCTION bf = {};

	bf.BlendOp = AC_SRC_OVER;
	bf.BlendFlags = 0;
	bf.SourceConstantAlpha = (int)info.Alpha;
	bf.AlphaFormat = 0;

	AlphaBlend(DC, 0, 0, m_FadeTex->GetWidth(), m_FadeTex->GetHeight(),
					m_FadeTex->GetDC(), 0, 0, m_FadeTex->GetWidth(), m_FadeTex->GetHeight(), bf);
}
```

## PNG 파일을 로딩해 비트맵으로 가져와보자!

Windows API에서는 PNG 이미지쓰는걸 비추..

→ 기본적으로 알파채널을 가지고 있음

비트맵만 로딩할 수 밖에 없으므로 비트맵으로 변환해야 하는데.. 이미지가 알파채널을 소실하는 것이기 때문에 원하는 이미지의 느낌을 잃어버리는 경우가 있다

따라서 PNG 이미지를 로딩하는 방법에 대해서 알아보자아

PNG 이미지를 비트맵 형태로 가져오기 위해서는 몇가지 헤더를 참조해야 한다!

> png 로딩 관련 헤더 추가
> 
- `#include <objidl.h>`
- `#include <gdiplus.h>`
- `#pragma comment(lib, “GdiPlus.lib”)`
- `using namespace GdiPlus;`

확장자명에 따른 분기처리 → `_wsplitpath_s();`

```cpp
int CTexture::Load(const wstring& _strFilePath)
{
	wchar_t szExt[50] = {};
	_wsplitpath_s(_strFilePath.c_str(), nullptr, 0, nullptr, 0, nullptr, 0, szExt, 50);

	if(!wcscmp(szExt, L".bmp") || !wcscmp(szExt, L".BMP"))
	{
		// 기존에 로딩하던 방법
		m_hBit = (HBITMAP)LoadImage(nullptr,

		if (nullptr == m_hBit)
		{
			MessageBox(nullptr, L"비트맵 로딩 실패", L"Asset 로딩 실패", MB_OK);
			return E_FAIL;
		}
	}
	else if (!wcscmp(szExt, L".png") || !wcscmp(szExt, L".PNG"))
	{
		// png 이미지를 비트맵으로 로딩하는 방법
		ULONG_PTR gdiplusToken = 0;
		GdiplusStartupInput gdiplusinput = {};
		GdiPlusStartup(&gdiplusToken, &gdiplusinput, nullptr);

		Image* pImg = Image::FromFile(_strFilePath.c_str(), 0);
		Bitmap* pBitmap = (Bitmap*)pImg->Clone();
		Gdiplus::Status status = pBitmap->GetHBITMAP(Color(0, 0, 0, 0), &m_hBit);
		assert(status == Gdiplus::Status::OK);
	}
	else
	{
		assert(nullptr);
	}

	GetObject(m_hBit, sizeof(BITMAP), &m_Info);

	 m_hDC = CreateCompatibleDC(CEngine::GetInst()->GetMainDC());
	 DeleteObject(SelectObject(m_hDC, m_hBit));
}
```

## Clone 추상화

### Clone과 복사생성자

`Clone()` → 나 자신을 복제해서 리턴시키는 개념 

ex) Player에서 사용된 Clone

현재 복사생성자를 구현한 적이 없기 때문에… default로 만들어주는 복사생성자를 이용해 Clone 되었던 것

player가 가지고 있는 값을 그대로 복사하면.. → 문제가 안생길까?

객체 하나가 만들어졌다면 본인만의 Component를 다르게 가져야 하는데, 복사대상자가 된 원본이 주인인 컴포넌트를 동일하게 가리키고 있게 된다

컴포넌트를 가지고 있어야 함은 맞되, 같은 컴포넌트를 가리키면 안된다

> 즉, Default 복사 생성자로는 우리가 원하는 Clone을 구현할 수 없는 경우가 있다!
> 

따라서 제대로 된 복사생성자를 직접 구현해주자 

<aside>
✍️ **복사생성자를 직접 구현할 때 유의할 사항**

A클래스를 상속하는 ← B클래스, B클래스를 상속하는 ← C클래스의 구조가 있다면,

필요한 클래스는 각 클래스에서 파트별 복사생성자를 통해 복사해야 한다

그리고 각 자식 클래스는, 부모의 복사 생성자를 자신의 복사 생성자에서 호출해 부모 파트에서 처리되어야 할 부분을 일임한다

</aside>

### 각 클래스 별로 복사 생성자의 요구사항하기

1. CEntity 클래스에서의 고민

**class CEntity 살펴보기**

```cpp
class CEntity
{
private:
	static UINT g_NextID;

private:
	const UINT	m_ID;		// 객체별 고유 ID
	wstring		m_strName;	

	//...
};
```

- `const UINT m_ID` → 각 객체가 고유하게 갖는 값이기 때문에 Default 복사 생성자로 그대로 복사하면 안된다! Clone에 사용될 복사 생성자를 새롭게 구현해 m_ID가 g_NextID를 활용해 설정되게끔 해야한다

**CEntity의 복사 생성자 구현**

```cpp
CEntity::CEntity(cosnt CEntity& _Other)
	: m_ID(g_NextID++)
	, m_strName(_Other.m_strName)
{
}
```

1. CLevel 클래스에서의 고민
    
    → Level은 복제해 똑같은 객체를 만들 이유가 없으니… CLevel 단계에서 `CLONE_UNABLED` 호출로 Clone기능 제외
    

1. CObj 클래스에서의 고민

```cpp
#pragma once
#include "CEntity.h"

#include "CEngine.h"
#include "CTimeMgr.h"
#include "CKeyMgr.h"
#include "CAssetMgr.h"
#include "CTexture.h"
#include "CCamera.h"

class CComponent;
class CCollider;
class CAnimator;
class CRigidBody;
class CFSM;

class CObj :
    public CEntity
{
private:
    Vec2                m_Pos;      // 위치
    Vec2                m_PrevPos;  // 이전 프레임에서의 위치
    Vec2                m_Scale;    // 크기
    vector<CComponent*> m_vecCom;   // 보유 컴포넌트들

    CAnimator*          m_Animator;

    LAYER_TYPE          m_Type;     // 소속 레이어
    bool                m_bDead;    // 삭제 예정상태

public:
    void SetPos(Vec2 _Pos) { m_Pos = _Pos; }
    void SetScale(Vec2 _Scale) { m_Scale = _Scale; }

    void SetPos(float _x, float _y) { m_Pos.x = _x; m_Pos.y = _y; }
    void SetScale(float _width, float _height) { m_Scale.x = _width; m_Scale.y = _height; }

    Vec2 GetPos() { return m_Pos; }
    Vec2 GetPrevPos() { return m_PrevPos; }
    Vec2 GetRenderPos() { return CCamera::GetInst()->GetRenderPos(m_Pos); }
    Vec2 GetScale() { return m_Scale; }
    LAYER_TYPE GetLayerType() { return m_Type; }
    bool IsDead() { return m_bDead; }

    void Destroy();

    CComponent* AddComponent(CComponent* _Component);

    template<typename T>
    T* GetComponent()
    {
        for (size_t i = 0; i < m_vecCom.size(); ++i)
        {
            T* pComponent = dynamic_cast<T*>(m_vecCom[i]);

            if(pComponent)
            {
                return pComponent;
            }
        }

        return nullptr;
    }

public:
    virtual void begin();
    virtual void tick();            // 오브젝트가 매 프레임마다 해야할 작업을 구현
    virtual void finaltick() final;// 오브젝트가 소유한 컴포넌트가 매 프레임마다 해야할 작업을 구현
    virtual void render();

    virtual void BeginOverlap(CCollider* _OwnCollider, CObj* _OtherObj, CCollider* _OtherCollider) {}
    virtual void OnOverlap(CCollider* _OwnCollider, CObj* _OtherObj, CCollider* _OtherCollider) {}
    virtual void EndOverlap(CCollider* _OwnCollider, CObj* _OtherObj, CCollider* _OtherCollider) {}
    

public:
    virtual CObj* Clone() = 0;

public:
    CObj();
    CObj(const CObj& _Other);
    ~CObj();

    friend class CLevel;
    friend class CTaskMgr;
};

```

CObj에서는 Clone을 그대로 두되… Default가 아닌 직접 생성한 복사 생성자 필요

부모 파트의 복사는 부모쪽의 복사생성자에서 진행, 그런데 복사생성자도 생성자이므로, 원하는 부모쪽 생성자의 버전을 명시해줘야 한다…?

```cpp
CObj::CObj(const CObj& _Other)
	: CEntity(_Other)                   // 부모 복사 생성자를 직접 구현한 버전으로 호출
	, m_Pos(_Oher.m_pos)
	, m_PrevPos(_Other.m_PrevPos)
	, m_Scale(_Other.m_Scale)
	, //m_vecCom                             // 요기 문제!
		CAnimator(null);
	, m_TYpe(LAYER_TYPE::NONE)
	, m_bDead(false)
{
	for size_t i = 0; i < _Other.m_vecCom.size(); ++i
	{
		// Collider 다형성으로 표현된 벡터더라도, 
		// Clone을 할 때에는 해당 세부적인 객체 타입에서 오버라이딩한 Clone이 수행
		AddComponent(_Other.m_vecCom[i]->Clone());
	}
}
```

```cpp
Cplayer::CPlayer(const CPlayer& _Other)
	: CObj(_Other)
	, m_Speed(_Other.m_Speed)
	, m_PlayerImg(_Other.m_PlayerImg)
	, m_DoubleJumpCount(_Other.m_DoubleJumpCount)
	, m_CurJumpCOunt(_Other.m_CurJumpCOunt)
	, m_BodyCol(nullptr)
	, m_Animator(nullptr)
	, m_RigidBody(nullptr)
{
	m_BodyCol = GetComponent<CCollider>();
	m_Animator = GetComponent<CAnimator>();
	m_RigidBody = GetComponent<CRigidBody>();
}
```

<aside>
✍️ **TODO**

- Component의 복사 생성자를 제대로 구현하기!
    - Collider의 복사 생성자
    - Animator의 복사 생성자
    - RigidBody의 복사 생성자
</aside>