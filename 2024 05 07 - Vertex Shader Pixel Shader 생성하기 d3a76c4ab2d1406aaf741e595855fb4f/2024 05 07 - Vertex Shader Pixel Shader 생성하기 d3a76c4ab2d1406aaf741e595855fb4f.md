# 24/05/07 - Vertex Shader / Pixel Shader 생성하기

태그: C++, DirectX11, 그래픽스, 중급
상위 항목: Week24 (Week24%20d7c7179c6b2f49348c9827ed8755dacc.md)
주차: 0011_Week20~29

### Render에서 필요한 작업들

Render에서는 각 렌더링 파이프라인의 단계별 설정이 이루어질 것이다!

현재까지 IA(Vertex Buffer 생성), Vertex Shader, Pixel Shader의 단계를 구현해봤으니, 해당 단계의 설정들을 진행해주자

참고로, 여기서 진행하는 작업들은 렌더링 파이프라인이 시작된다면 해당 옵션을 사용하라는 '*사전 설정*'일 뿐, 실제로 렌더링 파이프라인이 시작된 것은 아니다! 

여기서 명시하는 단계, 즉 호출 순서는 아무 상관이 없다 (단지 사전 설정일 뿐이니까!)

→ 나중에 Rendering을 시작하는 함수가 호출이 되어야 렌더링이 시작된다

```cpp
void TempRender()
{
	// ========== IA 단계용 Set ============
	UINT stride = sizeof(Vtx);     // 정점 버퍼에서 정점 간의 간격을 알려주기 위한 Vtx의 크기 전달
	UINT offset = 0;               // 버퍼의 시작 지점을 설정 (0이므로 처음부터 읽는다)
	
	// 입력 어셈블러에 정점 버퍼를 바인딩
	CONTEXT->IASetVertexBuffers(0, 1, &g_VB, &stride, &offset);
	// 입력 어셈블러에 사용할 Primitive Topology 지정        
	CONTEXT->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST);    
	
	// ===== Vertex Shader 단계용 Set =====
	CONTEXT->VSSetShader(g_VS, nullptr, 0);
	
	// ===== Pixel Shader 단계용 Set =====
	CONTEXT->PSSetShader(g_PS, nullptr, 0);
}
```

- **IASetVertexBuffers**
    
    : 입력 어셈블러에 정점 버퍼를 바인딩하는 함수로, 각각의 인자는 다음과 같다
    
    - *StartSlot* → `0`
        
         : 버퍼 바인딩을 시작할 입력 슬롯의 인덱스
        
    - *NumBuffers* → `1`
        
        : 바인딩할 정점 버퍼의 수, ppVertexBuffer에 포함된 버퍼의 수를 나타낸다
        
    - *ppVertexBuffers* → `&g_VB`
        
        : `ID3D11Buffer` 포인터를 지정, 파이프라인에 바인딩할 실제 정점 버퍼를 포함하는 배열을 전달
        
    - *pStrides* → `&stride`
        
        : 각 버퍼에 담긴 요소 간의 바이트 크기, 미리 담아놨던 stride를 전달했다
        
    - *pOffsets* → `&offset`
        
        : 각 버퍼에서 시작 정점까지의 바이트 오프셋으로, 처음으로 읽기 시작할 위치 지정
        

- **IASetPrimitiveTopology**
    - Primitive는 컴퓨터 그래픽스에서 선/원/곡선/다각형과 같은 기본적인 요소들을 의미하고, Topology는 구조, 형식을 의미한다
        
        → 즉, PrimitiveTopology는 컴퓨터 그래픽스에서 사용되는 기본 형식이라고 생각하면 된다!
        
        MSDN을 참고하면 다음과 같은 옵션이 존재한다 (Point List, Line List, Line Strip, Triangle List, Triangle Strip, …)
        
    - 따라서 현재는 `D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST` 옵션을 사용했으므로, 3개의 점을 하나의 삼각형으로 해석하게 되면서 해당 옵션대로 Rasterizer 단계에서 반영된다

- **VSSetShader**
    
    : 정점 셰이더를 파이프라인에 바인딩
    
- **PSSetShader**
    
    : 픽셀 셰이더를 파이프라인에 바인딩
    

<aside>
✍️ **쉐이더가 무엇일까?**

쉐이더란, 그래픽 프로그래밍에서 GPU가 수행하는 렌더링 작업을 의미하는 작은 프로그램이다. 특히 화면에 출력할 픽셀의 위치와 색상을 계산하는 함수인데, Shade라는 단어 자체가 ‘색의 농담, 색조, 명암 효과를 주다’ 라는 뜻을 가지고 있으며 접미사 ‘-er’이 붙은 단어라고 보면 된다. 즉, **화면에 존재하는 각 픽셀의 위치와 색상을 계산하는 주체가** 바로 쉐이더라고 생각하면 된다!

</aside>

### Vertex Shader 제작하기

- `<d3dcompiler.h>` → DX에 내장된 이 라이브러리를 통해서 우리가 HLSL로 작성한 셰이더 코드를 바이너리 코드로 컴파일해준다!
- Release 했을 때 해당 fx파일이 포함되어야 하므로, 셰이더 코드도 content로 같이 배포되어야 한다
    
    <aside>
    📁 Project … > OutputFile > content > shader
    
    </aside>
    
    그리고 배치 파일을 통해서 Engine에서 제작에 사용된 fx파일도 복사되도록 설정하기!
    
    ```cpp
    xcopy /s /y /exclude:exclude_list.txt ".\Project\Engine\*.fx" ".\OutputFile\content\shader"
    ```
    

```cpp
wstring strShaderPath = CPathMgr::GetInst()->GetContentPath();

HRESLUT hr = D3DCompileFromFile((strShaderPath + L"shader\\test.fx").c_str()
						, nullptr, D3D_COMPILE_STANDARD_FILE_INCLUDE
						, "VS_Test", "vs_5_0", D3DCOMPILE_DEBUG, 0, &g_VSBlob, &g_ErrBlob);
						
if (FAILED(hr))
{
	// HRESLUT의 결과로 2~3이 반환되거나,
	// ErrBlob이 생성되지 않았지만 컴파일이 실패한 경우 보통 "경로 및 파일 문제"
	if (g_ErrBlob != nullptr)
	{
		MessageBoxA(nullptr, (char*)g_ErrBlob->GetBufferPointer(), "쉐이더 컴파일 실패", MB_OK);
	}
	else
	{
		errno_t err = GetLastError();
		wchar_t szErrMsg[255] = {};
		swprintf_s(szErrMsg, 255, L"Error Code : %d", err);
		MessageBox(nullptr, szErrMgs, L"쉐이더 컴파일 실패", MB_OK);
	}
	return E_FAIL;
}

DEVICE->CreateVertexShader(g_VSBlob->GetBufferPointer(), g_VSBlob->GetBufferSize(), nullptr, &g_VS);	
```

- `D3DCompileFromFile()` 함수에 대해 알아보자
    
    : Direct3D 11에서 HLSL 코드로 작성된 쉐이더를 파일에서 컴파일하는 기능을 제공한다. HLSL 파일을 통해, 해당 쉐이더를 컴파일하고, 필요한 경우 컴파일 시 발생한 오류도 제공한다! → Blob 형태로
    
    - `pFileName` → *(strShaderPath + L”shader\\test.fx”).c_str()*
        
        : 쉐이더 파일의 경로 및 이름을 지정
        
    - `pDefines` → *nullptr*
        
        : 쉐이더 코드에 전달할 매크로 정의, nullptr일 경우 매크로 정의를 사용하지 않는다
        
    - `pInclude` → *D3D_COMPILE_STANDARD_FILE_INCLUDE*
        
        : #include 지시문을 사용할 때 포함할 파일을 찾기 위한 인터페이스로, 해당 옵션으로 기본 경로에서 파일을 포함시킬 수 있다
        
    - `pEntryPoint` → *“VS_TEST”*
        
        : 컴파일할 쉐이더 코드의 진입 함수
        
    - `pTarget` → *“vs_5_0”*
        
        : 타겟 쉐이더 버전. 정점 쉐이더라면 “vs_5_0”이, 픽셀 쉐이더라면 “ps_5_0”을 지정할 수 있다
        
    - `Flags1` → *D3DCOMPILE_DEBUG*
        
        : 컴파일 시 사용할 옵션 플래그로, D3DCOMPILE_DEBUG 옵션은 디버그 정보를 포함시킨다
        
    - `Flags2` → *0*
        
        : 효과를 위한 추가 플래그
        
    - `ppCode` → *&g_VSBlob*
        
        : 컴파일 된 쉐이더 코드를 담을 포인터
        
    - `ppErrorMsgs` → *&g_ErrBlob*
        
        : 컴파일 시 발생한 오류 메세지를 담을 포인터
        

- 여기서 Blob이란 개념이 사용되어 다음과 같이 전역 변수로 `ID3DBlob` 객체 역시 만들어줬다. ***Blob이 뭐길래?***
    
    ```cpp
    ID3DBlob*             g_VSBlob = nullptr;     // 컴파일 성공 시 블롭 제작됨 (셰이더 코드의 중간 결과, 바이너리 코드)
    ID3D11VertexShader*   g_VS = nullptr;         // 제작된 Vertex Shader
    
    ID3DBlog*             g_ErrBlob = nullptr;    // 컴파일 실패 시 블롭 제작됨 (셰이더 코드의 컴파일 실패 이유)
    ```
    
    <aside>
    ✍️ **Blob이란**
    
    Blob은 Binary Large Object의 약어로, Direct3D에서 데이터의 블록을 나타내는 인터페이스다! 다양한 종류의 구조화되지 않은 데이터를 관리할 때 사용되는데, 일반적으로 쉐이더 컴파일러와 같은 Direct3D API 기능에서 쉐이더 코드, 오류 메세지, 기타 바이너리 데이터 등을 저장하고 전달하는 데 사용된다.
    
    **ID3DBlob의 기본적인 기능**
    
    - 데이터 저장 : 쉐이더 컴파일 결과와 같은 바이너리 데이터를 메모리 내에 저장
    - 데이터 접근 : 저장된 데이터에 접근할 수 있으며, 데이터의 크기도 얻을 수 있음
    
    **주요 메소드들**
    - `GetBufferPointer` : 저장된 데이터에 대한 포인터를 반환, 이 포인터를 통해 데이터를 읽거나 데이터를 다른 API 함수에 전달할 수 있음
    -  `GetBufferSize` : Blob 내에 저장된 데이터의 크기(바이트)를 반환, 메모리 관리나 데이터 처리 시 필요한 정보 제공
    
    정리하자면, ID3DBlob은 Direct3D 프로그래밍에서 중요한 데이터 관리 도구로, 주로 쉐이더 컴파일 결과와 오류 메세지를 저장하는데 사용된다! 이런 Blob 객체는 메모리 효율성을 높이고, 복잡한 데이터 구조 없이 바이너리 데이터를 쉽게 다루게 해준다
    
    </aside>
    

- 결론적으로는, 이 Blob을 활용해서 CreateVertexShader 함수를 활용해 Vertex Sader를 제작했다
    
    CreateVertexShader 함수는 컴파일된 쉐이더 바이트 코드를 입력으로 받아, 그래픽 파이프라인에 사용할 수 있는 Vertex Shader 객체를 생성한다!
    
    - `pShaderBytecode` → *g_VSBlob->GetBufferPointer()*
        
        : 컴파일된 쉐이더 바이트 코드의 포인터, 바이트 코드는 HLSL 코드를 D3DCompileFromFile 같은 함수로 컴파일해 얻을 수 있다
        
        → Blob에 담겨있는 데이터가 바로 바이트 코드!
        
    - `ByteCodeLength` → *g_VSBlob→GetBufferSize()*
        
        : 전달된 쉐이더 바이트 코드의 길이
        
    - `pClassLinkage` → nullptr
        
        : 쉐이더 코드에서 동적으로 선택적 쉐이더 함수를 사용할 수 있게 하는 링키지를 지정. 일반적으로 nullptr로 지정한다
        
    - `ppVertexShader` → *&g_VS*
        
        : 생성된 정점 쉐이더 객체의 주소를 저장할 포인터의 주소, 성공적으로 쉐이더가 생성되면 포인터를 통해 정점 쉐이더에 접근 가능
        

### Pixel Shader 제작하기

마찬가지로 Pixel Shader도 제작하기 위해선 D3DCompileFromFile 함수를 통해 쉐이더 코드를 읽어야 하기 때문에, Blob 제작해주기!

```cpp
ID3DBlob*             g_PSBlob = nullptr;
ID3D11PixelShader*    g_PS = nullptr;
```

```cpp
HRESLUT hr = D3DCompileFromFile((strShaderPath + L"shader\\test.fx").c_str()
						, nullptr, D3D_COMPILE_STANDARD_FILE_INCLUDE
						, "PS_Test", "ps_5_0", D3DCOMPILE_DEBUG, 0, &g_PSBlob, &g_ErrBlob);
								
if (FAILED(hr))
{
	// HRESLUT의 결과로 2~3이 반환되거나,
	// ErrBlob이 생성되지 않았지만 컴파일이 실패한 경우 보통 "경로 및 파일 문제"
	if (g_ErrBlob != nullptr)
	{
		MessageBoxA(nullptr, (char*)g_ErrBlob->GetBufferPointer(), "쉐이더 컴파일 실패", MB_OK);
	}
	else
	{
		errno_t err = GetLastError();
		wchar_t szErrMsg[255] = {};
		swprintf_s(szErrMsg, 255, L"Error Code : %d", err);
		MessageBox(nullptr, szErrMgs, L"쉐이더 컴파일 실패", MB_OK);
	}
	return E_FAIL;
}

DEVICE->CreatePixelShader(g_PSBlob->GetBufferPointer(), g_PSBlob->GetBufferSize(), nullptr, &g_PS);	
```

### Temp의 Init, Render의 총정리

그럼 Temp의 `Init`에서 이뤄진 모든 과정을 정리해보자

- Vertex Buffer 생성
- Vertex Shader 생성
- Pixel Shader 생성

그리고, `Render`에서 이뤄진 과정을 정리해보자

- IA 단계의 Vertex Buffer 설정
- IA 단계의 프리미티브 토폴로지 설정
- Vertex Shader 설정
- Pixel Shader 설정
- Draw → 렌더링 파이프라인 실행

CEngine에서는 현재까지 이렇게 진행되고 있다

```cpp
int CEngine::Init(HWND _wnd, POINT _ptResolution)
{
	// ...
	if (FAILED(TempInit())
	{
		MessageBox(nullptr, L"장치 초기화 실패", L"CDevice 초기화 실패", MB_OK);
		return E_FAIL;
	}
}

void CEngine::Progress()
{
	TempTick();

	CDevice::GetInst()->Clear();

	TempRender();

	CDevice::GetInst()->Present();
}
```

### 레이아웃(InputLayout)과 시맨틱(Semantic)

사실 아직 IA 단계에서 Vertex Buffer의 데이터 처리를 위한 준비 과정이 끝나지 않았다!

→ **레이아웃(Input Layout)**이라는 단계가 남아있다

쉐이더에서 의미하는 레이아웃은, Vertex나 Pixel과 같은 데이터가 GPU의 그래픽 파이프라인으로 어떻게 전달되고 해석되는지 정의하는 개념이다. 특히 쉐이더가 GPU에서 실행될 때 각 데이터 요소를 어떻게 배치하고 해석해야 하는지를 지정하는데, **Vertex 데이터가 가진 각각의 속성이 Vertex Shader의 어떤 입력 변수에 대응하는지 레이아웃을 통해 정의**된다!

이는 입력 어셈블러(IA) 단계에서 Vertex Buffer의 데이터 구조와 Vertex Shader의 Input 사이의 관계를 정의해, GPU가 데이터를 정확하게 처리할 수 있도록 하는 것!

현재 경우를 예로 설명해보면,

```cpp
// Vertex Buffer의 데이터 구조
struct Vtx
{
	Vec3	vPos;
	Vec4	vColor;
};

```

```cpp
// Vertex Shader의 Input 타입
struct VTX_IN
{
	float3 vPos : POSITION;        // 이게 Semantic!
 	float4 vColor : COLOR;
}
```

→ 이 두 구조의 관계를 통해 GPU에게 데이터를 정확하게 처리할 수 있도록 매핑해주는 것!

이때 **Semantic**이라는 개념이 사용되는데, 

Semantic은 쉐이더 내에서 변수들이 어떤 목적으로 사용되는지에 대한 의미나 역할을 정의하는 태그다. 

쉐이더 코드와 파이프라인 사이의 인터페이스를 형성해, Vertex 데이터를 그래픽 파이프라인의 다른 단계로 올바르게 전달되도록 보장한다

### Input Layout 객체 생성

따라서 우리는 Layout을 통해 Vertex Buffer를 구성하는 데이터에 대한 정보를 Vertex Shader에게 전달할 것이다!

이를 위해 Layout 객체를 생성해주자

```cpp
// ...

D3D11InputLayout* g_Layout = nullptr;

// ...

int TempInit()
{
	// Vertex Buffer 생성
	// VS 생성
	// PS 생성
	// ...

	// Layout 생성
	D3D11_INPUT_ELEMENT_DESC Element[2] = {};

	Element[0].AlignedByteOffset = 0;
	Element[0].Format = DXGI_FORMAT_R32G32B32_FLOAT;          // 4byte 3개짜리 float
	Element[0].InputSlot = 0;                                 // 시작 위치로부터 정점의 위치
	Element[0].InputSlotClass = D3D11_INPUT_PER_VERTEX_DATA;
	Element[0].InstanceDataStepRate = 0;
	Element[0].SemanticName = "POSITION";                     // 일종의 키 값
	Element[0].SemanticIndex = 0;                             // 정점 내에서 이름이 겹칠 경우 구분하기 위한 INDEX                   
                                                            // (Maxtrix 전달과 같은 경우에 사용된다)

	Element[1].AlignedByteOffset = 12;
	Element[1].Format = DXGI_FORMAT_R32G32B32A32_FLOAT;       // 4byte 4개짜리 float
	Element[1].InputSlot = 0;                                 // 시작 위치로부터 정점의 위치
	Element[1].InputSlotClass = D3D11_INPUT_PER_VERTEX_DATA;
	Element[1].InstanceDataStepRate = 0;
	Element[1].SemanticName = "COLOR";
	Element[1].SemanticIndex = 0;

	DEVICE->CreateInputLayout(Element, 2, g_VSBlob->GetBufferPointer(), g_VSBlob.GetBufferSize(), &g_Layout);
	
	return S_OK;
}
```

- Layout을 구성하는 데에 사용된 `D3D11_INPUT_ELEMENT_DESC`에 대해 알아보자
    
    : Vertex Buffer의 개별 데이터 요소를 정의하는 데에 사용되는 이 구조체는, 입력 레이아웃을 생성할 때 각 Vertex 데이터 요소 (위치, 색상 등등..)의 특성을 지정할 수 있다!
    
    - `AlignedByteOffset` : 입력 슬롯에서 이 데이터 요소의 시작 바이트에 대한 오프셋을 지정, 예시를 보는게 더 이해가 빨리 될 것이다!
    - `Format` : 데이터 요소의 데이터 형식으로, DXGI_FORMAT 열거형의 값을 사용한다. Vec3는 `DXGI_FORMAT_R32G32B32_FLOAT`로 표현
    - `InputSlot` : 이 데이터 요소를 포함하는 입력 슬롯의 인덱스, 현재는 전달하는 Vertex Buffer가 하나이므로 둘 다 0이다
    - `InputSlotClass` : 입력 데이터가 어떻게 GPU에 의해 엑세스 되는지! 현재는 `D3D11_INPUT_PER_VERTEX_DATA`, 즉 정점당 데이터로 설정
    - `InstanceDataStepRate` : 입력 데이터가 `INSTANCE_DATA`일 경우 몇 개의 인스턴스 당 다음 데이터 요소로 넘어가는 지.. 현재는 영향 X
    - `SemanticName` : 이 요소가 사용하는 Semantic의 이름을 나타내는 문자열 (1byte 문자열이다!)
    - `SemanticIndex` : 같은 Semantic 이름을 가진 요소들을 구분하기 위한 인덱스로, 현재는 Semantic이 겹치는 경우가 없으므로 0

- `CreateInputLayout` 함수를 통해 Layout 객체를 생성했다
    
    : 그래픽 파이프라인의 IA 단계에서 사용될 데이터의 형식을 정의하는 역할을 위해, 입력 레이아웃 객체를 생성하는 함수!
    
    - `pInputElementDescs` → *Element*
        
        : **D3D11_INPUT_ELEMENT_DESC** 구조체 배열에 대한 포인터로, 각 Vertex 요소에 대한 정보를 전달해준다
        
    - `NumElements` → *2*
        
        : `pInputElementDescs` 배열에 있는 요소의 수, 즉 한 Vertex를 구성하는 요소의 수를 지정한다 
        
    - `pShaderBytecodeWithInputSignature` → *g_VSBlob→GetBufferPointer()*
        
        : 컴파일된 정점 셰이더의 바이트 코드에 대한 포인터로, 정점 셰이더가 입력으로 받을 데이터의 구조를 포함해야 한다
        
        즉, Vertex Shader Blob에 담긴 정보!
        
    - `BytecodeLength` → *g_VSBlob.GetBufferSize()*
        
        : `pShaderBytecodeWithInputSignature` 에서 가리키는 바이트 코드의 길이, 즉 Vertex Shader Blob에 담긴 정보!
        
    - `ppInputLayout` → &g_Layout
        
        : 생성된 입력 레이아웃 객체에 대한 포인터로, 이후 IA 단계에서 사용된다!
        

그럼, 이렇게 생성한 Layout 객체는 Temp의 Render 시 IA단계에서 활용되도록 설정해줘야 한다

```cpp
void TempRender()
{
	// ...
	CONTEXT->IASetInputLayout(g_Layout);
	
	// VS 설정
	// PS 설정
}
```

### 렌더링 파이프라인 시작하기

그럼 이제 렌더링 파이프라인을 시작할 준비가 모두 완료되었다!

context를 통해 `Draw` 함수(또는 `DrawIndexed` 함수)를 호출하면, 우리가 Temp의 Init과 Render 단계에서 설정한 그래픽 파이프라인과 데이터를 활용해 그래픽을 렌더링하기 위한 일련의 작업들이 수행되는 것이다! 

→ 이를 통해 GPU에서 그래픽을 그릴 준비가 되었다고 알려주고, GPU는 IA - VS - Rasterizer - PS - OM 등의 단계를 순차적으로 거쳐 원하는 이미지를 출력하게 된다

이때, 실행시킬 Vertex Shader의 개수를 인자로 같이 전달해줘야 한다

```cpp
void TempRender()
{
	UINT stride = sizeof(Vtx);
	UINT offset = 0;
	CONTEXT->IASetVertexBuffers(0, 1, &g_VB, &stride, &offset);
	CONTEXT->IASetPrimitiveTopology(D3D_PRIMITIVE_TOPOLOGY_TRIANGLELIST); // 3개의 점을 하나의 삼각형으로 해석
	CONTEXT->IASetInputLayout(g_Layout);

	CONTEXT->VSSetShader(g_VS, nullptr, 0);
	CONTEXT->PSSetShader(g_PS, nullptr, 0);

	CONTEXT->Draw(3, 0);       // 렌더링 파이프라인을 시작시키는 함수!
														 // 실행시킬 Vertex Shader의 개수를 인자로 전달
}
```

물론, 이렇게 DX API를 통해 생성한 객체들을 소멸자에서 Release 해주는 것을 잊지말기!

```cpp
void TempRelease()
{
	g_VB->Release();
	g_VSBlob->Release();
	g_VS->Release();

	g_PSBlob->Release();
	g_PS->Release();

	g_Layout->Release();

	if (nullptr != g_ErrBlob)
		g_ErrBlob->Release();
}
```