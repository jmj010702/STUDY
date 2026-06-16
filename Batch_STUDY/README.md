# 배치 고도화를 진행하던 중 모르는 부분이 많이 나와 작성하게 됨

# 문제 배경
중복된 작업을 막기 위해 ㄹ레디스 분산 락을 사용함 

# 문제
현재 배치를 진행하는 코드가 @Scheduled로 되어있음 로컬에서는 JVM안에 cron시각을 기다리다 fire하는 구조 파드별로 독립된 스케줄러가 돌아감 == 배포환경에서는 각 파드마다 같은 작업 3번 반복

# 원인 
그래서 redis 분산락을 통해서 첫 파드만 통과시키고 나머지는 skip하는 패턴을 쓰고 있음 
정합성 자체는 보호하지만 잡이 n번 돌아가는건 막지 못함 -> 내 의도와 다름 n번 돌아가는 걸 막고 싶음 
락 경쟁과 log.info가 매번 만복  
WorkGroup별 동적 스케줄이 현재 workGroup생성할 때 JVM에 ConcurrentHashMap으로 보관하고 있는 구조 만일 workGroup Crud가 발생하면 다른 파드에서는 인식하지 못함 고로 삭제되었더라도 계속해서 돌아감 
redis락이 중복은 막아주지만 올바른 시각에 올바른 작업은 아님 -> 내 의도와 다름 
또한 레디스가 항상 살아있다는 전제 위에 코드가 짜인것이기 때문에 다운될 경우에 잡 자체가 실행이 안되고 중복 작업이 돌아감. 

# 시도 방법
1) PartitionScheduler에 Redis에 분산락 적용 -> 단일 패턴 정합성 뺏김
2) shedLock 도입 -> 새 의존성 추가 + 기존 수동 락 패턴과 갈림
3) quartz Jdbc 클러스팅으로 통일 -> @Scheduled 자체를 제거하고 quartz JobDetail + CronTrigger로 전환 

# 해결
문제 해결 방안으로 quartz JDBC클러스터링을 채택함 
spring-boot-starter-quartz + org.quartz.jobstore.isClustered: true + Jdbc JobStore 적용 
동적 스케줄일 경우에 인메모리 핸들 -> Scheduler.scheduleJob/deleteJob 호출 시 DB 영속 -> 모든 파드 자동 동기화 
다양한 스케줄러 cron 잡도 동일 경로로 흡수함. redis락 코드 일괄 제거 
스케줄러 가용성을 db에 일원화하여 redis가용성과 디커플링,spof 1개 제거 
db를 통해 관리하기 때문에 한 서버가 장애시 다른 서버가 작업 인계, 누락 위험 제거 
SPOF 제거하고 디커플링으로 진행 


---

## Job implements를 받아야됨 
quartz 프레임워크와의 계약 Scheduler.scheduleJob은 Class<?, extends Job> 만 받기 때문에 표준 진입점이 org.quartz.job인터페이스의 excute(JobExcutionContext) 메서드 
실행 시각이 되면 quartz가 클래스를 reflection으로 인스턴스화 한뒤  excute호출. 이 때 메서드 시그니처명이 안 맞으면 NPE가 남 

## 필드 주입 vs private final + 생성자 주입 
일반적인 코드에서는 private final + 생성자 주입 패턴을 많이 쓰지만 Quartz Job은 예외. 왜냐 
항목 							일반 빈 							quartz Job
인스턴스 생성 주체 				spring 컨테이너 						quartz가 매 작동때마다 reflection으로 직접
호출되는 생성자 					@Autowired 붙은거 					빈 생성자(NoArgs)만 
의존성 주입 통로 				생성자 								필드 @AutoWired 
final이 가능한가 			가능(생성자에서 할당)						불가(빈 생성자에서는 final 못 채움)

&& 리플렉션 : 클래스 이름만 가지고 해당 클래스의 정보를 파헤쳐서 강제로 객체를 만들어내는 자바의 코드 
&& JobExecutionContext : quartz가 스케줄러가 작업을 실행할 때 넘겨주는 정보

---

## jobLauncher 경유 vs Service 직접 호출 
항목							jobLauncher 									Service 직접 호출 
중복 실행 차단 					Batch_JOB_INSTANCE Unique제약이 자동 차단 		없음, 두번 호출하면 두 번 실행
멱등 책임자 						프레임워크(DB 제약) 								Service 코드(로직 내에서 막아야함)
청크/재시작 						실패 지점부터 재시작 								없음
추적 							Batch_JOB_EXCUTION에 read/write 카운트, 상태 영속      로그 밖에 없음 

---

## Quartz로 바꾸면 Scheduler가 필요없지 않나? 
- 시각이 되면 작업 실행하는 엔진  -> quartz Schedular.
- 작업 실행되는 본체(service 호출)  -> @@Scheduler

 

---

## - JDBC 클러스터링이란 
여러 대의 서버가 하나의 Job 스케줄을 공유하고 관리하도록 만드는 설정 
DB를 중심으로 모든 서버가 동일한 DB테이블을 바라봐 Job을 시작하기전에 DB에 LOCK을 걸어 중복Job 방지 
DB에 어느 테이블을 이용해 어떤 서버가 작업을 선점했는지 기록 -> 서버 A가 중단되어도 서버 B가 DB를 확인하고 이어서 처리 가능하다 

Quartz 11개 테이블 
- Quartz는 잡/정의/트리거/락/이력을 DB에 저장함. 그걸 위한 표준 메타테이블이 11개  tables_mysql_innodb.sql는 그걸 만들어주는 공식 DDL 스크립트 (jar에 들어있음)
- MISFIRE 정책 : 트리거가 cron 시각에 Job실행 하지 못한 상황 발생
  - 모든 노드가 죽어있음
  - 스레드풀 포화
  - 클러스터 노드 인계 중
 quartz는 misfire가 발생했을 때 어떻게 처리할지 정책으로 지정
- FIRE_NOW : 놓친만큼 즉시 한 번 실행
- DO_NOTHING : 그냥 건너띔 , 다음 cron시간까지 대기
- IGNORE_MISFIRE_POLICY : 시각 지났어도 무조건 모두 실행 (놓친만큼 N번)
- 멱등성 보장이 안될 수 있지만 비용이 큰 Job같은 경우에는 해당 부분을 다시 추적해서 진행하기 보다는 다음날 알림으로 개발자가 직접 처리하는게 더 효율적임 
   

--- 
왜 사용하는가

- 고가용성 : 서버 한 대가 고장나도 다른 서버가 스케줄을 이어받아 실행함 (중단없는 서비스)
- 부하분산 : 여러 서버가 작업을 나눠서 처리할 수 있어 시스템 전체의 부담이 줄어듬
- 데이터 보존 : 서버가 재시작 해도 DB에 저장되어 있기에 작업이 사라지지 않음
- 병렬 작업 : 배포 환경에서 멀티 파드가 병렬적으로 job을 실행할 수 있음  

단점 
- 마찬가지로 DB가 죽으면 스케줄러도 같이 멈춤 -> redis의 문제와 같음
- 성능이 떨어짐 -> 매번 DB에 락을 걸고 상태를 업데이트해야 하므로 메모리 방식(기존에 사용하던 JVM)보다 속도가 느림
- 시간 동기화 : 클러스터에 참여하는 모든 서버의 시스템 시간이 일ㄹ치해야 함  -NTP(Network Time Protocol)를 사용해 해결 가능

## 배치는 잡별 misfore 정책을 다르게 생성할 수 있음 
misfore 정책은 트리거 단위로 설정 


- 마찬가지로 DB가 죽으면 스케줄러도 같이 멈춤 -> redis의 문제와 같음
- 성능이 떨어짐 -> 매번 DB에 락을 걸고 상태를 업데이트해야 하므로 메모리 방식(기존에 사용하던 JVM)보다 속도가 느림
- 시간 동기화 : 클러스터에 참여하는 모든 서버의 시스템 시간이 일ㄹ치해야 함  -NTP(Network Time Protocol)를 사용해 해결 가능
---


# 적용 방법 
	implementation 'org.springframework.boot:spring-boot-starter-quartz'   <- 의존성 추가 
    batch:
    job:
      enabled: false
    jdbc:
      initialize-schema: always

  quartz:
    job-store-type: jdbc
    jdbc:
      # local 전용: QRTZ_* 11개 메타테이블 자동 생성
      # prod 에서는 never + 수동 DDL (db/migration/quartz_tables_mysql_innodb.sql) 적용 권장
      initialize-schema: always
    properties:
      org.quartz.scheduler.instanceName: PeopleCoreClusteredScheduler
      # 노드별 자동 ID 생성 (HOSTNAME + 타임스탬프 조합) — 멀티 파드 식별용
      org.quartz.scheduler.instanceId: AUTO
      org.quartz.jobStore.class: org.quartz.impl.jdbcjobstore.JobStoreTX
      org.quartz.jobStore.driverDelegateClass: org.quartz.impl.jdbcjobstore.StdJDBCDelegate
      org.quartz.jobStore.tablePrefix: QRTZ_
      # 멀티 노드 환경에서 한 노드만 fire 보장 (DB row lock 기반)
      org.quartz.jobStore.isClustered: true
      # 노드 헬스체크 주기(ms) — QRTZ_SCHEDULER_STATE 폴링 간격. 노드 다운 감지에 영향
      org.quartz.jobStore.clusterCheckinInterval: 10000
      org.quartz.threadPool.class: org.quartz.simpl.SimpleThreadPool
      # 동시 fire 가능한 잡 수. vacation 6 + attendance 자동마감 + 파티션 합쳐 보수적으로 5 시작 → 운영 모니터링 후 조정
      org.quartz.threadPool.threadCount: 5
      org.quartz.threadPool.threadPriority: 5

