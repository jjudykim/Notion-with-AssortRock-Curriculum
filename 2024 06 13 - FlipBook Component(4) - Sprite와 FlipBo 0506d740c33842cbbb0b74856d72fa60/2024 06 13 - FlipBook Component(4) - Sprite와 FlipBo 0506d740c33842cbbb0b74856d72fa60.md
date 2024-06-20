# 24/06/13 - FlipBook Component(4) - Sprite와 FlipBook의 Save & Load, TileMap Component(1) - 개요

상위 항목: Week29 (Week29%208c950cb34a664d9ab3834144e1984a6f.md)
주차: 0011_Week20~29

> 1교시 녹음본
- [https://clovanote.naver.com/s/4A7fw8uNVtiY8gYQoTHryHS](https://clovanote.naver.com/s/4A7fw8uNVtiY8gYQoTHryHS)
2교시 녹음본
- [https://clovanote.naver.com/s/9LH4izstyw4vkrXTwKmhfFS](https://clovanote.naver.com/s/9LH4izstyw4vkrXTwKmhfFS)
> 

## Animation을 구성하는 Asset의 Save와 Load

우리가 FlipBook Component를 통해 애니메이션을 재생하기 위해서

두 Asset을 제작했다 → FlipBook과 이를 구성하는 Sprite

이제 두 에셋을 편집한 내용을 Save하고, 불러오는 Load 기능을 제작할 것이다

### Sprite의 Save & Load

Sprite를 Save하고 Load 해보자

```cpp
class CSprite :
	public CAsset
{
	// ...
public:
	virtual int Load(const wstring& _FilePath) override;
	virtual int Save(const wstring& _FilePath) override;
};
```

**Sprite의 Save 함수**

```cpp
int Sprite::Save(const wstring& _FilePath) override
{
	FILE* File = nullptr;
	_wfopen_s(&File, _FilePath.c_str(), L"wb");

	// 파일이 열리지 않을 경우 FAIL 처리
	if (File == nullptr)
		return E_FAIL;

	// 본인이 참조하고 있는 에셋(Atlas Texture) 정보를 저장
	SaveAssetRef(m_Atlas, File);

	// Sprite 정보를 저장
	fwrite(&m_LeftTopUV, sizeof(Vec2), 1, File);
	fwrite(&m_SliceUV, sizeof(Vec2), 1, File);
	fwrite(&m_BackgroundUV, sizeof(Vec2), 1, File);
	fwrite(&m_OffsetUV, siezof(Vec2), 1, File);
	
	fclose(File);

	return S_OK;
}
```

**Sprite의 Load 함수**

```cpp
int Sprite::Load(const wstring& _FilePath) override
{
	FILE* File = nullptr;
	_wfopen_s(&File, _FilePath.c_str(), L"rb");
	
	// 파일이 열리지 않을 경우 FAIL 처리
	if (File == nullptr)
		return E_FAIL;
	
	// 본인이 참조하고 있는 에셋(Atlas Texture)의 정보 불러오기
	LoadAssetRef(m_Atlas, File);
	
	// Sprite의 정보  
	fread(&m_LeftTopUV, sizeof(Vec2), 1, File);
	fread(&m_SliceUV, sizeof(Vec2), 1, File);
	fread(&m_BackgroundUV, sizeof(Vec2), 1, File);
	fread(&m_OffsetUV, sizeof(Vec2), 1, File);
	
	fclose(File);
	
	return S_OK;
}
```

이때, 우리는 본인(객체)가 참조하고 있는 에셋의 정보를 저장하거나 / 불러오는 함수를 활용하려고 한다

단지 Sprite에만 국한해서 Atlas Texture를 가져오는 것 뿐만 아니라,

FlipBook이 본인이 참조중인 Sprite들을 가져오는 등의.. 다양한 타입에서 활용될 것이기 때문에

본인이 사용하는 Asset을 참조해 가져올 수 있는 템플릿 함수로 작성해주자

- SaveAssetRef
- LoadAssetRef

**File에 Asset 참조 정보를 저장하는 SaveAssetRef 함수**

```cpp
template<typename T>
void SaveAssetRef(Ptr<T> Asset, FILE* _File)
{
	bool bAsset = Asset.Get();                   // 해당 Asset이 존재하는 지 확인
	fwrite(&bAsset, 1, 1, _File);                // Asset의 유무 저장
	
	if (bAsset)
	{
		SaveWString(Asset->GetKey(), _File);              // Asset의 Key 저장
		SaveWString(Asset->GetRelativePath(), _File);     // Asset의 Path 저장
	}
}
```

**File에 저장된 Asset 참조 정보를 불러오는 LoadAssetRef 함수**

```cpp
template<typename T>
void LoadAssetRef(Ptr<T> Asset, FILE* _File)
{
	bool bAsset = false;
	fread(&bAsset, 1, 1, _File);              // Asset의 유무를 확인
	
	if (bAsset)
	{
		wstring key, relativepath;
		LoadWString(key, _File);
		LoadWString(relativepath, _File);
	}
}
```

그런데 이때, wstring 객체 자체은 문자열을 저장하고 있는 힙 메모리 주소를 담고 있는 것이기 때문에,

직접 wstring 객체를 fwrite로 쓰게 되면, 객체의 내부 포인터나 메타데이터 등의 객체 정보가 저장된다

따라서, 문자열 데이터를 파일에 쓰기 위해선 wstring 객체가 가리키는 실제 문자열을 파일에 써야 한다!

→ 이를 함수로 따로 구현해, 전역으로 사용할 수 있도록 구현해놓자

**wstring을 파일에 쓰고, 읽는 함수 SaveWstring**

```cpp
// ...
void SaveWString(const wstring& _String, FILE* _File);
void LoadWString(wstring& _String, FILE* _File);
// ...
```

```cpp
void SaveWString(const wstring& _String, FILE* _File)
{
	size_t len = _String.length();
	fwrite(&len, sizeof(size_t), 1, _File);                  // wstring의 길이 저장
	fwrite(_String.c_str(), sizeof(wchar_t), len, _File);    // wstring의 길이만큼 문자열 저장
}

void LoadWstring(wstring& _String, FILE* _File)
{
	size_t len = 0;
	fread(&len, sizeof(size_t), 1, _File);
	
	_String.resize(len);                                    // wstring의 길이만큼 resize
	fread((wchar_t*)_String.c_str(), sizeof(wchar_t), len, _File);  // 읽어들인 문자열 저장
}
```

그런데 먼저 우리가 `CreateEngineSprite()` 함수를 통해 수동으로 만들었던 Sprite에는,

해당 Asset이 저장될 경로가 지정되어있지 않았기 때문에, 이 Sprite들에게 경로를 설정해줘야 한다

그 부분을 먼저 보완해주자!

**Sprite 생성 시 Asset의 경로 추가**

```cpp
void CAssetMgr::CreateEngineSprite()
{
	Ptr<CTexture> pAtlasTex = Load<CTexture>(L"texture\\link_32.png", L"Texture\\link.png");
	
	wstring strContentPath = CPathMgr::GetInst()->GetContentPath();
	Ptr<CSprite> pSprite = nullptr;
	
	for (int i = 0; i < 10; ++i)
	{
		wchar_t szKey[50] = {};
		swprintf_s(szKey, 50, L"Link_MoveDown_%d", i);

		pSprite = new CSprite;
		pSprite->Create(pAtlasTex, Vec2( (float)i * 120.f, 520.f), Vec2(120.f, 130.f));
		pSprite->SetBackground(Vec2(200.f, 200.f));
		
		AddAsset(szKey, pSprite);
		pSprite->SetRelativePath(wstring(L"Animation\\") + Buffer + L".sprite");
	}

	// ...	
}
```

### FlipBook의 Save & Load

이제 FlipBook도 저장해보자!

원하는 Sprite를 묶고 참조해, 이를 활용해 Animation을 출력할 수 있는 객체가 바로 FlipBook이다

따라서 참조하는 Sprite에 대한 정보와, FlipBook 자체에 대한 내용을 저장하고 불러올 수 있게 작성해보자

FlipBook을 Save하고 Load 해보자

```cpp
virtual int Save(wstring& _FilePath);
virtual int Load(wstring& _FilePath);
```

**FlipBook의 Save 함수**

```cpp
int CFlipBook::Save(wstring& _FilePath)
{
	FILE* File = nullptr;
	_wfopen_s(&File, _FilePath.c_str(), L"wb");

	// 파일이 열리지 않을 경우 FAIL 처리
	if (File == nullptr)
		return E_FAIL;

	// Sprite의 개수 저장
	size_t SpriteCount = m_vecSprite.size();
	fwrite(&SpriteCount, sizeof(size_t), 1, File);
	
	for(size_t i = 0; i < SpriteCount; ++i)
	{
		// 내가 참조하고 있는 Sprite들을 저장하기
		SaveAssetRef(m_vecSprite[i], File);
	}
	
	fclose(File);
	
	return S_OK;
}
```

**FlipBook의 Load 함수**

```cpp
int CFlipBook::Load(wstring& _FilePath)
{
	FILE* File = nullptr;
	_wfopen_s(&File, _FilePath.c_str(), L"rb");
	
	// 파일이 열리지 않을 경우 FAIL 처리
	if (File == nullptr)
		return E_FAIL;
		
	size_t SpriteCount = 0;
	fread(&SpriteCount, sizeof(size_t), 1, File);
	m_vecSprite.resize(SpriteCount);
	
	for (size_t i = 0; i < SpriteCount; ++i)
	{
		// 저장해놨던 Sprite들을 Load
		LoadAssetRef(m_vecSprite[i], File);
	}
	
	fclose(File);
	
	return S_OK;
}
```

FlipBook도 생성 시 경로를 추가해주고, Save 기능을 추가하자

```cpp
void CAssetMgr::CreateEngineSprite()
{
	// Sprite 생성, Sprite 저장
	// ...
	
	Ptr<CFlipBook> pFlipBook = nullptr;
	
	pFlipBook = new CFlipBook;
	
	for(int i = 0; i < 10; ++i)
	{
		wchar_t Buffer[50] = {};
		swprintf_s(Buffer, 50, L"Link_MoveDown_%d", i);
		pFlipBook->AddSprite(FindAsset<CSprite>(Buffer));
	}
	
	AddAsset(L"Link_MoveDown", pFlipBook);
	pPlipBook->SetRelativePath(L"Animation\\" + L"Link_MoveDown" + L".flip");
	
	pFlipBook->Save(StrContentPath + L"Animation\\" + L"Link_MoveDown" + L".flip");
}
```

그럼, Save과정을 거친 후에는 지정된 경로에 파일이 저장되어 있기 때문에 

CreateEngineSprite에서 수동으로 Sprite와 FlipBook을 만드는 과정을 생략하고 Flipbook을 Load 해와서 사용할 수 있다

```cpp
void CAssetMgr::CreateEngineSprite()
{
	wstring strContentPath = CPathMgr::GetInst()->GetContentPath();
	
	Ptr<CFlipBook> pFlipBook = new CFlipBook;
	Load<CFlipBook>(L"Link_MoveDown", L"Animation\\" + L"Link_MoveDown" + L".flip");
}
```

## TileMap Component

### TileMap Component의 개요

타일맵 컴포넌트를 제작하는 가장 큰 이유 중 하나 → 쉐이더 때문에!

Object에는 하나의 TileMap Component가 붙게 되는데,

이때 오브젝트를 어떤 크기의 n X n의 타일로 설정했느냐에 따라, 오브젝트 안에 있는 타일이 다수로 쪼개진다

그리고 이 말은 즉슨, n X n 짜리 타일맵을 하나의 Object로 취급하고 렌더링이 이루어진다는 것

→ 렌더링 최적화를 위해

그럴려면 고려해야하는 것이 여러가지가 생기는데..

- 하나의 오브젝트가 다수의 타일 조각을 합쳐서 렌더링하므로, 쉐이더적인 고민
- 타일의 정보를 렌더링할 때 넘겨야 하므로, 거대한 배열 형태의 데이터를 쉐이더로 전달해야
- 오브젝트 하나 내에 존재하는 TileMap의 행렬 수, 즉 Tile의 개수가 계속해서 달라짐
    
    → 전달될 데이터의 크기가 실시간으로 변동될 수 있다는 것인데…
    

<aside>
❗ 즉, 내가 쉐이더에게 데이터를 보낼 때,

상황에 따라서 가변적인 데이터가 있을 때에는 어떤 식으로 보내야 하는가?

</aside>

DX 9 정도에는 필요한 데이터가 가변적인 크기의 데이터였을 때, 텍스쳐를 활용했다고 한다!

텍스쳐 픽셀 하나의 포맷은, R32, G32, B32, A32로 즉 4바이트 플롯의 4개가 합쳐진 16바이트다

→ Vec4로 활용할 수 있다!

즉, 행렬 하나를 전달하고 싶다면, 픽셀 4개를 묶어서 전달하는 것 (float4 X 4니까!)

이런 식으로 텍스쳐를 통해 원하는 데이터를 담아서 보내곤 했었다

텍스쳐는 바인딩되는 사이즈가 매번 다르기 때문에, 이미 가변적인 데이터를 받고 있었다!

**Texcell을 활용**

: 텍스처에 단지 색상 정보가 아닌, 임의의 데이터를 저장

행렬 또는 벡터 데이터를 텍스처에 픽셀에 저장하고 쉐이더에서 읽어옴

텍스쳐를 통해 GPU에 가변적인 데이터를 보낼 수 있는 이유?

→ 레지스터 메모리가 속도 접근이 빠른 대신, 크기를 작게 갖고 있기 때문에

텍스쳐는 포인터 개념처럼, 

텍스쳐의 GPU와의 연동방식에 따라..

속도가 느리지만 참조 방식으로 보내기 때문에?

따라서 가변적인 상황에 대응할 수 있다

16바이트 단위로 떨어지도록 픽셀 단위로 나누고 묶어서 보내야 한다..?

DX11에서는 텍스쳐 레지스터를 이런 가변적인 데이터를 전송하는 기능을 채택해서 지원해주기 시작했다!

→ 구조화 버퍼 (StructuredBuffer)

```cpp
StructuredBuffer<Float4> g_Buffer : register(t11);
```

TileMap