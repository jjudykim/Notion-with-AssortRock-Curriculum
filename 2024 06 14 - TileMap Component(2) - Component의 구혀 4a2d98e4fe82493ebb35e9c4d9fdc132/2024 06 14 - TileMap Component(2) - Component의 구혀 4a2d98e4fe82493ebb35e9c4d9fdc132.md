# 24/06/14 - TileMap Component(2) - Component의 구현, Shader / Material 제작

상위 항목: Week29 (Week29%208c950cb34a664d9ab3834144e1984a6f.md)
주차: 0011_Week20~29

> 1교시 녹음본
- [https://clovanote.naver.com/s/BMa8yWcxmHVj2hnYiLBTN4S](https://clovanote.naver.com/s/BMa8yWcxmHVj2hnYiLBTN4S)

2교시 녹음본
- [https://clovanote.naver.com/s/qWjeL6PCfUsAxfDb9MJBzNS](https://clovanote.naver.com/s/qWjeL6PCfUsAxfDb9MJBzNS)
> 

### TileMap Component 클래스 생성하기

Tilemap Component의 클래스를 생성하고, 기본적인 틀을 작성해보자

**CTileMap 클래스 기본 틀 작성**

```cpp
class CTileMap :
	public CRenderComponent
{
private:
	int   m_Row;              // TileMap의 행 숫자
	int   m_Col;              // TileMap의 열 숫자
	
	Vec2 m_TileSize;         // Tile 1개의 크기

public:
	void SetRowCol(UINT _Row, UINT _Col);             // TileMap의 행/열 설정
	void SetTileSize(Vec2 _Size);                     // Tile 1개의 크기 설정

private:
	// 타일맵 전체의 크기가 변경되는 함수
	// : 행/열의 개수나 Tile의 크기가 변경되면, Transform을 통해 크기를 수정해야 한다
	void ChangeTileMapSize();

public:
	virtual void FinalTick() override;
	virtual void Render() override;

public:
	CLONE(CTileMap);
	CTileMap();
	~CTileMap();
};
```

```cpp
// ... #include ...
CTileMap::CTileMap()
	: CRenderComponent(COMPONENT_TYPE::TILEMAP)
	, m_Row(1)
	, m_Col(1)
{
	SetMesh(CAssetMgr::GetInst()->FindAsset<CMesh>(L"RectMesh"));
	SetMaterial(CAssetMgr::GetInst()->FindAsset<CMaterial>(L"TileMapMtrl"));
}

CTileMap::~CTileMap()
{
}

CTileMap::FinalTick()
{
}

void CTileMap::Render()
{
	Transform()->Binding();
	GetMaterial()->Binding();
	GetMesh()->Render();
}

void CTileMap::SetRowCol(UINT _Row, UINT _Col)
{
	m_Row = _Row;
	m_Col = _Col;
	
	ChangeTileMapSize();
}

void CTileMap::SetTileSize(Vec2 _Size)
{
	m_TileSize = _Size;
	
	ChangeTileMapSize();
}

void CTileMap::ChangeTileMapSize()
{
	Vec2 vSize = m_TileSize * Vec2((float)m_Col, (float)m_Row);
	Transform()->SetRelativeScale(vSize.x, vSize.y, 1.f);
	// 중점을 기준으로 크기가 커지기 때문에, 렌더링 될 때에는 좌상단을 위치의 기준으로 이동
	// -> Render시에 반영하기
}
```

편리한 사용을 위해 `GET_COMPONENT`, `GET_OTHER_COMPONENT` 매크로를 각각의 클래스에서 추가해주자

**GameObject 클래스에서 GET_COMPONENT 매크로 추가**

```cpp
class CGameObject :
    public CEntity
{
	// ...
public:
	GET_COMPONENT(TileMap, TILEMAP);
	// ...
};
```

**Component 클래스에서 GET_OTHER_COMPONENT 매크로 추가**

```cpp
class CComponent :
	public CEntity
{
	// ...
public:
	GET_OTHER_COMPONENT(TileMap);
	// ...
}
```

### TileMap Component를 가진, 테스트 객체를 생성해보자

TileMap 컴포넌트를 들고 있는 TileMap 전용 GameObject를 생성할 것이다

Level Manager의 Init 단계에서, GameObject를 생성하자

```cpp
void CLevelMgr::Init()
{
	// ...
	// TileMap Object
	CGameObject* pTileMapObj = new CGameObject;
	pTileMapObj->SetName(L"TileMap");

	pTileMapObj->AddComponent(new CTransform);
	pTileMapObj->AddComponent(new CTileMap);

	// Component 옵션 설정
	pTileMapObj->Transform()->SetRelativePos(Vec3(0.f, 0.f, 500.f));
	pTileMapObj->TileMap()->SetRowCol(1, 1);                            // 1X1 타일로 설정
	pTileMapObj->TileMap()->SetTileSize(Vec2(100.f, 100.f));            // 한 타일의 크기를 100으로 설정

	// 2번 레이어에 TileMap Object 추가
	m_CurLevel->AddObject(2, pTileMapObj);
}
```

추가로, TileMap은 기본적으로  RectMesh을 적용해 사용하는 것으로 설정하자

```cpp
CTileMap::CTileMap()
	: // ...
{
	// 생성 시에 사용할 메쉬 / 재질을 정해두자 
	// -> 외부에서 굳이 세팅하거나 변경해줄 필요가 X
	SetMesh(CAssetMgr::GetInst()->FindAsset<CMesh>(L"RectMesh"));
	SetMaterial(CAssetMgr::GetInst()->FindAsset<CMaterial>(L"TileMapMtrl"));
}
```

### TileMap 전용 Shader / Material 제작

타일맵을 통해 우리가 원하는 Atlas에서 원하는 위치의, 원하는 크기만큼의 타일을 출력하기 위해서

타일맵 전용 쉐이더와, 타일맵 Material을 새롭게 생성해주자

***먼저, Shader 제작부터!***

<aside>
📁 HLSL 필터 정리
01. Header > value.fx, func.fx
02. Render > debug.fx, std2d.fx, tilemap.fx

</aside>

타일맵 전용 쉐이더 파일 `tilemap.fx`을 작성해보자

기본적인 틀만 작성! 세부 구현은 일단 나중에 :3

```cpp
#ifndef _TILEMAP
#define _TILEMAP

#include "value.fx"

// 정점으로부터 받아올 Vertex Shader의 입력 구조체
struct VS_IN
{
	float3 vPos : POSITION;
	floa2 vUV : TEXCOORD;
};

// NDC 좌표계로 변환해 좌표를 전달하기 위한 출력 구조체
struct VS_OUT
{
	// SV_Position Semantic을 사용해야 
	// Rasterizer가 NDC 좌표계 데이터가 있음을 알 수 있음
	float4 vPosition : SV_Position;       
	float2 vUV : TEXCOORD;
};

VS_OUT VS_TileMap(VS_IN _in)
{
	VS_OUT output = (VS_OUT) 0.f;
	
	return output;
}

float4 PS_TileMap(VS_OUT _in) : SV_Target
{
	float4 vOutColor = (float4) 0.f;
	
	return vOutColor;
}

#endif
```

그리고 쉐이더 객체를 제작해, 해당 쉐이더를 사용할 수 있도록 등록해주자

**CAssetInit에서 쉐이더 객체 제작하기**

```cpp
void CAssetMgr::CreateEngineGraphicShader()
{
	// ...
	// TileMapShader
	pShader = new CGraphicShader;
	
	pShader->CreateVertexShader(L"shader\\tilemap.fx", "VS_TileMap");
	pShader->CreatePixelShader(L"shader\\tilemap.fx", "PS_TileMap");

	pShader->SetRSType(RS_TYPE::CULL_NONE);
	pShader->SetDSType(DS_TYPE::LESS);
	pShader->SetBSType(BS_TYPE::DEFAULT);

	// 알파(투명한 부분)가 존재하는 픽셀은 그냥 날려버리게끔!
	pShader->SetDomain(SHADER_DOMAIN::DOMAIN_MASKED);

	AddAsset(L"TileMapShader", pShader);
}
```

이제 해당 쉐이더를 사용하는 ***Material도 제작***해보자

**CAssetInit에서 Material 객체 제작**

```cpp
void CAssetMgr::CreateEngineMaterial()
{
	Ptr<CMaterial> pMtrl = nullptr;
	// ...
	// TileMapMtrl
	pMtrl = new CMaterial();
	pMtrl->SetShader(FindAsset<CGraphicShader>(L"TileMapShader"));
	AddAsset(L"TileMapMtrl", pMtrl);
}
```

<aside>
📝 **나중에는 엔진 상에서 만든 Asset과 파일로부터 로딩된 Asset이 구분되어야 한다**

예를 들어, Material에서 쉐이더에게 전달하는 값들을 중점으로 생각해보자

Material이 저장(파일로 Save)된다는 것은 쉐이더에 전달해 사용하려는 세팅 값들이 저장되는 것이고 나중에 그걸 불러와 사용하는 것인데, 

Engine에서 우리가 직접 코드로 제작한 Material들, 즉 직접 제작한 후에 Asset Manager에 저장하고 불러온 재질은 매번 프로그램이 시작될 때마다 새롭게 제작되는 것이기 때문에 모든 세팅값들이 초기화된다

그래서 지금은 각 쉐이더를 참조하는 샘플 Material을 만드는 과정일 뿐이기 때문에, Engine에서 Material을 제작해 사용하지만~ 나중에는 엔진에서 만든 Asset과 파일로부터 로딩한 Asset이 구분될 필요가 생긴다! 나중에는 에디터에서 제작한 Material들을 가져와 사용하고 할테니…

</aside>

### TileMap Component을 가진 Object를  Tile로 출력해보자

TileMap Component를 정의할 때, 설정한 Tile의 개수와 Tile 하나의 크기에 따라서 총 TileMap의 사이즈가 달라지게 되도록 작성했었다. 그런데 이때 사이즈가 축소 / 확대되는 기준은 TileMap 객체의 위치, 즉 중점을 기준으로 했는데…

우리는 TileMap과 각각의 Tile의 좌상단이 위치를 나타내는 것처럼 사용하고 싶기 때문에, 렌더링되는 위치를 조금 바꿔줄 것이다!

따라서 이를 TileMap의 쉐이더 코드에 반영해주자

**Shader 코드를 통해 TileMap의 렌더링 위치 조정**

```cpp
VS_OUT VS_TileMap(VS_IN _in)
{
	VS_OUT output = (VS_OUT) 0.f;
	
	// TileMap의 위치 기준이 좌상단이 되도록 
	// + Scale 조정 시 우측 하단으로 축소 / 확장되도록
	// -> Local RectMesh의 좌표를 수정한 후에 상태행렬을 연산
	_in.vPos.x += 0.5f;
	_in.vPos.y -= 0.5f;
	
	output.vPosition = mul(float4(_in.vPos, 1.f), matWVP);
	output.vUV = _in.vUV;
	
	return output;
}

float4 PS_TileMap(VS_OUT _in) : SV_Target
{
	float4 vOutColor = (float4) 0.f;
	
	return vOutColor;
}
```

이제 TileMap에 본격적으로 Atlas Texture를 설정하고, 원하는 크기만큼 Tile을 나눠 사용할 것이다!

따라서 필요한 멤버 변수 / 함수들을 추가해주자

**CTileMap 멤버 변수 / 멤버 함수 추가**

```cpp
class CTileMap :
	public CRenderComponent
{
private:
	// ...
	Vec2          m_TileSize;           // Tile 1개의 크기
	
	Ptr<CTexture> m_TileAtlas;          // Tile 개별 이미지들을 보유하고 있는 아틀라스 텍스쳐
	Vec2          m_AtalsResolution;    // Atlas 텍스쳐의 해상도
	Vec2          m_AtlasTileSize;      // Atals 텍스쳐 내에서 타일 1개의 크기 (m_TileSize와 다르다!)
	Vec2          m_AtlasTileSliceUV;   // m_AtlasTileSize를 UV로 환산한 값 (Slice UV)
	
	int           m_AtlasMaxRow;        // 아틀라스 텍스쳐가 보유하고 있는 타일의 최대 행 수
	int           m_AtlasMaxCol;        // 아틀라스 텍스쳐가 보유하고 있는 타일의 최대 열 수
	
	int           m_ImgIdx;             // 0 ~ (m_AtlasMaxRow * m_AtlasMaxCol - 1)까지 사용 가능
	
public:
	// ...
	void SetAtlasTexture(Ptr<CTexture> _Atlas);
	void SetAtlasTileSize(Vec2 _TileSize);
}
```

```cpp
// ...
void SetAtlasTexture(Ptr<CTexture> _Atlas)
{
	m_TileAtlas = _Atlas;
	
	m_AtlasResolution = Vec2((float)_Atlas->Width(), (float)_Atlas->Height());
	
	// 이 함수가 Size 설정보다 늦게 호출되어 설정된다면
	// 사이즈 재설정을 통해 UV값과 최대 행/열이 계산되도록 진행
	SetAtalsTileSize(m_AtalsTileSize);
}

void SetAtlasTileSize(Vec2 _TileSize)
{
	m_AtlasTileSize = _TileSize;
	
	// 아틀라스 텍스쳐가 존재할 때에만, 
	// Tile 사이즈의 UV값과 아틀라스 행/열의 최대 개수를 구해준다
	if (m_TileAtlas != nullptr)
	{
		m_AtlasTileSliceUV = m_AtlasTileSize / m_AtlasResolution;   // UV 값 계산

		m_AtlasMaxCol = m_AtlasResolution.x / m_AtlasTileSize.x;    // 타일 최대 열
		m_AtlasMaxRow = m_AtlasResolution.y / m_AtlasTileSize.y;    // 타일 최대 행
	}
}
```

이제 TileMap Component을 들고 있는 Object에, Atlas와 Tile의 사이즈를 설정해주자

```cpp
void CLevelMgr::Init()
{
	// ...
	// TileMap Object
	// ...
	pTileMapObj->TileMap()->SetRowCol(1, 1);
	pTileMapObj->TileMap()->SetTileSize(Vec2(64.f, 64.f));
	
	Ptr<CTexture> pTileAtlas = CAssetMgr::GetInst()->Load<CTexture>(L"TileAtlasTex", L"texture\\TILE.bmp");
	pTileMapObj->TileMap()->SetAtlasTexture(pTileAtlas);
	pTileMapObj->TileMap()->SetAtlasTileSize(Vec2(64.f, 64.f));
}
```

이제 렌더링하기 전에, Atlas에 대해 참조해야 되는 정보들을 Material을 통해 쉐이더에서 사용할 수 있도록 해주자

- 활용할 Atlas Texture 자체 → `m_ImgIdx`
- 어떤 Index 타일을 띄울 것인지 → `m_TileAtlas`
- 타일 하나가 얼만큼의 크기를 갖는지 → `m_AtlasTileSliceUV`
- 해당 Index가 몇 행 / 몇 열에 위치하는지?  → `m_AtlasMaxRow`, `m_AtlasMaxCol`
    
    → 타일이 시작하는 좌상단의 좌표를 알기 위해서
    

```cpp
void CTileMap::Render()
{
	GetMaterial()->SetScalarParam(TEX_0, m_TileAtlas);
	GetMaterial()->SetScalarParam(INT_0, m_ImgIdx);
	GetMaterial()->SetScalarParam(VEC2_0, m_AtlasTileSliceUV);
	GetMaterial()->SetScalarParam(INT_1, m_AtlasMaxRow);
	GetMaterial()->SetScalarParam(INT_2, m_AtlasMaxCol);
	
	GetMaterial()->Binding();
	Transform()->Binding();
	GetMesh()->Render();
}
```

TileMap 쉐이더는 재질을 통해서 값을 전달받긴 하지만, 

각 항목들이 단순히 타입명을 가진 변수로… 표현하기엔 직관적이진 않으니 전처리기를 활용해 변환해주자

```cpp
// ==================================
// TileMapShader
// parameter
#define AtalsTex       g_tex_0
#define IsAtlasBind    g_btex_0              // 아틀라스 텍스쳐가 바인딩 되었는지 체크
#define ImgIdx         g_int_0
#define TileSliceUV    g_vec2_0
#define AtlasMaxRow    g_int_1
#define AtalsMaxCol    g_int_2
// ==================================
// ...
```

이제 각 정점에 대한 샘플링이 진행될 수 있도록, Pixel Shader도 작성해주자

```cpp
// ...
float4 PS_TileMap(VS_OUT in) : SV_Target
{
	float4 vOutColor = (float4) 0.f;

	// 아틀라스 텍스쳐가 제대로 바인딩 되어 있을 때에만 진행
	if (IsAtlasBind)
	{
		// 이미지 인덱스를 통해 좌상단 UV를 계산
		int row = ImgIdx / AtlasMaxCol;
		int col = ImgIdx % AtalsMaxCol;
		
		float2 vLeftTopUV = float2(col, row) * TileSliceUV; 
	
		float2 vUV = vLeftTopUV + _in.vUV * TileSliceUV;
		vOutColor = AtlasTex.Sample(g_sam_1, );
	}
	else
	{
		vOutColor = float4(1.f, 0.f, 1.f, 1.f);
	}
	
	return vOutColor;
}
```