# 24/03/12 - UI를 활용한 Tile Load/Save, UI class 작성

태그: C++, WinAPI, 중급
날짜: 2024/03/12
상위 항목: Week16 (Week16%206fb66acd607f4a848a8ec7095f91c5df.md)
주차: 0010_Week10~19

## UI 를 제작해보자

### UI를 활용해서 타일 저장 / 불러오기

윈도우에서 기본적으로 제공되는 이런 창들!!

![Untitled](24%2003%2012%20-%20UI%E1%84%85%E1%85%B3%E1%86%AF%20%E1%84%92%E1%85%AA%E1%86%AF%E1%84%8B%E1%85%AD%E1%86%BC%E1%84%92%E1%85%A1%E1%86%AB%20Tile%20Load%20Save,%20UI%20clas%20f09c9e1aaeea4e2ab1f696d4000eff5b/Untitled.png)

이것도 윈도우에서 제공되는 기본적인 기능으로 구현할 수 있는 UI이지만, 우리가 직접 구현하기엔 어려움이 있으니 **예제 코드**를 활용해서 우리의 코드에 적용시켜 보자

- 타일 메뉴에서 하위항목 생성 → 타일 저장하기(`ID_TILESAVE`) / 타일 불러오기(`ID_TILELOAD`)
- 타일 정보를 저장하고 / 불러오는 기능은 Editor Level에 작성, 이를 불러와서 main의 프로시저 함수에서 호출

```cpp
LRESULT CALLBACK WndProc(HWND hWnd, UINT message, WPARAM wParam, LPARAM lParam)
{
	switch(message)
	{
	case WM_COMMAND:
		{
			int wmID = LOWORD(wParam);
			switch (wmId)
			{
				// ...
			case ID_TILESAVE:
			{
				CLevel_Editor* pLevel = dynamic_cast<CLevel_Editor*>(CLevelMgr::GetInst()->GetCurrentLevel());
				if (pLevel)
				{
					pLevel-> OpenSaveTile();
				}
			}
				break;
			
			case ID_TILELOAD:
			{
				CLevel_Editor* pLevel = dynamic_cast<CLevel_Editor*>(CLevelMgr::GetInst()->GetCurrentLevel());
				if (pLevel)
				{
					pLevel->OpenLoadTile();
				}
			}
				break;
			}
			// ...
		}
	}
}

```

CLevel_Editor에서는 해당 메뉴에 대한 `OpenSaveTile()`, `OpenLoadTile()` 함수를 구현해준다!

```cpp
void CLevel_Editor::OpenSaveTile()
{
	wchar_t szSelect[256] = {};                    // 최종 선택한 경로 / 파일이 담길 버퍼
	
	OPENFILENAME ofn = {};
	ofn.lStructSize = sizeof(ofn);
	ofn.hwndOwner = nullptr;
	ofn.lpstrFile = szSelect;
	ofn.lpstrFile[0] = '\0';
	ofn.nMaxFile = sizeof(szSelect);                                   // 파일 이름 
	ofn.lpstrFilter = L"Tile\0*.tile";                                 // 저장할 파일 형식
	ofn.nFilterIndex = 0;
	ofn.lpstrFileTitle = NULL;
	ofn.nMaxFileTitle = 0;

	// 탐색창 초기 위치 지정
	wstring strInitPath = CPathMgr::GetInst()->GetContehtPath();
	strInitPath += L"tile\\";
	ofn.lpstrInitialDir = strInitPath.c_str();

	ofn.Flags = OFN_PATHMUSTEXIST | OFN_FILEMUSTEXIST;

	// 여기까지는 Dialog를 통해서 파일을 탐색하고 최종 경로를 가져오는 역할만 하는 것이니,
	// 우리가 직접 해당 파일을 통해 저장하는 작업은 작성해줘야 한다.
	if (GetSaveFileName(&ofn))
	{
		m_EditTile->SaveToFile(szSelect);
	}	
}
```

```cpp
void CLevel_Editor::OpenLoadTile()
{
	whcar_t szSelect[256] = {};                // 최종 선택한 경로 / 파일이 저장될 버퍼

	OPENFILENAME ofn = {};
	
	ofn.lStructSize = sizeof(ofn);
	ofn.hwndOwner = nullptr;
	ofn.lpstrFile = szSelect;
	ofn.lpstrFile[0] = '\0';
	ofn.nMaxFile = sizeof(szSelect);                              // 파일 이름: 
	ofn.lpstrFilter = L"Tile\0*.tile";                            // 불러올 파일 형식
	ofn.FilterIndex = 0;
	ofn.lpstrFileTitle = NULL;
	ofn.nMaxFileTitle = 0;
	
	wstring strInitPath = CPathMgr::GetInst()->GetContentPath();
	strInitPath += L"tile\\";
	ofn.Flags = 
	
	// 여기까지는 Dialog를 통해서 파일을 탐색하고 최종 경로를 가져오는 역할만 하는 것이니,
	// 우리가 직접 해당 파일을 통해서 불러오는 작업은 작성해줘야 한다.
	if (GetOpenFileName(&ofn))
	{
		m_EditTile->LoadFromFile(szSelect);
	}
}
```

## 게임 상에서 존재하는 UI 제작하기

게임 내에서 존재하는 UI를 제작해보자! 스탯 창, 아이템 창 등등.. 

그런데 게임 내에서 사용하는 UI에 기본적인  윈도우 창을 띄우기에는 너무 못생기고.. 별로다..

따라서, 자체적으로 UI 오브젝트를 만들어서 사용하자!

### UI 클래스 작성하기

나만의 UI를 생성하기 위한 UI class를 작성해보자

**UI 클래스 생성**

<aside>
📁 03. Game > 05. Obejct > UI > `CUI.h`, `CUI.cpp` 생성!

</aside>

**CUI의 기본적인 틀 작성**

```cpp
class CUI
	: public CObj
{
private:

public:
	virtual void tick() override;
	virtual void render() override;
	
public:
	CLONE(CUI);
	CUI();
	~CUI();
}
```

```cpp
CUI::CUI()
{
}

CUI::~CUI()
{
}

void CUI::tick()
{
}

// 카메라 영향을 받지 않도록 직접적으로 render 구현 
// -> 카메라가 움직이더라도 위에 고정되어 있도록 한다!
void CUI::render()
{
	// CObj의 render를 호출해 부모 단의 render 작업이 선행되도록
	CObj::render();
	
	Vec2 vPos = GetPos();
	Vec2 vScale = GetScale();
	
	Rectangle(DC, (int)(vPos.x), (int)(vPos.y), 
								(int)(vPos.x + vScale.x), (int)(vPos.y + vScale.y));
}
```

**Editor Level에서, 기본적인 UI 객체를 하나 만들어 띄워보자!**

```cpp
void CLevel_Editor::Enter()
{
	// 화면 해상도 불러오기
	Vec2 vResol = CEngine::GetInst()->GetResolution();
	// ...
	
	// UI 추가
	CUI* pUI = new CUI;
	pUI->SetScale(Vec2(200.f, 100.f));
	
	// Pos를 우상단에서 x, y에 10만큼 Padding을 가지도록 설정
	pUI->SetPos(Vec2(vResol.x - (pUI->GetScale().x + 10), 10.f ));
	AddObject(LAYER_TYPE::UI, pUI)
}
```

### Click을 통해 상호작용을 할 수 있는 UI를 제작하려면?

Button과 같은 UI들의 기본적인 요구사항은, 마우스 Click을 통해 상호작용을 할 수 있다는 것이다!

그렇다면 그런 UI에는 

- 본인의 위에 Mouse가 올라와 있는지
- 그 상태에서 마우스의 Down, 또는 Up이 이루어졌는지

에 대한 확인이 이루어져야 할 것이다

이를 위해서

1. 현재 프레임, 이전 프레임에 Mouse의 위치가 UI 위에 올라와 있었는지 담는 멤버 변수 추가

→ `bool m_MouseOn`, `bool m_MouseOnPrev`

```cpp
class CUI
	: public CObj
{
private:
	bool      m_MouseOn;           // 현재 프레임에서 UI 위에 마우스가 위치하는지 bool
	bool      m_MouseOnPrev;       // 이전 프레임에서 UI 위에 마우스가 위치했는지 bool

public:
	// ...
}
```

1. 매 tick에서 이에 대한 상태를 업데이트하는 Check 함수를 호출할 수 있도록 멤버 함수 추가

→ `void CheckMouseOn();`

```cpp
class CUI
	: public Object
{
// ...
private:
	void CheckMouseOn();
// ...
}
```

```cpp
void CUI::tick()
{
	CheckMouseOn();                          // 매 tick에서 Mouse On 검사 진행!
	// ...
}

void CUI::CheckMouseOn()
{
	Vec2 vPos = GetPos();
	Vec2 vScale = GetScale();
	Vec2 vMousePos = CKeyMgr::GetInst()->GetMousePos();
	
	m_MouseOnPrev = m_MouseOn;
	
	// 마우스가 UI의 범위 내에 포함하는 위치일 경우
	if (vPos.x <= vMousePos.x &&
			vMousePos.x <= vPos.x + vScale.x &&
			vPos.y <= vMousePos.y &&
			vMousePos.y <= vPos.y + vScale.y)
	{
		// Mouse On
		m_MouseOn = true;
	}
	else
	{
		m_MouseOn = flase;
	}
}
```

### UI에 Click을 통한 작동을 부여하자!

 

Button이라는 UI를 만들어서, 해당 UI를 클릭하면 Tile의 정보 Dialog가 뜨도록 연결해보자!

그렇다면 ***UI에게 발생한 Event인 Click***이 이루어졌는지 감시하고 처리해줄 Manager가 필요하다

→ UI에게 발생한 Event를 감시하고 처리해줄 Manager 하나를 생성하자

**UI Manager 생성**

<aside>
📁 03. Manager > 10. UIMgr > `CUIMgr.h`, `CUIMgr.cpp` 생성!

</aside>

**UI Manager의 기본적인 틀 작성**

```cpp
class CUIMgr
{
	SINGLE(CUIMgr);
private:
	
public:
	void tick();
}
```

```cpp
CUIMgr::CUIMgr()
{
}

CUIMgr::~CUIMgr()
{
}

void CUIMgr::tick()
{
	// 현재 Level에 접근해, 해당 Level에 존재하는 모든 UI들을 가져온다
	CLevel* pCurLevel = CLevelMgr::GetInst()->GetCurrentLevel();
	const vector<CObj*>& vecUI = pCurLevel->GetObjects(LAYER_TYPE::UI);
	
	for (size_t i = 0; i < vecUI.size(); ++i)
	{
		// 해당 vector들에 대한 감시 및 처리를 시행!
		// ...
	}
}
```

Click과 관련된 Event 처리를 위해서 멤버 변수, 함수를 추가해주자!

- `bool m_MouseLbtnDown` : 현재 왼마우스가 눌려있는지에 대한 bool
- `void LButtonDown()` : 왼마우스가 눌렸을 때의 Event
- `void LButtonUp()` : 왼마우스가 떨어졌을 때의 Event
- `void LButtonClicked()` : 왼마우스가 클릭되었을 때의 Event

→ ***Down과 Click의 차이?***

Down Event : Mouse Pos가 해당 UI의 위치이고, Mouse가 `Tap`된 상태일 경우

Click Event : Mouse Pos가 해당 UI의 위치 안인 상황에서, `Down`하고 `Up`이 모두 된 경우

```cpp
class CUI
	: public CObj
{
private:
	//...
	bool m_MouseLbtnDown;

public:
	virtual void LButtonDown();         
	virtual void LButtonUP();
	virtual void LButtonClicked();
}

```

```cpp
void LButtonDown()
{
	m_MouseLbtnDown = true;
}

virtual void LButtonUP()
{
}

virtual void LButtonClicked()
{
}
```

그렇다면, 마우스의 상태에 대해 감시하고 이런 왼마우스에 대한 Event가 일어날 경우 처리하는 것을 담당하는 것이 바로 Mouse Manager가 되는 것이다!

따라서 UIMgr의 tick에서는 마우스 상태를 먼저 확인한 후 그에 맞는 Event를 처리해야 한다

```cpp
void CUIMgr::tick()
{
	// 마우스 상태 확인
	bool LBtnTap = KEY_TAP(KEY::LBTN);
	bool LBtnReleased = KEY_RELEASED(KEY::LBTN);
	
	// 현재 레벨에 있는 UI들의 이벤트를 처리
	// ...
	
	for (size_t i = 0; i < vecUI.size(); ++i)
	{
		CUI* pUI = (CUI*)vecUI[i];
		
		// 왼쪽 버튼이 눌렸고, 그게 해당 UI 위치 안에서 라면
		if (LBtnTap && pUI->IsMouseOn())
		{
			pUI->LButtonDown();           // 왼쪽 버튼 Down 호출
		}
		// 왼쪽 버튼이 떨어졌고, 그게 해당 UI 위치 안에서 라면
		else if (LBtnReleased && pUI->IsMouseOn())
		{
			pUI->LButtonUp();             // 왼쪽 버튼 Up 호출
		
			// 그런데 전에 해당 UI 위치 안에서 버튼이 눌렸었다면
			if (pUI->IsLbtnDowned())
			{
				PUI->LButtonClicked();     // 왼쪽 버튼 Click 호출
			}
		}
	}
}
```