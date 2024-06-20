# 24/05/03 - 3D 세계와 Vertex, Rendering Pipeline

상위 항목: Week23 (Week23%20394e612b264041cd8461f961c700ba03.md)
주차: 0011_Week20~29

### DirectX로 물체를 그려보자

DirectX 함수를 사용해 물체를 그려가는 과정을 한번 임시적으로! 구현해보자

Process를 통해 물체를 그리는 작업을 수행하기 위해선 4단계가 필요하다 

```cpp
void TempInit();         // 할당 및 생성
void TempTick();         // Tick
void TempRender();       // Render (그리기)
void TempRelease();      // 해제
```

```cpp
void TempInit()
{
}

void TempTick()
{
}

void TempRender()
{
}

void TempRelease()
{
}
```

그럼, CEngine에 Temp를 적용하고 이를 그리는 과정은 다음과 같을 것이다!

```cpp
CEngine::CEngine()
	: m_hWnd(nullptr)
	, m_ptResolution{}
{
	// ...
}

CEngine::~CEngine()
{
	TempRelease();                       // CEngine의 소멸자에서 Temp 해제
}

void CEngine::Init(HWND _wnd, POINT _ptResolution)
{
	// ...
	TempInit();                         // CEngine의 Init에서, Temp 역시 Init을 수행
	
	return S_OK;
}

void CEngine::Progress()
{
	TempTick();                         // 매 tick이 수행될 때마다 Temp의 연산 수행
	
	CDevice::GetInst()->Clear();        // Device의 화면 초기화
	
	TempRender();                       // Temp를 통해 물체가 그려지는 과정
	
	CDevice::GetInst()->Present();      // 그려진 물체를 SwapChain을 통해 화면에 표시
}
```

**필요한 부가적인 작업들**

원활한 진행을 위해서 WinAPI 때 사용했던 TimeManager, KeyManager, PathManager, TaskManager를 가져와서 적용해주자 

- 가져온 내용
    - TimeManager
        
        ```cpp
        #pragma once
        
        class CTimeMgr
        	: public CSingleton<CTimeMgr>
        {
        	SINGLE(CTimeMgr)
        private:
        	// LARGE_INTEGER - 8 바이트 정수 대용
        	LARGE_INTEGER	m_llCurCount;
        	LARGE_INTEGER	m_llPrevCount;
        	LARGE_INTEGER	m_llFrequency;
        
        	UINT			m_FPS;
        
        	float			m_DeltaTime;	// 프레임 간격 시간( 1 프레임 실행하는데 걸리는 시간 )
        	float			m_Time;			  // 프로그램이 켜진 이후로 진행된 시간
        
        public:
        	void init();
        	void tick();
        
        public:
        	float GetDeltaTime() { return m_DeltaTime; }
        	float GetTime() { return m_Time; }
        	UINT GetFPS() { return m_FPS; }
        };
        
        ```
        
        ```cpp
        #include "pch.h"
        #include "CTimeMgr.h"
        
        #include "CEngine.h"
        
        CTimeMgr::CTimeMgr()
        	: m_llCurCount{}
        	, m_llPrevCount{}
        	, m_llFrequency{}
        	, m_FPS(0)
        	, m_DeltaTime(0.f)
        	, m_Time(0.f)
        {
        
        }
        
        CTimeMgr::~CTimeMgr()
        {
        
        }
        
        void CTimeMgr::init()
        {
        	// 초당 1000 을 카운팅하는 GetTickCount 함수는 미세한 시간을 측정하기에는 정확도가 떨어진다.
        
        	// 1초에 셀수있는 카운트 기준을 얻는다.
        	QueryPerformanceFrequency(&m_llFrequency);
        
        	QueryPerformanceCounter(&m_llCurCount);
        	m_llPrevCount = m_llCurCount;	
        }
        
        void CTimeMgr::tick()
        {
        	// 현재 카운트 계산
        	QueryPerformanceCounter(&m_llCurCount);
        
        	// 이전 카운트와 현재 카운트의 차이값을 통해서 1프레임 간의 시간값을 계산
        	m_DeltaTime = (float)(m_llCurCount.QuadPart - m_llPrevCount.QuadPart) / (float)m_llFrequency.QuadPart;
        
        	// DT 보정
        	if (1.f / 60.f < m_DeltaTime)
        		m_DeltaTime = 1.f / 60.f;
        
        	// 누적시간을 통해서 프로그램이 실행된 이후로 지나간 시간값을 기록
        	m_Time += m_DeltaTime;
        
        	// 현재 카운트 값을 이전 카운트로 복사해둠
        	m_llPrevCount = m_llCurCount;
        
        	// 초당 실행 횟수(FPS) 계산
        	++m_FPS;
        
        	// 1초에 한번씩 TextOut 출력
        	static float AccTime = 0.f;
        	AccTime += m_DeltaTime;
        	
        	if (1.f < AccTime)
        	{
        		wchar_t szBuff[255] = {};
        		swprintf_s(szBuff, L"DeltaTime : %f, FPS : %d ", m_DeltaTime, m_FPS);
        		//TextOut(CEngine::GetInst()->GetMainDC(), 10, 10, szBuff, wcslen(szBuff));
        		SetWindowText(CEngine::GetInst()->GetMainWnd(), szBuff);
        		AccTime = 0.f;
        		m_FPS = 0;
        	}
        }
        ```
        
    - KeyManager
        
        Mouse Pos 부분 변경
        
        ```cpp
        #pragma once
        
        enum class KEY
        {
        	_0,	_1, _2, _3, _4, _5, _6, _7, _8, _9,
        	Q, W, E, R, T, Y, U, I, O, P, 
        	A, S, D, F, G, H, J, K, L,
        	Z, X, C, V, B, N, M,
        
        	LEFT, RIGHT, UP, DOWN,
        
        	ENTER,
        	SPACE,
        	ALT,
        	CTRL,
        	SHIFT,
        	ESC,
        
        	LBTN,
        	RBTN,
        	
        	KEY_END,
        };
        
        enum class KEY_STATE
        {
        	TAP,
        	PRESSED,
        	RELEASED,
        	NONE,
        };
        
        struct tKeyInfo
        {
        	KEY			eKey;		       // 키의 종류
        	KEY_STATE	eKeyState;	 // 키의 상태값
        	bool		bPressed;	     // 지금 눌려있는지
        };
        
        class CKeyMgr
        	: public CSingleton<CKeyMgr>
        {
        	SINGLE(CKeyMgr);
        private:
        	vector<tKeyInfo>	m_vecKeyInfo;
        
        	Vec2				m_MousePos;
        	Vec2				m_PrevMousePos;
        	Vec2				m_DragDir;
        
        public:
        	void init();
        	void tick();
        
        public:
        	KEY_STATE GetKeyState(KEY _Key) { return m_vecKeyInfo[(UINT)_Key].eKeyState; }
        	Vec2 GetMousePos() { return m_MousePos; }
        	Vec2 GetDragDir() { return m_DragDir; }
        };
        ```
        
        ```cpp
        #include "pch.h"
        #include "CKeyMgr.h"
        
        #include "CEngine.h"
        
        UINT g_RealKey[(UINT)KEY::KEY_END] =
        {
        	'0', '1', '2', '3', '4', '5', '6', '7', '8', '9',
        	'Q', 'W', 'E', 'R', 'T', 'Y', 'U', 'I', 'O', 'P',
        	'A', 'S', 'D', 'F', 'G', 'H', 'J', 'K', 'L',
        	'Z', 'X', 'C', 'V', 'B', 'N', 'M',
        
        	VK_LEFT, VK_RIGHT, VK_UP, VK_DOWN,
        
        	VK_RETURN,
        	VK_SPACE,
        	VK_MENU,
        	VK_CONTROL,
        	VK_LSHIFT,
        	VK_ESCAPE,
        
        	VK_LBUTTON,
        	VK_RBUTTON,
        };
        
        CKeyMgr::CKeyMgr()
        {
        
        }
        
        CKeyMgr::~CKeyMgr()
        {
        
        }
        
        void CKeyMgr::init()
        {
        	for (int i = 0; i < (int)KEY::KEY_END; ++i)
        	{
        		tKeyInfo info = {};
        		info.eKey = (KEY)i;
        		info.eKeyState = KEY_STATE::NONE;
        		info.bPressed = false;
        
        		m_vecKeyInfo.push_back(info);
        	}
        }
        
        void CKeyMgr::tick()
        {
        	// MainWindow 가 포커싱 상태이다.
        	if (CEngine::GetInst()->GetMainWnd() == GetFocus())
        	{
        		for (UINT i = 0; i < (UINT)m_vecKeyInfo.size(); ++i)
        		{
        			// 지금 키가 눌려있는지 체크
        			if (0x8001 & GetAsyncKeyState(g_RealKey[(UINT)m_vecKeyInfo[i].eKey]))
        			{
        				// 이전에도 눌려있었는지 
        				if (m_vecKeyInfo[i].bPressed)
        				{
        					m_vecKeyInfo[i].eKeyState = KEY_STATE::PRESSED;
        				}
        				// 이전에는 눌려있지 않았고, 지금은 눌려있다.
        				else
        				{
        					m_vecKeyInfo[i].eKeyState = KEY_STATE::TAP;
        				}
        
        				m_vecKeyInfo[i].bPressed = true;
        			}
        
        			// 키가 안눌려있다.
        			else
        			{
        				// 이전에는 눌려있었다.
        				if (m_vecKeyInfo[i].bPressed)
        				{
        					m_vecKeyInfo[i].eKeyState = KEY_STATE::RELEASED;
        				}
        
        				// 이전에도 안눌려있고, 지금도 안눌려있다.
        				else
        				{
        					m_vecKeyInfo[i].eKeyState = KEY_STATE::NONE;
        				}
        
        				m_vecKeyInfo[i].bPressed = false;
        			}
        		}
        
        		// 마우스 좌표 계산
        		m_PrevMousePos = m_MousePos;
        
        		POINT ptMousePos = { };
        		GetCursorPos(&ptMousePos);
        		ScreenToClient(CEngine::GetInst()->GetMainWnd(), &ptMousePos);
        		m_MousePos = Vec2((float)ptMousePos.x, (float)ptMousePos.y);
        		m_DragDir = m_MousePos - m_PrevMousePos;
        	}
        	
        	// 윈도우의 포커싱이 해제됨
        	else
        	{
        		for (UINT i = 0; i < (UINT)m_vecKeyInfo.size(); ++i)
        		{
        			if (m_vecKeyInfo[i].eKeyState == KEY_STATE::TAP
        				|| m_vecKeyInfo[i].eKeyState == KEY_STATE::PRESSED)
        			{
        				m_vecKeyInfo[i].eKeyState = KEY_STATE::RELEASED;
        			}
        			else
        			{
        				m_vecKeyInfo[i].eKeyState = KEY_STATE::NONE;
        			}
        
        			m_vecKeyInfo[i].bPressed = false;
        		}
        	}	
        }
        
        ```
        
    - PathManager
        
        ```cpp
        #pragma once
        
        class CPathMgr
        	: public CSingleton<CPathMgr>
        {
        	SINGLE(CPathMgr);
        private:
        	wstring		m_Content;
        	wstring		m_Solution;
        
        public:
        	void init();
        	void render();
        
        private:
        	void GetParentPath(_Inout_ wchar_t* _Buffer);
        
        public:
        	const wstring& GetContehtPath() { return m_Content; }
        	const wstring& GetSolutionPath() { return m_Solution; }	
        };
        ```
        
        ```cpp
        #include "pch.h"
        #include "CPathMgr.h"
        
        #include "CEngine.h"
        
        CPathMgr::CPathMgr()	
        {
        }
        
        CPathMgr::~CPathMgr()
        {
        }
        
        void CPathMgr::init()
        {
        	// 실행경로를 얻어낸다
        	wchar_t szBuffer[256] = {};
        	GetCurrentDirectory(256, szBuffer);
        
        	// bin 폴더의 상위폴더로 접근한다.
        	GetParentPath(szBuffer);
        
        	// \\Content\\ 를 붙여둔다
        	m_Content = szBuffer;
        	m_Content += L"\\content\\";	
        }
        
        void CPathMgr::render()
        {	
        }
        
        void CPathMgr::GetParentPath(_Inout_ wchar_t* _Buffer)
        {
        	size_t len = wcslen(_Buffer);
        	
        	for (size_t i = len - 1; 0 < i; --i)
        	{
        		if (L'\\' == _Buffer[i])
        		{
        			_Buffer[i] = L'\0';
        			break;
        		}
        	}
        }
        
        ```
        
    - TaskManager
        
        ```cpp
        #pragma once
        
        class CTaskMgr
        	: public CSingleton<CTaskMgr>
        {
        	SINGLE(CTaskMgr)
        private:
        	vector<tTask>	m_vecTask;
        	vector<CObj*>	m_GC; // Garbage Collector;
        
        public:
        	void tick();
        	void AddTask(const tTask& _Task) { m_vecTask.push_back(_Task); }
        
        private:
        	void ClearGC();
        	void ExecuteTask();	
        };
        ```
        
        ```cpp
        #include "pch.h"
        #include "CTaskMgr.h"
        
        #include "CLevelMgr.h"
        #include "CLevel.h"
        #include "CObj.h"
        #include "CUI.h"
        
        CTaskMgr::CTaskMgr()
        {}
        
        CTaskMgr::~CTaskMgr()
        {}
        
        void CTaskMgr::tick()
        {
        	ClearGC();
        
        	ExecuteTask();
        }
        
        void CTaskMgr::ClearGC()
        {
        	Safe_Del_Vec(m_GC);
        
        	m_GC.clear();
        }
        
        void CTaskMgr::ExecuteTask()
        {
        	static bool bLevelChanged = false;
        	bLevelChanged = false;
        
        	for (size_t i = 0; i < m_vecTask.size(); ++i)
        	{
        		switch (m_vecTask[i].Type)
        		{
        		case TASK_TYPE::SPAWN_OBJECT:
        		{
        			CLevel* pSpawnLevel = (CLevel*)m_vecTask[i].Param1;
        			LAYER_TYPE Layer = (LAYER_TYPE)m_vecTask[i].Param2;
        			CObj* pObj = (CObj*)m_vecTask[i].Param3;
        
        			if (CLevelMgr::GetInst()->GetCurrentLevel() != pSpawnLevel)
        			{
        				delete pObj;
        			}
        			pSpawnLevel->AddObject(Layer, pObj);
        			pObj->begin();
        		}
        		break;
        		case TASK_TYPE::DELETE_OBJECT:
        		{
        			CObj* pObject = (CObj*)m_vecTask[i].Param1;
        			if (pObject->m_bDead)
        			{
        				continue;
        			}
        			pObject->m_bDead = true;
        
        			// GC 에서 수거
        			m_GC.push_back(pObject);
        		}
        		break;
        
        		case TASK_TYPE::CHANGE_LEVEL:
        		{
        			assert(!bLevelChanged);
        			bLevelChanged = true;
        			LEVEL_TYPE NextType = (LEVEL_TYPE)m_vecTask[i].Param1;
        			CLevelMgr::GetInst()->ChangeLevel(NextType);
        		}
        			break;
        
        		case TASK_TYPE::UI_LBTN_DOWN:
        		{
        			CUI* pUI = (CUI*)m_vecTask[i].Param1;
        			bool bLbtnDown = (bool)m_vecTask[i].Param2;
        			pUI->m_MouseLbtnDown = bLbtnDown;
        		}
        			break;
        		}
        	}
        
        	m_vecTask.clear();	
        }
        ```
        

- 필요한 라이브러리들 포함
    
    ```cpp
    // ...
    #include <string>
    using std::string;
    using std::wstring;
    
    #include <vector>
    using std::vector;
    
    #include <list>
    using std::list;
    
    #include <map>
    using std::map;
    using std::make_pair;
    
    // ...
    #include "define.h"
    #include "struct.h"
    ```
    
- 엔진에 윈도우 핸들 받아오는 함수 GetMainWnd 추가
    
    ```cpp
    class CEngine
    	: public CSingleton<CEngine>
    {
    	// ...
    public:
    	HWND GetMainWnd() { return m_hWnd; }
    	
    public:
    	// ...
    }
    ```
    
- GetDevice, GetContext 매크로 생성 → 	이 둘은 워낙 자주 사용하니까, 매크로로 작성해놓자
    
    ```cpp
    #define DEVICE	CDevice::GetInst()->GetDevice();
    #define CONTEXT	CDevice::GetInst()->GetContext();
    ```
    

### 3D 세계와 Vertex

DirectX에서 **3D**는 세 개의 차원을 나타내는 말로, 그래픽에서는 X, Y, Z 축을 기반으로 한 3차원 공간으로 표현한다!

Vertex는 점 또는 꼭지점을 나타내는 말로, 선이나 면의 시작점 / 끝점을 나타내는데 사용된다

3D 그래픽에서는 3D 모델을 구성하는 다수의 Vertex들이 서로 연결되어 모양을 형성하고, 이런 모양들이 모여서 객체를 구성한다

Vertex는 공간적 위치, 즉 위치 값 뿐만 아니라 그 이외의 정보들(색상, 법선 벡터, 텍스쳐 좌표 등)을 담고 있으며, 이 정보들은 그래픽 파이프라인에서 다양한 계산 및 렌더링 작업에 사용된다. 따라서 이를 통해 좀 더 복잡한 렌더링 효과를 구현할 수 있다!

### Vertex를 활용해 삼각형을 그려보자

<aside>
📁 Engine > 01. Header > struct.h 생성

</aside>

앞으로 Vertex에 대해 표현할 구조체인 `Vtx`를 만들어주자

```cpp
struct Vtx             // Vertex            
{
		Vec3 vPos;         // 좌표 정보
		Vec4 vColor;       // 색상 정보
};
```

→ Vertex는 단순히 좌표 정보만 담고 있지 않으므로, 일단 색상 정보까지 추가해줬다!

Vertex는 표준이 없기 때문에, 좌표 정보 + 우리가 담고 싶은 정보를 포함해 구현해주면 된다

우리가 그리고 싶은 삼각형의 세 꼭지점을 정점으로 삼고, 그 점들을 이어 생기는 Surface에 색상을 채우는 방식으로 생성할 것이다!

이때, 해당 삼각형을 그리기 위해서 짚고 넘어가야 할 두 가지 개념이 있다

→ **Vertex Buffer**와 **NDC좌표계** 

- **Vertex Buffer (정점 버퍼)**
    
    : 3D 모델의 Vertex 데이터를 저장하는데 사용되는, 그래픽 파이프라인의 한 부분이다
    
    GPU에 Vertex 데이터를 제공해서, 렌더링 작업을 수행하는 데 필요한 정보를 제공한다
    
    - Vertex 데이터를 저장
    - CPU→GPU로 Vertex 데이터를 전송
    
    즉, 렌더링 작업의 핵심 요소 중 하나로, GPU는 Vertex Buffer에서 데이터를 읽어와 그래픽 파이프라인을 통해 렌더링을 수행한다
    

- **NDC(Normalized Device Coordinates) 좌표계 (정규화 장치 좌표계)**
    
    : NDC란 정규화된 좌표계를 말한다!
    
    플레이어는 모니터로 게임의 화면을 보게 되는데, 해당 화면은 2D인 2차원이다
    
    게임의 공간이 3D여도 결국 렌더링을 통해 2D로 변환되는데, 이런 변환을 **투영**이라고 한다
    
    ![Untitled](24%2005%2003%20-%203D%20%E1%84%89%E1%85%A6%E1%84%80%E1%85%A8%E1%84%8B%E1%85%AA%20Vertex,%20Rendering%20Pipeline%2098f8c4c1607f491f84aee56abf3ad6bf/Untitled.png)
    
    투영 변환을 통해 보는 화면은 View Plane이라고 되어 있고, 이 화면이 실제 플레이어가 보게되는 화면과 같다
    
    이렇게 3D 물체가 투영 변환을 통해 2D 공간에 변환되면서 가지는 좌표계를 NDC라고 한다
    
    → 그래서 NDC 좌표계가 뭔데?!
    
    ![Untitled](24%2005%2003%20-%203D%20%E1%84%89%E1%85%A6%E1%84%80%E1%85%A8%E1%84%8B%E1%85%AA%20Vertex,%20Rendering%20Pipeline%2098f8c4c1607f491f84aee56abf3ad6bf/Untitled%201.png)
    
    - 모니터 화면의 해상도가 사용자마다 제각각이기 때문에, 정규화된 좌표계를 이용해 계산한다
    - 가로, 세로가 (-1 ~ 1), 즉 2인 좌표 시스템! 중앙이 (0, 0)을 시작으로, 좌측, 하단 끝은 -1, 우측, 상단 끝은 1이라고 가정
    - 따라서 NDC 좌표계를 사용한다면, 우리 화면의 종횡비 (가로:세로 비율)을 알고 있어야 우리가 원하는 정확한 그림을 그릴 수 있다
    - 깊이 범위는 0 ~ 1

**Vertex Buffer (정점 버퍼) 정의해주기**

```cpp
#include "pch.h"
#include "Temp.h"

#include "CDevice.h"

// 버텍스 버퍼
ID3D11Buffer* g_VB = nullptr;
Vtx g_Vtx[3] = {};

void TempInit()
{
	// Vertex Buffer 생성
	g_Vtx[0].vPos = Vec3(0.f, 1.f, 0.f);
	g_Vtx[0].vColor = Vec4(1.f, 1.f, 1.f, 1.f);
	
	g_Vtx[1].vPos = Vec3(1.f, -1.f, 0.f);
	g_Vtx[1].vColor = Vec4(1.f, 1.f, 1.f, 1.f);
	
	g_Vtx[2].vPos = Vec3(-1.f, -1.f, 0.f);
	g_Vtx[2].vColor = Vec4(1.f, 1.f, 1.f, 1.f);

	D3D11_BUFFER_DESC tVtxBufferDesc = {};
	tVtxBufferDesc.ByteWidth = sizeof(Vtx) * 3;
	tVtxBufferDesc.BindFlags = D3D11_BIND_VERTEX_BUFFER;
	tVtxBufferDesc.Usage = D3D11_USAGE_DEFAULT;
	tVtxBufferDesc.CPUAccessFlags = 0;
	tVtxBufferDesc.MiscFlags = 0;
	tVtxBufferDesc.StructureByteStride = 0;
	
	// CreateBuffer의 2번째 인자로 들어가는 것은 초기 데이터 값
	D3D11_SUBRESOURCE_DATA tSub = {};
	tSub.pSysMem = g_Vtx;
	
	if (FAILED(DEVICE->CreateBuffer(&tVtxBufferDesc, &tSub, &g_VB)))
	{
		MessageBox(nullptr, L"VertexBuffer 생성 실패", L"Temp 초기화 실패", MB_OK);
		return E_FAIL;
	}
}

// ...

void TempRelease()
{
	if(g_VB != nullptr)
		g_VB->Release();
}
```

- `DEVICE->`CreateBuffer 함수를 통해 버퍼를 생성한다!  그래픽 리소스를 생성하는 작업과 같으니까, Device가 담당하게 된다
- `IDI3D11Buffer` 객체도, `ID3D11Texture2D` 객체도 *GPU공간 안에 있는 데이터를 관리하려는 용도*다보니 사용되는 옵션이 비슷하다
- Vertex Buffer의 Description Option 살펴보기
    - **ByteWidth** → `sizeof(Vtx) * 3`
        
        : 버퍼의 크기 (바이트 단위)를 지정. 버퍼에 필요한 메모리 크기이기 때문에, 정점 데이터의 총 크기로 설정하면 된다! 
        
    - **BindFlags** → `D3D11_BIND_VERTEX_BUFFER`
        
        : 버퍼를 파이프라인의 어디에 바인딩할지 결정하는 플래그인데.. 풀어서 말하자면 버퍼가 그래픽 파이프라인에서 어떻게 사용될지 결정하는 플래그다! 이 버퍼는 지정한 용도로만 사용할 수 있게 된다
        
    - **Usage** → `D3D11_USAGE_DEFAULT`
        
        : 버퍼의 사용 패턴, 읽기/쓰기 패턴? CPU에 의한 수정이 필요한지를 결정하는 듯 하다. 우리가 설정한 DEFAULT 옵션은 GPU가 읽고 쓰기에 적합한 버퍼임을 나타내는 것!
        
    - **CPUAccessFlags** → `0`
        
        : CPU가 이 버퍼에 접근할 수 있는지를 지정. `0`이라면 CPU에 의한 버퍼 액세스 권한이 없음을 의미하는데, 어차피 Usage가 Default 설정이므로 보통 GPU 전용이기 때문에 0으로 설정된다
        
    - **MiscFlags** → `0`
        
        : 추가적인 기능 지정. 이 값이 0이므로 이 버퍼는 특별한 용도에 사용되지 않는다
        
    - **StructureByteStride** → `0`
        
        : 버퍼 내 각 구조체의 바이트 단위 크기 지정. 구조체 버퍼가 아닌 경우에는 0으로 설정된다
        

### DirectX11 3D Graphic Rendering Pipeline

먼저 **파이프라인**이란,

어떤 업무를 하는 단계, 절차, 흐름이라고 말할 수 있다.

즉, 그래픽 파이프라인은 → 그래픽이 렌더링되면서 수행되는 일련의 절차를 의미하는 것이다

DirectX 3D의 그래픽 파이프라인을 살펴보자

![Untitled](24%2005%2003%20-%203D%20%E1%84%89%E1%85%A6%E1%84%80%E1%85%A8%E1%84%8B%E1%85%AA%20Vertex,%20Rendering%20Pipeline%2098f8c4c1607f491f84aee56abf3ad6bf/Untitled%202.png)

우리는 여기서 주요 작업만을 집어서 확인할 것인데,

크게 

- Input Assemble
- Vertex Shader Stage
- RasterRizer Stage
- Pixel Shader Stage
- Output Merger Stage

로 나눠서 볼 수 있다

**Input Assemble (IA)** 

: 렌더링할 때 필요한 데이터들이나 세팅 값들 (대표적으로 정점 버퍼, 인덱스 버퍼 등…)을 입력을 받는다

현재 경우처럼 정범 버퍼의 정점 데이터들을 입력 받으면, 다른 파이프라인 단계에서 사용할 primitive로 조립한다

즉, 메모리에서 기하 자료(정점 데이터와 Index)를 읽어 기하학적 기본 도형을 조립한다.

**Vertex Shader Stage** 

: 입력으로 들어온 정점들을 하나씩 처리한다, 즉 각 정점에 대한 연산을 수행한다 (병렬로 동시에 처리)

정점 쉐이더는 항상 모든 정점들에 대해 한 번씩, 한 번만 호출되며, 하나의 출력 정점을 생성한다! 즉, 정점마다 Vertex Shader 호출

파이프라인단계에서 항상 수행이 되어야 하며, 정점에 대한 변환이 필요하지 않아도 정점 쉐이더를 생성해 연결해야 한다

Vertex Shader 함수의 구체적인 내용은 프로그래머가 구현해서 GPU에 전달하고, 각 함수는 정점에 대해 GPU에서 실행

**Rasterizer Stage**

 : Vertex 데이터가 실제 픽셀로 변환되는 단계! Vertex Shader로부터 나온 Vertex 데이터가 삼각형이나 선, 점과 같은 Primitive로 구성이 되고, 이 단계에서 적절한 Primitive로 그룹화된다. 이 Primitive에 해당하는 영역이 픽셀로 변환되고, 렌더 타겟의 실제 픽셀 위치에 매핑된다. 

**Pixel Shader Stage** 

: 각 픽셀의 데이터를 생성하고, 모든 픽셀에 해당하는 값을 하나씩 처리 (병렬로 동시에 처리)한다. 즉, 픽셀마다 Pixel Shader 호출

Pixel Shader 함수의 구체적인 내용은 프로그래머가 구현해서 GPU에 전달하고, 각 함수는 픽셀에 대해 GPU에서 실행한다

**Output Merge State**

: 최종적으로 픽셀의 색상을 생성해 렌더 타겟으로 출력하는 단계인데, 이 단계에서 일부 픽셀 조각(pixel fragment)들이 깊이 판정이나 스텐실 판정에 의해 버려지게 되며, 버려지지 않은 픽셀 조각은 후면 버퍼에 기록된다

1) *Depth Stencil State*를 고려한다, 깊이 판정, 깊이가 0에 더 가까운 Depth 정보가 이미 존재할 경우 버려지게 된다

2) *Blend State,* 색상 혼합으로, 최종 출력하려는 픽셀에 색상을 블렌딩하는 역할… 보통 옵션은 DEFAULT (픽셀 셰이더 그대로)

→ 최종적으로 우리가 원하는 그림이 그려지게 되는 것!

따라서…

우리가 해야될 것들이 몇 가지 생겼는데,

- Vertex Shader 세팅
- Pixel Shader 세팅

Vertex Shader / Pixel Shader를 호출하고 실행하는 것은 GPU이기 때문에, **쉐이더 언어**로 작성을 해줄 것이다

→ HLSL(High Level Shader Language) 

(원래는 어셈블리어로 작성하는 것이지만.. DX에 내장된 컴파일러를 통해 쉐이더 언어 → 어셈블리어로 변환이 가능해진다)

**쉐이더 파일 작성하기**

<aside>
📁 Engine > HLSL > test.fx 생성

</aside>

파일 확장자는 .fx

쉐이더 형식 → /fx, 셰이더 모델 5버전으로 설정!

![Untitled](24%2005%2003%20-%203D%20%E1%84%89%E1%85%A6%E1%84%80%E1%85%A8%E1%84%8B%E1%85%AA%20Vertex,%20Rendering%20Pipeline%2098f8c4c1607f491f84aee56abf3ad6bf/Untitled%203.png)

HLSL은 #pragma once가 적용되지 않기 때문에 C 버전으로 작성해줘야 한다

C언어 버전의 #pragma once는 이렇게 작성할 수 있다!

```cpp
#ifndef _TEST
#define _TEST

// 여기에 내용 작성...

#endif
```

→ _TEST가 정의되어 있지 않을 때에만 다음 항목이 _TEST로써 define되도록 하는 흐름! (즉, 한번만 호출된다)

```glsl
#ifndef _TEST
#define _TEST

struct VTX_IN
{
	float3 vPos : POSITION;
	float4 vColor : COLOR;
};

struct VTX_OUT
{
	float4 vPosition : SV_Position;
	float4 vColor : COLOR;
};

// 버텍스 쉐이더
VTX_OUT VS_Text(VTX_IN _in)
{
	VTX_OUT output = (VTX_OUT) 0.f;
	
	output.vPosition = float4(_in.vPos, 1.f);
	output.vColor = _int.vColor;
	
	return output;                                  // 입력으로 들어온 정점을 래스터라이저화 해 리턴
}

// 픽셀 쉐이더
float PS_Text(VTX_OUT _in) : SV_Target
{
	return float4(1.f, 0.f, 0.f, 1.f);              // 입력으로 들어온 픽셀을 빨간색으로 처리해 리턴
}

#endif
```

이제 Temp의 구현부로 돌아와서,

```cpp
// 버텍스 버퍼
// ...

// 버텍스 쉐이더 단계에서 수행할 작업의 포인터를 전달
ID3D11VertexSahder* g_VS = nullptr;

// 픽셀 쉐이더 단계에서 수행할 작업의 포인터를 전달
ID3D11PixelShader* g_PS = nullptr;

int TempInit() { ... }
void TempTick() { ... }
void TempRender() { ... }
void TempRelease() { ... }
```

→ 이제 g_VS와 g_PS를 생성해주면 된다. 이 Vertex Shader와 Pixel Shader를 생성하는 부분은 다음 시간에 진행하기!

[참고자료]

렌더링 파이프라인(Rendering Pipeline)  [https://novemberfirst.tistory.com/27](https://novemberfirst.tistory.com/27)