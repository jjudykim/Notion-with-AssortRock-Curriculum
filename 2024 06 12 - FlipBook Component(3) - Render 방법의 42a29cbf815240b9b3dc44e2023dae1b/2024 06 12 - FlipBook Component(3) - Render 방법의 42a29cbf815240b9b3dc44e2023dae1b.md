# 24/06/12 - FlipBook Component(3) - Render 방법의 개선

상위 항목: Week29 (Week29%208c950cb34a664d9ab3834144e1984a6f.md)
주차: 0011_Week20~29

> 1교시 녹음본
- [https://clovanote.naver.com/s/Eh2oaphyhyh4eWgpfLxx27S](https://clovanote.naver.com/s/Eh2oaphyhyh4eWgpfLxx27S)
2교시 녹음본
- [https://clovanote.naver.com/s/dkMZJFpQT4JXAUQJXv2uXNS](https://clovanote.naver.com/s/dkMZJFpQT4JXAUQJXv2uXNS)
> 

## FlipBook Component의 Render 방법에 대한 개선

### AddFlipBook 시에 Index 설정하기

AddFlipBook 시, 원하는 Index에 넣을 수 있도록 구현하자

우리가 Index를 enum으로 사용할 시에는 원하는 순번이 있을텐데, 지금 구조는 순차적으로 넣어야 하니,

지정된 인덱스로 넣을 수 있도록 변경

```cpp
void CFlipBookComponent::AddFlipBook(int Idx, Ptr<CFlipBook> _Flipbook)
{
	// 만약 Idx만큼 Vector의 크기가 resize 되지 않을 수 있으니...
	if (m_vecFlipBook.size() <= Idx)
	{
		m_vecFlipBook.resize(_Idx + 1);
	}
	
	m_vecFlipBook[_Idx] = ;
}
```

그럼 AddFlipBook을 사용할 때에, Index를 넣어 지정할 수 있게 된다

```cpp
pObject->FlipBookComponent()->AddFlipBook(5, CAssetMgr::GetInst()->FindAsset<CFlipBook>(L"Link_MoveDown));
```

<aside>
🗒️ reverse vs resize 차이?

</aside>

FlipBook의 렌더링 방법 개선

지금은 재질(Material)을 빌려서 사용하고 있는데…

나중에 재질을 실제로 사용하려 한다면?

항상 FlipBook이 Material의 레지스터를 점유하고 있기 때문에, 사용시 이를 유의해야 한다!

따라서 FlipBook이 사용하는 전용 별도 상수버퍼를 통해, 출력하려는 Sprite의 데이터를 전달해주는 방식으로 변경해보자!

그럼 애니메이션 데이터를 담는 상수버퍼가 필요할 것이다!

**상수버퍼 타입에 추가**

```cpp
enum class CB_TYPE
{
	TRANSFORM,
	MATERIAL,
	SPRITE,               // 추가!
	GLOBAL,
	END,
};
```

상수버퍼 연동 구조체도 작성하자

```cpp
struct tSpriteInfo
{
	Vec2 LeftTopUV;
	Vec2 SliceUV;	
	Vec2 BackGroundUV;
	Vec2 OffsetUV;
	
	int UseFlipbook;       // FlipBook의 사용 여부
	int Padding[3];        // 16바이트로 맞춰주기 위한 Padding
};
```

Device에서 상수버퍼를 만들 때 같이 만들어줘야 한다

```cpp
pCB = new CConstBuffer;
if (FAILED(pCB->Create
{
}
```

```cpp
cbuffer SPRITE_INFO : register(b2)
{
	float2 LeftTopUV;
	float2 SliceUV;
	float2 BackGroundUV;
	float2 OffsetUV;             // 새 멤버 추가! 아직 사용하진 않을 것이다
	int UseFlipBook;             // 새 멤버 추가! 아직 사용하진 않을 것이다
	
	int3 SpriteInfoPadding;
}
```

이렇게 레지스터를 통해 상수 버퍼를 보내게 될 것이다

<aside>
🗒️ [추가 사항]
Sprite의 멤버가 추가 되었으니, CSprite에서 멤버와 그 Get, Set 함수 역시 추가해주자

```cpp
void CSprite::SetBackground(Vec2 _Background)
{
	Vec2 AtlasResolution = Vec2((float)m_Atlas->Width(), (float)m_Atlas->Height());
	m_BackgroundUV = _Background / AtlasResolution;
}
```

</aside>

MeshRender에서는 바인딩의 과정에서 전처럼 데이터를 나눠서 바인딩하는 것이 아닌,

FlipBook Component가 존재한다면 그 자체를 바인딩하라고 명령을 할 것이다

그럼 이제 FlipBook Component에는 Binding 함수가 필요하게 됐다!

FlipBook Component의 Binding() 함수 작성

```cpp
class CFlipBookComponent :
    public CComponent
{
public:
	// ...
	void Binding();
};
```

```cpp
void CFlipBookComponent::Binding()
{
	// FlipBook Component는 현재 재생중인 FlipBook이 있을 때에만 바인딩을 할 것이다
	if (m_CurFrmSprite != nullptr)
	{
		// 현재 출력해야 하는 Sprite의 정보를 Sprite 전용 상수버퍼를 통해 GPU에 전달할 것!
		tSpriteInfo tInfo = {};
		
		m_CurFrmSprite->GetAtlasTexture();
		tInfo.LeftTopUV = m_CurFrmSprite->GetLeftTopUV();
		tInfo.SliceUV = m_CurFrmSprite->GetSliceUV();
		tInfo.BackGruondUV = m_CurFrmSprite->GetBackGroundUV();
		tInfo.OffsetUV = m_CurFrmSprite->GetOffsetUV();
		tInfo.UseFlipBook = 1;
		
		static CConstBuffer* CB = CDevice::GetInst()->GetConstBuffer(CB_TYPE::SPRITE);
		
		CB->SetData(&tInfo);
		CB->Binding();
		
		// FlipBook Sprite 아틀라스 텍스쳐 전용 레지스터 번호 10에 바인딩
		Ptr<CTexture> pAtlas = m_CurFrmSprite->GetAtlasTexture();
		pAtlas->Binding(10);
	}
	else
	{
		Clear();
	}
}
```

만약 FlipBook Component가 존재하지 않는 상황이라면?

아무 작업도 해주지 않으면 되는거 아닌가? 라고 생각할 수 있지만!

쉐이더에 세팅해놨던 값들이 그대로 남아있기 때문에, 별도 처리를 꼭 해줘야 한다

FlipBook Component의 객체가 없는 상황에서도 호출이 가능해야 하기 때문에

정적 멤버 함수로 만들어 호출해줄 것이다! → `Clear`

**FlipBook Component의 객체가 존재하지 않을 때, 상수버퍼를 비워주는 함수 Clear 작성**

```cpp
class CFlipBookComponent:
	public CComponent
{
	// ...
	// 객체없이 호출될 수 있는 정적 멤버함수로 만들기 위해 static 키워드를 붙여준다!
	static void Clear();
};
```

```cpp
void CFlipBookComponent::Clear()
{
	tSpriteInfo tInfo = {};
	static CConstBuffer* CB = CDevice::GetInst()->GetConstBuffer(CB_TYPE::SPRITE);
	CB->SetData(&tInfo);
	CB->Binding();
}
```

그럼 MeshRender에서 FlipBook Component를 바인딩할 때 부터도, 호출이 가능할 것이다

```cpp
// FlipBook 컴포넌트가 있으면
if (FlipBookComponent())
	FlipBookComponent()->Binding();
else
	CFlipBookComponent::Clear();
```

추가한 두 멤버 변수에 대하여!

**BackgroundUV**

먼저 BackgroundUV가 고안된 배경에 대해 생각해보면!

비율과 크기가 다 다른 Sprite를 RectMesh로 띄우게 되면 변형이 일어나게 되니, 

보통 RectMesh 자체를 출력하려는 Sprite에 맞춰 변경한다 → 그러나 이렇게 되면 발생하는 문제점들.. 

- Collider가 영향을 받는다거나
- Transform 자체가 영향을 받게 된다

그래서!

RectMesh를 변형하는 일 없이, 

이미지를 RectMesh의 비율과 동일하게 Background를 포함해 Atlas에서 추출해온다

*WinAPI 때 쿠키런에서 애니메이션을 제작할 때 사용했던 방법을 생각하면 된다!*

그럼 이 Background는, Sprite 내에서 추출하려는 이미지 중 가장 사이즈가 큰 이미지를 기준으로 해서, 그보다 더 크게 가져와주어야 한다

OffsetUV → 애니메이션 프레임의 위치를 조정하는 수치

BackGround 적용해보기

Create Sprite시에, BackGround를 같이 적용해보자

```cpp
pSPrite->Create(pAtalsTex, Vec2((float)i * 120.f, 520.f), Vec2(120.f, 130.f));
pSPrite->SetBackground(Vec2(150.f, 150.f));
```

그럼 쉐이더에서도 Background를 고려해주어야 하는데,

```cpp
if (UseFlipbook)
{
	
}
```

Atlas 내에서는, 출력하려는 Sprite의 Background의 LeftTop 위치부터 Background 영역만큼이 RectMesh에 대응하게 될 것이다

Background의 좌표를 알기 위해서?

그럼 이제 추출한 Background 내에서, 우리가 출력하는 Sprite의 영역이 어느 위치에 존재하는 지를 알아야 하는데..!

Background의 x 길이의 절반 - SliceUV의 x길이의 절반을 연산하면, LeftTop끼리의 x 좌표만큼의 차이가 나올 것이고

Background의 y 길이의 절반 - SliceUV의 y길이의 절반을 연산하면, LeftTop끼리의 y좌표 만큼의 차이가 나올 것이다

```cpp
float2 BackGroundLeftTop = LeftTopUV - (BackGroundUV - Slice) / 2.f;
float2 vSpriteUV = BackGroundLeftTop + (_in.vUV * BackGroundUV);

// 출력하고 싶은 원래 Sprite의 영역을 한정해, 그 부분만 샘플링해주자!
if (LeftTopUV.x <= vSpriteUV.x && vSpriteUV.x <= LeftTopUv.x + SlizeUV.x
		&& LefTopUV.y <= vSpriteUV.y && vSpriteUV.y <= LeftTopUV.y + SliceUV.y)
{
	// 샘플링
	vColor = 
}
else
{
	// 해당되지 않는 부분은 출력X
	
}
```

### Offset

Offset이 움직이는 만큼, UV값은 반대로 움직여야 한다?

ex) Offset이 오른쪽으로 적용한다면, 추출해오는 Background의 LeftTop은 왼쪽으로 더 이동하게 된다

```cpp
vSpriteUV -= OffsetUV;
```

따라서, Offset을 적용할 여지가 있는 Atlas와 Sprite라면 Background의 사이즈를 Offset을 적용할 만큼 더 고려해줘야 한다!

Background와 Offset을 모두 적용한 코드를 보면 다음과 같다