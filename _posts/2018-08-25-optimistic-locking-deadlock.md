---
title: Optimistic Locking 을 원했으나 Deadlock이 난다?
layout: post
categories: mysql
---

Pessimistic Locking 과 Optimistic Locking에 대해서는 이미 머리속에 있으나 실제 Optimisitic Locking은 어떻게 사용할까? 어떤 경우에? 그리고 난 Optimistic Locking 처리가 되길 원했는데 Deadlock이 난다면?

E-Commerce 환경에서 주문정보는 잦은 변경이 일어나기 때문에 동시 변경 요청이 많아지게 되고 이 경우 Optimistic Lock을 사용하여 처리하게 된다. 
먼저 아래와 같은 테이블이 있다고 가정하자. (현실세계에서는 더욱 복잡하지만 지금 포인트는 Optimistic Locking이므로 최대한 단순하게 설계해보자.)

![no_foreign_key_deadlock](/assets/img/deadlock/no_foreign_key_deadlock.png)

아래는 Update소스이다.

````
    @Transactional
    public boolean updateOrder(Long id, String delivery) {
        Optional<Orders> ordersOptional = ordersRepository.findById(id);

        if (ordersOptional.isPresent()) {
            Orders orders = ordersOptional.get();
            orders.setDelivery(delivery);

            OrdersUpdateInfo ordersUpdateInfo = new OrdersUpdateInfo();
            ordersUpdateInfo.setUpdateInfo("updated");
            ordersUpdateInfo.setOrders(orders);
            ordersUpdateInfoRepository.save(ordersUpdateInfo);

            orders.getOrdersUpdateInfoList().add(ordersUpdateInfo);

            ordersRepository.save(orders);
        }


        return true;
    }
```
Orders Table에 Optimistic Lock 모드를설정했다. 
```
public interface OrdersRepository extends JpaRepository<Orders, Long> {
    @Lock(LockModeType.OPTIMISTIC)
    Optional<Orders> findById(Long id);
}
```

그리고 위와 같이 Order를 Update해보자. 
```
    @PostMapping("/order")
    public boolean order(@RequestBody OrderDto orderDto) {
        try {
            return orderService.updateOrder(orderDto.getId(), orderDto.getDelivery());
        } catch (OptimisticLockingFailureException e) {
            throw new RuntimeException(e);
        }
    }
```

아래처럼 Optimistic Locking이 일어나도록 해보자. (Test는 iTerm의 동시 요청을 이용했다.)

![multi_request1](/assets/img/deadlock/multi_request1.png)

와우 우리가 예상한대로 Optimistic Locking Fail이다. 이런 경우 OptimisticLockingFailureException 를 잡아서 적절하게 처리해주면 된다. 

자 다시한번 아래처럼 시도해보자

![multi_request_deadlock](/assets/img/deadlock/multi_request_deadlock.png)

어라 이번엔 OptimisticLockingFail이 아니고 LockAquisitionException 이다. 실제 서버로그를 봐도 아래 처럼 나온다.
```
com.mysql.jdbc.exceptions.jdbc4.MySQLTransactionRollbackException: Deadlock found when trying to get lock; try restarting transaction
```

필자가 무슨 마법을 부린건 아니다. 단지 아래와 같이 추가해주었을 뿐이다.

```
ALTER TABLE orders_update_info ADD FOREIGN KEY (orders_id) REFERENCES orders(id);
```

Intellij 의 DB Tool로 이용하면 
![foreign_key_deadlock_diagram](/assets/img/deadlock/foreign_key_deadlock_diagram.png)

단지 FK를 추가했을뿐이다.  사실 고의로 FK를 추가한게 아니다. 필자도 몇몇 회사를 다니다보니 알게된건데 어떤데는 FK를 꼭 넣어서 쓰고, 어떤데는 FK를 절대 넣지 않는다. (물론 대다수가 FK를 사용하지 않았지만 현재 다니고 있는 회사에서 이미 FK가 추가된 상태에서 Optimistic Locking을 예상했는데 deadlock이 나오게 되서 디테일안에 있는 악마를 찾아보기 위해서다.)

그럼 단순히 FK를 제거하면 될까? 물론 그래도 된다. 마음의 평안이 온다.. 하지만 항상 The devil is in the detail. 이다. 문제에 직면했으니 명확하게 왜 이런지 이해하고 가보자. 
mysql에 들어가 Deadlock을 유발하는 녀석이 누군지 찾아보자.
```
------------------------
LATEST DETECTED DEADLOCK
------------------------
2018-08-25 18:34:04 70000eb92000
*** (1) TRANSACTION:
TRANSACTION 9776, ACTIVE 0 sec starting index read
mysql tables in use 1, locked 1
LOCK WAIT 5 lock struct(s), heap size 1184, 2 row lock(s), undo log entries 1
MySQL thread id 3, OS thread handle 0x70000ebd6000, query id 158 localhost 127.0.0.1 root updating
update orders set delivery='2', version=9 where id=1 and version=8
*** (1) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 32 page no 4 n bits 520 index `PRIMARY` of table `optimistic`.`orders` trx id 9776 lock_mode X locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 8; hex 8000000000000001; asc         ;;
 1: len 6; hex 000000002604; asc     & ;;
 2: len 7; hex 03000001a10ba7; asc        ;;
 3: len 1; hex 38; asc 8;;
 4: len 4; hex 80000008; asc     ;;

*** (2) TRANSACTION:
TRANSACTION 9777, ACTIVE 0 sec starting index read
mysql tables in use 1, locked 1
5 lock struct(s), heap size 1184, 2 row lock(s), undo log entries 1
MySQL thread id 2, OS thread handle 0x70000eb92000, query id 159 localhost 127.0.0.1 root updating
update orders set delivery='1', version=9 where id=1 and version=8
*** (2) HOLDS THE LOCK(S):
RECORD LOCKS space id 32 page no 4 n bits 520 index `PRIMARY` of table `optimistic`.`orders` trx id 9777 lock mode S locks rec but not gap
Record lock, heap no 2 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 8; hex 8000000000000001; asc         ;;
 1: len 6; hex 000000002604; asc     & ;;
 2: len 7; hex 03000001a10ba7; asc        ;;
 3: len 1; hex 38; asc 8;;
 4: len 4; hex 80000008; asc     ;;

*** (2) WAITING FOR THIS LOCK TO BE GRANTED:
RECORD LOCKS space id 32 page no 4 n bits 520 index `PRIMARY` of table `optimistic`.`orders` trx id 9777 lock_mode X locks rec but not gap waiting
Record lock, heap no 2 PHYSICAL RECORD: n_fields 5; compact format; info bits 0
 0: len 8; hex 8000000000000001; asc         ;;
 1: len 6; hex 000000002604; asc     & ;;
 2: len 7; hex 03000001a10ba7; asc        ;;
 3: len 1; hex 38; asc 8;;
 4: len 4; hex 80000008; asc     ;;
```

보면 2번 트랜잭션이 S(Shared) Lock을 잡고 있는 상태에서 1,2번이 동시에 update order를 하려고 아니 primary key의 index 에 대해 X(Exclusive) Lock을 잡으려고 하니 Deadlock이 발생한 것이다. 

음.. 그런데 왜 갑자기 생뚱맞게 S Lock이지.. Optimistic Lock이라는게 마지막에 Transaction이 Commit될때 Commit될거라고 생각하고 락을 잡지않고 가다가 막판에 락잡고 하는거 아닌가...

여기서 FK의 함정이 나타난다. JPA로 우리가 Update하려고 할때 먼저 OrderDetail을 업데이트고 보니 여기 FK인 orders_id를 저장한 OrderDetail  Table의 lock덕분에 해당 테이블에 CONSTRAINT를 갖고 있는 Orders 테이블이 S Lock이 걸리게 된 것이다.