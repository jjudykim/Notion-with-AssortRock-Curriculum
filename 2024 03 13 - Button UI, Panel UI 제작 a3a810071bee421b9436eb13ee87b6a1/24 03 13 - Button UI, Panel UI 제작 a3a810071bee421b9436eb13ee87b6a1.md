# 24/03/13 - Button UI, Panel UI 제작

상위 항목: Week16 (Week16%206fb66acd607f4a848a8ec7095f91c5df.md)
주차: 0010_Week10~19

### 그럼 UI의 m_MouseLbtnDown은, 언제 다시 false로 돌아가야 할까?

**m_MouseLBtnDown이 false로 변환되어야 하는 타이밍은?**

단순히 생각한다면, `void CUI::LButtonUp()` 에서 m_MouseLBtnDown 처리를 해주면 되는 것 아닌가? 라고 생각할 수 있다! 

→ 그러나 이 타이밍에 false 처리를 해버리면, Manger의 tick에서는, 해당 tick에서의 MouseOn의 상태에 따라 Event 호출이 달라지기 때문에, button이 down되고 up 되었던 사실을 정확하게 파악하기가 어렵다!

따라서, 우리는 왼마우스의 Up에 대한 Event을 Task로 처리할 것이다!

**Task로 처리하기 위한 TASK_TYPE에 UI_LBTN_DOWN 추가**

```cpp
enum class TASK_TYPE
{
	SPAWN_OBJECT,
	DELETE_OBJECT,
	CHANGE_LEVEL,
	// 추가
	UI_LBTN_DOWN,      // 1: UI Object Adress, 2: true or false (눌림 / 눌리지않음)
}
```

**해당 Task에 대한 case 작성**

```cpp
void CTaskMgr::ExecuteTask()
{
	static bool bLevelChanged = false;
	bLevelChanged = false;

	for (size_t i = 0; i < m_vecTask.size(); ++i)
	{
		switch(m_vecTask[i].Type)
		{
		//...
		case TASK_TYPE::UI_LBTN_DOWN:
		{
			CUI* pUI = (CUI*)m_vecTask[i].Param1;
			bool bLbtnDown = (bool)m_vecTask[i].Param2;
	
			pUI->m_MouseLbtnDown = bLbtnDown;
		}
	}
}	
```

그렇다면 tick에서 현재 마우스 상황이

- m_MouseLbtnDown이 true일 때 (전 프레임에서 마우스를 Down했을 때)
- Key가 떼어졌을 때

의 두가지 조건을 충족하면, 해당 Tick에서 마우스가 눌렸다 떼어진 상태이니까! 

UI_LBTN_DOWN에 대한 Task를 추가해줘서 m_MouseLbtnDown이 false로 바뀔 수 있도록 해준다

```cpp
void CUI::tick()
{
	CheckMouseOn();
	
	//
	if (KEY_RELEASED(KEY::LBTN) && m_MouseLbtnDown)
	{
		CTaskMgr::GetInst()->AddTask{tTask{TASK_TYPE::UI_LBTN_DOWN, 
																			(DWORD_PTR)this, false});
	}
}
```

이로써 상황에 맞게 Down / Up / Clicked Event 분기를 나누고, UI의 MouseLbtnDown 여부를 on/off 할 수 있게 되었다!

## UI를 상속받은, Button UI를 제작해보자

Button의 기능을 갖고있는 UI는 가장 많이 사용하는 UI 중 하나므로, CUI를 상속받아 아예 Button이라는 UI 타입을 만들자!

### Button 기능을 하는 UI를 클래스로 제작하자

**CButton의 기본적인 틀 작성**

```cpp
class CButton
	: pbulic CUI
{
public:
	// 이를 위해서는 CUI에 선언되어있는 LButtonClicked 함수를 virtual로 만들어야 한다.
	// 상속받은 Button에서 구현하고 싶은 LButton의 Click 상호작용이 있을 수 있으니!
	virtual void LButtonClicked() override;

public:
	virtual void render() override;

public:
	CLONE(CButton);
	CButton();
	~CButton();
}
```

```cpp
CButton()
{
}

~CButton()
{
}

virtual void LButtonClicked() override
{
	// Button에 마우스 왼클릭이 일어났을 때!
}
```

→ 그럼 Button이 눌렸을 때, 즉 `LButtonClicked()`가 호출되었을 때의 상호작용을 어떻게 해주는 것이 좋을까?

- Button은 다양하게 사용되어지는 상황이 많으니, 콜백 함수를 통해 해당 이벤트가 발생했을 때 해당 함수를 호출하는 것이 낫겠지!

**CALLBACK 생성 및 추가**

```cpp
typedef void(*BUTTON_CALLBACK)(void);

class CButton
	: public CUI
{
private:
	// Callback 함수를 담아놓을 멤버 함수
	BUTTON_CALLBACK      m_Func;	

public:
	void SetCallBack(BUTTON_CALLBACK _Func) { m_Func = _Func; }
}
```

callback으로 보내주기 위한 전역 함수를, CLevel_Editor에서 하나 작성해주자!

```cpp
// 멤버 함수가 아닌 전역함수!!!
void ButtonCallBackFunc()
{
	CLevel* pLevel = CLevelMgr::GetInst()->GetCurrentLevel();
	CLevel_Editor* pEditorLevel = dynamic_cast<CLevel_Editor*>(pLevel);
	if (nullptr == pEditorLevel)
		return;
		
	// 현재 레벨이 Editor Level일 때, 타일 저장 UI를 여는 함수 실행!
	pEditorLevel->OpenSaveTile();
}
```

**Button 객체의 생성과 CallBack 함수의 세팅**

그렇다면, Button의 LButtonClick 시에는 *본인의 멤버로 저장해놓은 callback 함수*를 호출하면 되는 것이고

```cpp
void CButton::LButtonClicked()
{
	if (m_Func != nullptr)
	{
		m_Func();
	}
}
```

Editor Level에서 Button UI 객체를 생성하면서, CallBack 함수를 세팅해주면 해당 함수를 클릭 시 실행할 수 있게 된다

```cpp
void ButtonCallBackFunc()                  // -> 요 함수를 콜백으로 세팅!
{
	// ...
}

void CLevel_Editor::Enter()
{
	// Button UI 생성!
	CButton* pUI = new CButton;
	
	pUI->SetCallBack(&ButtonCallBackFunc);
	// ...
}
```

### Button의 상태에 따라 이미지를 설정해보자

보통 Button의 이미지를 생각해보면, 3개 정도의 분기가 있는 것으로 볼 수 있다!

- 기본적인 Button의 이미지
- Button 위에 마우스가 올려져 있을 때의 이미지
- Button 위에 마우스가 Down 되었을 때의 이미지

따라서 우리의 Button UI에도 3가지 이미지를 가지도록 해주자!

```cpp
class CButton
	: public CUI
{
private:
	CTexture*  m_NormalImg;          // 기본적인 Button 이미지
	CTexture*  m_HoverImg;           // 마우스가 올려져 있을 때의 이미지
	CTexture*  m_DownImg;            // 버튼이 눌렸을 때의 이미지
	// ...	
	
public:
	void SetNormalImage(CTexture* _Tex) { m_NormalImg = _Tex; }
	void SetHoverImage(CTexture* _Tex) { m_HoverImg = _Tex; }
	void SetDownImage(CTexture* _Tex) { m_DownImg = _Tex; }
	
public:
	// ...
	virtual void render() override
}
```

```cpp
void CButton::render()
{
	// 일단 적당히 출력할만한 이미지가 없으니까 분기만 나눠주자... :3
	
	if (IsLbtnDowned())                     // 버튼이 눌려졌을 때
	{
		if(m_DownImg == nullptr)
			CUI::render();
	}
	else if (IsMouseOn())                   // 버튼에 마우스가 올라와 있을 때
	{
		if(m_HoverImg == nullptr)
			CUI::render();
	}
	else
	{
		if (m_NormalImg == nullptr)           // 기본 상태
			CUI::render();
	}
}
```

## 여러 UI를 소유하고 있는, Panel UI를 만들어보자

사실 UI는 굉장히 작게 쪼갤 수 있다!

우리가 한 UI라고 지칭하는 파일 선택 창의 경우에도.. 아이콘, 버튼, 텍스트, 모든 것이 UI이다

그렇다면 우리가 제작하는 UI도, 여러 UI가 그 안에 속할 수 있도록 창(Pannel)의 역할을 하는 UI를 만들고 안에 다른 UI들을 배치할 수 있을 것이다!

이런 식으로!

![Untitled](24%2003%2013%20-%20Button%20UI,%20Panel%20UI%20%E1%84%8C%E1%85%A6%E1%84%8C%E1%85%A1%E1%86%A8%20a3a810071bee421b9436eb13ee87b6a1/Untitled.png)

### UI의 <부모 - 자식> 계층 관계 형성

그럼 UI 내에서 부모 - 자식 개념이 생기는 것이므로…

CUI 단에서 부모 UI와 자식 UI를 가질 수 있도록 멤버 변수들을 정의해주자

이때, 부모 UI는 단 하나일 수밖에 없고, 자식 UI는 여러개일 수 있으니

부모 UI를 가리키는 `m_ParentUI`는 단 CUI* 타입으로,

자식 UI들을 가리키는 `m_ChildUI`는 CUI* 타입의 vector로 구성하자!

```cpp
class CUI
	: public CObj
{
private:
	CUI*           m_ParentUI;
	vector<CUI*>   m_vecChildUI;
	// ...
}
```

### 창 역할을 하는 UI, CPanel 을 작성해보자

**CPanel의 기본적인 틀 작성**

```cpp
class CPanel
	: public CUI
{
public:
	virtual void tick() override;
	virtual void LButtonDown() override;
	
public:
	CLONE(CPanel);
	CPanel();
	~CPanel();
}
```

```cpp
CPanel::CPanel()
{
}

CPanel::~CPanel()
{
}

void CPanel::tick()
{

}

void CPanel::LButtonDown()
{
}
```

일단 Level_Editor에서, 기본적인 구성만 되어있는 Panel UI 객체를 하나 만들어 추가해보자!

- Enter에서 UI를 만들어주는 부분이 너무 길어져서 → `CreateUI()`를 따로 구현해줬다

```cpp
class CLevel_Editor
	: public CLevel
{
// ...
private:
    void CreateUI();
// ...
}
```

```cpp
void CLevel_Editor::Enter()
{
	// ...
	CreateUI();
}

void CLevel_Editor::CreateUI()
{
	// ...
	// Panel UI 추가
	CPanel* pPanelUI = new CPanel;

	pPanelUI->SetScale(Vec2(400.f, 500.f))
	pPanelUI->SetPos(Vec2(vResol.x - (pPanelUI->GetSCale().x + 10), 10.f));
	// ...
}
```

### Panel을 드래그해 위치 옮기기

이렇게 제작한 CPanel을 드래그를 이용해 위치를 옮길 수 있도록 기능을 구현해주자

<aside>
🗒️ **CPanel의 Drag 기능을 위한 TODO list**

- 1️⃣ 마우스 L버튼이 눌렸을 때, L버튼이 눌린 시점에서의 마우스 위치를 기록
    
    → 이를 담기 위해 `m_vDownPos` 멤버 변수 추가
    
- 2️⃣ 매 tick마다 마우스 L버튼이 눌렸는지 체크
    
    → L버튼이 눌렸다면, 현재 마우스 위치와 기록했던 m_vDownPos를 계산해 이동할 위치를 계산
    
</aside>

그런데 이때 고려해야 할 점이 있다!!

우리는 지금 만든 Panel 안에 부모나 자식 UI를 속하지 않았는데..

- 만약 우리의 Panel에 부모 UI가 있다면 → 우리는 부모 UI의 위치(Pos)를 따라가야 하고
- 만약 우리의 Panel에 자식 UI가 있다면 → 내 자식 UI들은 내 위치(Pos)를 따라가야 한다

따라서, 먼저 CUI 단에서 부모 UI의 위치에 영향을 받는 최종 위치인 `m_FinalPos` 멤버 변수를 추가해주고, 이를 Get할 수 있는 함수도 추가해주자!

**UI의 최종 위치, m_FinalPos**

```cpp
class CUI
	: public CObj
{
private:
	// ...
	Vec2    m_vFinalPos;          // UI의 최종 위치값
}
```

```cpp
void CUI::tick()
{
	// ...
	// 매 tick에서 FinalPos를 계산해주자
	m_vFinalPos = GetPos();
	
	if (m_ParentUI)
		m_vFinalPos += m_ParentUI->GetFinalPos();
		
	// ...
}
```

→ 그리고 UI들의 render나, CheckMouseOn 같은 함수에서도 *기준 위치를 FinalPos로* 해줘야 할 것이다!

그럼 이제, Panel을 Drag해서 옮겨보자!

1️⃣ 마우스 L버튼이 눌렸을 때, L버튼이 눌린 시점에서의 마우스 위치를 기록 

```cpp
void CPanel::LButtonDown()
{
	// 오버라이딩하더라도 부모쪽에서 구현된 부분들을 실행하기 위해 부모쪽 함수를 호출
	CUI::LButtonDown();
	
	// 왼쪽 버튼이 눌린 시점에서의 마우스 위치값 기록
	m_vDownPos = CKeyMgr::GetInst()->GetMousePos();
}
```

2️⃣ 매 tick마다 마우스 L버튼이 눌렸는지 체크

```cpp
void CPanel::tick()
{
	
	CUI::tick();
	
	if (IsLbtnDowned())
	{
		Vec2 vCurMousePos = CKeyMgr::GetInst()->GetMousePos();
		Vec2 vDiff = vCurMousePos - m_vDownPos;                 // Panel이 이동해야 할 거리
		
		Vec2 vNextPos = GetFinalPos() + vDiff;
		SetPos(vNextPos);
		
		m_vDownPos = vCurMousePos;          // 다음 tick에서 이동할 거리를 계산하기 위해 갱신
	}
}
```

### Panel에 자식 UI를 생성해보자

Editor Level에서 기존에 만들었던 Button UI를 Panel UI의 자식으로 넣어보자

1. 어느 UI든지 자신의 자식 UI를 만들 수 있어야 하므로, CUI 단에서 `AddChildUI` 함수 추가

```cpp
class CUI
	: public CObj
{
// ...
public:
	void AddChildUI(CUI* _UI)
	{
		m_vecChileUI.push_back(_UI);
		_UI->m_ParentUI = this;
	}
}
```

1. Level_Edtior에서 Panel과 Button UI를 생성하고, Button UI를 panel의 자식으로 Add해준다

```cpp
void CLevel_Editor::CreateUI()
{
	// ...
	// Panel UI 추가
	CPanel* pPanelUI = new CPanel;
	// Panel UI 세팅
	// ...
	
	// Button UI 추가
	CButton* pUI = new CButton;
	// Button UI 세팅
	// ...
	
	pPanelUI->AddChildUI(pUI);
	AddObject(LAYER_TYPE::UI, pPanelUI);
}
```

1. 자식 UI가 생김으로써 해야하는 일들!

- UI의 소멸자에서 본인의 자식 UI들 삭제
    
    → 생성된 자식UI들은 직접적으로 AddObject 되지 않아 메모리 해제를 따로 해줘야 한다!
    

```cpp
CUI::~CUI()
{
	Safe_Del_Vec(m_vecChildUI);
}
```

- 자식 UI들의 tick 및 render 호출

```cpp
void CUI::tick()
{
	// ...	
	for (size_t i = 0; i < m_vecChileUI.size(), ++i)
	{
		m_vecChildUI[i]->tick();
	}
}

void CUI::render()
{
	// ...
	for (size_t i = 0; i < m_vecChileUI.size(), ++i)
	{
		m_vecChildUI[i]->render();
	}
}
```

**기본적인 UI에서 이루어져야 하는 tick과, 이를 상속받는 자식 UI에서 이루어져야 하는 tick의 구분**

ex)

- 기본적인 UI에서 공통적으로 이루어져야 하는 tick 내의 작업들
    - FinalPos의 계산
    - Mouse on 체크
    - 마우스 L버튼을 눌렀다 뗐는지 체크 → Task 추가
    - 본인이 소유한 자식 UI들의 tick 호출
    
- 상속받은 자식 UI에서 이루어져야 하는 tick 내의 작업들
    - 드래그 기능

이때 부모의 tick → 자식의 tick 순서대로 호출이 진행되어야 하는데…

만약 자식 UI에서 tick을 override 해버리면, 

- 공통적으로 이루어져야 하는 UI단의 tick의 작업은 수행되지 못해 직접 구현해줘야 하고
- 다른 방법으로는 `CUI::tick()`을 호출할 수 있지만 구현하게 될 자식 UI마다 호출을 해줘야 함

CUI 단에서의 `tick`과 `render`를 더 이상 override할 수 없도록 final로 선언한 후에,

`tick_ui`, `render_ui`라는 순수 가상함수를 만들어 해당 객체만의 (Panel, Button 등…) tick 또는 render 작업을 수행할 수 있도록 작성하자!

정리하자면,  

UI 공통으로 이루어져야 하는 작업은 기존의 tick, render에서

UI를 상속받은 객체가 해야하는 자신만의 작업은 tick_ui, render_ui에서 진행함으로써,

 UI단에서는 tick에서 tick_ui를 호출, render에서 render_ui를 호출해 공통/개별로 진행되는 작업을 구분해줄 수 있다

```cpp
class CUI
	: public CObj
{
private:
	// ...

public:
	// ...
	virtual void tick() final;
	virtual void render() final;
	
	virtual void tick_ui = 0;
	virtual void render_ui = 0;
	// ...
}
```

```cpp
void CUI::tick()
{
	// ...
	// UI 단에서 공통적으로 이루어져야 하는 작업들
	// ...
	
	tick_ui();

	// 자식 vector의 tick() 호출
	// ...
}

void CUI::render()
{
	CObj::render();
	
	render_ui();
	
	// 자식 vector의 render() 호출
	// ...
}
```

각각의 `tick_ui()`, `render_ui()`에서는 자식 객체가 수행해야 할 작업 구현

```cpp
void CPanel::tick_ui()
{
	if (IsLbtnDowned())
	{
		// ...
	}
}

void CPanel::render_ui()
{
	// Panel ui만의 render
}
```

```cpp
void CButton::tick_ui()
{
	// Button ui 만의 render
	if (IsLbtnDowned())
	{
		// ...
	}
	// ...
}
```