---
title: 엑셀 업로드를 하였는데 db update 2000건 처리가 4시간? (feat. OSIV)
layout: post
categories: hibernate
---

서비스를 개발하다 보면 가끔 그런 요청이 온다.

엑셀 파일에 주문정보가 한 2천건쯤 있고 해당 건들에 대해 추가적으로 이런저런요런 처리를 해주세요. 라고..
그리고 급해요~ 라고..

그럼 가장 1차적으로 생각하게 되는 것은 해당 엑셀을 업로드 할 수 있는 기능을 만들고, 해당 엑셀을 row by row로 읽으면서 주문 정보를 들고 와서 업데이트를 진행한다.

이 경우 해당 주문 정보를 변경하는 method를 하나의 Transaction으로 묶고 해당 method를 for문을 돌면서 실행하면서 실패하는 경우 try - catch 로 묶어서 실패한 주문번호들을 모아 최종적으로 return한다.
```
public List<Long> process() {
    List<Long> failedOrderList = new ArrayList<>();
    Stream.iterate(1L, a -> a + 2).limit(600).collect(Collectors.toList())
    .forEach(a -> {
        try {
            orderService.updateOrder(a, String.valueOf(a));
        } catch (Exception e) {
            failedOrderList.add(a);
        }
    });

    return failedOrderList;
}
```

단순하게 표현하자면 위와 같다. 

필자의 경우 해당 기능을 구현 후 외부에 일이 있어 옆 개발자에게 해당 엑셀 업로드를 운영이 요구한 시간에 맞추어 업로드만 해주면 된다고 하고 외부에 나갔으나 옆 개발자께서 필자님 이거 끝날 생각을 안하는데요~ 
하면서 아래와 같은 그래프를 떡하고 보여주었다.
![osiv_memory](/assets/img/osiv/osiv_memory.png)

결국 4시간을 기달려 해당 처리는 마무리가 되었고, 해당 이슈에 대해 파보기로 하였다. 

위의 그림은 필자의 회사가 서버에 로그를 남길때마다 ELK에서 관련 로그를 쌓는데, 해당 request의 transactionId를 기준으로 조회했을때의 모습이다. 한개의 처리를 할때 관련 로그가 10개 가까이 남기 때문에 위의 그림처럼 2만개 가까운 로그가 남은것이다. 

문제는 로그의 개수는 그렇다치고 고작 2천개의 db처리를 하는데 이 정도의 시간이 걸려야 하는 것이었다. 그리고 그래프의 그림이 메모리를 한번 크게 점유한 후 해당 메모리가 해제되면서 처리되는 모습으로 보였다. 

이 문제의 원인은 OSIV였다. 자 그럼 왜 OSIV인지 알아보자.
![entitymanager](/assets/img/osiv/entitymanager.png)

위의 그림은 
> 자바 ORM 표준 JPA 프로그래밍
에서 참조 했다.

그림에서 보는 바와 같이 API 요청이 들어오면 그 즉시 EntityManager가 생성이 돼고 해당 Entity Manager는 Request의 요청이 끝날 때까지 살아 있게 된다. 

자 그럼 다시 우리 문제로 돌아와서 OSIV 가 이 문제에 어떤 영향을 미쳤는지 생각해보자. (참고로 Spring Boot의 경우 따로 설정하지 않으면 spring.jpa.open-in-view의 값이 true이다...........)

EntityManager는 request가 들어올때 생성되고 request가 종료될때 같이 종료된다. (즉 메모리에서 사라진다.). 필자의 구현은 for문안에 Transaction method를 2천번 실행하는 동안 해당 Transaction은 종료가 되었지만 해당 Transaction에서 사용된 (정확히는 주문 정보를 변경하기 위해 orderRepository.findOne(id); 에서 사용한 Orders Entity) Entity들이 PersistenceContext에는 Request가 끝날때까지는 살아남는다라는 OSIV의 정책을 지키고자 마구 마구 EntityManager에 해당 Entity가 저장된 것이다. 이로 인해 메모리 이슈가 발생하게 됨으로 인해 고작 2천건을 처리하는데 저렇게 많은 시간이 걸리게 되었다.

OSIV 세팅을 spring.jpa.open-in-view=false로 하여 해당 문제를 해결할 수 있었으나 이미 프로젝트는 거대해졌고 해당 OSIV 설정을 false로 하게 되면 어떤 다른곳에 side-effect가 발생할지 예측할 수 없어서 아래와 같이 해결하였다.

```
public List<Long> process() {
    List<Long> failedOrderList = new ArrayList<>();
    Stream.iterate(1L, a -> a + 2).limit(600).collect(Collectors.toList())
    .forEach(a -> {
        try {
            orderService.updateOrder(a, String.valueOf(a));
            entityManager.clear();
        } catch (Exception e) {
            failedOrderList.add(a);
        }
    });

    return failedOrderList;
}
```

메모리가 정말로 느는지에 대해서는 해당 부분을 검증해 볼 수 있는 Code 및 시연결과를 빠른 시일내에 업데이트 하도록 하겠다.