# 우아한 스케줄러

### 개요

스프링 스케줄러를 구성할때 일반적으로 하나의 스레드로 작업을 실행하도록 구성한다. </br>
스케줄 서버를 배포할 때 인스턴스가 동시에 뜨는걸 방지하기 위해 작업 중단을 한 후 새로운 서버에 스케줄러를 배포하면 교체 타이밍에 작업 실행이 누락될 수 있다. </br>
위 문제 해결을 위해 무중단 배포를 수행한다면(블루/그린 배포) 동일 작업이 같은 타이밍에 두번 실행되는 이슈가 발생할 수 있다.</br>
우아한 스케줄러는 항상 하나의 작업만 누락없이 실행되는 걸 보장한다. </br>
분산된 환경에서 작업 획득을 DB 혹은 Redis를 이용해 하나의 서버만 실행되도록 보장한다.</br>



### 기능 요구사항
- 두개의 인스턴스에 스케줄러가 실행되어도 하나의 스케줄러만 작업 실행하도록 보장하는 단일 작업 설정 (클러스터링 기능/ 쿼츠 참고)
- 트리거되는 작업이 DB에 들어오는 경우에만 실행 (ex 이벤트 메시지 발행 작업 처리) 
- 여러개의 인스턴스에서 DB에 들어온 작업을 각 한번만 처리하도록 보장한다.   

![스크린샷 2022-08-23 오전 1 26 07](https://user-images.githubusercontent.com/25260024/185971279-f043964c-09af-4427-9776-ea824c8233f1.png)


#### Scheduled
- 기본 스프링 스케줄러 작업
- 작업 할당도 스프링 스케줄러로 설정한다.
- 작업 할당은 10초마다 돌면서 JobCoordinator에 작업 할당을 요청한다.
#### MethodInterceptor
- Scheduled 어노테이션을 포함한 새로운 어노테이션을 만들어 스케줄 작업 메서드 실행전에 작업 실행 정보를 확인한다.
  - JobManager에서 작업정보를 요청한다. 
  - 작업 정보에서 실행 여부 확인 중 lock이 걸리거나 alloc되어 있지 않으면 실행하지 않는다.
#### JobCoordinator
- getJob() , alloc() 을 제공하는 인터페이스
- JpaJobCoordinator는 jpa와 rdb를 사용해서 작업을 관리한다
- jobAlloc 호출 시 job의 row락을 활용해 하나의 인스턴스만 작업 정보를 가져와서 연장한다.
- JobAlloc/JobAllocHistory
  - JobAlloc은 유니크 id로 인스턴스 id를 가진다. 
  - 새로운 인스턴스가 alloc() 메서드를 호출하고 JobAlloc의 만료시간을 확인해서 아직 만료 시간이 안지났으면 새로운 인스턴스가 JobAlloc을 가져간다 (updateAllocId)
  - 작업이 새로운 인스턴스가 가져갈때 JobAllocHistory가 쌓인다 
  - 같은 인스턴스가 alloc()을 호출하면 만료시간만 증가시키고 계속 스케줄 작업을 수행한다.
