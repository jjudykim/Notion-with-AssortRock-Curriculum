# 2024/01/18 - wWinMain 분석하기(2) - message loop

태그: C++, WinAPI, 중급
날짜: 2024/01/18
상위 항목: Week8 (Week8%2097066e5626ef4fb4ac1b17db8d09a64b.md)
주차: 0001_Week1~9

### +) 기본자료형이 typedef로 다른 이름이 된 경우가 많다!

Windows API에서는 

```cpp
// int타입을 대체하는 MYINT, int*타입을 대체하는 PMYINT 타입을 선언
typedef int MYINT, *PMYINT; 
```

→ 상황에 따른 가독성을 향상시킬 수 있다!

### 3) Main message loop 분석하기

지난 시간에 이어서, `wWinMain` 함수에서 중요한 부분의 3번째에 해당하는 main message loop에 대해서 분석해보자

```cpp
HACCEL hAccelTable = LoadAccelerators(hInstance, 
																			MAKEINTRESOURCE(IDC_GAMECLIENT)); 
MSG msg;

 while (GetMessage(&msg, nullptr, 0, 0))
 {
     // 단축키 조합 체크하는 테이블
     if (!TranslateAccelerator(msg.hwnd, hAccelTable, &msg))
     {
         ::TranslateMessage(&msg);
         ::DispatchMessage(&msg);
     }
 }

 return (int) msg.wParam;
```

---

**위 코드 톺아보기**

```cpp
HACCEL hAccelTable 
= LoadAccelerators(hInstance, MAKEINTRESOURCE(IDC_GAMECLIENT));
```

: 엑셀레이터(단축키) 테이블을 로드해 ***엑셀레이터(단축키) 테이블 핸들***을 얻는 부분

- `IDC_GAMECLIENT`라는 정수를 매크로(`MAKEINTRESOURCE`)를 통해 리소스 식별자를 만들고,
- `LoadAccelerators` 함수를 통해 (1) 현재 실행중인 프로그램의 인스턴스 핸들과 (2) 엑셀레이터 테이블 리소스의 식별자로 얻은 ***엑셀레이터 테이블 핸들***을, `hAccelTable`이라는 `HACCEL` 타입으로 저장한다.
- 해당 핸들은 message loop에서 키 이벤트를 처리할 때 사용된다 
→ 메뉴 / 단축키 / UI의 특정 동작 관리

```cpp
MSG msg;
```

: *메세지 정보*를 받을 구조체 변수

내부적으로 tagMSG라는 구조체가 생성되어 반환되는 것!

```cpp
while (GetMessage(&msg, nullptr, 0, 0))
 {
     // 단축키 조합 체크하는 테이블
     if (!TranslateAccelerator(msg.hwnd, hAccelTable, &msg))
     {
         ::TranslateMessage(&msg);
         ::DispatchMessage(&msg);
     }
 }
```

1) `while (GetMessage(…)) {}` : `GetMessage` 함수를 사용해 loop를 돌린다

- `GetMessage` 함수의 특징
    - 메세지 큐에서 메세지가 있을 때까지 대기하고, ***메세지가 있으면 해당 메세지를 return***해 가져온다. 메세지가 없으면 대기 상태로 들어간다
    - 주로 이벤트 기반의 프로그램에서 사용
    - return 조건 → 메세지 큐에서 메세지가 있으면 return 한다
    - `WM_QUIT`이 발생하면 GetMessage 함수는 false를 반환, 
    `WM_QUIT` 이외의 메세지면 GetMessage 함수는 true를 반환
    
    <aside>
    📒 **WM_QUIT**
    : 프로그램이 종료될 때 발생하는 메세지로, `PostQuitMessage` 함수를 호출해 메세지 큐에 추가된다. 해당 함수를 호출하면 프로그램은 메세지 루프에서 빠져나가게되고, 종료 조건이 충족되면 ‘WM_QUIT’ 메세지가 메세지 큐에 추가되면서 메세지 루프가 종료된다
    
    현재 코드에서는 `WndProc`(윈도우 프로시저) 함수의 case 중 `WM_DESTROY`인 경우에 PostQuitMessage(0)이 실행되어 WM_QUIT이 메세지 큐에 추가되고, 메세지 루프가 종료되며 프로그램이 종료된다.
    
    </aside>
    

2) `if (!TranslateAccelerator(msg.hwnd, hAccelTable, &msg))`

→ 단축키 별에 해당하는 특정 동작을 수행할 수 있는 조건문

- TranslateAccelerator는 인수로 전달한 msg의 현재 윈도우 핸들과, 엑셀레이터 테이블에 대한 핸들,  메세지 정보를 저장할 구조체를 통해 지정된 윈도우 프로시저에 직접 보내고, **윈도우 프로시저가 메세지를 처리할 때까지 반환되지 않는다**
- `hAccelTable`에 정의된 단축키 키를 처리하고, ⭐만약 단축키 키에 해당하지 않는 메세지라면 (즉, if문에 해당하는 경우라면) 일반적인 메세지 처리 (`TranlateMessage`와 `DispatchMessage`)가 이루어진다⭐

3) `TranslateMessage(&msg);`

: 주로 키보드 입력과 관련이 있고, 키 이벤트에 대한 메세지를 생성해 윈도우 프로시저에 전달

→ 키 이벤트에 대한 가상 키 메세지들을 생성하고, 변환된 메세지들이 메세지 큐에 추가되어 윈도우 프로시저에서 이벤트를 처리할 수 있도록 한다

4) `DispatchMessage(&msg);`

: GetMessage로부터 받은 메세지를 실제 윈도우 프로시저로 전달해 현재 메세지의 hwnd 필드에 해당하는 윈도우의 프로시저가 호출된다

- 메세지를 처리하기 위해 윈도우 프로시저에 메세지를 전달한다
- 윈도우 프로시저의 반환값은 DispatchMessage 함수의 반환값으로 전달된다
- 이를 통해 메세지 루프에서 다음 메세지를 처리하게 한다

→ 사용자 입력 및 시스템 이벤트에 대한 응답을 윈도우 프로시저를 통해 수행하게 함

여기서 반복해서 등장하는 *윈도우 프로시저*란..

<aside>
✍️ **윈도우 프로시저(Window Procedure)**

: Windows 프로그램에서 이벤트와 메세지에 응답하는 코드를 포함하는 함수로, **각 윈도우는 자체의 윈도우 프로시저를 가지고 있다** (MyRegisterClass를 생성할 때 lpfnWndProc 변수로 설정)

여기서는 switch문을 통해 여러 종류의 메세지에 대한 처리를 수행했다. 메세지에 대한 특정한 동작을 정의해, 윈도우 이벤트에 대한 응답을 담당하게 된다.

</aside>

```cpp
LRESULT CALLBACK WndProc(HWND hWnd, UINT message, WPARAM wParam, LPARAM lParam)
{
    switch (message)
    {
    case WM_LBUTTONDOWN:
        ShowWindow(g_hWnd, false);
        break;

    case WM_COMMAND:
        {
            int wmId = LOWORD(wParam);
            // 메뉴 선택을 구문 분석합니다:
            switch (wmId)
            {
            case IDM_ABOUT:
                // Dialog Box 생성 함수
                DialogBox(hInst, MAKEINTRESOURCE(IDD_ABOUTBOX), hWnd, About);
                break;
            case IDM_EXIT:
                DestroyWindow(hWnd);
                break;
            default:
                return DefWindowProc(hWnd, message, wParam, lParam);
            }
        }
        break;
    case WM_PAINT:
        {
            PAINTSTRUCT ps;
            HDC hdc = BeginPaint(hWnd, &ps);
            // TODO: 여기에 hdc를 사용하는 그리기 코드를 추가합니다...
            EndPaint(hWnd, &ps);
        }
        break;
    case WM_DESTROY:
        PostQuitMessage(0);
        break;
    default:
        return DefWindowProc(hWnd, message, wParam, lParam);
    }
    return 0;
}
```

→ 윈도우 프로시저 함수인 `WndProc`는 CALLBACK 함수로 정의되었음을 알 수 있다

근데 Call back이 뭔데?!

### CallBack 함수

: ***다른 함수에 의해 호출되는 함수***로, 정확히는 “호출되기를 기다리는” 함수라고 생각할 수 있다

> 함수의 주소(함수포인터를 통해)를 알려줘서, 
특정 상황(조건)이 맞으면 알려준 함수가 호출되는 구조!
> 

- window(일반적인 윈도우)
    
    →생성을 위한 전달해야 할 정보가 방대하기 때문에, 윈도우 클래스를 생성해 전달했다. (MyRegisterClass)
    

- dialog box(ABOUT을 통해 생성된 윈도우)
    
    → 리소스에서 디자인을 통해 간단하게 만들 수 있는 것으로, 새로운 dialog를 생성하고 싶다면 리소스를 추가해야 한다
    

그럼 여기서 또 callback 함수가 적용되었던 예인 Dialog Box의 생성 과정에 대해서 알아보자

### Dialog Box와 ABOUT

아까 wndProc 함수를 보면, `WM_COMMAND`의 case에서 전달받은 메세지의 ID가 `IDM_ABOUT`일 경우 아래 코드가 실행된다

```cpp
DialogBox(hInst, MAKEINTRESOURCE(IDD_ABOUTBOX), hWnd, About);
```

이때 IDM_ABOUT은 리소스 뷰의 Accelerator에 작성되어 있는데, 해당 단축키를 입력하면 윈도우 프로시저에서 위 작업이 시행되는 것!

`IDD_ABOUTBOX`로 정의되어 있는 디자인 리소스를 활용해, DialogBox 함수를 통해 정보 대화 상자를 만드는 것인데.. 이때 프로시저로 `About`을 사용한다!

```cpp
INT_PTR CALLBACK About(HWND hDlg, UINT message, WPARAM wParam, LPARAM lParam)
{
    UNREFERENCED_PARAMETER(lParam);
    switch (message)
    {
    case WM_INITDIALOG:
        return (INT_PTR)TRUE;

    case WM_COMMAND:
        if (LOWORD(wParam) == IDOK || LOWORD(wParam) == IDCANCEL)
        {
            EndDialog(hDlg, LOWORD(wParam));
            return (INT_PTR)TRUE;
        }
        break;
    }
    return (INT_PTR)FALSE;
}
```

→ 자세히 보면 `WndProc` 함수와 상당히 유사하다는 것을 볼 수 있는데, 반환 타입이 달라보여도 사실 내부적으로 typedef 됐을 뿐 __int64라는 동일한 타입을 띈다!

따라서, **CALLBACK 함수는 일반적으로 함수 포인터 형식을 따르는 함수**라고 할 수 있다!

Windows API에서 CALLBACK이라는 용어 자체가 해당 함수가 호출될 때 시스템이 사용자 코드를 콜백한다는 것을 나타내면서, 함수 포인터로 전달되어야 한다는 뜻!

### message loop를 활용해 어떻게 게임을 구현할 수 있을까?

메세지 루프를 활용해 게임을 구현하는 방법에 대해서 고민해보자

> 👎 **방법 1. GetMessage를 활용하기 위해 메세지를 계속해서 넣어주는 방법!**
> 

`GetMessage` 함수는, 메세지 큐에 메세지가 들어올 때까지 대기하는 함수였기에 메세지가 들어오지 않는다면 동작을 하지 않았다

→ 그럼 메세지 큐에 메세지를 계속 넣어준다면?!

`Timer`를 통해 메세지 큐에 메세지를 계속해서 삽입해주고,

프로시저에 case를 만들어 case WM_TIMER로 실행하려는 명령을 수행하면 된다?

setTimer(g_hWnd, 0, 50, nullptr);

killTimer(g_hWnd, 0);

→ 너무 비효율적이얍. 이런 구조를 쓰진 않을 것이다!

message loop를 변경해서 사용하자 → GetMessage를 사용하지 않을거다!!

> 👍 **방법 2. PeekMessage 를 활용하는 방법!** ⭐⭐
> 
- 여기서 Peek은 엿보다라는 뜻!
- 메시지 큐에서 메시지를 확인하는 역할을 하면서, 
메세지가 없으면 즉시 반환하고, 메시지가 있으면 해당 메세지를 가져온다
- 메시지를 큐에서 제거하거나, 제거하지 않을 수 있다!
    
    *option* : `PM_REMOVE` → 확인 후 제거, `PM_NOREMOVE` → 제거하지 않고 확인만
    
- 주로 메시지 루프에서 주기적으로 메시지를 확인하고 다른 작업을 수행할 때 사용
- ⭐**메세지 큐에 메세지가 있든, 없든, return은 된다!** ⭐(우리가 peekMessage를 채택한 이유) → 리턴값의 의미 : 메세지가 있어야 true, 없다면 false

```cpp
while(true)
{
	// 메세지가 있다면, 메세지 큐에서 메세지를 제거
	if (PeekMessage(&msg, nullptr, 0, 0, PM_REMOVE)
	{
		// while문을 벗어날 조건이 필요 (꺼내온 메세지가 QUIT이라면 종료)
		if (msg.message == WM_QUIT)
			break;

		if (!TranslateAccelerator(msg.hwnd, hAccelTable, &msg))
		{
			TranslateMessage(&msg);
	    DispatchMessage(&msg);
		}
	}
	else
	{
		// 메세지가 큐에 없을 때에는 게임 코드 실행
		// Game Logic을 여기에 구현
	}
}
```

### 미리 컴파일된 헤더 (pre-compiled header)

= pcheader

**사용하는 이유**

1) 공통적인 헤더들을 참조하는 목적

2) 컴파일 속도를 높이기 위해

**만드는 방법**

프로젝트 속성 > 미리 컴파일된 헤더 > 만들기 (pch.h를 만들어줬다!)

→ pch.h와 매칭되는 cpp 파일 역시 만들어 줘야 하고,

pch.cpp 의 파일 속성 > 미리 컴파일된 헤더 > 사용으로 변경해주기

이제부터 `#include “pch.h”`이 강제된다!

`pch.h`에는, 많이 사용할만한 전처리기들과 작업들을 명시해놓자

```cpp
#pragma once

#include <Windows.h>
#include <vector>      // 동적 배열
#include <list>        // 연결형 리스트
#include <map>         // 이진 탐색 트리

using std::vector;
using std::list;
using std::map;
using std::make_pair;
```

최상위 부모 클래스인 CEntity를 작성해보자

```cpp
#pragma once

class CEntity
{
private:
	UINT	 m_ID;          // 해당 오브젝트마다 절대 겹쳐지지 않을 본인만의 고유한 식별자
	
public:
	CEntity();
	virtual ~CEntity();
};
```

→ 다음 시간에 이어서..