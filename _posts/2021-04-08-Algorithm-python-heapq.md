---
title: 파이썬 - 힙 큐
author: bseony2
date: 2021-04-08 00:00:00 -0000
categories: [Algorithm, data-structure]
tags: [알고리즘, 자료구조]
---

데이터를 정렬된 상태로 저장하기 위해서 사용하는 파이썬의 heapq(힙큐)

## 힙 자료구조
heapq 모듈은 이진 트리(binary tree) 기반의 최소 힙(min heap) 자료구조를 제공한다
자바의 PriorityQueue 클래스와 비슷하다

min heap을 사용하면 원소들이 항상 정렬된 상태로 추가되고 삭제되며, min heap의 최소값 언제나 인덱스 0, 즉, 이진 트리의 루트에 위치합니다. 내부적으로 min heap 내의 모든 원소(index i)는 항상 자식 원소들(2i+1, 2i+2)보다 크기가 작거나 같도록 원소가 추가되고 삭제된다

```heap[k] <= heap[2*k+1] and heap[k] <= heap[2*k+2]```

아래 그림은 위 공식을 만족시키는 간단한 min heap의 구조를 보여주고 있다

![](https://images.velog.io/images/bseony2/post/84039bae-4efd-4bb9-9452-6c49a62cbdfa/image.png)
## 모듈 임포트
우선 **heapq** 모듈은 내장 모듈이기 때문에 아래처럼 임포트만 시키면 사용할 수 있다

```import heapq```
## 최소 힙 생성
**heapq** 모듈은 파이썬의 보통 리스트를 최소 힙처럼 다룰 수 있도록 도와준다 
자바의 **PriorityQueue** 클래스처럼 하나의 데이터 타입이 아니다

때문에, 힙으로 사용할 리스트를 생성 후 heapq 모듈을 호출하여 리스트를 인자로 넘겨야 한다
다시말해, 파이썬에서는 heapq 모듈을 사용하여 요소를 추가 / 삭제 한 **리스트**를 최소힙으로 사용한다.
```heap = []```

## 힙에 원소 추가
**heapq** 모듈의 **heappush()** 함수를 이용하여 힙에 원소를 추가할 수 있다.
첫번째 인자는 원소를 추가할 대상 리스트이며 두번째 인자는 추가할 원소를 넘긴다.
```
heapq.heappush(heap, 4)
heapq.heappush(heap, 1)
heapq.heappush(heap, 7)
heapq.heappush(heap, 3)
print(heap)
```
```
[1, 3, 7, 4]
```
가장 작은 1이 인덱스 0에 위치하며, 인덱스 1(= i)에 위치한 3은 인덱스 3(= 2i + 1)에 위치한 4보다 크므로 힙의 공식을 만족한다.
내부적으로 이진 트리에 원소를 추가하는 **heappush()** 함수는 **O(logN)**의 시간 복잡도를 가집니다.

## 힙에서 원소 삭제
**heapq** 모듈의 **heappop()** 함수를 이용하여 힙에서 원소를 삭제 및 추출(?)
원소를 삭제할 대상 리스트를 인자로 넘기면, 해당 힙의 루트를 꺼낸다.
```
print(heapq.heappop(heap))
print(heap)
```
```
1
[3, 4, 7]
```
가장 작았던 1이 삭제되어 리턴되었고, 그 다음으로 작었던 3이 인덱스 0으로 올라왔다.
```
print(heapq.heappop(heap))
print(heap)
```
```
3
[4, 7]
```
가장 작었던 3이 삭제되어 리턴되었고, 그 다음으로 작았던 4가 인덱스 0으로 올라왔다. 
내부적으로 이진 트리로 부터 원소를 삭제하는 **heappop()** 함수도 역시 **O(logN)**의 시간 복잡도를 가진다.

## 최소값 삭제하지 않고 얻기
**heappop**은 해당 힙에서 루트를 삭제한다.
힙에서 최소값을 삭제하지 않고 단순히 읽기만 하려면 일반적으로 리스트의 첫번째 원소에 접근하듯이 인덱스를 통해 접근해야한다.
```
print(heap[0])
4
```
여기서 주의사항은 인덱스 0에 가장 작은 원소가 있다고 해서, 인덱스 1에 두번째 작은 원소, 인덱스 2에 세번째 작은 원소가 있다는 보장은 없다. 최소 힙이라는 자료구조는 그저 부모노드가 자식노드보다 작다를 만족시킬 뿐이기 때문이다.

따라서 두번째로 작은 원소를 얻으려면 바로 heap[1]으로 접근하면 안되고, 반드시 heappop()을 통해 가장 작은 원소를 삭제 후에 heap[0]를 통해 새로운 최소값에 접근해야 한다.

## 기존 리스트를 힙으로 변환
**heapify()**함수를 통해 기존 리스트를 힙으로 변경할 수도 있다.
```
heap = [4, 1, 7, 3, 8, 5]
heapq.heapify(heap)
print(heap)
[1, 3, 5, 4, 8, 7]
```
**heapify()** 함수에 리스트를 인자로 넘기면 리스트 내부의 원소들의 위에서 다룬 힙 구조에 맞게 재배치되며 최소값이 0번째 인덱스로 바뀐다.

## [응용] 최대 힙
heapq 모듈은 최소 힙(min heap)을 기능만을 동작하기 때문에 최대 힙(max heap)으로 활용하려면 약간의 요령이 필요합니다. 바로 힙에 튜플(tuple)를 원소로 추가하거나 삭제하면, **튜플 내에서 맨 앞에 있는 값을 기준**으로 최소 힙이 구성되는 원리를 이용한다.

따라서, 최대 힙을 만들려면 각 값에 대한 우선 순위를 구한 후, **(우선 순위, 값)** 구조의 **튜플(tuple)**을 힙에 추가하거나 삭제하면 된다. 그리고 힙에서 값을 읽어올 때는 각 튜플에서 인덱스 1에 있는 값을 취하면 된다. (우선 순위에는 관심이 없으므로 )
```
import heapq

nums = [4, 1, 7, 3, 8, 5]
heap = []

for num in nums:
  heapq.heappush(heap, (-num, num))  # (우선 순위, 값)

while heap:
  print(heapq.heappop(heap)[1])  # index 1
8
7
5
4
3
1
```
## [응용] K번째 최소값/최대값
최소 힙이나 최대 힙을 사용하면 K번째 최소값이나 최대값을 효츌적으로 구할 수 있다.
```
import heapq

def kth_smallest(nums, k):
  heap = []
  for num in nums:
    heapq.heappush(heap, num)

  kth_min = None
  for _ in range(k):
    kth_min = heapq.heappop(heap)
  return kth_min

print(kth_smallest([4, 1, 7, 3, 8, 5], 3))
4
```
K번째 최소값을 구하기 위해서는 주어진 배열로 힙을 만든 후, **heappop()** 함수를 K번 호출하면 됩니다.

## [응용] 힙 정렬
힙 정렬(heap sort)은 위에서 설명드린 힙 자료구조의 성질을 이용한 대표적인 정렬 알고리즘입니다.
```
import heapq

def heap_sort(nums):
  heap = []
  for num in nums:
    heapq.heappush(heap, num)
  
  sorted_nums = []
  while heap:
    sorted_nums.append(heapq.heappop(heap))
  return sorted_nums

print(heap_sort([4, 1, 7, 3, 8, 5]))
[1, 3, 4, 5, 7, 8]
```

## heapq를 응용한 우선순위 큐 
heappush(heap, item)를 사용할 때 item 파라미터에는 꼭 정수만이 아닌 리스트를 넣어도 된다.
그말인즉슨 리스트 내의 (정렬 기준 값 : 데이터)형식dmf 
ex) => [[no, item], [no, item]]을 heapq.heapify() 사용하여 힙 구조를 만들 수 있다는 뜻이다.
이 [우선순위 값, 데이터] 형식을 사용하면 heapq를 사용하여 우선순위 큐를 구현할 수 있게된다.

python에는 PriorityQueue 클래스가 있어 굳이 heapq를 사용하지 않아도 우선순위 큐를 사용할 수 있다. 하지만 heapq는 thread-safe를 지원하지 않는 모듈이고 PriorityQueue는 heapq을 사용한 thread-safe를 지원하는 클래스이다.
즉, 내부적인 lock을 사용하냐 마냐의 차이가 있다는 의미이다. 이 lock에 의해 heapq와 PriorityQueue는 속도 측면에서 차이를 보이기 때문에 멀티 쓰레딩 환경이 아니라면 PirorityQueue보다는 heapq를 사용하는 것이 유리하다.