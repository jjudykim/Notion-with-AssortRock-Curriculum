# 2024/06/27 - Light 2D Component

태그: C++, DirectX11, 그래픽스, 중급
상위 항목: Week31 (https://www.notion.so/Week31-e62530ed86dd4b30a456fab03e5bfd81?pvs=21)
상태: 메모
주차: 0100_Week30~39

> 1교시 녹음본
-
2교시 녹음본
-
> 

ImGUI의 기본적인 구조나 틀은 제작해뒀으니, Engine 쪽의 기능에 더 집중해보자

## Light2D Component

광원을 다루는 Component를 제작해보자

### 광원(빛)에는 어떤 정보들이 담겨야 할까?

먼저 클래스를 제작하기 전에, 담겨야 할 정보들에 대해서 정리해보고 이를 구조체로 선언하자

- `tLight` : 광원(빛)을 구성하는 기본적인 정보들
    - Color : 광원의 색상
    - Ambient : 광원으로부터 발생하는 환경광(물체나 배경에 스며든 불분명한 일정한 빛)
    - Specular : 반사광의 세기 (3D에서 사용되는 개념, 2D인 현 단계에서는 다루지 말자)
- `tLightInfo` : 한 광원 객체를 구성하는 기본적인 정보들
    - Light : 현재 다루고 있는 광원 객체
    - WorldPos : 광원의 월드 상에서의 좌표, 위치
    - WorldDir : 광원의 월드 상에서의 진행 방향
    - Radius : 광원의 반경, 영향범위

```cpp
struct tLight
{
	Vec4    Color;     
	Vec4    Ambient;
	// Vec4    Specular;
};

struct tLightInfo
{
	tLight          Light;
	Vec3            WorldPos;
	Vec3            WorldDir;
	float           Radius;
	float           Angle;
	LIGHT_TYPE      Type;
};
```

또, 이렇게 제작 가능한 광원들을 여러 타입으로 나눌 수 있도록 enum을 제작해 Light Info에서 다룰 수 있는 정보로 추가해뒀다

```cpp
enum class LIGHT_TYPE
{
	DIRECTIONAL,                // 빛이 평행하게 방향성을 가지고 퍼짐 (비현실적인 요소)
	POINT,                      // 한 점에서부터 뻗어나가는 빛
	SPOT,                       // 
}
```

여기서 타입으로 분류된 빛의 종류들을 정리해보자

- Directional : 빛이 평행하게 방향성을 가지고 퍼지는 빛
    
    → 전역적으로 사용하는 빛, 멀리서부터 오는 빛을 표현할 때 주로 사용한다 (광원의 진원지가 멀리 떨어져있어 광각을 무시해도 될 정도일 때, 비현실적인 요소이긴 하나 제작의 편의성을 위해) ex) 태양광, 달빛
    
- Point : 한 점에서부터 뻗어나가는 빛
    
    → 우리가 현실에서 생각하는 광원과 가장 유사한 개념으로, 광원과의 위치에 따라 광원으로부터 영향을 받는 방향이 달라진다 ex) 횃불, 모닥불
    
- Spot : 지정된 각도로, 한 쪽 방향으로 모아서 쏘는 빛
    
    → 광원 자체는 Point와 같으나, 반사를 활용해 지정된 각도와 지정된 방향으로만 빛이 나아가도록 제약되는 빛을 말한다 ex) 손전등, 스포트라이트
    

지금까지 정리해둔 구조체를 기반으로,  Light Component 클래스를 작성해보자!

**Light2D Component의 기본적인 틀 작성**

```cpp
class CLight2D:
	 public CComonent
{
private:
	

public:
	CLight2D();
	~CLight2D();
};
```

```cpp
void CLight2D::FinalTick()
{
	m_Info.WorldPos = Transform()->GetWorldPos();
	m_Info.WorldDir = Transform()->GetWorldDir(DIR::RIGHT);  
	// 광원이 진행하는 방향을 Right 기준으로 생각
}
```

광원에 대한 정보가 실제로 쉐이더 코드에서 영향을 줄 수 있어야 한다

→ 즉, 광원이 있는지, 없는지에 따라서 우리가 작성했던 std2D 파일은 또 변경될 수 있다

광원 객체의 개수는 유동적이니까.. 구조화 버퍼를 활용해서 광원 정보를 렌더링할 수 있을 것이다

그럼 광원 정보들을 Render Manager 쪽에서 체크를 해줄 수 있도록..

Render Manager에서 광원 정보 체크

```cpp
vector<CLight2D*>          m_vecLight2D;
CStructured
```

그럼 광원은, 매 Final Tick에서 자신의 대한 정보를 Render Manager 쪽에 등록해야 한다

```cpp
void CLight2D::FinalTick()
{
	// ...
	// 자신을 Render Manager에 등록
	CRenderMgr::GetInst()->RegisterLight2D(this);
}
```

그럼 우리 광원 구조체를 구조화 버퍼로 전달하기 위해서, 패딩이 들어가야 하는지 확인해보자

```cpp
struct tLightInfo
{
	tLight          Light;        // 32 byte
	Vec3            WorldPos;     // 12 byte
	Vec3            WorldDir;     // 12 byte
	float           Radius;       // 4 byte
	float           Angle;        // 4 byte
	LIGHT_TYPE      Type;         // 
};
```

그리고 쉐이더 측에서도, 우리의 구조체와 연동해 사용할 구조체를 작성해주자

```cpp
struct tLightInfo
{
	tLight light;
	float3 WorldPos;
	float3 WorldDir;
	
}
```

<aside>
🗒️ [수정 사항]

Render Manager의 Tick의 구조를 조금 변경해줬다!
Render가 시작될 때 처리해야 되는 작업들을 RenderStart라는 함수를 통해 한 곳으로 정리해줬다

</aside>

```cpp
void CRenderMgr::RenderStart()
{
	// 렌더 타겟을 지정
	Ptr<CTexture> pRTTex = 
	Ptr<CTexture> pDSTex =
	CONTEXT->OMSetRenderTargets(1, pRTTex->GetRTV().GetAddressOf(), pDSTex->GetDSV().Get());
	
	// Light2D 정보 업데이트 및 바인딩
	vector<tLightInfo> vecLight2DInfo;
	for(size_t i = 0; i < m_vecLight2D.size(); ++i)
	{
		vecLight2DInfo.push_back(m_vecLight2D[i]->GetLightInfo());
	}
	
	if (m_Light2DBuffer->GetElementCount() < vecLight2DInfo.size())
	{
		m_Light2DBuffer->Create(sizeof(tLightInfo), vecLight2DInfo.size());
	}
	
	m_Light
}
```

광원 정보를 담는 구조화 버퍼의 레지스터 할당은 11, 12번을 사용해줬다!

```cpp
StructuredBuffer<tLigthInfo> g_Light2D : register(t11);       //  2D 광원
StructuredBuffer<tLigthInfo> g_Light2D : register(t12);       //  3D 광원 (추후 사용)

```

이렇게 Redner Start를 통해 담은 정보들을 깨끗하게 비워주는 함수 Clear를 제작해주자

```cpp
void CRenderMgr::Clear()
{
	
}
```

<aside>
🗒️ [수정 사항]

새로운 컴포넌트를 추가함으로써 수정해야 하는 몇몇 부분들

```cpp
GET_COMPONENT(Light2D, LIGHT2D);
```

```cpp
GET_OTHER_COMPONENT(Light2D);
```

</aside>

### 광원(Light2D) 전용 Object를 추가해보자

```cpp
pObject = new CGameObject;
pObject->SetName(L"Directional");
pObject->AddComponent(new CTransform);
pObject->AddComponent(new CLight2D);

m_CurObject 

```

광원이 존재함을 알리는 Debug에 대한 아이디어 → Editor Object 활용

### Editor UI를 통해 Light Component를 컨트롤해보자

Light Component 담당 UI를 제작해, Light 정보를 제어할 수 있도록 작성해보자

```cpp
class Light2DUI
 : ComponentUI
{
public:
	virtual void Update() override;
	
public:
	Light2DUI();
	
};
```

```cpp
Light2DUI::Light2DUI()
	: ComponentUI(COMPONENT_TYPE::LIGHT2D)
{
}

Light2DUI::Update()
{
	Title();
}
```

UI에 띄워줘야 하는 광원의 정보들을 추려보자

- Type : 광원의 종류
- light : 광원의 색상 및 주변광
- Raius : 광원의 반경 (Point / Spot 광원에만 해당되는 사항)
- Angle : 광원의 범위 각도 (Spot 광원에만 해당되는 사항)

```cpp

```

Inspector의 Target Object도, 우리가 방금 생성한 Light Object로 설정해주자

```cpp

```