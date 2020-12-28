---
title: 원형 Queue(큐)에서의 삽입&삭제 연산
author: LEE YEONGUK
date: 2020-06-10 00:22:00 +0900
categories: [Leature, Data Structure]
tags: [Queue, Data Structure]
---

## 원형 큐(Queue)
- `front`와 `rear`의 초기 값은 0으로 설정한다. `front`와 `rear`은 시계 방향으로 진행되며, `front = rear`이 되면 `Queue`에서는 데이터가 저장되어 있지 않은 빈 상태를 뜻한다.
- 원형 `Queue`를 유지하기 위해서 `mod` 연산을 해 주는 `%`를 사용한다.
- 데이터를 `Queue`에 삽입하기 위해서는 `rear`를 하나씩 증가시킬 때 : `rear ← (rear+1)%ArraySize`
- 데이터를 `Queue`로부터 삭제할 때 : `front ← (front+1)%ArraySize`

<br/><br/>

## 연결 자료구조를 활용한 연결 Queue에서의 삽입&삭제 알고리즘
<br/>

### 연결 자료구조를 활용한 연결 Queue의 삽입 알고리즘
~~~bash
enQueue(LQ, item)
   newnode <- getNode();   
   newnode.data <- item;
   newnode.link <- null;
   if(front=null) then {  
      rear <- newnode;
      front <- newnode;
   } else {   
      rear.link <- newnode;
      rear <- newnode;
   }
end enQueue()
~~~

1. `newnode ← getNode()` : 삽입할 새 노드를 생성하여 데이터 필드에 `item`을 저장한다.
2. `if(front=null) then { ~ }` : 새 노드를 삽입하기 전에 연결 `Queue`가 공백인지 아닌지를 검사한다. 연결 `Queue`가 공백인 경우에는 삽 입할 새 노드가 `Queue`의 첫 번째 노드이자 마지막 노드이므로 `front`와 `rear`가 모두 새 노드를 가리키도록 설정한다.
3. `else { ~ }` : `Queue`가 공백이 아닌 경우, (노드가있는경우) 현재 `Queue`의 마지막 노드 뒤에새 노드를 삽입한다. 그리고 마지막 노드를 가리키는 `rear`가 삽입한 새 노드를 가리키도록 설정한다.


<br/><br/>

### 연결 자료구조를 활용한 연결 Queue의 삭제 알고리즘
~~~bash
deQueue(LQ)
   if(isEmpty(LQ)) then Queue_Empty();
   else {
      old <- front;         
      item <- front.data;
      front <- front.link;   
      if (isEmpty(LQ)) then rear <- null; 
      returnNode(old);     
      return item;
}
end deQueue()
delete(LQ) 
         if(isEmpty(LQ)) then Queue_Empty(); 
         else { 
                 old ← front; 
                 front ← front.link; 
                 if(isEmpty(LQ)) then rear ← null; 
                 returnNode(old); 
         } 
end delete()
~~~
> `Queue`에서 삭제 연산은 `Queue`가 공백이 아닌 경우에만 수행한다.

1. `old ← front` : `front`가 가리키는 노드를 포인터 `old`가 가리키게 하여 삭제할 노드를 지정한다.
2. `front ← front.link` : 삭제연산 후에는 현재 `front` 노드의 다음 노드가 `front`(첫번째) 노드 가 되어야 하므로 포인트 `front`를 재설정한다.
3. `if(isEmpty(LQ)) then rear ← null` : 현재 `Queue`에 노드가 하나인 경우 삭제연산 이후 공백 `Queue`가 되는 경우에는 `Queue`의 마지막 노드가 없어지므로 포인터 `rear`를 `null`로 설정한다.
4. `returnNode(old)` : `old`가 가리키고 있는 노드를 삭제하여 메모리 공간을 시스템에 반환한다.

