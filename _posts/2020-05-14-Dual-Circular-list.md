---
title: 이중 원형 연결리스트에서의 노드 삽입&삭제 알고리즘
author: LEE YEONGUK
date: 2020-05-14 10:53:00 +0900
categories: [Leature, Data Structure]
tags: [Circular List, Data Structure]
---

## 이증 원형 연결리스트에서 노드를 삽입&삭제하는 알고리즘

~~~cs
insertNode(DL, pre, x)
    new <- getNode();
    new.data <- x;
    new.rlink <- pre.rlink;
    pre.rlink <- new;
    new.llink <- pre;
    new.rlink.llink <- new;
end insertNode()
~~~
> 이중 연결 리스트에서의 삽입 알고리즘    

<br/>

1. `new.rlink ← pre.rlink` : 노드 `pre`의 `rlink` 값을 노드 `new`의 `rlink`에 저장한다. 즉, 노드 `pre`의 오른쪽 노드(`pre` 다음 노드의 `llink`)를 삽입할 노드 `new`의 오른쪽 노드로 연결한다.
2. `pre.rlink ← new` : 새 노드 `new`의 참조값을 노드 `pre`의 `rlink`에 저장한다. 즉, 노드 `new`를 노드 `pre`의 오른쪽 노드로 연결한다.
3. `new.llink ← pre` : 참조변수 pre의 값을 삽입할 노드 new의 llink에 저장한다. 즉, 노드 `pre`를 노드 `new`의 왼쪽 노드로 연결한다
4. `new.rlink.link ← new `: `new`의 값을 노드 `new`의 오른쪽 노드(`new.rlink`)의 `llink`에 저장한다. 즉, 노드 `new`의 오른쪽 노드의 왼쪽 노드로 노드 `new`를 연결한다.