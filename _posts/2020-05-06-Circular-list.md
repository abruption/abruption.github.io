---
title: 원형리스트에서의 노드 삽입 알고리즘
author: LEE YEONGUK
date: 2020-05-06 15:06:00 +0900
categories: [Leature, Data Structure]
tags: [Circular List, Data Structure]
---

## 원형연결리스트에서 노드를 삽입하는 알고리즘

~~~cs
insertFirstNode(CL, x)
new ← getNode();
new.data ← x;
if (CL = null) then { // ① 
CL ← new; // ①-ⓐ
new.link ← new; // ①-ⓑ
}
temp ← CL; // ②
while (temp.link ≠ CL) do // ③ 
temp ← temp.link;
new.link ← temp.link; // ④
temp.link ← new; // ⑤
CL ← new; // ⑥
end insertFirstNode()
~~~
> 원형연결리스트에서 처음 노드로 삽입하는 알고리즘   

<br/>

- 원형 연결 리스트에서의 삽입연산은 거의 단순 연결 리스트와 동일하다.
- 마지막 노드의 링크를 첫 번째 노드로 연결하는 것만 단순 연결 리스트와 다르다.
- 리스트의 마지막에 노드를 삽입하는 것이 리스트의 처음에 노드를 삽입하는 것과 같다. 
- 따라서, 처음에 노드를 삽입하는 경우와 중간 노드로 삽입하는 경우로 나눌 수 있다.

<br/>

1. `if(CL = null)` : 공백 리스트인 경우 삽입하는 노드 `new`가 리스트의 첫 번째 노드이자 마지막 노드이다.
   - `CL ← new;` : 리스트의 시작노드에 대한 참조 값을 저장하는 참조변수 CL에 새 노드 new에 대한 참조 값을 삽 입한다. 즉, `CL`이 노드 `new`를 첫 번째 노드로 가리키게 한다.
   - `new.link ← new;` : 참조변수 `new`의 값을 `new`가 가리키는 새 노드의 링크 필드에 저장한다. 즉, `new`가 자기 자신을 가리키게 함으로 new를 원형 연결 리스트 CL의 첫 번째이나 마지막 노드로 지정한다.
2. `temp ← CL` : CL의 첫 번째 노드에 대한 참조 값(시작 위치)을 임시 순회 참조변수 `temp`에 저장한다.
3. `while (temp.link ≠ CL) do temp <- temp.link;` : `while`문으로 `temp`가 마지막 노드를 가리키도록 한다.
4. `new.link ← temp.link;` : temp가 가리키는 노드(마지막 노드)의 링크 값을 new의 링크에 저장한다. 즉, 노드 `new`가 노드 `temp`의 다음 노드를 가리키게 한다. `CL`은 원형 연결 리스트이므로 마지막 노드의 다음 노드는 첫 번째 노드가 된다.
5. `temp.link ← new;` : `new`의 값을 `temp`가 가리키고 있는 마지막 노드의 링크에 저장한다. 즉, 리스트의 마지막 노드가 노드 `new`를 가리키게 한다.
6. `CL ← new;` : `new`의 값(새 노드에 대한 참조 값)을 `CL`에 저장하여 노드 `new`가 리스트의 첫 번째 노드가 되게한 다.
