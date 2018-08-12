---
title: Synchronous Asynchronous Blocking NonBlocking
layout: post
categories: common
---

Synchronous, Asynchronous, Blocking, NonBlocking

위의 4가지 용어는 개발을 하다보면 참 많이 듣게 된다. 
동기, 비동기, 블록킹, 논블록킹의 언어로 많이 듣는다. 

내가 Aysnchronous를 처음 듣게 된것은 Ajax를 처음으로 접했을 때 Ajax는 비동기로 처리를 해! 라고 하였을때다.
NonBlocking이란 단어를 처음 듣게 된것은 Netty라는 책을 보면서 Netty는 Client의 Request를 Non Blocking하게 처리 해! 라고 하였을때다.

그래서 그냥 둘다 비동기 아니야? 라고 이해하게 된다. 

그냥 그렇게 이해하고 넘어가다 2년전쯤이었던 것으로 기억하는 한 IT회사의 면접에서 
Synchronous, Asynchronous, Blocking, NonBlocking 의 차이점을 설명해보라고 하신다. 
음.. 응?? 응???? 응???????

자 이제 자기가 알고 있다고 생각하지만 모르는 것을 아는 것으로 정리해보자.

![asyncsyncblockingnonblocking](/assets/img/syncasyncblockingnonblocking/asyncsyncblockingnonblocking.png)

위의 4가지 용어에 대해 공부하다 보면 위의 그림을 많이보게된다. 그리고 그 위의 그림에 대해 많이 설명된 좋은 Post를 Google께서 많이 찾아주신다. Post 들을 읽다보면 개념은 이해가 되는 것 같은데 몇 일 후에 다시보면 다시 새롭게 보인다. 즉 이해가 가지 않은데 이해가 가는 것처럼 느끼게 된다.

이럴때 가장 좋은 건 자기만의 예제를 해당 개념에 적용해보는 것이다. 

아래의 코드를 보자. 아주 간략화 하였지만 주문 정보를 만든다고 하자. 해당 주문을 만드는데 배송 정보는 API로 가져와야 한다고 가정하자. (점점 MSA를 차용하는 곳이 많아지면서 이런 케이스들을 많이 보게 된다.)

![asyncsyncblockingnonblocking](/assets/img/syncasyncblockingnonblocking/synchronous_blocking.png)

위 코드의 경우 해당 코드는 모두 순선대로 진행되어야 한다.  그럼 아래 코드를 보자

![synchronous_nonblocking](/assets/img/syncasyncblockingnonblocking/synchronous_nonblocking.png)

차이점이 보이는가 **Syncrhonous_Blocking** 과 **Synchronous_NonBlocking** 의 차이를 보여준다. 

delivery정보를 가져오는 API가 API Latency가 5초, makeOrder의 처리시간이 5초이면 위의 케이스는 주문을 만드는데 총 10초의 시간이 걸리게 된다. 
즉 delivery정보를 가져오는 동안 해당 Thread는 펑펑 놀면서 5초를 기다린 후 해당 정보를 받은 후에 주문정보를 만들게 된다.

그럼 두번째 케이스를 보자. Java의 CompletableFutrue를 이용하였다. 해당 케이스는 일단 배달정보를 가져오라고 요청한 후 해당 Thread는 열심히 주문 정보를 만들기 시작한다. 주문 정보를 다 만든 후에 deliveryFutrue에게 물어본다. 배송정보를 다 가져왔는지, 그리고 다 가져왔다면 해당 정보를 조합하여 주문정보 작업을 완료한다. 즉, 배송정보를 가져오는 작업에 대해서 해당 Thread가 Blocking 되지 않는다.

**나는 방금 위에서 Concurrency를 보았다.**

자 그럼 이제 Aysnchronous를 생각해보자. Synchronous라는 개념을 고려하지 않을때 Blocking, NonBlocking만 보았을 때 아 Blocking이 되지 않고 작업이 된다라고 이해가 된다. Asynchronous를 우리는 비동기 처리라고 한다. 비동기란 말을 풀어보면 동기가 아닌거다. 동기가 아닌건 무슨 뜻일까. 사실 동기라는 뜻을 찾아보면 교류 장치에 있어서의 주파수의 일치 (https://hanja.dict.naver.com/search?query=%EB%8F%99%EA%B8%B0) 라고 나온다...

비동기라고 생각하지 말고 Syncrhonous / Asynchronous 단어 자체로 개념을 이해하는게 좋을것 같다는 생각이 들었다. Apple이 사과가 아니라 우리가 먹는 과일중의 하나의 이미지로 받아들이듯이..(웅??)

![asynchronous_nonblocking](/assets/img/syncasyncblockingnonblocking/asynchronous_nonblocking.png)

AysncRestTemplate를 이용해서 Async를 구현했다. 우리는 여전히 배송 정보를 가져와서 주문정보를 가져왔다. 그런데 Spec이 변경되었다. 배송정보를 가져와서 해당 정보를 이용해서 고객의 배송 정보를 변경하고 싶다. 그래서 해당 정보를 가져온 후에 Asynchronous하게 고객정보 업데이트를 요청했다. 

자 이제 개념정리를 잠시 하고 가보자. 

위에서도 한번 짚었듯이 Blocking, NonBlocking은 일의 순서에 있어서 해당일을 기다려야(Blocking)할지 기다리지 말아야 할지를 정의한다. 그럼 Syncronous와 Asynchronous를 본다면 어떨까. 위의 두 개념을 잘 이해해보고자 Caller, Callee 를 적용해보자 

주문을 만들어야하는 Caller입장에서 Order의 Address 정보를 만들기 위해서는 Callee에게 받은 정보를 마지막까지 신경써서 setAddress를 한다. **Synchronous** 이다.   하지만 고객의 정보를 변경하는 부분은 주문을 만드는 Caller입장에서는 관심이 없다. 배송정보를 가져오면서 Callback을 통해 고객정보를 변경하라고 한다. 여기서 헷갈릴 수 있는 부분인데 이 Callback Function은 주문을 만들어야하는 Caller의 관심사가 아니다. Callback Function이 하는 거다. **Asynchronous** 이다.

* Synchronous Asynchornous
	* 내가 호출한 결과물에 관심이 있으면 Synchronous, 내가 호출한 결과물에 관심이 없으면 Asynchronous. (이렇게 생각해보니 교류 장치에 있어서의 주파수의 일치 가 말이 되는것 같다..)
* Blocking Non Blocking
	* 내가 작업을 호출 한 후 다음 작업까지 기다려야한다면 Blocking, 다음 작업을 바로 실행할 수 있으면 Non Blocking ( **Concurrency** 내가 자꾸 Concurrency를 언급하는 이유는 은근히 이걸 Parallel과 혼동하는 경우가 있다)