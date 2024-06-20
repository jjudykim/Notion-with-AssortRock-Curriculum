# 24/04/30 - Engine 제작과 Singleton Pattern/Template

태그: C++, DirectX11, 중급
상위 항목: Week23 (Week23%20394e612b264041cd8461f961c700ba03.md)
주차: 0011_Week20~29

### 본격적인 Engine 제작하기, 첫번째 단계!

먼저, 프로젝트에 새롭게 정적 라이브러리 프로젝트 Enigne을 생성해주자. 이 파일에 엔진을 제작해줄 것이다!

엔진 클래스를 정적 라이브러리 파일인 Engine에 생성해보자

```cpp
class CEngine
{
private:
	// Engine 객체를 싱글톤 패턴으로 구현해 줄 것이다..
	// ...
};
```

이때, 싱글톤 패턴을 구현하는 방법 2가지를 다시 생각해보자!

**1) 동적할당 시**

```cpp
class CEngine
{
private:
	static CEngine* pEngine;

public:
	static CEngine* GetInst()
	{ 
		pEngine = new CEngine; 
		return pEngine;
	}
	
	static void Destroy()
	{
		if (nullptr != pEngine)
			delete pEngine;
	}
	
	private:
		CEngine();
		CEngine(const CEngine& _Other) = delete;
		~CEngine();
};
```

장점 : 런타임 도중에 생성 및 해제가 가능하다

단점 : 동적할당이 이루어졌기 때문에 해제 작업도 구현해줘야 한다! 객체가 만들어져 있다면 지우고 nullptr로 초기화하는 과정이 필요

**2) 정적 변수의 선언으로 데이터 영역에 할당 시**

```cpp
class CEngine
{
public:
	static CEngine* GetInst()
	{
		static CEngine engine; // 정적 변수 초기화는 한번만
		return &engine;
	}
};
```

장점 : 간결하다, 직접적으로 메모리에서 해제할 필요 없음!

단점 : 프로그램 실행 시작부터 끝까지 계속해서 존재한다. 즉, 런타임 내내 전역변수로써 존재하기 때문에 도중에 해제할 수 없다

<aside>
🗒️ **저번에 배웠던 내용이다! 참조해서 복습하기**
→ [2024/01/19 - 모든 객체들의 최상위 클래스 구현, 현재 게임 클라이언트에 대한 관리자 구현](2024%2001%2019%20-%20%E1%84%86%E1%85%A9%E1%84%83%E1%85%B3%E1%86%AB%20%E1%84%80%E1%85%A2%E1%86%A8%E1%84%8E%E1%85%A6%E1%84%83%E1%85%B3%E1%86%AF%E1%84%8B%E1%85%B4%20%E1%84%8E%E1%85%AC%E1%84%89%E1%85%A1%E1%86%BC%E1%84%8B%E1%85%B1%20%E1%84%8F%E1%85%B3%E1%86%AF%E1%84%85%E1%85%A2%E1%84%89%E1%85%B3%20%E1%84%80%E1%85%AE%E1%84%92%E1%85%A7%2019dd9871e72b4088a9b430372abcb367.md)

</aside>

이번 수업에서는 두 가지 장단점을 합쳐서 → 동적할당을 하면서도 직접적으로 해제할 필요가 없도록 작성해볼 것이다!

**싱글톤 템플릿을 제작하자**

```cpp
template<typename T>
class CSingleton
{
private:
	static T* g_Inst;
	
	// 함수 포인터 타입 EXIT 정의
	typedef void (*EXIT)(void);
	
public:
	static T* GetInst()
	{
		if (g_Inst == nullptr)
		{
			g_Inst = new T;
		}
		retrun g_Inst;
	}	
	
	static void Destroy()
	{
		if (g_Inst)
		{
			delete g_Inst();
			g_Inst = nullptr;
		}
	}
	
public:
	CSingleton()
	{
		atexit((EXIT)&CSingleton<T>::Destroy);
	}
	CSingleton(const CSingleton& _Other) = delete;
	virtual	~CSingleton() {}
};

// 이건 무슨 의미인지 모르겠네... 찾아봐서 설명 추가하기
template<typename T>
T* CSingleton<T>::g_Inst = nullptr;
```

- atexit 사용을 위해서 Default/pch.h 에 `#include <stdlib.h>` 추가
- atexit는 프로그램이 종료될 때 등록받아놨던 함수를 호출해주는 함수!
    
    → 따라서 생성했던 Singleton 객체들의 생성자에서 Destroy 함수 호출을 예약해놨기 때문에 프로그램이 종료될 때 자동으로 호출된다
    

싱글톤 기능이 존재하는 헤더파일(`singleton.h`)을 제작한 것이므로, 싱글톤 기능을 사용할 곳에서는 해당 헤더파일을 include 해줘야 한다 → 매번 사용하기 귀찮으니, 이를 간략화 해보자!

1) 제작한 Singleton Template을 define을 통해 간략화

```cpp
#pragma once

#define SINGLE(Type) private:\
											Type();\
											~Type();\
											friend class CSingleton<Type>;
```

2) define.h과 singleton.h을, `global.h`라는 *헤더파일 참조용 헤더파일*을 만들어 넣어주자

```cpp
#pragma once

#include <Windows.h>

#include "singleton.h"
#include "define.h"
```

3) 해당 global.h을 default pch.h에서 참조해주기

```cpp
// ...
#include <stdlib.h>
#include "global.h"
// ...
```

### 싱글톤 템플릿을 활용해 Engine 작성하기

```cpp
#pragma once

class CEngine
	: public CSingleton<CEngine>
{
	SINGLE(CEngine);
	
private:
	HWND     m_hWnd;             // 윈도우 핸들 값
	POINT    m_ptResolution      // 윈도우 창 크기	
	
public:
	void Init(HWND _wnd, POINT _ptResolution);
	void ChangeWindowScale(UINT _Width, UINT _Height);
}
```

```cpp
#include "pch.h"
#include "CEngine.h"

CEngine::CEngine()
	: m_hWnd(nullptr)
	, m_ptResolution{}
{
}

CEngine::~CEngine()
{
}

void CEngine::Init(HWND _wnd, POINT _ptResolution)
{
	m_hWnd = _wnd;
	m_ptResolution = _ptResolution;
	ChangeWindowScale(_ptResolution.x, _ptResolution.y);
}

void CEngine::ChangeWindowScale(UINT _Width, UINT _Height)
{
	bool bMenu = false;     // 메뉴바 없이
	if (GetMenu(m_hWnd))
		bMenu = true;
	
	RECT rt = { 0, 0, _Width, _Height };   // 인자로 받은 POINT를 활용해 RECT로 변경
	AdjustWindowRect(&rt, WS_OVERLAPPEDWINDOW, bMenu);
	SetWindowPos(m_hWnd, nullptr, 0, 0, rt.right - rt.left, rt.bottom - rt.top, 0);
}

```