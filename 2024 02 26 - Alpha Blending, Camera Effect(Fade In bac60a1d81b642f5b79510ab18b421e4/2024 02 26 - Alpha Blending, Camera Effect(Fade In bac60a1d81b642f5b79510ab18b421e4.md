# 2024/02/26 - Alpha Blending, Camera Effect(Fade In, Fade Out)

태그: C++, WinAPI, 중급
날짜: 2024/02/26
상위 항목: Week14 (Week14%202545b77261d946f984f92ede76a23ddb.md)
주차: 0010_Week10~19

## AlphaBlending에 대해 알아보자

### Alpha Blending이란?

먼저 우리가 지금 리소스로 사용하고 있는 bmp파일은 픽셀 하나 하나가 R,G,B, 즉 1byte씩 총 3byte의 용량을 갖고 있는 24bit 표현 체계의 이미지이다.

그런데 24bit는 2의 승수로 나눠떨어지지 않아서, 32bit 포맷의 이미지를 사용해야 비트맵을 사용하더라도 메모리 액세스 부분에서 최적화가 용이할 수 있다

그럼 따라서 나머지 1byte를 어떻게 채우느냐! → 알파 채널(투명도)의 추가로!

### 알파(Alpha) 채널이란

알파 채널은 이미제에서 각 픽셀의 투명도 정보를 저장하는 방식으로, 기본적으로 이미지는 RGB의 세 가지 색상 채널로 구성되어 있고, 이를 통해 다양한 색상을 표현할 수 있다.

여기에 알파(Alpha, A) 채널이 추가되면, 각 픽셀의 투명도 정보를 포함하게 된다!

그러나 모든 이미지 파일 형식이 알파 채널을 지원하는 것은 아니다! 대표적으로는 PNG, GIF, TIFF… (그래서  나중에 PNG 파일로 애니메이션을 구성하는 방법을 배우는 듯!)를스터에 애니메이션 추가 → Figther.bmp 파일로 

이를 활용해서 몬스터에 애니메이션을 추가해보자

→ Figther.bmp 파일로 

```cpp
class CMonster
	: public CObj
{
private:
	// ...
	CTexture*  m_Img;
}
```

```cpp
CMonster::CMonster()
{
	// ...
	m_Img = CAssetMgr::GetInst()->LoadTexture(L"texture\\Fighter.bmp", 
																						L"texture\\Fighter.bmp")
}

CMonster::render()
{
	// ...
	TransparentBlt(DC, (int)(vPos.x - m_Img->GetWidth() / 2.f)
									 , (int)(vPos.y - m_Img->GetHeight() / 2.f)
									 , m_Img->GetWidth(), m_Img->GetHeight(),
									 , m_Img->GetDC(), 0, 0,
									 , m_Img->GetWidth(), m_Img->GetHeight(), RGB(255, 0, 255));
}
```

지금까지는 알파 채널을 사용하지 않고 TransparentBlt 함수를 사용해서 특정 색상을 제외하고 출력하는 방식으로 투명한 배경 효과를 낼 수 있었다.

이제부턴 TransparentBlt이 아닌 **AlphaBlending**을 활용해, 알파값이 0인 이미지는 투명하게 처리하는 방식으로 사용해보자! → 어느 부분을 투명하게 할지 알파값이 다 정해져있기 때문에 투명하게 처리할 부분에 대한 정보를 넘기지 않아도 된다

### 구조체 BLENDFUNCTION과 AlphaBlend() 함수

> **`struct BLENDFUNCTION`**
> 

알파 블렌딩 작업에서 사용되는 블렌딩 파라미터를 정의하는 구조체로, 이 구조체가 `AlphaBlend` 함수에 전달되어 소스 이미지와 대상 이미지간의 블렌딩 방식을 지정한다

**BLENDFUNCTION의 정의**

```cpp
typedef struct _BLENDFUNCTION {
  BYTE BlendOp;                     // AC_SRC_OVER 옵션만 사용 가능
  BYTE BlendFlags;                  // 항상 0
  BYTE SourceConstantAlpha;         // 전체 소스 이미지에 적용될 알파 값
  BYTE AlphaFormat;                 // AC_SRC_ALPHA 옵션을 사용해,
																    // 리소스가 자체적 픽셀 단위 알파값을 가짐을 나타냄
} BLENDFUNCTION, *PBLENDFUNCTION;
```

- 여기서 특히 `SourceConstantAlpha`라는 멤버는 **고정 알파값**으로, 알파 블렌딩이 끝난 후 전체 소스 이미지에 한번 더 적용되어 블렌딩에 사용되는 값이다! 자세한건 알파 블렌딩의 연산 과정과 같이 알아보자

**사용 예**

```cpp
BLENDFUNCTION bf = {};

bf.BlendOp = AC_SRC_OVER;
bf.BlendFlags = 0;
bf.SourceConstantAlpha = 255;            
bf.AlphaFormat = AC_SRC_ALPHA;

AlphaBlend(DC, (int)(vPos.x - m_Img->GetWidth() / 2.f)
						 , (int)(vPos.y - m_Img->GetHeight() / 2.f)
						 , m_Img->GetWidth, m_Img->GetHeight()
						 , m_Img->GetDC(), 0, 0,
						 , m_Img->GetWidth(), m_Img->GetHeight());
```

이를 Monster의 render에 적용해보면…

```cpp
void CMonster::render()
{
	Vec2D vPos = GetRenderPos();
	Vec2D vScale = GetScale();

	static float alpha = 0;
	static float dir = 1;
	
	alpha += DT * 400.f * dir;           // 점진적으로 투명/불투명해지도록 속도 설정

	if (255.f <= alpha)                  // 완전히 불투명해지면 다시 투명해지도록
	{
		dir *= -1.f;
	}
	else if (alpha < 0.f)                // 완전히 투명해지면 다시 불투명해지도록
	{
		dir *= -1.f;
	}

	BLENDFUNCTION bf = {};

	bf.BlendOp = AC_SRC_OVER;
	bf.BlendFlags = 0;
	bf.SourceConstantAlpha = (int)alpha;
	bf.AlphaFormat = AC_SRC_ALPHA;

	AlphaBlend(DC, (int)(vPos.x - m_Img->GetWidth() / 2.f)
							 , (int)(vPos.y - m_Img->GetHeight() / 2.f)
					     , m_Img->GetWidth(), m_Img->GetHeight()
					     , m_Img->GetDC(), 0, 0, m_Img->GetWidth(), m_Img->GetHeight(), bf);
}
```

### TransparentBlt과 Alpha Blending의 출력 방식 차이와 연산 공식

TransparentBlt과 AlphaBlend는 모두, 이미지를 다른 이미지 위에 그리는 방법을 제공하지만, 그 방식과 목적이 다르다!

> ***TransparentBlt 함수***는..
> 

Windows GDI에서 제공하는 함수로, 한 이미지를 다른 이미지 위에 투명하게 그릴 수 있게 해준다. 소스 이미지에서 특정 색상을 투명하게 처리해, 그 색상이 있는 부분을 제외하고 이미지를 목적지에 복사하는 것이다.

따라서 투명하게 보이는 것은 특정 색상을 아예 배제하는 것이기 때문에, 픽셀이 완전히 불투명하거나 완전히 투명한 상태만을 지원하고, 중간 단계의 투명도는 표현할 수 없다

> ***Alpha Blending***은..
> 

이미지의 픽셀마다 다른 투명도(알파값)을 가지고 있을 때 사용하는 기술로,즉 이미지가 32bit 표현 방식을 가질 때 사용할 수 있다는 뜻이다. 알파 값은 픽셀의 불투명도를 나타내며, 0에서 255사이의 값을 가질 수 있다 (0은 완전 투명, 255는 완전 불투명)

AlphaBlend는 각 픽셀의 알파값을 사용해 최종 색상을 결정해 출력하는 방식으로, 다음과 같은 연산 방식이 사용된다!

$$
결과\ 색상 = (A \ * 소스의\ RGB)\ + \ (1 - A) \ * \ (배경의\ RGB) 
$$

이에 추가로, Windows API에서 알파 블렌딩 작업을 수행할 때 사용되는 매개 변수인 `BLENDFUNCITON` 구조체 중 `SourceConstantAlpha` 값을 사용해 소스 이미지와 대상 이미지 간의 블렌딩에 사용될 고정 알파값을 연산해준다

```cpp
typedef struct _BLENDFUNCTION {
  BYTE BlendOp;                     // AC_SRC_OVER 옵션만 사용 가능
  BYTE BlendFlags;                  // 항상 0
  BYTE SourceConstantAlpha;         // 전체 소스 이미지에 적용될 알파 값
  BYTE AlphaFormat;                 // AC_SRC_ALPHA 옵션을 사용해,
																    // 리소스가 자체적 픽셀 단위 알파값을 가짐을 나타냄
} BLENDFUNCTION, *PBLENDFUNCTION;
```

$$
결과\ 색상 = AlphaBlend로\ 연산한\ 결과값\ *\ SourceConstantAlpha
$$

이 두 연산을 통해서 최종적으로 출력될 결과 색상을 조절할 수 있다

## FadeIn, FadeOut

이를 응용해서 Camera에 적용해, FadeIn, FadeOut 효과를 연출해보자!

### Camera Effect 정의하기

```cpp
enum class CAM_EFFECT       // 카메라 이펙트 유형에 대한 enum class 정의
{
	FADE_IN,
	FADE_OUT,
	NONE,
};

class CCamera
{
	SINGLE(CCamera)
private:
	// ...

	CAM_EFFECT  m_Effect;     // 카메라 이펙트 유형
	float       m_Duration;   // 해당 이펙트의 생애 주기
	float       m_Time;       // 소요 시간

public:
	// ...
	// 카메라 이펙트 세팅 함수
	void SetCameraEffect(CAM_EFFECT _Effect, float _Duration);
}

```

```cpp
// ...

void SetCameraEffect(CAM_EFFECT _Effect, float _Duration)
{
	m_Effect = _Effect;
	m_Duration = _Duration;
	m_Time = 0.f;
}
```

### Camera의 tick에서 일어나는 일을 세분화하자

Camera의 tick에서 일어나는 작업들을 정리해 함수로 세분화할 수 있는 작업들을 나열해보자

- 카메라 이동에 대한 일 → Move 함수를 작성해 호출
- 카메라 효과와 관련된 일 → CameraEffect 함수를 작성해 호출
- 해상도 중심과 카메라가 바라보고 있는 지점 간의 차이값 계산 → 그대로 tick에서..

그럼 이런 형태가 된다

```cpp
void CCamera::tick()
{
	Move();

	// 해상도 중심과 카메라가 바라보고 있는 지점 간의 차이 값 계산
	//...

	CameraEffect();
}
```

각각의 함수를 정의해보자

```cpp
void CCamera::Move()
{
	// 카메라 이동
	if (KEY_PRESSED(KEY::W))
	// ... 기존에 작성했던 부분 ...
}
```

```cpp
void CCamera::CameraEffect()
{
	// 만약 설정된 카메라 효과가 NONE이라면 바로 return
	if(CAM_EFFECT::NONE == m_Effect)
		return;

	// tick이 진행되는 동안 소요 시간이 누적되고
	m_Time += DT;
	// 소요 시간이 설정한 생애 주기(Duration)을 넘어서면 카메라 효과를 NONE으로 전환
	if( m_Duration < m_Time )
	{
		m_Effect = CAM_EFFECT::NONE;
	}

	// 만약 설정된 카메라 효과가 FADE_IN이라면 진행할 작업
	if(CAM_EFFECT::FADE_IN == m_Effect)
	{
		// ... 작성 예정!
	}
	// 만약 설정된 카메라 효과가 FADE_OUT이라면 진행할 작업
	else if (CAM_EFFECT::FADE_OUT == m_Effect)
	{
		// ... 작성 예정!
	}
}
```

 

### 카메라의 render 함수 추가

 CEngine에서 render시 호출해 현재 적용된 카메라 효과가 render 되도록 함수를 정의해주자

일단 틀만 작성하고.. 자세한 내용은 Fade In/Out 구현한 후에 기능 붙이기!

```cpp
void CCamera::render()
{
	// ... 작성 예정!
}
```

```cpp
void CEngine::Progress()
{
	// ...
	CLevelMgr::GetInst()->render();
	CCamera::GetInst()->render();
	CDbgRender::GetInst()->render();    
	// 카메라 효과가 적용된 중에도 디버그는 그대로 보이도록 순서 배치
}
```

## Fade In, FadeOut에 사용될 검은 텍스쳐를 생성하기

### 먼저 DC, bitmap 생성을 Texture로 컨트롤하기

우리가 만든 CTexture 클래스를 보면 멤버가 다음과 같았다

```cpp
class CTexture :
    public CAsset
{
private:
    HDC         m_hDC;          // DC
    HBITMAP     m_hBit;         // Bitmap
    BITMAP      m_Info;         // Bitmap에 대한 정보

// ...
```

→ 어랏, 우리가 화면을 그리고 출력하기 위해서는 DC와 Bitmap이 필요한데.. 그럼 Texture 객체 하나로 이 둘을 모두 컨트롤할 수 있겠구나!!

그러므로 기존에 DC, Bitmap을 개별적으로 관리했던 방식을, 우리의 CTexture를 활용해 화면을 그리는 작업을 대체해보자!

**원하는 크기의 Texture 객체를 생성하는 Create 함수 추가**

```cpp
class CTexture:
	public CAsset
{
	// ...
	void Create(UINT _Width, UINT _Height);
	// ...
}
```

```cpp
int CTexture::Create(UINT _Width, UINT _Height)
{
	// DC 생성
	m_hDC = CreateCOmpatibleDC(CEngine::GetInst()->GetMainDC());

	// Bitmap 생성
	m_hBit = CreateCompatibleBitmap(CEngine::GetInst()->GetMainDC(), _Width, _Height)

	// SubDC가 SubBitmap을 지정하게 함
	Hbitmap hPrevBitmap = (hbitmap)SelectObject(m_hDC, m_hBit);
	DeleteObject(hPrevBitmap);

	// 로드된 비트맵의 정보를 확인
	GetObject(m_hBit, sizeof(BITMAP), &m_Info);

	return S_OK;
}
```

그리고 Texture는 Asset으로 관리되기 때문에, AssetMgr을 통해 Texture 객체를 생성할 수 있도록 함수를 작성해준다!

**CAssetMgr의 CreateTexture 함수 추가**

```cpp
class CAssetMgr
{
	SINGLE(CAssetMgr);
private:
	//...
public:
	// ...
	CTexture* CreateTexture(const wstring& _Key, UINT _Width, UINT _Height);
}
```

```cpp
CTexture* CAssetMgr::CreateTexture(const wstring& _Key, UINT _Width, UINT _Height)
{
	// 이미 해당 키로 등록된 텍스쳐가 있으면 assert
	assert(!FindTexture(_Key));
		
	// 텍스쳐 객체 생성
	CTexture* pTex = new CTexture;
	if (FAILED(pTex->Create(_Width, _Height)))
	{
		MessageBox(nullptr, _Key.c_str(), L"텍스쳐 생성 실패", MB_OK);
		delete pTex;
		return nullptr;
	}

	// map 에 로딩된 텍스쳐를 등록
	m_mapTex.insert(make_pair(_Key, pTex));

	// 텍스쳐 에셋에 본인의 키값을 알려줌
	pTex->m_Key = _Key;	

	return pTex;
}
```

이를 CEngine의 SubDC 생성 함수(`CreateDefaultGDIObject`)에서 호출해, 이제부터 SubDC, SubBitmap이 아닌 SubTexture 하나의 변수로 관리하자!

```cpp
class CEngine
{
	SINGLE(CEngine)
private:
	// ...
	// HDC      m_hSubDc;
	// HBITMAP  m_ hSubBitmap;   이 둘은 이제 사용하지 않는다!

	CTexture* m_SubTex;          // 더블 버퍼링 용도 텍스쳐
}
```

```cpp
CEngine::CreateDefaultGDIObject()
{
	// ...
	// 서브 텍스쳐 
	// : 메인 비트맵(윈도우)에 출력하기 전에 먼저 그림들이 그려지는 텍스쳐
	m_subTex = CAssetMgr::GetInst()->CreateTexture(L"SubTexture", 
																								(UINT)m_Resolution.x, 
																								(UINT)m_Resolution.y);
}

// Texture의 멤버 함수에 접근해야 하므로 cpp에서 구
HDC CEngine::GetSubDC()
{
	return m_SubTex->GetDC();
}
```

### 카메라에서 검은색 텍스쳐 생성

이렇게 새로운 화면을 그리는 Texture 객체를 생성하는 방식을 응용해, Fade In/Out 효과에 활용할 검은색 텍스쳐를 카메라에서 생성하자

```cpp
class CCamera
{
	// ...
	CTexture* m_FadeTex;
}
```

```cpp
void CCamera::init()
{
	// Resolution 가져오기 + m_LookAt 위치 설정
	// ...
	// 윈도우 해상도랑 동일한 크기의 검은색 텍스쳐를 생성
	m_FadeTex = CAssetMgr::GetInst()->CreateTexture(L"FadeTexture", 
																									(UINT)vResol.x, 
																									(UINT)vResol.y);
}
```

### Fade In/Out 기능 구현하기

이제 Fade In/Out 을 구현하기 위한 모든 준비가 완료됐으니, CameraEffect와 이를 그리는 render를 모두 작성해보자!!

**CCamera::CameraEffect()**

```cpp
void CCamera::CameraEffect()
{
	if (CAM_EFFECT::NONE == m_Effect)
		return;

	m_Time += DT;

	if (m_Duration < m_Time)
	{
		m_Effect = CAM_EFFECT::NONE;
	}

	// 만약 설정된 카메라 효과가 FADE_IN이라면 alpha가 1(255)에서 0이 되어야 하므로
	if(CAM_EFFECT::FADE_IN == m_Effect)
	{
		m_Alpha = (1.f - (m_Time / m_Duration)) * 255.f;
	}

	// 만약 설정된 카메라 효과가 FADE_OUT이라면 alpha가 0에서 1(255)이 되어야 하므로
	else if (CAM_EFFECT::FADE_OUT == m_Effect)
	{
		m_Alpha = (m_Time / m_Duration) * 255.f;
	}
}
```

**CCamera::Render()**

```cpp
void CCamera::render()
{
	if (m_Alpha <= 0.f)
	{
		return;
	}

	BLENDFUNCTION bf = {};
	
	bf.BlendOp = AC_SRC_OVER;
	bf.BlendFlags = 0;
	bf.SourceConstantAlpha = (int)m_Alpha;
	bf.AlphaFormat = 0;

	AlphaBlend(DC, 0, 0, 
						 m_FadeTex->GetWidth(), m_FadeTex->GetHeight(), 
						 m_FadeTex->GetDc(), 0, 0,
						 m_FadeTex->GetWidth(), m_FadeTex->GetHeight(), bf);
}
```

**카메라 효과 적용**

stage01이 시작할 때 FadeIn이 적용되기 위해서 해당 레벨(`CLevel_Stage01`)에서 begin 오버라이딩

```cpp
class CLevel_Stage01
	: public Clevel
{
private:
	// ...

public:
	virtual void begin() override;
	// ...
}
```

```cpp
void CLevel_Stage01::begin()
{
	CCamera::GetInst()->SetCameraEffect(CAM_EFFECT::FADE_IN, 2.f);
}
```

→ 이로써 카메라 효과를 추가한 결과로 스테이지가 시작할 때 밝아지는 효과(FADE_IN)이 적용된 걸 볼 수 있다!