## JobListener 
 - quartz가 job라이프사이클의 특정 시점마다 자동으로 호출해주는 콜백(hook). java 인터페이스로 구현해서 Scheduler에 등록만 하면 모든 잡 실행 시 끼워들 수 있는 자리가 생김

== 호출 되는 시점 
 jobToBeExcuted(잡 실행 직전) -> Job.excute(본체 실행) -> jobWasExcuted(잡 실행 직후 (job 성공/실패 결과, null로 오면 성공/값이 있으면 실패)

모든 job에 성공 실패 여부를 jobListener로 알 수 있음. 
 Job 실패시 throw new JobExcutionException(e,false)로 받아줘야 함
 Why?  예ㄹ외를 Quartz까지 보내야 JobListener가 감지 가능 -> false일 때 즉시 refire(misfire정책에 따름) 

 ---
 흐름 
 job 실패 -> catch -> throw new JobExcutionException -> Quartz -> jobListener.jobWasExcuted -> DiscordWebhookSender.send -> Discord 채널 알림 

새 잡을 추가해도 JobListener가 자동 적용 


---

Quartz와 Spring Batch는 엄연히 다른 것 
quartz가 "00:00 시각에 한 노드만 실행"까지 책임지고, spring batch가 회사 단위로 처리 결과 영속 + 청크 commit+ 카운트 집계 책임짐 

## 배치 고도화할 종류 
 - Spring Batch retry 정책	현재 .skip(Exception.class) 만 → 일시 네트워크 장애도 즉시 skip. .retry(TransientException).retryLimit(3) 추가	중간	일시 장애 자동 복구
 - Multi-thread Step	청크를 여러 스레드 병렬 처리 → 처리 속도 ↑	트랜잭션 / 멱등 신중
 - AnnualGrantHire ListItemReader → JdbcCursorItemReader	매치자 폭증 시 인메모리 위험 사전 대응	YAGNI — 현재 회사당 수십 명
