# 2024/03/08 - Tile, TileMap 제작, Asset을 활용한 Tile 출력, Tile Click

태그: C++, WinAPI, 중급
날짜: 2024/03/08
상위 항목: Week15 (Week15%204207aa697c6145e0959981e524666a76.md)
주차: 0010_Week10~19

## Tile을 구성하는 Component, TileMap 제작하기

한 Tile, 즉 정사각형의 크기를 갖는 한 이미지를 Object로 만들게 된다면?

→ 한 Level에서 관리되는 Object가 무수히 증가하게 되고, Object에서 실행되는 tick()에 대한 부담이 엄청나게 커질 것이다!

그럼 각각 타일 하나를 Component로 만들어, 해당 Component의 조합을 가지는 Tile 전체를 하나의 Object로 관리한다면 어떨까?

이를 구현할 수 있는 ***Tile Object***와 ***TileMap Component***를 작성해보자

**타일 하나를 구성하는 Component, CTileMap의 기본적인 틀 작성**

```cpp
class CTileMap
	: public CComponent
{
private:
	UINT         m_Row            // 행
	UINT         m_Col;           // 열
	
	Vec2         m_TileSize;      // 타일의 개당 사이즈
	CTexture*    m_AtlasTex;      // 타일이 사용하는 아틀라스 텍스쳐

public:
	virtual void finaltick() override;
	void render();
		
	CLONE(CTileMap);
	
	CTileMap();
	//CTileMap(CTileMap _Other);  // 복사 생성자가 필요할지에 대해선 아직 모르니 주석!
	~CTileMap();	
}
```

```cpp
CTileMap::CTileMap()
	: m_Row(1)
	, m_Col(1)
	, m_TileSize(Vec2(TILE, TILE));
{
}

CTileMap::~CTileMap()
{
}
```

**CTileMap을 Component로 갖는 CTile Object 작성**

```cpp
class CTile
	: public CObj
{
private:
	CTileMap*    m_TileMap;             // 본인이 갖고 있는 TileMap Component

private:
	virtual void begin() override;
	virtual void tick() override;
	virtual void render() override;
	
public:
	CLONE(CTile);
	
public:
	CTile();
	// CTile(const CTile& _Other);
	~CTile();
};
```

```cpp
CTile::CTile()
{
	// 자신의 컴포넌트로 TileMap 추가
	m_TileMap = (CTileMap*)AddComponent(new CTileMap);
}

CTile::~CTile()
{
}

void CTile::begin()
{
}

void CTile::tick()
{
}

void CTile::render()
{
	// render에 대한 과정은 자신의 컴포넌트인 TileMap에게 일임!
	m_TileMap->render();
}
```

## TileMap을 통해 원하는 크기의 Tile을 출력해보자

### 1. Tile의 크기를 결정하는 Row, Col 세팅하기

먼저 한 Tile의 크기가 얼마나 될 지를 결정하는 것은, 

→ TileMap의 Row와 Col이 각각 몇 칸을 갖게 될 지에 대한 내용과 같다!

따라서 Tile을 통해 TileMap의 Row와 Col을 설정함으로써 TileMap의 크기를 설정할 수 있다

```cpp
class CTile
	: public CObj
{
private:
	// ...
public:
	void SetRow(UINT _Row);
	void SetCol(UINT _Col);
	void SetRowCol(UINT _Row, UINT _Col);
	// ...
}
```

```cpp
// ...
void CTile::SetRow(UINT _Row)
{
	m_TileMap->SetRow(_Row);
}

void CTile::SetCol(UINT _Col)
{
	m_TileMap->SetCol(_Col);
}

void CTile::SetRowCol(UINT _Row, UINT _Col)
{
	m_TileMap->SetRowCol(_Row, _Col);
}
// ...
```

→ 일단 이렇게 Tile 단위에서 TileMap을 통해 Row, Col을 설정할 수 있도록 작성해놨으니, TileMap 단위에서의 구현은 나중에 하도록 하자!

<aside>
🗒️ TODO : CTileMap에서 `SetRow`, `SetCol`, `SetRowCol` 함수 작성하기

</aside>

### 2. Editor Level에서 원하는 크기의 Tile 객체를 만들자

우리가 원하는 것은 

→ Editor Level로 Level 전환이 이루어졌을 때 **원하는 크기의 Tile 객체**가 생기는 것!

따라서 Editor Level 진입 시 (`Enter()`) Tile 객체를 생성해 현재 Level의 Object로 추가해보자

**Editor Level에 Tile 객체 생성**

```cpp
void CLevel_Editor::Enter()
{
	CTile* pTile = new CTile;
	pTile->SetRowCol(3, 3);            // Size를 3 X 3으로 설정해줬다

	AddObject(LAYER_TYPE::TILE, pTile);
}
```

### 3. 만든 Tile Object를 Grid 형태로 출력해보자!

아직까지 Tile의 AtlasTex를 직접 호출하진 말고, 그 경계만 표현할 수 있는 **Green Grid**를 그려보자!

Tile Object를 render하기 위해서는 TileMap 단위에서의 render를 작성해줘야 한다. 

→ 우리가 Tile의 render에서 해당 Tile이 가진 TileMap의 render가 호출되도록 일임했으니까

```cpp
void CTileMap::render()
{
	// 본인을 소유하고 있는 오브젝트의 위치를 그대로 반영해주자
	// 그런데 이제 render 중이니까 **renderPos**로 반영!!
	Vec2 vRenderPos = GetOwner()->GetRenderPos();
	
	USE_PEN(DC, PEN_TYPE::PEN_GREEN);
	USE_BRUSH(DC, BRUSH_TYPE::BRUSH_HOLLOW);
	
	// 한 타일의 시작 위치는 좌상단이 기준인 걸 생각하기!
	for (UINT i = 0; i < m_Row; ++i)
	{
		for (UINT j = 0; j < m_Col; ++j)
		{
			Rectangle(DC, (int)(vRenderPos.x + m_TileSize.x * j) // 현재 행 Tile의 x 시작 좌표
									, (int)(vRenderPos.y + m_TileSize.y * i) // 현재 열 Tile의 y 시작 좌표
									, (int)(vRenderPos.x + m_TileSize.x * j + m_TileSize.x) // 현재 행 Tile의 x 마무리 좌표
									, (int)(vRenderPos.y + m_TileSize.y * i + m_TileSize.y) // 현재 열 Tile의 y 마무리 좌표
		}
	}
}

```

<aside>
🗒️ **Rectangle의 좌표 계산 한눈에 이해하기!**

→ 만약 현재 반복문에서 그려지는 TileMap이 3행 2열의 타일이었다면?

![Untitled](2024%2003%2008%20-%20Tile,%20TileMap%20%E1%84%8C%E1%85%A6%E1%84%8C%E1%85%A1%E1%86%A8,%20Asset%E1%84%8B%E1%85%B3%E1%86%AF%20%E1%84%92%E1%85%AA%E1%86%AF%E1%84%8B%E1%85%AD%E1%86%BC%E1%84%92%201a77a0c6bd3440afa2eeca5d924dd7a6/Untitled.png)

vRenderPos가 (0, 0)이라고 가정했을 때, 

x 좌표를 결정하는 것은 열이기 때문에 → 시작좌표는 `m_TileSize.x` * (현재 Col - 1)

y 좌표를 결정하는 것은 행이기 때문에 → 시작 y좌표는 `m_TileSize.y` * (현재 Row - 1)

⇒ `Rectangel(DC, 64, 128, 128, 192);`

</aside>

<aside>
💡 나중에 최적화가 필요하다면, 현재 Screen, 즉 화면 상에 나오는 부분만 render 되도록 최적화 할 수도 있을 것이다!

</aside>

그렇다면 이 render 과정이 원활하게 이뤄지기 위해서는,

아까 미뤄뒀던 TileMap에서의 Row, Col의 setting이 이루어져야 할 것이다! 구현해보자

**TileMap의 Row, Col Setter 작성**

```cpp
class CTileMap
	: public CComponent
{
//...
public:
	void SetRow(UINT _Row);
	void SetCol(UINT _Col);
	void SetRowCol(UINT _Row, UINT _Col);
// ...
}
```

```cpp
// ...
void CTileMap::SetRow(UINT _Row)
{
	m_Row = _Row;
}

void CTileMap::SetCol(UINT _Col)
{
	m_Col = _Col;
}

void CTileMap::SetRowCol(UINT _Row, UINT _Col)
{
	m_Row = _Row;
	m_Col = _Col;
}
```

### 4. render를 세분화해 Atlas Texture를 이용해 Tile을 출력하기

현재는 TileMap이 Row, Col 칸의 개수만큼 Green Grid 형태로 출력되고 있지만,

render를 세분화해 Atlas Texture를 이용해 출력하는 기능과, Green Grid도 출력하는 기능으로 나눠볼 것이다!

### 4 - 1) Tile로 그릴 Atlas Texture 세팅하기

먼저, Tile을 통해서 TimeMap이 사용할 AtlasTex를 세팅할 수 있어야 하므로, 

**CTile에서 AtlasTex를 세팅하는 함수 추가**

```cpp
class CTile
	: public CObj
{
// ...
public:
	void SetAtlasTex(CTexture* _Atlas);
// ...
}
```

```cpp
// ...
void CTile::SetAtlasTex(CTexture* _Atlas)
{
	m_TileMap->SetAtlasTex(_Atlas);
}
// ...
```

**마찬가지로, CTileMap에서 AtlasTex를 세팅하는 함수 추가**

```cpp
class CTileMap
	: public CComponent
{
	
}
```

그럼 Tile Object를 갖고 있는 CLevel_Editor에서 해당 Tile의 AtlasTex를 설정할 수 있게 된다

```cpp
void CLevel_Editor::Enter()
{
	CTile* pTile = new CTile;
	pTile->SetRowCol(10, 10);
	
	pTile->SetAtlasTex(CAssetMgr::GetInst()->LoadTexture(L"texture\\TILE.bmp", 
																											 L"texture\\TILE.bmp"));
}
```

그럼 이제 CTileMap의 render의 세분화에 대해서 얘기해보자!

우리가 방금까지 render를 통해 출력했던 과정을 → `render_grid()`로,

불러온 AtlasTextrue를 통해 타일을 출력하는 과정을 → `render_tile()`로 나눠보자

또, `render_grid()`는 DbgRender를 통해서 Debug Render의 출력 여부에 따라 출력이 되도록 작성할 것이다

이렇게!!

```cpp
void CTileMap::render()
{
	render_tile();
	
	if (CDbgRender::GetInst()->IsDbgRender())
	{
		render_grid();
	}
}
```

### 4 - 2) render_tile 작성하기

**render_tile() : 실제 타일 이미지를 렌더링**

```cpp
void CTileMap::render_tile()
{
	if (m_AtlasTex == nullptr)
		return;
		
	Vec2 vRenderPos = GetOwner()->GetRenderPos();
	
	for(UINT i = 0; i < m_Row; ++i)
	{
		for (UINT j = 0; j < m_Col; ++j)
		{
			BitBlt(DC, (int)(vRenderPos.x + m_TileSize.x * j)
							 , (int)(vRenderPos.y + m_TileSize.y * i)
							 , m_TileSize.x, m_TileSize.y
							 , m_AtlasTex->GetDC(), 0, 0, SRCCOPY);
		}
	}
}
```

 `render_grid()`와 비슷하게 각 타일을 BitBlt 함수를 통해 Atlas Texture의 첫번째 그림을 출력했다!

그런데 이렇게 해버리면 모두 다 똑같은 타일만 출력되버리네…🤔

⇒ 이를 해결하기 위해서는 **각각의 타일은 정보를 갖고, 해당 타일 정보에 따라서 render가 되도록** 해야한다! 

따라서 `TileInfo`라는 구조체 정의하고 → TileMap을 구성하는 타일 하나에 대한 정보 담기

그럼 어떤 정보가 담겨야 할지 고민해보면, 해당 Atlas Texture에서 타일이 갖게 되는 이미지의 순서를 알아야 하므로! 이를 Index 값으로 저장해 가지면 될 것이다

단, 이렇게 Tile에 Index를 부여하는 경우에는 Atlas Texture 자체가 적절하게 편집되어 있어야 한다. 

![Untitled](2024%2003%2008%20-%20Tile,%20TileMap%20%E1%84%8C%E1%85%A6%E1%84%8C%E1%85%A1%E1%86%A8,%20Asset%E1%84%8B%E1%85%B3%E1%86%AF%20%E1%84%92%E1%85%AA%E1%86%AF%E1%84%8B%E1%85%AD%E1%86%BC%E1%84%92%201a77a0c6bd3440afa2eeca5d924dd7a6/Untitled.bmp)

요런식으로…

**따라서 TileInfo 구조체를 정의해보면,**

```cpp
struct tTileInfo
{
	int ImgIdx;            // 각 Atals Texture의 한 타일마다 Index를 부여!
												 // 즉, 해당 Atlas에서 첫번째 이미지는 0의 index를 갖는다
	tTileInfo()
		: ImgIdx(0)          // 모든 TileInfo의 index를 0으로 초기화할 수 있다
	{}
};
```

이렇게 각각 타일의 정보가 담긴 TileInfo 구조체의 집합을, 한 TileMap이 갖게 된다

```cpp
class CTileMap
	: public CComponent
{
private:
	// ...
	vector<tTileInfo>  m_vecTileInfo;
}
```

그럼 각각의 타일에 대한 정보는, *TileMap의 Row * Col 수만큼 존재*해야 하므로 Row와 Col을 Set하는 시점에서 **해당 정보를 담는 vector도 해당 크기만큼 resize** 해준다

```cpp
void CTileMap::SetRow(UINT _Row)
{
	m_Row = _Row;
	m_vecTileInfo.clear();
	m_vecTileInfo.resize(m_Row * m_Col);
}

void CTileMap::SetCol(UINT _Col)
{
	m_Col = m_Col;
	m_vecTileInfo.clear();
	m_vecTileInfo.resize(m_Row * m_Col);
}

void CTileMap::SetRowCol(UINT _Row, UINT _Col)
{
	m_Row = _Row;
	m_Col = _Col;
	
	m_vecTileInfo.clear();
	m_vecTileInfo.resize(m_Row * m_Col);
}
```

<aside>
✍️ **vector의 resize와 reserve 차이점**

resize와 reserve 함수는 둘 다 벡터의 크기를 조정하는데 사용되지만, 큰 차이점이 있다!

- `resize()`
: resize(n)은 벡터의 크기를 n으로 변경한다! 만약 n이 현재 벡터의 크기보다 크다면, 벡터의 끝에 새로운 요소들이 추가되고, 이 요소들은 기본 값으로 초기화된다. 반면 n이 현재 벡터의 크기보다 작다면, 벡터의 끝에서 요소들이 제거되어 크기가 n이 된다.

- `reserve()`
: reserve(n)은 벡터의 용량(capacity)를 미리 확보하는데 사용되는 것으로, (*여기서 용량은 벡터가 저장할 수 있는 요소의 수를 말한다*) n이 현재 벡터의 용량보다 큰 경우에만, 메모리가 재할당되어서 벡터의 용량이 n으로 증가한다. 용량만 늘어날 뿐, 벡터의 실제 크기는 변경되지 않는다. 요소의 개수는 변하지 않는다는 뜻!

</aside>

그럼 이제 TileInfo Vector를 통해서 각 타일의 이미지 인덱스를 통해 출력하는 `render_tile()`을 구성할 수 있을 것이다!

이때 TileInfo Vector에 접근해야 할 **Row * Col**개의 인덱스를 정확하게 알아야 하는데… 

우리는 각 타일을 Row, Col개념의 2차원 개념으로 생각하고 있지만 vector는 1차원 개념이다!

따라서 2차원의 반복을 1차원으로 변환해 연산하는 방식이 필요하다

마치 이 그림처럼!!

![Untitled](2024%2003%2008%20-%20Tile,%20TileMap%20%E1%84%8C%E1%85%A6%E1%84%8C%E1%85%A1%E1%86%A8,%20Asset%E1%84%8B%E1%85%B3%E1%86%AF%20%E1%84%92%E1%85%AA%E1%86%AF%E1%84%8B%E1%85%AD%E1%86%BC%E1%84%92%201a77a0c6bd3440afa2eeca5d924dd7a6/Untitled%201.png)

→ 이와 같은 연산 과정을 말로 풀어써보면…

한 타일의 인덱스는, ***현재 행 * 열의 최대 개수 + 현재 열***로 정리할 수 있다

그리고 우리가 Index로 전달한 순서의 이미지를 Atlas Texture에서 추출해야 하기 때문에, 

Atlas에서 Index를 부여할 수 있는 Row와 Col도 변수로 저장해두자

→ `UINT m_MaxImgRow`, `UINT m_MaxImgCol` 추가

정리해서, render의 연산 과정으로 옮겨 적어보면…

```cpp
void CTileMap::render_tile()
{
	if (m_AtlasTex == nullptr)
		return
		
	Vec2 vRenderPos = GetOwner()->GetRenderPos();
}
for(UINT i = 0; i < m_Row; ++i)
{
	for(UINT j = 0; j < m_Col; ++j)
	{
		// 현재 i(행), j(열)을 1차원 인덱스로 바꾸어서, 
		// 렌더링하려는 타일의 정보를 Vector에서 꺼내온다
		// 현재 타일의 index = 열의 최대 개수 * 현재 행(i) + 현재 열
		int TileIdx = m_Col * i + j;
		
		// 해당 타일 정보에서 이미지 인덱스를 확인하고,
		// 아틀라스 텍스쳐에서 이미지 인덱스에 맞는 부위를 화면에 출력한다
		// 타일이미지의 행 = Col 개수로 나눈 몫
		// 타일이미지의 열 = Col 개수로 나눈 나머지
		UINT TileImgRow = m_vecTileInfo[TileIdx].ImgIdx / m_MaxImgCol;
		UINT TileImgCol = m_vecTileInfo[TileIdx].ImgIdx % m_MaxImgCol;
		
		// 해당 타일의 이미지 인덱스가 아틀라스가 보유한 타일 행, 열을 초과한 경우
		assert(!(m_MaxImgRow <= TileImgRow));
		
		// 따라서 Atlas에서 index에 따라 참조해야 할 위치는
		// TileImgCol * m_TileSize.x
		// TileImgRow * m_TileSize.y
		// 가 된다
		BitBlt(DC, (int)(vRenderPos.x + m_TileSize.x * j)
						 , (int)(vRenderPos.y + m_TileSize.y * i)
						 , m_TileSize.x, m_TileSize.y
						 , m_AtlasTex->GetDC()
						 , TileImgCol * m_TileSize.x, TileImgRow * m_TileSize.y, SRCCOPY); 
	}
}
```

## 마우스로 타일을 클릭하면, 다음 이미지로 변경

### 1. Editor Level → 마우스 클릭이 발생하면, Tile Object에게 알려주자!

Editor Level에서는 마우스 클릭 발생 시, 해당 레벨에 존재하는 Tile Object에게 전달할 수 있도록 작성하기

```cpp
class CLevel_Editor
	: public CLevel
{
private:
	CTile* m_EditTile;             // 해당 Level의 Tile 포인터를 멤버 변수로 추가
	
public:
	// ...		
}
```

```cpp
void CLevel_Editor::tick()
{
	CLevel::tick();
	
	// 마우스 클릭 발생 시, 타일 오브젝트에게 알림
	if (KEY_TAP(KEY_::LBTN))
	{
		m_EditTile->Clicked(CKeyMgr::GetInst()->GetMousePos());
	}
}
```

### 2. Editor Level에서 전달받은 Click에 대한 TileMap의 상호작용 작성하기

이를 위해서 Tile, TileMap 단에서는 Clicked 함수를 구현해줘야 한다! 

(Click이 발생한 마우스 위치와 함께)

```cpp
void CTile::Clicked(Vec2 _vMousePos)
{
	m_TileMap->Clicked(_vMousePos);
}
```

```cpp
void Clicked(Vec2 _vMousePos)
{
	// 마우스 좌표를 실제 게임 공간 좌표로 변경한다! (RealPos 활용)
	_vMousePos = CCamera::GetInst()->GetRealPos(_vMousePos);
	Vec2 vObjPos = GetOwner()->GetPos();
	
	// 클릭한 지점이 타일 영역을 벗어나 있으면 return하기
	if (   _vMousePos.x < vObjPos.x
			|| _vMousePos.y < vObjPos.y
			|| vObjPos.x + m_Col * m_TileSize.x < _vMousePos.x
			|| vObjPos.y + m_Row * m_TileSize.y < _vMousePos.y)
	{
		return;
	}
	
	// 타일 Object 자체가 이동된 경우를 고려해 마우스 위치를 보정해준다
	_vMousePos -= vObjPos;
	
	// 마우스로 클릭한 지점이 몇행 몇열의 타일인지 계산하기
	int ClickCol = (int)_vMousePos.x / m_TileSize.x;
	int ClickRow = (int)_vMousePos.y / m_TileSize.y;
	
	// 클릭한 지점의 타일을 1차원 인덱스로 변경해서 타일 벡터에서 해당 정보에 접근!
	int TileIdx = m_Col * ClickRow + ClickCol;
	
	// 해당 타일 정보에서 참조 이미지의 인덱스를 1 증가시킨다
	m_vecTileInfo[TileIdx].ImgIdx += 1;
	
	// 만약 이미지 인덱스가, 아틀라스에서 제공하는 이미지 개수를 초과해서 참조하게 되면
	// 다시 첫번째 이미지를 참조하도록 이미지 인덱스를 0으로 설정한다
	if (m_MaxImgCol * m_MaxImgRow <= m_vecTileInfo[TileIdx].ImgIdx)
	{
		m_vecTileInfo[TileIdx].ImgIdx = 0;
	}
}
```

### +) 마우스 좌표의 보정 자세히 알아보기!

1. renderPos가 RealPos와 달라질 경우에는 좌표 계산이 제대로 이루어지지 않는다
    
    → ClickPos는 RenderPos를 기준으로 했으니까!
    
    따라서 TileMap에서는 Clicked를 통해 _vMousePos를 받게 된다면, 렌더링된 화면에서 클릭된 마우스 좌표를 ***실제 게임 공간 좌표로 변경***해줘야 한다!
    

```cpp
_vMousePos = CCamera::GetInst()->GetRealPos(_vMousePos);
```

1. TileMap의 Owner인 Tile Object 자체가 Pos를 변경하게 된다면, 정확한 위치의 Col, Row를 잡지 못한다
    
    → 클릭된 행/열에 대한 계산 자체는 Tile의 시작 좌표가 (0, 0)임을 기준으로 하기 때문에!
    
    따라서 해당 연산 과정이 0, 0으로 이루어짐을 감안하고, 현재 마우스 위치에서 Owner Tile의 Pos만큼을 빼주면서 마우스의 좌표를 보정해줘야 한다
    

```cpp
// 타일 전체가 이동된 경우를 생각해 마우스 위치를 보정
_vMousePos -= vObjPos;
```