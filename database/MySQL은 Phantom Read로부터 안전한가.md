## REPEATABLE READ : 한 트랜잭션 내에서 동일한 결과를 보장

일반적으로 RDBMS는 변경 전의 레코드를 언두 공간(Undo space)에 백업해둔다.

변경 전/후 데이터를 모두 가지므로, 동일한 레코드에 대해 여러 버전의 데이터가 존재한다고 하여 이를 MVCC(다중 버전 동시성 제어) 라고 한다. 

MVCC를 통해 트랜잭션이 롤백된 경우에 데이터를 복원할 수 있고, 서로 다른 트랜잭션간에 접근 가능한 데이터를 제어할 수 있다.

트랜잭션은 각각의 고유번호(ID)가 존재하며 언두 영역에 저장되는 백업 로그에는 트랜잭션 번호가 기록된다.

B가 SELECT문으로 조회를 하고, 결과는 1건이다.

아직 해당 트랜잭션은 종료 전이다.

![Untitled (22)](https://github.com/wonjunYou/WIL/assets/59856002/9da88f9c-07f7-42ce-a9be-76f409fee9a7)

![Untitled (23)](https://github.com/wonjunYou/WIL/assets/59856002/cc8cfc67-ac02-4a5c-8827-787e7314969e)

이때 다른 사용자 A의 트랜잭션에서 id=50인 레코드를 갱신하는 상황이라고 하자.

그러면 MVCC를 통해 기존 데이터는 변경되지만 백업된 데이터가 언두 로그에 남게 된다.

![Untitled (24)](https://github.com/wonjunYou/WIL/assets/59856002/1616e9b4-b6ac-4d38-9061-3501850dd499)

이때 B가 처음 시작하여 종료되지 않은 트랜잭션에서 다시 select문을 실행하면?

b 트랜잭션은 a 트랜잭션 시작 전에 이미 시작되었다. 이때 repeatable read는 자신보다 먼저 실행된 트랜잭션의 데이터를 조회한다. 만약 테이블에 자신보다 이후에 실행된 트랜잭션 데이터가 commit 되었다면? → 이때 언두로그를 참고하여 데이터를 조회한다.

따라서 B의 조회 결과는 A가 변경한 결과가 반영되지 않은 이전 데이터를 그대로 얻는다. 즉 mvcc를 이용해 phantom read는 발생하지 않는 것이다.

그렇다면 언제 유령 읽기가 발생하는가? 바로 잠금이 사용되는 경우이다. MySQL은 다른 RDBMS와 다르게 특수한 갭 락이 존재한다.

사용자 B가 데이터를 조회할 때, select ~~ for update를 이용해 쓰기 잠금(베타적 잠금, 비관적 잠금)을 건다. 읽기 잠금은 select ~~ for share 이다.

![Untitled (25)](https://github.com/wonjunYou/WIL/assets/59856002/c18b9da7-ec1b-4f8b-bab2-b8a5e4b812b7)

하지만 id = 50만 락이 걸리므로, 결과적으로 Phantom read가 발생한다.

이 현상은 MVCC로 해결할 수 없다. 왜일까?

MVCC는 데이터를 먼저 테이블에 반영하고, 언두로그에 백업한다.

위의 경우에는 SELECT FOR UPDATE로 잠금을 걸어도 테이블에는 커밋되나, 언두로그에는 이전 트랜잭션의 데이터가 쌓인다.

언두로그는 append only 형태이므로 락을 걸 수 없고, select for update, select for share 로 레코드 조회시 언두 영역 데이터가 아니라 테이블의 레코드를 가져오게 되어 Phantom Read가 발생한다.

### 하지만 MySQL에서는 갭 락이 존재하므로, MVCC의 한계를 극복한다.

![Untitled (26)](https://github.com/wonjunYou/WIL/assets/59856002/d77b044e-9c61-4f73-9446-63b2c311e4cb)

MYSQL의 갭 락이란 ********레코드 락 + 넥스트 키 락을********  합친 형태로, 해당 ID = 50에는 레코드 락, 50 이후의 범위에는 넥스트 키 락을 건다. 따라서 A의 INSERT 쿼리는 B의 트랜잭션이 종료된 이후에 발생한다.

따라서 일반적인 MySQL의 REPEATABLE READ에서는 Phantom Read가 발생되지 않는다.

### 그러나 MySQL도 Phantom Read로부터 완전하게 안전한 것은 아님!

하지만 이 첫 조회를 잠금 없이 하고, 두번째에 select for update로 조회를 하면?

![Untitled (27)](https://github.com/wonjunYou/WIL/assets/59856002/37cb8177-470c-4414-9c26-a9dcd68e5593)

언두 로그가 아닌 테이블에서 레코드를 조회하므로 Phantom Read가 발생한다.

그러나 이런 케이스는 굉장히 드물기에, MYSQL PEPEATABLE READ에서는 Phantom Read가 발생하지 않는다고 한다.

마지막으로 트랜잭션 내에서 실행되는 SELECT와 트랜잭션 없이 실행되는 SELECT의 차이를 살펴보도록 하자. REPEATABLE READ에서는 트랜잭션 번호를 바탕으로 실제 테이블 데이터와 언두 영역의 데이터 등을 비교하며 어떤 데이터를 조회할 지 판단한다. 즉, 트랜잭션 안에서 실행되는 SELECT라면 항상 일관된 데이터를 조회하게 된다. 하지만 트랜잭션 없이 실행된다면, 데이터의 정합성이 깨지는 상황이 생길 수 있다. 커밋된 데이터만을 보여주는 READ COMMITTED 수준에서는 둘의 차이가 거의 없다

### References
* https://mangkyu.tistory.com/299