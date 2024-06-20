# 2023/12/26 - 템플릿, namespace, ::, new와 delete, 인라인 함수, STL vector의 구현

태그: C++, CS, 자료구조, 중급, 초급
날짜: 2023/12/26
상위 항목: Week5 (Week5%200a092a69a72545ed811b51af82f1994a.md)
주차: 0001_Week1~9

### 템플릿

C++이 가지는 프로그래밍 언어로서의 특징 중 하나!

→ 바로 ***일반화 프로그래밍 (generic programming)***

: 프로그램의 알고리즘에 중점을 두는 프로그래밍! 대표적인 기능 중 하나가 바로 템플릿 (template)

**템플릿(template)이란,**

매개변수의 타입에 따라 함수나 클래스를 생성하는 메커니즘

즉, 타입마다 별도의 함수나 클래스를 만들지 않고도, 여러 타입에서 동작할 수 있는 단 하나의 함수나 클래스를 작성하는 것이 가능해진다!

→ 타입이 매개변수에 의해 표현되므로, `매개변수화 타입(parameterized type)`이라고도 한다

**함수 템플릿(function template)**

: 함수의 일반화된 선언으로, 같은 알고리즘을 기반으로 하면서 서로 다른 타입에서 동작하는 함수를 한 번에 정의할 수 있다

```cpp
template <typename 타입이름>
함수의 원형
{
    // 함수의 본체
}
```

ex)

```cpp
template <typename T>
void Swap(T& a, T& b)
{
	T temp;
	temp = a;
	a = b;
	b = temp;
}

int main()
{
	int a = 3;
	int b = 5;
	Swap(a, b);          // Int 타입의 Swap이 실행됨
	cout << "a: " << a << " b: " << b << endl;
}
```

→ **함수 템플릿의 인스턴스화**가 진행된다!

함수 템플릿이 각각의 타입에 대해 처음으로 호출될 때, C++컴파일러는 해당 타입의 인스턴스를 생성한다. 이렇게 생성된 인스턴스는 ***해당 타입에 대해 특수화된 템플릿 함수***이다. 이 인스턴스는 함수 템플릿에 해당 타입이 사용될 때마다 호출된다.

- `Swap<int>(3, 5)`와  `Swap(1.2f, 8.2f)`의 차이

: 원칙적으로는 함수 템플릿에 타입을 지정해줘야 하지만, 컴파일러가 입력 인자의 타입을 분석해서 자동으로 특정 타입을 정해 함수 템플릿을 작성한다!

→ 그렇기에 `Swap(1.2f, 8.2f)`의 경우, 겉으로 보기에 마치 일반 전역 함수를 호출하는 것처럼 보일 수 있지만, 엄연히 함수 템플릿을 사용한 것!

**클래스 템플릿(class template)**

: 클래스의 일반화된 선언으로, 타입에 따라 다르게 동작하는 클래스를 만들 수 있다!

즉, 클래스 템플릿에 전달되는 `템플릿 인수(template argument)`에 따라 (타입이나.. 명시된 타입의 상숫값이나..) 별도의 클래스를 만들 수 있게 된다

```cpp
template <typename 타입이름>
class 클래스템플릿이름
{
	// 클래스 멤버의 선언
}
```

ex)

```cpp
template <typename T>
class Data
{
private:
    T m_data;

public:
    Data(T dt);
    T get_data();
};
```

***클래스 템플릿에서 유의할 점***

<aside>
⭐ 클래스 템플릿은 헤더에 전부 구현해야 한다

</aside>

→ 헤더와 cpp를 구분한 채로 클래스 템플릿을 구현해버리면, 링크 시점에 연결할 실제로 구현된 cpp가 없기 때문에

이 개념을 그대로 사용해, ***클래스를 템플릿화 해*** 동적 배열을 작성해보는 것이 목표!

### namespace(이름공간)과 ::(스코프 연산자, 범위지정 연산자)

네임스페이스(namespace)는

변수명, 함수명, 클래스명, 매크로 이름 등등… 모든 부분에서 이름적으로 겹치는 일이 없도록 각자 자기만의 namespace를 지정해 기능들을 구현할 수 있는 특별한 선언!

즉, 내부 식별자(identifier)에 사용될 수 있는 유효한 범위를 제공하는 선언적인 영역이다

- 이름 공간을 나눠서 구현하더라도 통합적으로 관리된다
- 중첩으로도 사용이 가능하다

네임스페이스를 정의한 후에는 해당 네임스페이스로 접근할 수 있는 방법이 필요한데, 이때 범위 지정 연산자 `::` 를 사용한다!

```cpp
namespace MY_SPACE
{
	namespace MY_INNER_SPACE
	{
		int inner_a;
	}
	int a;
}

namespace MY_SPACE
{
	int b;
}

int a;

void testNamespace()
{
	MY_SPACE::a = 50;
	MY_SPACE::MY_INNER_SPACE::inner_a = 100;
	MY_SPACE::b = 60;

	a = 30;
}
```

**using을 사용해 간소화된 네임스페이스로의 접근**

: using 지시자는 명시한 네임스페이스에 속한 이름을 모두 가져와, 범위 지정 연산자를 사용하지 않고도 사용할 수 있게 해준다

- 전역 범위에서 사용된 using 지시자는
    
    → 해당 네임스페이스의 모든 이름을 전역적으로 사용할 수 있게 해줌
    
- 지역 범위(블록 내)에서 사용된 using 지시자는
    
    → 해당 블록 안에서만 해당 네임스페이스의 모든 이름을 사용할 수 있게 해줌
    
- 또, using 선언은 단 하나의 이름만을 범위 지정 연산자를 사용하지 않고도 사용할 수 있게 해준다

```cpp
// using 지시자
using namespace 네임스페이스이름;
// : 해당 네임스페이스를 무효화 처리, 
// namespace 안에 모든 것들을 해제시키기 때문에 원래의 목적을 상실한다

// using 선언
using 네임스페이스이름::이름;
// : 해당 네임스페이스 안의 특정 기능(멤버)만을 namespace 범위 해제
```

**전역 범위를 지칭하는 scope 연산자**

```cpp
::a;
```

→ scope 연산자를 통해 namespace만 지정하는 것이 아니라, 전역 범위도 지정할 수 있다!

- 함수 지역 내에서 호출할 변수의 우선순위를 전역변수로 바꾸고 싶은 경우
- 멤버 함수 내에서 호출할 함수의 우선순위를 전역함수로 바꾸고 싶은 경우

### C++ 스타일의 동적 할당과 해제

기존에 C 스타일에서 사용하던 동적할당과 해제는 `malloc`, `free`를 사용했다

C++에서는 이제 동적할당을 `new`, `delete` 템플릿으로 처리한다!

→ 힙 메모리에 요청할 메모리의 크기를 전달한 자료형 타입의 크기로 처리하고,

해당 공간을 전달한 자료형 단위로 쓰인다고 판단한다

해당 자료형이 클래스라면? 그 클래스의 생성자 or 소멸자까지 호출해준다

```cpp
// new의 기능을 하는 템플릿을 재현해보자
template<typename T>
void MyNew()
{
	int size = sizeof(T);
	T* pNew = (T*)malloc(size);
	pNew->T::T();
}

// delete도 템플릿이라고 했으니..
template<typename T>
void MyDelete()
{
	// ...
	T::~T();
	free(T);
}
```

new와 delete는 기존에 malloc과 free가 해줄 수 없던 부분을 보완해준다!

- `malloc`은 컴파일러 입장에서 자동으로 생성자의 호출이 불가능했다
    
    → byte에 대한 메모리 공간의 할당만 가능했지, 어떤 자료형으로 그 공간을 처리하고 해석할지는 나중에 결정됐기 때문에!
    
- 같은 이유로, `free`도 해당 메모리 공간에 대한 해제만 담당할 뿐 자동으로 소멸자를 호출할 수 없었다

```cpp
CArr* pArr1 = new CArr;
```

→ 동적할당을 함과 동시에 해당 공간을 어떤 타입(용도)로 쓸 지를 명시했기 때문에, 해당 타입의 생성자가 자동으로 호출된다

### 인라인(inline) 함수

멤버함수를 헤더에서 정의하면, **인라인 함수**가 된다!

→ 함수의 호출 부분에서 *해당 함수의 실행 부분이 그대로 복사*되는 함수

따라서, 실제 호출했을 때  별도의 스택이 할당되지 않아 별도의 스택 생성 및 해제의 비용이 없다! 매크로 함수와 유사한 개념

단, 인라인 함수를 남발하는 경우에 문제가 발생하게 되는데

→ 함수 구문이 여기저기 호출하는 곳마다 복사되어, 코드의 양이 엄청나게 늘어날 수 있다!

따라서 적절한 인라인 처리를 하는 것이 중요하다

- 특정 멤버 값을 get하거나 set하는 함수 (getter, setter)
- 구문이 짧고, 기능이 간단하면서, 자주 호출되는 함수

### 클래스 템플릿으로 동적 배열 구현하기 - STL의 vector 따라해보기

```cpp
template<typename T>
class CArr
{
private:
	int*	m_pData;
	int		m_MaxCount;
	int		m_CurCount;

	void Realloc();

public:
	void push_back(const T& _Data);

	int sizeOfArr() const { return m_CurCount; }
	int capacity() const { return m_MaxCount; }
	int at(int _idx) { return m_pData[_idx]; }

	CArr();
	~CArr();
};

template<typename T>
CArr<T>::CArr()
	: m_pData(nullptr)
	, m_CurCount(0)
	, m_MaxCount(3)
{
	// m_pData = (int*)malloc(sizeof(int) * m_MaxCount);
	m_pData = new T[m_MaxCount];
}

template<typename T>
CArr<T>::~CArr()
{
	// free(m_pData);
	delete[] m_pData;
}

template<typename T>
void CArr<T>::push_back(const T& _Data)
{
	// 데이터가 꽉 차있다면
	if (m_MaxCount <= m_CurCount)
	{
		// 저장공간 추가 할당
		Realloc();
	}

	m_pData[m_CurCount++] = _Data;
}

template<typename T>
void CArr<T>::Realloc()
{
	// 1. 새로운 공간 할당
	m_MaxCount *= 2;
	T* pNew = new T[m_MaxCount];

	// 2. 기존 데이터 이동
	for (int i = 0; i < m_CurCount; i++)
	{
		pNew[i] = m_pData[i];
	}
	// 3. 기존 공간 해제
	free(m_pData);

	// 4. 새로운 공간 가리키기
	m_pData = pNew;
}
```

→ 클래스 템플릿이기 때문에, 구현 부분 역시 헤더파일에 같이 작성되어야 한다!

```cpp
#include "CArr.h"
#include <iostream>
#include <vector>

using namespace std;

int main()
{
	// 표준 라이브러리 동적배열 vector는 클래스 템플릿!
	vector<int>		vecInt;
	vector<float>	vecFloat;

	// 우리가 만든 클래스 템플릿 동적배열
	CArr<int> arr;

	int i = 0;
	i = arr.sizeOfArr();
	i = arr.capacity();

	arr.push_back(10);
	arr.push_back(20);
	arr.push_back(30);
	arr.push_back(40);

	int a = arr.at(2);

	// 연속된 공간에 동적 할당하기
	CArr<float>* arr2 = new CArr<float>[10];

	delete arr;
	// 연속된 공간에 대한 delete를 각각의 객체마다 연쇄적으로 실행해주는 delete[] 
	delete[] arr2;

}
```