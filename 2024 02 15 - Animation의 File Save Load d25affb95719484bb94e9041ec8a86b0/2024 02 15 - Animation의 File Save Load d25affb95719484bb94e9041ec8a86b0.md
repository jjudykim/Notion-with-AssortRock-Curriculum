# 2024/02/15 - Animation의 File Save/Load

태그: C++, WinAPI, 중급
날짜: 2024/02/15
상위 항목: Week12 (Week12%20c3bc54596db14ec19bbc3c480be90207.md)
주차: 0010_Week10~19

## Atlas를 구성하는 Sprite의 크기가 다른 경우

움직이는 이미지를 표현하는 Animation을 제작하면서, 한 Frame을 구성하는 Sprite의 크기가 서로 다를 때 위치가 맞지 않는 문제를 해결해보자!

### Anchor Points? Offset?

**Anchor Points(앵커 포인트)**

: 각 스프라이트 내에서 **고정된 위치(ex: 중앙, 하단 중앙 등)를 앵커 포인트로 설정**해, 모든 애니메이션 프레임에서 이 앵커 포인트를 기준으로 스프라이트를 정렬하면, 객체의 위치가 일관되게 유지될 수 있다고 한다!

**Offset(오프셋)**

: 기준 위치로부터의 상대적인 거리를 나타내는 값을 설정해, 스프라이트나 오브젝트의 위치를 조정하는 데 사용된다. 각각의 스프라이트에 대해서 기준점으로부터 얼마나 떨어져 있어야 하는지를 명시적으로 지정하기 때문에 다양한 크기와 형태의 스프라이트를 보다 세밀하게 제어할 수 있다!

따라서 둘 중 어느 방법을 선택하느냐는.. 프로젝트의 특성과 목표에 따라서 달라진다고 하는데, *앵커 포인트와 오프셋을 결합해서 사용하는 방식*을 사용하기도 한다고 한다! 

<aside>
✍️ 예를 들면

- 앵커 포인트 설정 : 모든 애니메이션 프레임에서 캐릭터의 발 위치를 앵커 포인트로 설정해, 캐릭터가 점프를 하더라도 발이 항상 동일한 지면 위치에 있게 된다
- 오프셋 적용 : 캐릭터의 암 펌핑이나 몸의 기울기 등 프레임마다 달라지는 상체의 움직임을 조정하기 위해 각 프레임에 오프셋 값을 적용한다. 보통 캐릭터의 머리와 상체가 프레임마다 조금씩 다른 위치에 있어야 할 때 유용하다
</aside>

일단 그렇다는 것만 알아두고, 이 프로젝트에서는 Offset을 사용해서 스프라이트의 위치를 조정해보자!

### Offset의 구현

1. **Offset을 표현할 멤버 추가**

offset에 대한 정보는 frame마다 적용해야 하는 것이므로, (frame마다 기준 위치로부터의 상대적인 거리를 지정해야 하기 때문에) Animation Frame 정보를 저장하는 구조체였던 `tAnimFrm`에 멤버를 추가해주도록 하자

```cpp
struct tAnimFrm
{
	Vec2    StartPos;
	Vec2    SliceSize;
	Vec2    Offset;
	float   Duration;
};
```

1. **Frame에 대한 Offset 설정**

Player가 갖고 있는 Animator에 Animation을 생성하고 추가하는 함수(`CreateAnimation()`)에서는, offset을 따로 작성하는 작업은 구현되지 않았다

따라서 `FindAnimation`을 통해 해당 Animation의, 해당 Frame에 접근해 Offset의 수치를 직접적으로 변경해주자

Animation의 Frame Index를 통한 프레임 접근을 위한 함수 구현

```cpp
tAnimFrm& GetFrame(int _Idx) { return m_vecFrm[_Idx]; }
```

Animation의 프레임에 접근해 Offset값을 변경

```cpp
m_Animator->FindAnimation(L"IDLE_RIGHT")->GetFrame(1).Offset = Vec2(1.f, -50.f);
```

1. **render할 위치에 Offset을 적용**

그럼 이 변경된 위치 값이 render 시에 적용이 되도록 객체의 pos에 offset이 적용되도록 변경해줘야 한다

```cpp
void CAnimation::render()
{
	// ...
	TransparentBlt(DC,
								 vPos.x - frm.SliceSize.x / 2.f + frm.Offset.x,
								 vPos.y - frm.SliceSize.y / 2.f + frm.Offset.y,
								 frm.SliceSize.x, frm.SliceSize.y,
								 m_Atlas->GetDC(),
								 frm.StartPos.x, frm.StartPos.y,
								 frm.SliceSize.x, frm.SliceSize.y,
								 RGB(255, 0, 255));
}
```

## Animation에 대한 정보를 저장하고, 로드해서 사용하자!

Animation Data를 효율적으로 관리하고 재사용할 수 있도록 Animation을 구성하는 정보들을 File로 저장하고, 로드해 불러오는 방식으로 사용할 것이다! 그렇다면 File에는 어떤 내용들이 담겨야하고, 어떻게 저장하고, 어떻게 불러올 수 있을까?

### File에 Animator의 어떤 주요한 정보들을 저장해야 할까?

- `m_vecFrm` : Frame을 구성하는 정보들, 즉 메타데이터가 저장되어야 프레임마다 필요한 정보를 load해 사용하기 용이하다!
- `m_Atlas` : 사용되는 Texture Atlas에 대한 정보를 담고 있어야 Frame 정보에 따라 Sprite를 추출하기에 용이하다

<aside>
✍️ **“*메타데이터*”**란 용어에 대해 짚고 넘어가보자

메타데이터(Metadata)는 데이터에 대한 데이터로 정의될 수 있다! 즉, 다른 데이터를 설명하거나 분류, 관리하기 위해 사용되는 정보의 집합으로.. 보통 데이터의 구조나 속성, 특징 등을 설명하는 역할을 담당하기 때문에 데이터를 이해하고, 사용하고, 관리하는 데에 용이하게 사용이 된다.

여기서 프레임 정보를 담고 있는 `tAnimFrm`도 하나의 메타데이터라고 볼 수 있는데, 해당 프레임에 사용되는 스프라이트의 위치, 크기, 오프셋, 생명 주기 등을 담고 있어 각 스프라이트의 특성을 쉽게 이해하고, 프로그램 내에서 효율적으로 사용할 수 있다!

이처럼 메타데이터는 데이터를 효과적으로 관리하고 검색을 가능하게 만들며, 특히 대용량의 데이터를 다루는 경우에 필수적인 요소가 된다! 따라서 데이터의 효과적인 사용과 장기적인 보존을 위해서 메타데이터를 정의하고 관리하는 것에 주의를 기울여야 한다~

</aside>

### 파일을 생성하고, 쓰고, 저장해보자! Save 함수 제작하기

1. **디렉토리 생성**

먼저, 디렉토리를 생성해 우리가 생성한 파일에 대한 경로를 지정해줄 것이다

우리는 지금 Animation에 관련된 내용 중, Player를 관리하고 있으므로

<aside>
📁 content > animation > player

</aside>

의 경로를 가지는 디렉토리를 먼저 생성해주자!

1. **Save 함수의 틀 생성**

```cpp
void Save(const wstring& _strRelativeFolderPath);
```

→ 파일이 저장될 폴더의 **상대경로**(content부터 시작되는 경로)를 인자로 받아, 해당 공간에 저장하는 함수를 구현할 것이다!

1. **Save 함수의 구현**
    
    1️⃣ 파일의 경로 및 이름 지정
    
    ```cpp
    void CAnimation::Save(const wstring& _strRelativeFolderPath)
    {
    	// 디렉토리 경로 설정
    	wstring strFilePath = CPathMgr::GetInst()->GetContentPath();
    	strFilePath += _strRelativeFolderPath;
    
    	// 파일 이름(경로 설정의 마무리) + 확장자 설정
    	strFilePath += GetName();       // 해당 애니메이션 이름을 파일명으로 설정!
    	strFilePath += L".anim";        // .anim이라는 우리만의 확장자를 사용하자
    }
    ```
    
    2️⃣ 파일을 생성하고, 파일을 “쓰기 모드”로 열어보자!
    
    ```cpp
    void CAnimation::Save(const wstring& _strRelativeFolderPath)
    {
    	// 디렉토리 경로 설정
    	// 파일 이름 + 확장자 설정
    	
    	// ⭐파일 열기⭐
    	FILE* pFile = nullptr;
    	_wfopen_s(&pFile, strFilePath.c_str(), L"wb");
    
    	// pFile이 nullptr일 경우의 예외 처리도 해주자
    	if (pFile == nullptr)
    	{
    		LOG(LOG_TYPE::DBG_ERROR, L"애니메이션 저장 실패");
    		return;
    	}
    
    	// ⭐파일 닫기⭐
    	fclose(pFile);
    }
    ```
    
    - `FILE*` : C style의 표준 입출력 라이브러리인 `<stdio.h>`의 파일 스트림을 위한 포인터 타입! → 파일 생성 및 데이터 입력에 주로 *fopen*, *fprintf*, *fputs*, *fwrite* 등이 사용된다
    - `_wfopen_s` : 파일을 안전하게 열기 위해 사용되는 함수, 그 중 파일 이름과 파일 모드를 나타내는 문자열이 Wide 문자열인 경우 사용!
        - 프로토타입
            
            ```cpp
            errno_t _wfopen_s(
            	FILE** pFile,                // 포인터의 포인터 형태!             
            	const wchar_t *filename,
            	const wchar_t *mode
            );
            ```
            
        - `FILE** pFile` : 열린 파일에 대한 포인터를 저장할 FILE 객체의 포인터의 주소
        - `const wchar_t *filename` : 열고자 하는 파일의 이름
        - `const char *mode` : 파일 열기 모드를 지정하는 문자열 (ex. *”wb”* : 쓰기(바이너리))
        
        → 그럼 우리가 입력한 인자에 대해서 해석해보면,
        
        - *&pFile* : 우리가 생성한 FILE 객체의 포인터인 pFile의 주소를 전송했다
        - *strFilePath.c_str()* : 우리가 지정한 파일 경로 + 확장자를 전달해 파일을 open!
        - *L(”wb”)* : **쓰기** 모드로 파일을 open, 그런데 **b**가 붙으면서 *바이너리 모드*도 적용!
    - `fclose` : 열린 파일 스트림을 닫는데 사용되는 함수로, 열려있는 ***FILE**** 스트림에 대한 모든 버퍼를 비우고 (즉, 버퍼에 저장된 데이터를 실제 파일에 작성한다) 해당 스트림을 닫는다!

3️⃣ 애니메이션의 정보 저장하기

```cpp
void CAnimation::Save(const wstring& _strRelativeFolderPath)
{
	// 디렉토리 경로 설정
	// 파일 이름 + 확장자 설정
	// 파일 열기
	
	// 애니메이션의 정보 저장하기 =======================
	// [1] Frame에 대한 정보들을 저장
	// 1-1. 프레임 개수 쓰기
	size_t FrmCount = m_vecFrm.size();
	fwrite(&FrmCount, sizeof(size_t), 1, pFile);
	
	// 1-2. 각각의 프레임 정보를 쓰기
	for (size_t i = 0; i < m_vecFrm.size(); ++i)
	{
		fwrite(&m_vecFrm[i], sizeof(tAnimFrm, 1, pFile);
	}

	// [2] Texture Atlas 정보를 저장
	bool bAtlasTex = false;
	if (m_Atlas != nullptr)
		bAtlasTex = true;

	// Texture Atlas의 존재 유무를 쓰기
	fwrite(&bAtlasTex, sizeof(bool), 1, pFile);

	if (bAtlasTex)
	{
		wstring strKey = m_Atlas->GetKey();
		size_t len = strKey.length();
		fwrite(&len, sizeof(size_t), 1, pFile);              // 'Key의 길이' 쓰기
		fwrite(strKey.c_str(), sizeof(wchar_t), len, pFile); // 'Key' 쓰기

		wstring strRelativePath = m_Atlas->GetRelativePath();
		len = strRelativePath.length();
		fwrite(&len, sizeof(size_t), 1, pFile);              // 'Path의 길이' 쓰기
		fwrite(strRelativePath.c_str(), sizeof(wchar_t), len, pFile); // 'Path'쓰기
	}
	
	// 파일 닫기
}
```

- `fwrite` : 바이너리 데이터를 파일에 쓰는 데 사용되는 함수로, 지정된 크기의 여러 개의 데이터 요소를 파일 스트림에 쓴다!
    - 프로토타입
        
        ```cpp
        size_t fwrite(const void* ptr, size_t size, size_t nmemb, FILE *stream);
        ```
        
    - `const void* ptr` : 쓰기를 수행할 데이터를 가리키는 포인터, void* 타입이니 어떤 타입이든 가능
    - `size_t size` : 각 데이터 요소의 크기
    - `size_t nmemb` : 파일에 쓰고자 하는 데이터 요소의 개수
    - `FILE* stream` : 데이터를 쓰기 위한 파일 스트림의 ‘*파일 포인터*’

- `CAnimation::Save()` 전체보기
    
    ```cpp
    void CAnimation::Save(const wstring& _strRelativeFolderPath)
    {
    	wstring strFilePath = CPathMgr::GetInst()->GetContentPath();
    	strFilePath += _strRelativeFolderPath;
    	strFilePath += GetName();
    	strFilePath += L".anim";
    
    	FILE* pFile = nullptr;
    	_wfopen_s(&pFile, strFilePath.c_str(), L"wb");
    
    	if (pFile == nullptr)
    	{
    		LOG(LOG_TYPE::DBG_ERROR, L"애니메이션 저장 실패");
    		return;
    	}
    
    	size_t FrmCount = m_vecFrm.size();
    	fwrite(&FrmCount, sizeof(size_t), 1, pFile);
    
    	for (size_t i = 0; i < m_VecFrm.size(); ++i)
    	{
    		fwrite(&m_vecFrm[i], sizeof(tAnimFrm), 1, pFile);
    	}
    
    	bool bAtlasTex = false;
    	if (m_Atlas != nullptr)
    		bAtlasTex = true;
    
    	fwrite(&bAtlasTex, sizeof(bool), 1, pFile);
    
    	if (bAtlasTex)
    	{
    		wstring strKey = m_Atlas->GetKey();
    		size_t len = strKey.length();
    		fwrite(&len, sizeof(size_t), 1, pFile);
    		fwrite(strKey.c_str(), sizeof(wchar_t), len, pFile);
    	
    		wstring strRelativePath = m_Atlas->GetRelativePath();
    		len = strRelativePath.length();
    		fwrite(&len, sizeof(size_t), 1, pFile);
    		fwrite(strRelativePath.c_str(), sizeof(wchar_t), len, pFile);
    	}
    	fclost(pFile);
    }
    ```
    

현재까지의 단계에서 정상적으로 Animation에 대한 File이 생성되었는지 확인하기 위해 Player에서 Save를 호출해보자!

```cpp
CPlayer::CPlayer()
{
	// ...
	m_Animator->FindAnimation(L"WALK_DOWN")->Save(L"animation\\player\\");
}
```

![Untitled](2024%2002%2015%20-%20Animation%E1%84%8B%E1%85%B4%20File%20Save%20Load%20d25affb95719484bb94e9041ec8a86b0/Untitled.png)

잘 생성되었다! 신기신기 🤩

내용물을 메모장으로 열어보면…

![Untitled](2024%2002%2015%20-%20Animation%E1%84%8B%E1%85%B4%20File%20Save%20Load%20d25affb95719484bb94e9041ec8a86b0/Untitled%201.png)

이렇게 뭔가 괴랄하게 적혀있는데, 당연하긴 하다! 바이너리 코드를 억지로 문자열로 읽으려고 했기 때문에, 메모장에서는 이렇게 나오지만.. 

Load를 구현함으로써 이 데이터들을 활용해 Animation의 정보로 활용할 수 있을 것이다!

### 저장한 파일을 열고 읽어들여보자! Load 함수 제작하기

1. **Load 함수의 틀 생성**
    
    ```cpp
    int CAnimation::Load(const wstring& _strRelativeFilePath)
    ```
    
    → 파일이 저장되어 있는 폴더의 **상대경로**(content부터 시작되는 경로)를 인자로 받아, 해당 공간에서 불러오는 함수를 작성할 것이다
    
    - 성공/실패 값을 받아오기 위해 반환 타입을 int로 설정 S_OK, E_FAIL

1. **Load 함수의 구현**
    
    1️⃣ 파일의 경로 및 이름 지정
    
    ```cpp
    void CAnimation::Save(const wstring& _strRelativeFolderPath)
    {
    	// 디렉토리 경로 설정
    	wstring strFilePath = CPathMgr::GetInst()->GetContentPath();
    	strFilePath += _strRelativeFolderPath;
    }
    ```
    
    2️⃣ 파일을 생성하고, 파일을 “읽기 모드”로 열어보자!
    
    ```cpp
    void CAnimation::Save(const wstring& _strRelativeFolderPath)
    {
    	// 디렉토리 경로 설정
    	
    	// ⭐파일 열기⭐
    	FILE* pFile = nullptr;
    	_wfopen_s(&pFile, strFilePath.c_str(), L"rb");
    
    	// pFile이 nullptr일 경우의 예외 처리도 해주자
    	if (pFile == nullptr)
    	{
    		return E_FAIL;
    	}
    
    	// 애니메이션 정보 읽기
    
    	// ⭐파일 닫기⭐
    	fclose(pFile);
    	return S_OK;
    }
    ```
    
    - *L(”rb”)* : **읽기** 모드로 파일을 open, 그런데 **b**가 붙으면서 *바이너리 모드*도 적용!

3️⃣ 애니메이션의 정보 저장하기

```cpp
void CAnimation::Save(const wstring& _strRelativeFolderPath)
{
	// 디렉토리 경로 설정
	// 파일 열기
	
	// 애니메이션의 정보 불러와 읽기 =======================
	// [1] Frame에 대한 정보들을 불러오기
	// 1-1. 프레임 개수 읽기
	size_t FrmCount = 0;
	fread(&FrmCount, sizeof(size_t), 1, pFile);
	
	// 1-2. 각각의 프레임 정보를 읽기
	for (size_t i = 0; i < FrmCount; ++i)
	{
		tAnimFrm frm{};
		fread(&frm, sizeof(tAnimFrm), 1, pFile);
		m_vecFrm.push_back(frm);
	}

	// [2] Texture Atlas 정보를 저장
	bool bAtlasTex = false;
	fread(&bAtlasTex, sizeof(bool), 1, pFile);

	if (bAtlasTex)
	{
		wstring strKey;
		wstring strRelativePath;

		wchar_t buff[256] = {};

		size_t len = 0;
		fread(&len, sizeof(size_t), 1, pFile);      // 'Key의 길이' 읽기
		fread(buff, sizeof(wchar_t), len, pFile);   // 'Key' 읽기
		strKey = buff;

		wmemset(buff, 0, 256);                      // 버퍼 초기화

		fread(&len, sizeof(size_t), 1, pFile);      // 'Path의 길이' 읽기
		fread(buff, sizeof(whcar_t), len, pFile);   // 'Path' 읽기
		strRelativePath = buff;

		// Key, Path로 Texture 가져와 저장하기
		m_Atlas = CAssetMgr::GetInst()->LoadTexture(strKey, strRelativePath);
	}

	// 파일 닫기
}
```

- `fread` : 파일로부터 바이너리 데이터를 읽는 데 사용되는 함수로, 지정된 크기의 여러 개의 데이터 요소를 파일 스트림으로부터 읽어와서, 저장된 버퍼에 저장한다
    - 프로토타입
    
    ```cpp
    size_t fread(void *ptr, size_t size, size_t nmemb, FILE *stream);
    ```
    
    - `void* ptr` : 읽어온 데이터를 저장할 버퍼 가리키는 포인터, void* 타입이니 어떤 타입이든 가능
    - `size_t size` : 각 데이터 요소의 크기
    - `size_t nmemb` : 파일로부터 읽고자 하는 데이터 요소의 개수
    - `FILE* stream` : 데이터를 쓰기 위한 파일 스트림의 ‘*파일 포인터*’
    
    → fread 함수는 성공적으로 읽혀진 데이터 요소의 개수를 반환한다!
    

1. **Animator에 `LoadAnimation()`이라는 함수를 추가해 불러온 애니메이션을 추가하자**
    
    ```cpp
    void CAnimator::LoadAnimation(const wstring& _AnimName, 
    															const wstring& _strRelativeFilePath)
    {
    	m_CurAnim = FindAnimation(_AnimName);
    	if (m_CurAnim != nullptr)
    	{
    		LOG(LOG_TYPE::DBG_ERROR, L"중복된 애니메이션 이름");
    		return;
    	}
    
    	// 애니메이션을 만들어서 지정된 경로로부터 로딩을 진행
    	CAnimation* pNewAnim = new CAnimation;
    	if (FAILED(pNewAnim->Load(_strRelativeFilePath)))
    	{
    		delete pNewAnim;
    		LOG(LOG_TYPE::DBG_ERROR, L"애니메이션 로딩 실패");
    		return;
    	}
    
    	pNewAnim->SetName(_AnimName);
    	pNewANim->m_Animator = this;
    	m_mapAnim.insert(make_pair(_AnimName, pNewAnim));
    }
    ```
    
    → 그럼 이렇게 사용이 가능하다!
    
    ```cpp
    CPlayer::CPlayer()
    {
    	// ...
    	// 불러오기
    	m_Animator->LoadAnimation(L"IDLE_DOWN", L"animation\\player\\IDLEDOWN.anim");
    }
    ```
    

1. Animator를 통해 Play해보자 (+ 예외처리)
    
    ```cpp
    void CAnimator::Play(const wstring& _AnimName, bool _Repeat)
    {
    	m_CurAnim = FindAnimation(_AnimName);
    	
    	// 찾은 애니메이션이 없는 경우의 예외처리!
    	if (m_CurAnim == nullptr)
    	{
    		LOG(LOG_TYPE::DBG_ERROR, L"Play할 애니메이션을 찾을 수 없음");
    		return;
    	}
    
    	m_CurAnim->Reset();
    	m_Repeat = _Repeat;
    }
    ```