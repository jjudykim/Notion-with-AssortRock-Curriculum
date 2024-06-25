# 2024/06/25 - Camera UI (2), MeshRender UI

태그: C++, DirectX11, 과제수행필요, 중급
날짜: 2024/06/25
상위 항목: Week31 (https://www.notion.so/Week31-e62530ed86dd4b30a456fab03e5bfd81?pvs=21)
상태: 작성중
주차: 0100_Week30~39

> 1교시 녹음본
- [https://clovanote.naver.com/s/yzzQm7nuVGheTxVvqa6Bd3S](https://clovanote.naver.com/s/yzzQm7nuVGheTxVvqa6Bd3S)

2교시 녹음본
- [https://clovanote.naver.com/s/rheAR4HeGsJVDidPWyDDiCS](https://clovanote.naver.com/s/rheAR4HeGsJVDidPWyDDiCS)
> 

### Camera UI (2)

**Projection Type - 카메라 투영방식을 설정**

```cpp
void CameraUI::Projection()
{
	// Projection Type UI에 Label 붙이기
	ImGui::Text("Projection");
	ImGui::SameLine(100);
	
	// ComboBox의 길이 설정
	ImGui::SetNextItemWidth(200);
	
	// ComboBox 설정 =====================================
	CCamera* pCam = GetTargetObject()->Camera();
	PROJ_TYPE Type = pCam->GetPro
	
	const char* items[] = {"Orthographic", "Perspective"};
	const char* combo_preview_value = items[Type];
	static bool bOpen = false;
	
	if (ImGui::BeginCombo("##ProjectionCombo", combo_preview_value))
	{
		for(int i = 0; i < 2; ++i)
		{
			const bool is_selected = (Type == i);
			
			if (ImGui::Selectable(items[i], is_selected))
			{
				Type = (PROJ_TYPE)i;
			}
			
			// ComboBox를 열었을 때 기본적으로 포커싱이 되도록 설정
			if (is_selected)
				ImGui::SetItemDefaultFocus();
		}
		ImGui::EndCombo();
	}
	
	// ComboBox에서 선택한 투영 방식이 반영되도록 설정
	pCam->SetProjType(Type);
}
```

→ 이런 식으로 여러 선택지 중 하나를 선택할 수 있는 ComboBox가 완성된다

![Untitled](2024%2006%2025%20-%20Camera%20UI%20(2),%20MeshRender%20UI%200cf4c18d326d4adb9cca342c87377b98/Untitled.png)

**Width / Height / AspectRatio / Far / FOV → 카메라의 투영 범위를 설정**

```cpp

// 사실 선생님은 따로 함수로 빼놓진 않으셨는데,
// 난 Update 함수가 더 깔끔했으면 해서 함수로 따로 분리했다!
void CameraUI::ProjectionTransform()
{
	float Width = pCam->GetWidth();
	ImGui::Text("Width");
	ImGui::SameLine(100);
	ImGui::InputFloat("##Width", &Width);
	pCam->SetWidth(Width);
	
	float Height = pCam->GetHeight();
	ImGui::Text("Height");
	ImGui::SameLine(100);
	ImGui::InputFloat("##Height", &Height);
	pCam->SetHeight(Height);

	float AR = pCam->GetAspectRatio();
	ImGui::Text("AspectRatio");
	ImGui::SameLine(100);
	ImGui::InputFloat("##AspectRatio", &AR, ImGuiInputTextFlags_::ImGuiInputTextFlags_ReadOnly);  
	// 해상도는 Width / Height에 의해 자동 설정되는 값으로, ReadOnly로 설정
	
	float Far = pCam->GetFar();
	ImGui::Text("Far");
	ImGui::SameLine(100);
	ImGui::InputFloat("##Far", &Far);
	pCam->SetFar(Far);
	
	//Perspective 전용 투영 범위 설정
	float FOV = pCam->GetFOV();
	FOV = (FOV / XM_PI) * 180.f;
	
	// 현재 투영 모드가 Perspective일때만 FOV UI 활성화 설정
	if (pCam->GetProjType() != PROJ_TYPE::PERSPECTIVE)
	{
		ImGui::BeginDisabled();
		
		ImGui::Text("FOV");
		ImGui::SameLine(100);
		ImGui::InputFloat("##FOV", &FOV);
		
		ImGui::EndDisabled();
	}
	else
	{
		ImGui::Text("FOV");
		ImGui::SameLine(100);
		ImGui::InputFloat("##FOV", &FOV);
	}
	FOV = (FOV / 180.f) * XM_PI;
	pCam->SetFOV(FOV);
	
	// New!) Orthographic 전용 투영 범위 설정
	// 현재 투영 모드가 Orthgraphic일때만 Scale UI 활성화 설정
	if (pCam->GetProjType() != PROJ_TYPE::ORTHOGRAPHIC)
	{
		ImGui::BeginDisabled();
		
		float Scale = pCam->GetScale();
		ImGui::Text("Scale");
		ImGui::SameLine(100);
		ImGui::inputFloat("##Scale", &Scale);
		pCam->SetScale(Scale);
		
		ImGui::EndDisabled();
	}
	else
	{
		float Scale = pCam->GetScale();
		ImGui::Text("Scale");
		ImGui::SameLine(100);
		ImGui::InputFloat("##Scale", &Scale);
		pCam->SetScale(Scale);
	}
}	
```

- FOV의 Set, Get 함수는 그대로 라디안 단위를 사용하더라도, Editor에서는 60분법으로 전환해서 출력하는 것이 나을테니 이 점을 반영해서 작성해줬다
    
    Cam FOV를 가져와 60분법으로 사용할 때 → `FOV = (FOV / XM_PI) * 180.f;`
    
    변경한 FOV를 다시 라디안 단위로 변경해 저장 → `FOV = (FOV / 180.f) * XM_PI;`
    
- FOV는 투영 모드가 Perspective일 때, Projection Scale은 Orthographic 일 때에만 활성하기 위해서 투영 모드에 따라 분기 처리를 해줬다!

<aside>
🗒️ [수정 사항]
Camera Component에서 Projection Scale이라는 멤버 함수를 추가했다!
→ 직교 투영(Orthographic)에서 zoom in/out 하듯이 투영의 범위를 좁히거나 늘릴때 사용할 수 있다

```cpp
class CCamera :
	public CComponent
{
private:
	// ...
	float           m_FOV;                // Perspective 전용
	float           m_ProjectionScale;    // Orthographic 전용
}
```

```cpp
void CCamera::FinalTick()
{
	// ...
	m_matProj = XMMatrixOrthographicLH(m_Width * m_ProjectionScale, m_Height * m_ProjectionScale, 1.f, m_Far);
}
```

그리고 키보드 입력을 통해 Orthgraphic 모드에서의 투영 범위를 Zoom in/out 하듯이 조정할 수 있고, 이 수치를 UI에 표시함으로써 사용자는 자유자재로 조절할 수 있다

```cpp
void CEditorCameraScript::OrthoGraphicMove()
{
	// ...
	float Scale = Camera()->GetScale();
	if (KEY_PRESSED(KEY::_8))
		Scale += EngineDT;
		
	if (KEY_PRESSED(KEY::_9))
		Scale -= EngineDT;
		
	Camera()->SetScale(Scale);
}
```

</aside>

→ 각각의 수치를 확인하고 조절할 수 있는 Camera의 Projection Transform UI 완성

![Untitled](2024%2006%2025%20-%20Camera%20UI%20(2),%20MeshRender%20UI%200cf4c18d326d4adb9cca342c87377b98/Untitled%201.png)

**Inspector의 자식 UI로 추가**

```cpp
Inspector::Inspector()
{
	// ...
	// Camera UI
	m_arrComUI[(UINT)COMPONENT_TYPE::CAMERA] = new CameraUI;
	m_arrComUI[(UINT)COMPONENT_TYPE::CAMERA]->SetName("CameraUI");
	m_arrComUI[(UINT)COMPONENT_TYPE::CAMERA]->SetChildSize(ImVec2(0.f, 200.f));
	AddChild(m_arrComUI[(UINT)COMPONENT_TYPE::CAEMRA]);
}
```

**완성 화면**

Camera Component의 전체적인 구성이 완성되었다

![Untitled](2024%2006%2025%20-%20Camera%20UI%20(2),%20MeshRender%20UI%200cf4c18d326d4adb9cca342c87377b98/Untitled%202.png)

> 번외) UI 제작 순서 변경하기
> 

다음 UI의 제작에 들어가기 전에, 원래 Editor Manager에서 Inspector를 가장 먼저 생성했기 때문에, 매번 이렇게 Inspector의 자식 UI를 생성해 추가할 때마다 다른 UI들 (Outliner, Content … )들이 ID가 계속 밀리면서 새롭게 정의되어 윈도우의 모양이 변형되었었다

앞으로 Inspector에 추가될 UI들이 많이 남아있으니, 번거롭지 않게 Inspector를 가장 마지막에 생성되어 ID가 등록되게끔 작성해주자

```cpp
void CEditorMgr::CreateEditorUI()
{
	EditorUI * pUI = nullptr;
	
	// Content
	// ...
	
	// Outliner
	// ...
	
	// Insepctor (가장 마지막으로 순서 변경)
	pUI = new Inspector;
	pUI->SetName("Insepctor");
	m_mapUI.insert(make_pair(pUI->GetName(), pUI));
}
```

### MeshRender UI

**MeshRender UI를 통해서 표시해 줄 정보들**

- MeshRender가 어떤 Mesh를 참조하는지
- MeshRender가 어떤 Material을 참조하는지

```cpp
class MeshRenderUI :
	public ComponentUI
{
private:
	virtual void Update() override;

public:
	MeshRenderUI();
	~MeshRenderUI();
}
```

```cpp
MeshRenderUI::MeshRenderUI()
	: ComponentUI(COMPONENT_TYPE::MESHRENDER)
{
}

MeshRenderUI::~MeshRenderUI()
{
}

void MeshRenderUI::Update()
{
	Title();
	
	// Mesh 정보
	
	// Material 정보
}
```

**Mesh - Mesh Render가 어떤 Mesh를 참조하는지**

```cpp
void MeshRenderUI::Update()
{
	Title();
	
	CMeshRender* pMeshRender = GetTargetObject()->MeshRender();

	// Mesh 정보
	Ptr<CMesh> pMesh = pMeshRender->GetMesh();
	
	string MeshName = string(pMesh->GetKey().begin(), pMesh->GetKey().end());
	
	ImGui::Text("Mesh");
	ImGui::SameLine(100);
	ImGui::InputText("##MeshKey", MeshName.c_str(), ImGuiInputTextFlags_::ImGuiInputTextFlags_ReadOnly);
	
	// Material 정보
	// ...
}
```

**Material - Mesh Render가 어떤 Material를 참조하는지**

```cpp
void MeshRenderUI::Update()
{
	Title();
	
	CMeshRender* pMeshRender = GetTargetObject()->MeshRender();

	// Mesh 정보
	// ...
	// Material 정보
	Ptr<CMaterial> pMtrl = pMeshRender->GetMaterial();

	string MtrlName = string(pMtrl->GetKey().begin(), pMtrl->GetKey().end());
	ImGui::Text("Material);
	ImGui::SameLine(100);
	ImGui::InputText("##MaterialKey", (char*)MtrlName.c_str(), ImGuiInputTextFlags_::ImGuiInputTextFlags_ReadOnly);
}
```

 

**Inspector의 자식 UI로 추가**

```cpp
Inspector::Inspector()
{
	// ...
	// MeshRender UI
	m_arrComUI[(UINT)COMPONENT_TYPE::MESHRENDER] = new MeshRenderUI;
	m_arrComUI[(UINT)COMPONENT_TYPE::MESHRENDER]->SetName("MeshRenderUI");
	m_arrComUI[(UINT)COMPONENT_TYPE::MESHRENDER]->SetChildSize(ImVec2(0.f, 100.f));
	AddChild(m_arrComUI[(UINT)COMPONENT_TYPE::MESHRENDER]);
}
```

→ MeshRender를 Insepctor의 자식으로 추가해줌으로써, Inspector에서 우리가 구현한 UI가 추가되게끔 해주자

**완성 화면**

![Untitled](2024%2006%2025%20-%20Camera%20UI%20(2),%20MeshRender%20UI%200cf4c18d326d4adb9cca342c87377b98/Untitled%203.png)

다음은 과제로 직접 구현해 볼 UI들! 추후 업데이트 예정

### FlipBookCom UI

FlipBook 객체가 아닌, FlipBook Component에 대한 UI임을 유의하기

### TileMap UI

### ParticleSystem UI