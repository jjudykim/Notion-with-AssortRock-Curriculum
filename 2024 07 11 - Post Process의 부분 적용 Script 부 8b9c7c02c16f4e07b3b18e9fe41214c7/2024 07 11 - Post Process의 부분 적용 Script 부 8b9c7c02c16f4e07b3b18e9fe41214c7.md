# 2024/07/11 - Post Process의 부분 적용 / Script 분리를 위한 프로젝트 생성

상위 항목: Week33 (https://www.notion.so/Week33-c25c856dcd954fe0af5f24db92f6af83?pvs=21)
상태: 메모
주차: 0100_Week30~39

> 1교시 녹음본
- [https://clovanote.naver.com/s/YmT6kg7kwGtzcCmkBE527VS](https://clovanote.naver.com/s/YmT6kg7kwGtzcCmkBE527VS)

2교시 녹음본
- [https://clovanote.naver.com/s/3KGUA4w7G69pY7RcuP3FofS](https://clovanote.naver.com/s/3KGUA4w7G69pY7RcuP3FofS)
> 

### 부분적으로 적용되는 Post Process

Lens Flare나, Weapon trail Distortion 같은 경우를 생각해보자

→ 화면의 전체가 아닌, 화면의 일부에 Post Process가 적용되는 경우이다!

Object로서 존재하면서, 자신이 렌더링될 때에만 해당 효과가 눈에 보이게 되는 것

이제부턴 부분적으로 적용하려는 Post Process를 적용시킬 물체의 위치가 중요하게 된다!

생성했던 PostProcess Object를 활용해, Transform으로 크기 / 위치를 설정해보자

```cpp
// PostProcess Object
CGameObject* pPostProcessObj = new CGameObject;
pPostProcessObj->SetName(L"GrayFilter");
pPostProcessObj->AddComponent(new CTransform);
pPostProcessObj->AddComponent(new CMeshRender);

pPostProcessObj->Transform()->SetRelativeScale(150.f, 150.f, 1.f);

pPostProcessObj->MeshRender()->SetMesh(CAssetMgr::GetInst()->FindAsset<CMesh>(L"RectMesh"));
pPostProcessObj->MeshRender()->SetMaterial(CAssetMgr::GetInst()->FindAsset<CMaterial>(L"DistortionMtrl"));

m_CurLevel->AddObject(0, pPostProcessObj);
```

- Set된 Material → 새로운 Shader / Material을 사용해 적용할 것이라, 새로운 이름인 “DirstortionMtrl”을 가져와줭ㅆ다!

**새로운 Dirstortion 전용 Shader를 한번 작성해보자**

```cpp
// ===========================
//      DistortionShader
// ===========================
//            Info
// --------------------------- 
// Mesh : RectMesh
// DS_TYPE : NO_TEST_NO_WRITE
// g_text_0 : TargetCopyTex
// g_text_1 ~ 3 : NoiseTexture             // Noise Texture 추가!
// ============================

VS_OUT VS_Distortion(VS_IN _in)
{
	VS_OUT output = (VS_out) 0.f;
	
	output.vPosition = mul(float4(_in.vPos, 1.f), matWVP);    // W, V, P 모두 적용된 행렬로 계산
	output.vUV = _in.vUV;
	
	return output;
}

float4 PS_Distortion(VS_OUT_in) : SV_Target
{
	// 1️⃣ Render Target 해상도 정보 -> g_Resolution 활용
	// 2️⃣ Pixel Shader가 호출된 Pixel 본인의 현재 좌표 -> SV_Position 활용
	// -> 즉, 1️⃣/2️⃣ 연산을 통해서 UV 좌표계와 같은 비율값을 알 수 있게 된다
	float2 vScreenUV = _in.vPosition.xy / g_Resolution;
	float2 vNoiseUV = vScreenUV;
	
	vScreenUV.x += g_EngineTime * 0.1f;
	float4 vNoise = g_tex_3.Sample(g_sam_0, vNoiseUV);
	
	vNoise = (vNoise * 2.f - 1.f) * 0.01f;
	vScreenUV = vScreenUV + vNoise.xy;
	float4 vColor = g_tex_0.Sample(g_sam_0, vScreenUV);
	
	return v
}
```

- 전에 구현했던 쉐이더와는 다르게, Render Target의 부분에 적용하려고 하는 것이므로..
    
    1️⃣ Render Target 전체의 실제 해상도 정보
    
    2️⃣ 전체 해상도를 기준으로 픽셀 쉐이더가 호출된 픽셀들의 좌표 정보
    
    평소에 사용하던 SV_Position이 이미 본인의 좌표를 갖고 있었다
    
- 

→ 전체 해상도를 기준으로 알 수 있어야 한다

현재 PosProcess가 적용되려고 하는 Object의, 보간된 Rect Mesh의 정보를 

이렇게 후처리 효과를 구현하고 적용하는 방법을 알았으니, 다양한 효과를 직접 제작해볼 수 있을 것이다!

<aside>
🗒️ [참조할 수 있는 사이트]

→ Shadertoy

</aside>

<aside>
✏️ [과제]

Post Process 효과 구현해보기 (5개 이상)

</aside>

<aside>
✏️ [수정 사항]

UI까지 후처리가 적용되면 안되니까, UI의 Shader Domain을 새롭게 제작해 분류해주고,

가장 마지막에 UI에 대한 Rendering이 이루어질 수 있도록 적용하기

```cpp
enum Shader_DOMAIN
{
	// ...
	DOMAIN_UI;                // UI
	DOMAIN_DEBUG;             // 디버그
};
```

```cpp
class CCamera :
	public CComponent
{
private:
	// ...
	vector<CGameObject*> m_VecUI;              // UI 오브젝트들

public:
	// ...
}

```

</aside>

### Script를 별도로 관리하자

Engine은 하나의 Tool일 뿐이지, Client를 통해 게임을 제작하는 과정이 이루어져야 하는데..

현재 구조상 Engine에서 각 물체들에 대한 Script가 작성되어 있기 때문에, 이제부턴 분리를 해서 따로 관리해줄 것이다! 그러나 Client에 Script가 포함되게 되면, Editor를 위해 사용되는 Script와 혼재될 수 있기 때문에…

→ Script 전용 프로젝트를 따로 구현해, Client에서 참조해 사용할 수 있도록 구현하자

그런데 이때, Script에 담긴 내용들은 Engine에서 구현한 기능들을 참조하기 때문에..

빌드 순서 및 종속성은 Engine → Script → Client가 되어야 한다

<aside>
📁 **New Project**
-*Script 프로젝트 생성-*
New Project > “Scripts” > 정적 라이브러리

</aside>

추가로, Engine에서 초기화하던 Level 구성 역시 Client에서 Editor를 통해 진행되어야 하지만…

Client 쪽에서는, 임시 레벨을 제작하는 단계를 따로 구현해 코드로 Level을 설정하고 제작하는 과정을 분류해 작성해줄 것이다

<aside>
📁 **New Filter & New Class**
Client > Manager > 02. TestLevel > CTestLevel 클래스 제작

</aside>

**Test Level을 생성하고 초기 세팅을 해주는 CreateTestLevel 함수 작성**

```cpp
#include <Engine/CAssetMgr.h>
#include <Engine/assets.h>

#include <Engine/CLevelMgr.h>
#include <Engine/CLevel.h>
#include <Engine/CLayer.h>
#include <Engine/CGameObject.h>
#include <Engine/CCollisionMgr.h>

void CTestLevel::CreateTestLevel()
{
	// Level 생성
}
```

Level을 변경하는 Change_Level Task를 구현하기

- Param_0 : 변경하려는 레벨 (Level Address)
- Param_1 : 변경하려는 레벨의 초기 상태 (Level State)

```cpp
case TASK_TYPE::CHANGE_LEVEL :
{
	// Param_0 : Level Address, Param_1 : Level State
	CLevel* pLevel = (CLevel*)task.Param_0;
	LEVEL_STATE NextState = (LEVEL_STATE)task.Param_1;
	
	CLevelMgr::GetInst()->ChangeLevel(pLevel);
	pLevel->ChangeState(NextState);
}
	break;
```

```cpp
class CLevelMgr
	: public CSingleton<CLevelMgr>
{

	
};
```

```cpp
bool CLevelMgr::ChangeLevel(CLevel* _NextLevel)
{
	if (m_CurLevel == _NextLevel)
		return false;
		
	if(nullptr != m_CurLevel)
		delete m_CurLevel;
		
	m_CurLevel = _NextLevel;
	
	return true;
}
```

```cpp
void ChangeLevel(CLevel* _Level, LEVEL_STATE _NextLevelState);
```

```cpp
void ChangeLevel(CLevel* _Level, LEVEL_STATE _NextLevelState)
{
	tTask task = {};
	task.Type = TASK_TYPE::CHANGE_LEVEL;
}
```

이제, Create Test Level 시에 마지막에 해당 레벨로 현재 레벨을 변경해주자

```cpp

```

그런데 이때 Task Manager를 통해서 Level을 변경해주는 것이기 때문에…

프로그램이 시작되고 1프레임 정도의 상황에서는 현재 Level 자체가 nullptr인 상태이기 때문에

이에 대한 예외를 설정해주자

```cpp

```

여태 Engine에서 작성해줬던 Script 파일들을, Sciprts 프로젝트로 옮겨주자

<aside>
📁 **New Filter**
Scripts Project > Scripts
→ 기존의 Script 파일들을 해당 Filter로 옮겨주기 (실제 파일들도 경로 변경)

</aside>

<aside>
📁 New Folder
External > Include > Scripts, External > Library > Scripts

</aside>

<aside>
📁 Scripts 프로젝트의 속성 설정
→ Scipts 프로젝트가 빌드된다면, 헤더파일은 Include 쪽으로.. 실행 lib 파일은 Library 쪽으로 빠지도록 포함/출력 디렉토리 설정하기

- 포함 디렉토리 : $(SolutionDir)External\Include\
- 출력 디렉토리 : $(SolutionDir)External\Library\
</aside>

빌드 시 헤더파일들이 자동으로 Include로 이동할 수 있도록 새롭게 Script Header Copy 배치 파일 작성

Level Manager에서, Level이 변경되거나 / Level 내의 구성원들이 변경되었을 때, (즉, Level 내에서 변경점이 발생되었을 때) Outliner와 같이 현재 Level의 구성원들에 영향을 받는 것들을 위한 Trigger를 하나 만들어줘보자

→ `IsLevelChanged()`

Level Manager는 매 Progress에서 Level Changed를 false로 Off 해놓지만

→ Task Manager가 Level 변경 시에 Level Chnaged를 true로 On 해놓음으로써 변경이 반영될 수 있도록

```cpp
void CLevelmgr::LevelChanged()
{
	CTaskMgr::GetInst()->
}
```