---
title: 연결리스트에서의 노드 삽입 알고리즘
author: LEE YEONGUK
date: 2020-05-04 00:56:00 +0900
categories: [Leature, Data Structure]
tags: [Linked List, Data Structure]
---

## 연결리스트에서 마지막 노드에 새로운 노드를 삽입하는 경우

~~~cs
insertLastNode(list, x)
	newnode ← getNode();
	newnode.data ← x;
	newnode.link ← null;
	//list가 null인 경우
	if (list = null) then {............①
		list ← newnode;
		return;
	}	                ............②
	temp ← list;              ..........②-ⓐ
	//list가 null이 아닌 경우
	while (temp.link ≠ null) do
	        temp ← temp.link;   ............②-ⓑ
	temp.link ← newnode;    ............②-ⓒ
end insertLastNode()
~~~
> 연결리스트에서 마지막 노드에 삽입하는 경우의 알고리즘   

<br/>

- 마지막 노드로 `newnode`를 삽입하기 위해 마지막 노드를 찾아야한다.
- 임시 참조변수 `temp`를 이용하여 리스트의 노드를 순회하여 마지막 노드를 찾는다.

<br/>

1. `if(list = null)` : `list`가 공백 리스트인 경우에 마지막 노드 삽입 연산은 중간노드로 삽입하는 과정과 같다
2. `list`가 공백이 아닌 경우
    - `temp←list` : 마지막 노드로 삽입하기 위해서는 마지막 노드를 먼저 찾아야 한다. 따라서 링크 필드를 따라 노드를 순회할 임시 참조변수 `temp`에 리스트의 첫 번째 노드 주소를 저장한다.
    - `while(temp.link ≠ null) do temp←temp.list` : `while` 반복문을 수행하는 동안 `temp`가 노드의 링크 필드를 따라 이동한다. 그리고 링크 필드가 null인 마지막 노드를 찾는다.
    - `temp.link←newnode` : `temp`가 가리키는 노드(마지막 노드)의 링크 필드에 삽입할 새 노드 `newnode`의 참조 값 을 저장한다. 즉, 리스트 `list`의 마지막 노드 뒤에 새 노드 `newnode`를 연결하게 한다.
