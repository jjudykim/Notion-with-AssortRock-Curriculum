# 2024/07/10 - Post Process(후처리) (2) - 쉐이더 코드 직접 작성 & 기존의 Texture를 적용

태그: C++, DirectX11, 게임수학, 고급, 그래픽스, 자료보충필요
날짜: 2024/07/10
상위 항목: Week33 (https://www.notion.so/Week33-c25c856dcd954fe0af5f24db92f6af83?pvs=21)
상태: 메모
주차: 0100_Week30~39

> 1교시 녹음본
- [https://clovanote.naver.com/s/3cy5wwscj2WeWqi4EpcdieS](https://clovanote.naver.com/s/3cy5wwscj2WeWqi4EpcdieS)

2교시 녹음본
- [https://clovanote.naver.com/s/j6JJsVwwqaSYArkHjsorcUS](https://clovanote.naver.com/s/j6JJsVwwqaSYArkHjsorcUS)
> 

### Post Process의 쉐이더 코드 작성해보기

```cpp
#ifndef _POSTPROCESS
#define POSTPROCESS

// ===========================
//      GrayFilterShader
// ===========================
//            Info
// --------------------------- 
// Mesh : RectMesh
// DS_TYPE : NO_TEST_NO_WRITE
// g_text_0 : TargetCopyTex
// g_text_1 : NoiseTexture

struct VS_IN
{
	float3 vPos : POSITION;
	float2 vUV : TEXCOORD;
};

struct VS_OUT
{
	float4 vPosition : SV_Position;
	float2 vUV : TEXCOORD;
};

VS_OUT VS_GrayFilter(VS_IN _in)
{
	VS_OUT output = (VS_OUT) 0.f;
	
	// Projection 행렬을 곱한 결과는 각 xyz에 자신의 Viewz가 곱해져 있는 형태!
	// w 자리에 자신의 Viewz가 출력되기 때문에, w 값으로 각 x, y, z를 나누어야 실제 Projection 좌표가 나온다
	// 따라서 Rasterizer State에 투영행렬을 곱한 결과를 전달하면 각 xyz를 
	output.vPosition = float4(_in.vPos.xy * 2.f, 0.f, 1.f);            // w에 1을 넣어주는 이유?
	output.vUV = _in.vUV;
	
	return output;
}

float4 PS_GrayFilter(VS_OUT _in) : SV_Target
{
	float4 vColor = g_tex_0.Sample(g_sam_0, _in.vUV);
	float Average = (vColor.x + vColor.y + vColor.z) / 3.f
	vColor = float4(Average, Average, Average, 1.f);
	
	return vColor;
}

#endif
```

- w에 1을 넣어주는 이유?

<aside>
📁 **원근 투영의 원리**

원근 투영의 변환은 *원근 투영 행렬*을 통해 이루어지는데, 4x4 행렬을 활용해 3D 공간의 점들을 클립 공간으로 변환한다!

클립 공간의 좌표는 (x, y, z, w)의 형태로 표현되는데, 이 클립 공간 좌표를 NDC 공간으로 변환하기 위해서 래스터라이저 단계에서 x, y, z를 w로 나누는 과정인 동차나누기가 활용된다

동차 나누기는 다음과 같이 수행된다
x = x / w
y = y / w
z = z / w

즉, 우리의 입장에서는 View 좌표인 (Vx, Vy, Vz, 1.f) 가 투영행렬과 연산된 결과가

</aside>

$$

$$

: 투영(Projection) 시에 원근 투영이 적용될 때, Raseterizer에서 각 x, y, z를 w로 나눠 연산하게 된다?

→ 원근 투영의 변환은 원근 투영 행렬을 통해 이루어지는데, 이때 4x4 행렬을 활용해 3D 공간의 점들을 클립 공간으로 변환한다! 

- 원근 투영 변환이 적용된 후의 좌표는 클립 공간에 위치하게 되며, 좌표는 ***(x, y, z, w)***의 형태로 표현된다
- 클립 공간의 좌표는 래스터라이저 단계에서 x, y, z를 w로 나누는 과정을 거쳐, NDC 공간으로 변환된다 → ***동차 나누기(Homogeneous Division)***
    
    <aside>
    ✍️ 동차 나누기(Homogeneous Division)란?
    
    </aside>
    

클립 공간(Clip Space)에서 NDC(Normalized Device Coordinates) 공간으로의 변환을 위한 필수적인 단계

→ (Vx Vy Vz 1.f) * Proj = (Vx * Vz, Vy *  Vz, Vz * Vz, Vz)

- 독특한 효과를 적용해보자
    - Distortion의 예시
    
    ```cpp
    float4 PS_GrayFilter(VS_OUT _in) : SV_Target
    {
    	float2 vUV = _in.vUV;
    	
    	vUV.y += cos(vUV.x);
    	
    	float4 vColor = g_tex_0.Sample(g_sam_0, vUV);
    
    	return vColor;
    }
    ```
    
    - 0 ~ 2π 가 0~1로 보간될 수 있도록 → `vUV.y += cos(vUV.x * PI * 2.f)`
        
        ![Untitled](2024%2007%2010%20-%20Post%20Process(%E1%84%92%E1%85%AE%E1%84%8E%E1%85%A5%E1%84%85%E1%85%B5)%20(2)%20-%20%E1%84%89%E1%85%B0%E1%84%8B%E1%85%B5%E1%84%83%E1%85%A5%20%E1%84%8F%E1%85%A9%E1%84%83%207cbdc5aae75841dfbe9cee9a705df126/Untitled.png)
        
    - 시간의 흐름에 따라 효과가 적용되게끔 하고싶다면 → Engine DT를 활용해서
        
        `vUv.y += cos((vUV.x + g_EngineTime * 0.1f) * PI * 12.f) * 0.01`
        
    

### Noise Texture를 활용해 Post Process의 효과로 적용해보기

가져온 Noise Texture의 이미지를 Asset Manager에서 Texture 로딩시 함께 등록될 수 있도록 설정

```cpp
void CAssetMgr::CreateEngineTexture()
{
	// Post Process 용도 텍스쳐 생성
	
	// Noise Texutre
	Load<CTexture>(L"texture\\noise\\noise_01.png", L"texture\\noise\\noise_01.png");
	Load<CTexture>(L"texture\\noise\\noise_02.png", L"texture\\noise\\noise_02.png");
	Load<CTexture>(L"texture\\noise\\noise_03.jpg", L"texture\\noise\\noise_03.jpg");
}
```

쉐이더에서 무작위성을 구현할 때 노이즈를 활용해보기

```cpp
// ...

// ===========================
//      GrayFilterShader
// ===========================
//            Info
// --------------------------- 
// Mesh : RectMesh
// DS_TYPE : NO_TEST_NO_WRITE
// g_text_0 : TargetCopyTex
// g_text_1 ~ 3 : NoiseTexture             // Noise Texture 추가!
// ============================

float4 PS_GrayFilter(VS_OUT _in) : SV_Target
{
	// GrayFilter
	// ...
	// Distortion
	// ...
	// Noise
	float2 vUV = _in.vUV;
	vUV.x += g_EngineTime * 0.01f;	
	// 시간 개념을 추가해 시간의 흐름에 따라 효과가 적용되도록

	float4 vNoise = g_tex_1.Sample(g_sam_0, vUV);
	vNoise = (vNoise * 2.f - 1.f) * 0.001f;
	// 0에서 1사이의 값을 -> 0 ~ 2 사이의 값으로 -> -1 ~ 1 범위로 -> -0.001 ~ 0.001 범위로
	
	vUV += _in.vUV + vNoise.xy;
	float4 vColor = g_tex_0.Sample(g_sam_0, vUV);
	
	return vColor;
}
```

→ Noise Texture의 UV 값으로 Render Target의 UV 값을 변형?