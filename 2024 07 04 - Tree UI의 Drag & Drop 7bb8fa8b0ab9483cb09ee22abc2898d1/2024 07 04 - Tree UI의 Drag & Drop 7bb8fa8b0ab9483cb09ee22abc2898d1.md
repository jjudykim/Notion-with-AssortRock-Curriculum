# 2024/07/04 - Tree UI의 Drag & Drop

상위 항목: Week32 (https://www.notion.so/Week32-ba138587ed0c43049aaeb6a6d6683658?pvs=21)
상태: 메모
주차: 0100_Week30~39

> 1교시 녹음본
- [https://clovanote.naver.com/s/H7S2ZGVgGHepJG6SCb6WxVS](https://clovanote.naver.com/s/H7S2ZGVgGHepJG6SCb6WxVS)

2교시 녹음본
- [https://clovanote.naver.com/s/v7JDi42gGVojwFw5xxqNPXS](https://clovanote.naver.com/s/v7JDi42gGVojwFw5xxqNPXS)
> 

## Tree UI에 Drag & Drop 기능을 추가해보자

### 흠냐리…

먼저, 현재 우리가 `IsItemClicked`을 사용했을 때에 클릭이 판정되는 기준은,

→ Mouse L Button에 대한 Pressed 입력이 들어왔을 때! 즉, 마우스를 누르는 순간 바로 실행되었다!

그러나 Drag는 Pressed - Drag - Released의 과정을 거쳐야하는데, Click의 조건을 그대로 Pressed 입력 시인 상태로 둔다면…?

→ Drag를 위해 입력한 마우스 Pressed가, `IsItemClicked` 조건에 걸려 Drag하려는 의도대로 사용하지 못할 수 있으므로.. 

즉, *Click / Drag를 구분하기 위해 마우스의 판정 조건을 변경*해줘야 한다

Drag & Drop를 세부적으로 나눠보면

- Drag 기능 : Click된 상태로
- Drop 기능 : Tree 쪽에서도 Drop을 받을 수 있게 할 것인지?

또, Drag & Drop을 사용하는 많은 case들이 있지만 대표적으로는 두 가지 case가 있다

- 

**Drag & Drop 기능을 사용할 것인지에 대한 멤버 변수**

```cpp
class TreeUI :
	public EditorUI
{
private:
	//  ...
	bool      m_UseDrag;
	bool      m_UseDrop;

	// ...
	void UseDrag(bool _Drag) { m_UseDrag = _Drag; }
	void UseDrop(bool _Drop) { m_UseDrop = _Drop; }

	bool IsDrag() { return m_UseDrag; }
	bool IsDrop() { return m_UseDrop; }
}
```

Drag나 Drop을 당하는 주체는 각각의 노드들?

Outliner → Drag & Drop 모두 사용

```cpp
Outliner::Outliner()
{
	// 트리 옵션 세팅
	// ...
	// Drag & Drop
	m_Tree->UseDrag(true);
	m_Tree->UseDrop(true);
}
```

Tree에서 Drag를 체크?

```cpp
if (ImGui::TreeNodeEx(Name, Flag))
{
	if (m_Owner->IsDrag())
	{
		if (ImGui::BeginDragDropSource()) // 마우스가 눌린 상태에서, 조금이라도 위치가 움직인 경우
		{
			// DragDrop을 통해 보내려는 payload (payload : 탑재, 수하물 등등 ...)
			// 드래그가 시도된 노드의 주소값 원본을 데이터로 전송하기 위해 
			// this의 주소가 담긴 이중 포인터,
			// 노드 포인터의 size를 할당
			TreeNode* pThis = this;
			ImGui::SetDragDropPayload(m_Owner->GetName().c_str(), this, sizeof(TreeNode));
			ImGui::Text("test");           // 드래그 시 마우스와 함께 이동되는 Text
			ImGui::EndDragDropSource();
		}
	}
}
```

- Tree의 이름을 통해, 어떤 Tree에서 보내왔는지를 확인
- Drag가 시도된 Node의 주소값 원본을 payload를 통해 데이터로 전송하기 위해서,
    - this (현재 Node)의 주소가 담긴 이중 포인터
    - TreeNode* 의 size를 할당해
    
    → 해당 데이터를 확인하는 곳에서는 data를 접근해보면, TreeNode의 포인터를 받을 수 있어 해당 포인터로 TreeNode의 원본에 접근하는 것!
    

지금 Drag 되고 있는 / Drop되고 있는 노드를 기록해두기

```cpp
class TreeUI

void SetDragedNode(TreeNode* _Node);
void SetDroppedNode(TreeNode* _Node);
```

```cpp
void TreeUI::SetDragedNode(TreeNode* _Node)
{
	m_DragedNode = _Node;
}

void TreeUI::SetDroppedNode(TreeNode* _Node)
{
}
```

마우스가 Release되는 순간에는 Drag & Drop된 노드가 모두 해제

```cpp
if (ImGui::IsMouseReleased(ImGuiMouseButton_Left))
{
	m_DroppedNode = m_DragedNode = nullptr;
}
```

Mesh Render에서 InputText에 Drop을 받게 하려면 → InputText 자체가 Drop 체크를 해야 한다!

```cpp
if (ImGui::BeginDrapDropTarget())
{
	const ImGuiPayload* payload = ImGui::AcceptDragDropPayload("ContentTree");
	
	if (payload)           // payload가 nullptr이 아닐 경우
	{
		TreeNode* pNode = *((TreeNode**)payload->Data);
		Ptr<CAsset> pAsset = (CAsset*)pNode->GetData();
		
		// Mesh일 경우에만 MeshRender의 
		if (ASSET_TYPE::MESH == pAsset->GetAssetType())
		{
			pMeshRender->SetMesh((CMesh*)pAsset.Get());
		}
	}
	
	ImGui::EndDragDropTarget();
}
```

Tree UI의 노드에게도 Drop이 이루어지길 원한다면, Drop Check를 받으면 된당 

```cpp

```

+ 노드 쪽 입장에서는 Drag & Drop을 함수화 시키기

```cpp
// Drag Check
// Drop Check
```

```cpp
void TreeNode::DragCheck()
{
	isDrag
}
```

```cpp
void TreeNode::DropCheck()
{
	if (!m_Owner->IsDrop())
		return;
		
	if (ImGui::
}
```

Tree 입장에서 자신이 Payload를 받을 목록들이 있어야 한다?

→ payLoad의 이름을 미리 받아놓거나..

```cpp

void SetDropPayLoadName() { };                     // 드랍될 때 PayLoad의 이름을 세팅
const string GetPayLoadName() { return m_DropPayLoadName; }
```

```cpp

{
	m_DroppedNode = _Node;
	
	// ...
}
```

case 1️⃣) 외부에서 Drag가 , Tree UI에서 Drop이 이루어진 경우 (외부 데이터가 트리로 드랍된 경우)

case 2️⃣) Drag된 노드, Drop된 노드 둘 다 TreeUI 소속인 경우 → Self로 Drag & Drop이 이루어졌다!

```cpp
void TreeUI::SetDroppedNode(TreeNode* _Node)
{
	// 1️⃣ 
	if (m_DragedNode == nullptr)
	{
	}
	// 2️⃣
	else
	{
		// 마우스를 Drag한 후에 Hover만 해도 조건이 들어오기 때문에, 
		// PayLoad를 활용해 Release때 동작하도록 사용
		const ImGuiPayload* payload = ImGui::AcceptDragDropPayload(GetName().c_str();
		if (payload)
		{
			m_DroppedNode = _Node;
			
			if (m_SelfDragDropInst && m_SelfDragDropFunc)
			(m_SelfDragDropInst->*m_SelfDragDropFunc)((DWORD_PTR)m_DragedNode, (DWORD_PTR)m_DroppedNode);
		}
	
	}
}
```

→ Delegate 선언

```cpp
EditorUI*    m_SelfDragDropInst;
DELEGATE_2   m_SelfDragDropFunc;

EditorUI*    m_DropInst;
DELEGATE_1   m_DropFunc;

// ...

void AddDragDropDelegate(EditorUI* _Inst, DELEGATE_2 _Func) { m_SelfDragDropInst = _Inst; m_SelfDragDropFunc = _Func; }
```

Outliner → Self Drag Drop Delegate를 등록

```cpp
m_Tree->AddDragDropDelegate(this, (DELEGATE_2)&Outliner::GameObjectAddChild));
```

부모 자식 관계로 계층 구조화

```cpp
void GameObjectAddChild(DWORD_PTR _Param1, DWORD_PTR _Param2);
```

```cpp
void Outliner::GameObjectAddChild(DWORD_PTR _Param1, DWORD_PTR _Param2)
{
	TreeNode* pDragNode = 
}
```

Outliner에서 외부에 있는 노드를 받아와본다면? (test용 기능)