# 2024/06/28 - Light2D Component를 UI로 조작하기

태그: C++, DirectX11, 게임수학, 그래픽스, 중급
날짜: 2024/06/28
상위 항목: Week31 (https://www.notion.so/Week31-e62530ed86dd4b30a456fab03e5bfd81?pvs=21)
상태: 완료
주차: 0100_Week30~39

> 0교시 (보강) 녹음본
- [https://clovanote.naver.com/s/k4fKcfzMLY2GuwpgXRqaeNS](https://clovanote.naver.com/s/k4fKcfzMLY2GuwpgXRqaeNS)

1교시 녹음본
- [https://clovanote.naver.com/s/aUVdafF3qy6tqWTd8tkkJPS](https://clovanote.naver.com/s/aUVdafF3qy6tqWTd8tkkJPS)

2교시 녹음본
- [https://clovanote.naver.com/s/ufdZcNYV5usG289spgDxdFS](https://clovanote.naver.com/s/ufdZcNYV5usG289spgDxdFS)
> 

## Light2D UI 제작하기

### Light2D UI로 컴포넌트를 컨트롤해보자

Light Component 담당 UI를 제작해, Light 정보를 제어할 수 있도록 작성해보자

**Light2D UI의 기본적인 틀 작성**

```cpp
class Light2DUI :
	public ComponentUI
{
public:
	virtual void Update() override;
	
public:
	Light2DUI();
	~Light2DUI();
};
```

```cpp
Light2DUI::Light2DUI()
	: ComponentUI(COMPONENT_TYPE::LIGHT2D)
{
}

Light2DUI::~Light2DUI()
{
}

Light2DUI::Update()
{
	Title();
}
```

Light2D UI를 Inspector에서 확인할 수 있도록 띄우려고 하니,

Inspector에서

- 생성자에서 Light2DUI 객체를 생성해 본인에게 AddChild
- Target Object를 생성해놨던 Light 전용 Object인 “*Directional*”로 설정

```cpp
Insepctor::Inspector()
	: m_TargetObject(nullptr)
	, m_arrComUI{}
{
	// ...
	m_arrComUI[(UINT)COMPONENT_TYPE::LIGHT2D] = new Light2DUI;
	m_arrComUI[(UINT)COMPONENT_TYPE::LIGHT2D]->SetName("Light2DUI");
	m_arrComUI[(UINT)COMPONENT_TYPE::LIGHT2D]->SetChildSize(ImVec2(0.f, 200.f));
	AddChild(m_arrComUI[(UINT)COMPONENT_TYPE::LIGHT2D]);
}

// ...
void Inspector::Update()
{
	if (nullptr == m_TargetObject)
	{
		//SetTargetObject(CLevelMgr::GetInst()->FindObjectByName(L"Player"));
		//SetTargetObject(CLevelMgr::GetInst()->FindObjectByName(L"MainCamera"));
		SetTargetObject(CLevelMgr::GetInst()->FindObjectByName(L"Directional"));
		return;
	}
	
	// ...
}
```

### Light2D UI 구성하기

UI에 띄워줘야 하는 광원의 정보들을 추려보자

- Type : 광원의 종류
- light : 광원의 색상 및 주변광
- Radius : 광원의 반경 (Point / Spot 광원에만 해당되는 사항)
- Angle : 광원의 범위 각도 (Spot 광원에만 해당되는 사항)

이를 하나씩 구현해보자

1️⃣ Type : 광원의 정보 → Combo Box로 구성

```cpp
LIGHT_TYPE	Type = pLight->GetLightType();

const char* items[] = { "DIRECTIONAL", "POINT", "SPOT" };
const char* combo_preview_value = items[(UINT)Type];

// Label
ImGui::Text("Light Type");
ImGui::SameLine(100);
ImGui::SetNextItemWidth(180);

// Combo Box
if (ImGui::BeginCombo("##LightTypeCombo", combo_preview_value))
{
	for (int i = 0; i < 3; i++)
	{
		const bool is_selected = ((UINT)Type == i);

		if (ImGui::Selectable(items[i], is_selected))
		{
			Type = (LIGHT_TYPE)i;
		}

		if (is_selected)
			ImGui::SetItemDefaultFocus();
	}
	ImGui::EndCombo();
}

pLight->SetLightType(Type);
```

2️⃣ light : 광원 색상 정보 (+ 환경광) -> Color Picker(selectors) 활용

```cpp
const tLightInfo& info = pLight->GetLightInfo();         // 현재 색상 정보를 띄우고

// Color Label
ImGui::Text("Light Color");
ImGui::SameLine(100);
ImGui::ColorEdit3("##LightColor", info.light.Color);     // Color Picker UI를 함께 게시

// Ambient Label
ImGui::Text("Light Ambient");
ImGui::SameLine(100);
ImGUi::ColorEdit3("##LightAmbient", info.light.Ambient);
```

Radius와 Angle처럼, 광원의 타입에 따라 UI의 활성화 / 비활성화 여부를 설정해야 하는 경우를 생각해보자

<aside>
🗒️ [수정 사항]
경우에 따라 UI를 able / disable하게 만드는 손 쉬운 방법
→ `BeginDisabled` 함수를 활용
: 인자로 들어가는 조건이 true일 때 해당 항목을 disabled

```cpp
// Perspective가 아닐 때 diabled (즉 Orthographic 시에만 활성화)
ImGui::BeginDisabled(pCam->GetProjType() != PROJ_TYPE::PERSPECTIVE);
ImGui::Text("FOV");
ImGui::SameLine(100);
ImGui::InputFloat("##FOV", &FOV);
ImGui::EndDisabled();

// Orthographic이 아닐때, diabled (즉 Orthographic 시에만 활성화)
ImGui::BeginDisabled(pCam->GetProjType() != PROJ_TYPE::ORTHOGRAPHIC);   
float Scale = pCam->GetScale();
ImGui::Text("Scale");
ImGui::SameLine(100);
ImGui::InputFloat("##Scale", &Scale);
pCam->SetScale(Scale);
ImGui::EndDisabled();
```

</aside>

3️⃣ Radius : Point / Spot 광원에서, 광원의 반경을 설정

```cpp
ImGui::BeginDisabled(Type == LIGHT_TYPE::DIRECTIONAL);

ImGui::Text("Light Radius");
ImGui::SameLine(100);
ImGui::DragFloat("##DragRadius", (float*)&info.Radius, 0.1f);

ImGui::EndDisabled();
```

4️⃣ Angle: Spot 광원에서, 광원의 각도 범위를 설정

```cpp
ImGui::BeginDisabled(Type != LIGHT_TYPE::SPOT);

float Angle = info.Angle;
Angle = (ANGLE / XM_PI) * 180.f;

ImGui::Text("Light Angle");
ImGui::SameLine(100);
ImGui::DragFloat("##DragAngle", &Angle, 0.1f);

Angle = (Angle / 180.f) * XM_PI;
pLight->SetAngle(Angle);

ImGui::EndDisabled();
```

🟦 이를 종합해, UI를 구성하기 위해 작성한 Update의 함수를 전체적으로 보면 다음과 같다!

```cpp
void Light2DUI::Update()
{
	Title();

	CLight2D* pLight = GetTargetObject()->Light2D();

	// 광원 종류
	LIGHT_TYPE	Type = pLight->GetLightType();

	const char* items[] = { "DIRECTIONAL", "POINT", "SPOT" };
	const char* combo_preview_value = items[(UINT)Type];

	ImGui::Text("Light Type");
	ImGui::SameLine(100);
	ImGui::SetNextItemWidth(180);

	if (ImGui::BeginCombo("##LightTypeCombo", combo_preview_value))
	{
		for (int i = 0; i < 3; i++)
		{
			const bool is_selected = ((UINT)Type == i);

			if (ImGui::Selectable(items[i], is_selected))
			{
				Type = (LIGHT_TYPE)i;
			}

			// Set the initial focus when opening the combo (scrolling + keyboard navigation focus)
			if (is_selected)
				ImGui::SetItemDefaultFocus();
		}
		ImGui::EndCombo();
	}

	pLight->SetLightType(Type);

	// 광원 색상정보	
	const tLightInfo& info = pLight->GetLightInfo();
	
	ImGui::Text("Light Color");
	ImGui::SameLine(100);
	ImGui::ColorEdit3("##LightColor", info.light.Color);

	ImGui::Text("Light Ambient");
	ImGui::SameLine(100);
	ImGui::ColorEdit3("##LightAmbient", info.light.Ambient);

	// 광원의 반경 ( Point, Spot )
	ImGui::BeginDisabled(Type == LIGHT_TYPE::DIRECTIONAL);

	ImGui::Text("Light Radius");
	ImGui::SameLine(100);
	ImGui::DragFloat("##DragRadius", (float*)&info.Radius, 0.1f);

	ImGui::EndDisabled();

	// 광원의 범위 각도
	ImGui::BeginDisabled(Type != LIGHT_TYPE::SPOT);

	float Angle = info.Angle;
	Angle = (Angle / XM_PI) * 180.f;

	ImGui::Text("Light Angle");
	ImGui::SameLine(100);
	ImGui::DragFloat("##DragAngle", &Angle, 0.1f);

	Angle = (Angle / 180.f) * XM_PI;
	pLight->SetAngle(Angle);

	ImGui::EndDisabled();	
}
```

### 광원의 정보를 쉐이더에게 전달하기 (With Global Data)

쉐이더 쪽에서 자주 필요로 하는 몇몇 데이터들이 있는데,  쉐이더에서 특별히 분류하기 힘든 데이터들을 모아서, 새로운 Global Data 전용 상수 버퍼를 제작해 바인딩받을 수 있도록 하자

**Global Data 전용 상수 버퍼 작성**

```cpp
cbuffer GLOBAL_DATA : register(b3)
{
	float        g_DT;               // Delta Time
	float        g_EngineDT;         // Engine Delta Time
	float        g_Time;             // Time
	float        g_EngineTime;       // Engine Time
	
	float2       g_Resolution;       // 현재 지정된 렌더타겟의 해상도 정보

	// 바인딩 된 구조화 버퍼에 광원이 몇 개 들어있는지
	int          g_Light2DCount;     // 2D 광원의 개수
	int          g_Light3DCount;     // 3D 광원의 개수
}
```

- 여기서 Global Data로 광원의 개수가 존재하는 이유?
    
    → 현재 광원이 몇 개가 존재하는지에 대해서는, 우리가 전달한 구조화 버퍼만으로는 알 수 없다!
    
    Light2D의 정보를 담은 Structured Buffer의 개수를 알기 위해서, 광원의 개수 역시 포함해줬다
    

이와 연결할 상수 버퍼 연동 구조체 역시, Engine쪽에서 작성해주자

```cpp
struct tGlobalData
{
	// 시간 관련 정보
	float g_DT;
	float g_EngineDT;
	float g_Time;
	float g_EngineTime;
	
	// 현재 지정된 렌더타겟의 해상도 정보
	Vec2  g_Resolution;
	
	// 바인딩 된 구조화버퍼에 광원이 몇 개 들어 있는지
	int   g_Light2DCount;
	int   g_Light3DCount;
};
extern tGlobalData g_GlobalData;                          // 전역 변수 선언
```

선언한 `g_GlobalData`에 대한 구현 역시 extern에서 작성해줬다

→ 멤버들은 각각 적절한 위치에서 설정해주기!

```cpp
tGlobalData g_GlobalData = {};
// ...
```

**시간 관련 정보 갱신**

```cpp
void CTimeMgr::Tick()
{
	// ...
	// GlobalData
	g_GlobalData.g_DT = m_DeltaTime;
	g_GlobalData.g_EngineDT = m_E_DeltaTime;
	g_GlobalData.g_Time = m_Time;
	g_GlobalData.g_EngineTime = m_E_Time;
}
```

**렌더타겟의 해상도 / 광원 정보 갱신**

```cpp
void CRenderMgr::RenderStart()
{
	// ...
	g_GlobalData.g_Resolution = Vec2((float)pRTTex->Width(), (float)pRTTex->Height());
	g_GlobalData.g_Light2DCount = (int)m_vecLight2D.size();
	// ...
}

```

또 Device에서 상수 버퍼들을 생성하는 시점에서, Global Data 버퍼 역시 생성해주자!

```cpp
int CDevice::CreateConstBuffer()
{
	// ...
	
	pCB = new CConstBuffer;
	if (FAILED(pCB->Create(CB_TYPE::GLOBAL, sizeof(tGlobalData))))
	{
		MessageBox(nullptr, L"상수버퍼 생성 실패", L"초기화 실패", MB_OK);
		return E_FAIL;
	}
	m_arrCB[(UINT)CB_TYPE::GLOBAL] = pCB;
	
	return S_OK;
}
```

그리고 Render Manager의 RenderStart 함수를 통해,

마지막에 Global data 상수 버퍼를 바인딩해주는 작업까지 마무리해주자

```cpp
void CRenderMgr::RenderStart()
{
	// ...
	// GlobalData 바인딩
	static CConstBuffer* pGlobalCB = CDevice::GetInst()->GetConstBuffer(CB_TYPE::GLOBAL);
	pGlobalCB->SetData(&g_GlobalData);
	pGlobalCB->Binding();
}
```

<aside>
🗒️ [수정 사항]

Render Target을 Clear해주는 함수였던 Device의 Clear 함수를, 
Render Manager에서 일괄적으로 Render 관련된 작업들을 컨트롤하기 위해서 해당 작업을 Render Start로 옮겨줄 것이다

```cpp
void CRenderMgr::RenderStart()
{
	// ...
	// TargetClear
	float color[4] = { 0.f, 0.f, 0.f, 1.f };
	CONTEXT->ClearRenderTargetView(pRTTex->GetRTV().Get(), color);
	CONTEXT->ClearDepthStencilView(pDSTex->GetDSV().Get(), D3D11_CLEAR_DEPTH | D3D11_CLEAR_STENCIL, 1.f, 0);
	
	// ...
}

```

</aside>

### 쉐이더를 통해 물체들이 Light2D에 영향을 받을 수 있도록 해보자

이제 광원의 유무가 쉐이더에 영향을 끼칠 수 있도록 쉐이더 코드를 수정해주자

아직까지 광원의 개수까지는 신경쓰지 말고, 당장은 우리가 광원 하나만 신경써보자!

→ `g_Light2D[0]` 으로 사용 (우리가 광원 오브젝트 1개는 생성해놨으니!)

```cpp
float4 PS_Std2D(VTX_OUT _in) : SV_Target
{
	// ...
	// 광원 적용

	// Light Type이 0, 즉 Directional Light인 경우
	if (g_Light2D[0].Type == 0)                                   
	{
		vColor.rgb = g_Light2D[0].light.Color.rgb * vColor.rgb       // 원광
		           + g_Light2D[0].light.Ambient.rgb * vColor.rgb;    // 환경광(ambient)
	}
	// Light Type이 1, Point Light인 경우
	else if (g_Light2D[0].Type == 1)
	{
		// 광원의 위치 / 광원의 반경을 통해 광원에게 영향을 받는 픽셀에 대해 알아야 한다
		// 각각의 픽셀들이 본인의 World상에서의 위치를 광원의 위치와 비교해 
		// 본인이 반경에 들어있는지를 확인! -> 반경에 들어있는 경우에만 광원 효과를 내야 한다
		// 단, 2D상에서는 x,y좌표만을 고려해 계산해야 한다 
		// (깊이를 표현하기 위해 z를 활용하지만, 광원 효과는 그대로 받아야 하기 때문에)
		
		// 점광원과 픽셀까지의 거리
		float fDist = distance(g_Light2D[0].WorldPos, _in.vWorldPos);
		
		// 빛의 세기를 연산하는 두 가지 방법
		// 1) 광원으로부터 떨어진 거리에 비례하는 빛의 세기 (거리값에 따른 비율로 치환)
		float fPow1 = saturate(1.f - fDist / g_Ligth2D[0].Radius);     
		
		// 2) 중심점이 더 밝고, 반경 경계로 갈수록 급속하게 약해지는 빛의 세기
		float fPow2 = saturate(cos(saturate(fDist / g_Light2D[0].Radius) * (PI / 2.f))));
		
		// 최종 색상 계산 = 빛의 색 * 거리에 따른 광원의 세기
		vColor.rgb = vColor.rgb * g_Light2D[0].light.Color.rgb * fPow2;    // (2번 연산 채택)
	}
	
	return vColor;
}
```

- HLSL에서는 두 점 사이의 거리를 구해주는 함수가 정의되어 있다 → `distance`
- Point 광원은, 원광원의 위치에서 멀어질수록 빛의 세기가 점점 감소하다가 반경 외로는 광원의 영향을 받지 않도록 제작해야 한다
    
    → 극단적으로 항상 일정한 빛이 비추다가 반경 외로 영향을 받지 않는다면 상당히 부자연스럽겠지?!
    
    따라서 거리에 따른 빛의 세기를 계산해줘야 한다 → `fPow`
    
- `saturate` (0-1 사이의 값으로 치환하는 함수) 를 사용한 이유?
    
    : 빛의 세기라는 것이 음수로 갈 수는 없으니, 연산결과가 0과 1 사이의 비율로 나타날 수 있도록 제약!
    
- 여기서 조금 더 디테일을 추가하자면, 현재는 거리에 따른 빛의 세기의 감소가 일정하다 (1번 연산)
    
    그러나 더 현실적이게 보이기 위해선, 일정치 거리의 수준까지는 빛의 세기가 유지되다가 영향 반경의 근처에 갈 경우 급격하게 어두워지도록 설정하는 것이 자연스럽다! (2번 연산)
    
    둘을 그래프로 그려보자면 대강 이런 모양
    
    ![Untitled](2024%2006%2028%20-%20Light2D%20Component%E1%84%85%E1%85%B3%E1%86%AF%20UI%E1%84%85%E1%85%A9%20%E1%84%8C%E1%85%A9%E1%84%8C%E1%85%A1%E1%86%A8%E1%84%92%E1%85%A1%E1%84%80%E1%85%B5%200fedbaf0db204c4aad8786bc738216e8/Untitled.png)
    
    ![Untitled](2024%2006%2028%20-%20Light2D%20Component%E1%84%85%E1%85%B3%E1%86%AF%20UI%E1%84%85%E1%85%A9%20%E1%84%8C%E1%85%A9%E1%84%8C%E1%85%A1%E1%86%A8%E1%84%92%E1%85%A1%E1%84%80%E1%85%B5%200fedbaf0db204c4aad8786bc738216e8/c32164d1-f4a7-4a1b-a398-cf167f5d84a0.png)
    
    그래프를 보면, 우리가 만드려는 광원 효과가  cos 그래프와 유사하므로 이를 활용해 연산해주겠다!
    
    →  `fPow = saturate(cost((fDist / g_Light2D[0].Radius) *  (PI / 2.f)));` (2번 연산)
    
    즉, 둘 사이의 거리를 각도로 치환해서 거리에 따른 빛의 세기를 cos 그래프 형태로 사용하는 것
    
    이렇게 빛의 세기를 통해 원하는 연출을 선택해 표현할 수 있다~!
    

이렇게 작성한 코드를 다른 Material이나 Shader에 적용할 수 있어야 하므로, 

간편하게 사용하기 위해서 함수로 제작해주자

**광원을 연산하는 함수 CalculateLight2D 제작**

```cpp
// 인자 : 적용할 광원의 Index, 광원을 적용받을 Pixel의 위치, (&)광원 세기의 합 (색 제외)
void CalculateLight2D(int _LightIdx, float3 _WorldPos, inout tLight _Light)
{
	// 현재 사용할 광원
	tLightInfo Info = g_Light2D[_LightIdx];
	
	if (Info.Type == 0)
	{
		_Light.Color = Info.light.Color;
		_Light.Ambient = Info.light.Ambient;
	}
	else if (g_Light2D[0].Type == 1)
	{	
		float fDist = distance(g_Light2D[0].WorldPos, _in.vWorldPos);
	
		// 1) 광원 세기 연산 1 (거리 비례)
		float fPow1 = saturate(1.f - fDist / g_Ligth2D[0].Radius);
	
		// 2) 광원 세기 연산 2 (중심점 위주)
		float fPow2 = saturate(cos(saturate(fDist / g_Light2D[0].Radius) * (PI / 2.f))));
	
		// 최종 색상 계산
		_Light.Color = Info.light.Color.rgb * fPow2;
		_Light.Ambient = Info.light.Ambient.rgb;
}
```

- 인자인 `_Light` 를 **inout**으로 설정한 이유!
    
    : 읽고 쓰기가 모두 가능한 인자여야 하기 때문에 
    
    (즉, C++로 따지면 참조(`&`)를 통해 원본 값을 변경해야 하는 것과 동일하다)
    
- 현재 index의 광원을 연산할 때마다, 나(픽셀)에게 현재 적용되어야 할 광원을 전달받은 후*(입력, in)*에 광원의 세기에 따른 Color 및 Ambient를 연산한 후, 이를 다시 인자로 받은 _Light 변수에 결과값을 적용*(출력, out)*하게 된다
    
    → 즉, 각 픽셀마다 광원의 영향을 순차적으로 받게끔 해준 것이다
    

그럼 Shader에서는, 해당 함수를 활용해서 간편하게 현재 적용되는 광원들을 연산할 수 있게 된다

```cpp
float4 PS_Std2D(VTX_OUT _in) : SV_Target
{ 
	// ...
	// 광원 적용      
  tLight Light = (tLight) 0.f;
    
  for (int i = 0; i < g_Light2DCount; ++i)
  {
	  CalculateLight2D(i, _in.vWorldPos, Light);
  }
  
  // 최종 색상 계산
  vColor.rgb = vColor.rgb * Light.Color.rgb
					   + vColor.rgb * Light.Ambient.rgb;
	
	return vColor;
}
```

- 적용되는 빛과 색상 정보를 분리하고 연산한 이유?
    
    → 하나의 물체에 *여러 개의 빛*이 적용될 수 있기 때문에!
    
    : 각 광원들이 주는 빛(세기)을 모두 합친 후에 Object에 영향을 미치게끔 해야 한다
    
    빛을 중첩해서 영향을 받는 경우, 받는 빛에 따른 색상을 연산하며 중첩하게 된다면 우리가 원하는 결과와 다르게 빛이 적용될 수 있다
    

<aside>
📝 [과제]

Spotlight 광원 구현해보기

```cpp
// SpotLight 인 경우
else
{
    float fDist = distance(Info.WorldPos.xy, _WorldPos.xy);
    
    // 광원의 방향 벡터 정규화
    float2 lightDir = normalize(_WorldPos.xy - Info.WorldPos.xy);        
		
		// 두 벡터 간의 내적
    float fAngleToCos = dot(lightDir.xy, Info.WorldDir.xy);              

    if (fAngleToCos > cos(Info.Angle / 2.f))
    {
        float fPow = saturate(cos(saturate(fDist / Info.Radius) * (PI / 2.f)));
        _Light.Color.rgb += Info.light.Color.rgb * fPow;
        _Light.Ambient.rgb += Info.light.Ambient.rgb;
    }
}    
```

</aside>

광원 전용 오브젝트를 두 개 제작해서, 서로 영향을 미치게끔 해보자

```cpp
void CLevelMgr::Init()
{
	// ...
	// 광원 오브젝트 추가
	pObject = new CGameObject;
	pObject->SetName(L"PointLight 1");
	pObject->AddComponent(new CTransform);	
	pObject->AddComponent(new CLight2D);

	pObject->Light2D()->SetLightType(LIGHT_TYPE::POINT);
	pObject->Light2D()->SetLightColor(Vec3(1.f, 1.f, 1.f));
	pObject->Light2D()->SetRadius(500.f);
	pObject->Transform()->SetRelativePos(Vec3(-300.f, 0.f, 100.f));
	
	m_CurLevel->AddObject(0, pObject);

	pObject = new CGameObject;
	pObject->SetName(L"PointLight 2");
	pObject->AddComponent(new CTransform);
	pObject->AddComponent(new CLight2D);

	pObject->Light2D()->SetLightType(LIGHT_TYPE::POINT);
	pObject->Light2D()->SetLightColor(Vec3(0.2f, 0.2f, 0.8f));
	pObject->Light2D()->SetRadius(500.f);
	pObject->Transform()->SetRelativePos(Vec3(300.f, 0.f, 100.f));

	m_CurLevel->AddObject(0, pObject);
	
	// ...
}
```