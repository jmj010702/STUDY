# eks 환경에서의 RestClient를 사용할 때 @LoadBalanced가 일으키는 문제 
문제사항

전자결재 문서 기안 시 일부 요청에서 SERVICE_UNAVAILABLE 와 함께 "HR 서비스 연결 실패: 회사 정보를 조회할 수 없습니다." 메시지 노출
동일한 사용자가 직전 요청은 성공하다가, 일정 시간 후 또는 신규 회사 ID 로 처음 진입할 때만 재현되는 산발적 장애
collaboration-service → hr-service 뿐 아니라 hr-service → collaboration-service, search-service(AI Copilot) → 양쪽 사내 호출 모두 동일 패턴 잠복
사용자 입장에서는 "방금 됐는데 다시 누르니 에러" → 결재 기안 자체가 막혀 업무 중단
원인분석

사내 호출 클라이언트가 Redis 캐시 → 미스 시 RestClient 의 2단 구조로 동작 → 캐시가 비어 있는 순간(TTL 1시간 만료, 재배포 직후, 신규 ID 첫 조회)에만 RestClient 호출이 실제로 나가며 실패
RestClient.Builder 빈에 @LoadBalanced 가 붙어 있어, http://hr-service 호출 시 호스트명을 DNS 가 아닌 Spring Cloud LoadBalancer 의 service-id 로 해석 시도
prod 환경은 eureka.client.enabled: false (EKS 만 사용, Eureka 미배포) → DiscoveryClient 가 인스턴스 0개를 반환 → 네트워크 호출 자체가 발생하지 않고 실패
@CircuitBreaker fallback 이 발동되어 BusinessException("HR 서비스 연결 실패…") 로 변환되며 원인이 가려짐
동일 패턴이 3개 모듈(collaboration-service / hr-service / search-service)에 분산 → 외관상 무관해 보이는 산발 장애로 인지
시도방법

에러 메시지 발신 지점을 추적해 HrServiceClient 의 @CircuitBreaker fallback 까지 거슬러 올라감 — RestClient 호출이 실패해 fallback 으로 빠지는 구조 확인
application-prod.yml 에서 Eureka/디스커버리 설정 점검 → 모든 서비스 모듈에서 Eureka 비활성, K8s 디스커버리 의존성도 미도입 상태 확인
RestClientConfig / CollaborationApiConfig / InternalRestClientConfig 3곳에 @LoadBalanced 가 일관되게 부착되어 있음을 grep 으로 전수 확인
정상 동작 중인 호출(예: face-api, redis, kafka)은 모두 K8s Service DNS + 포트를 직접 쓰며 @LoadBalanced 미사용임을 비교 분석 → "K8s Service 가 이미 서버사이드 LB 를 수행하므로 클라이언트 LB 는 중복" 결론
해결방법

3개 모듈의 사내 호출용 빈에서 @LoadBalanced 어노테이션 및 미사용 import 제거 → kube-dns 가 직접 호스트명을 해석하고 K8s Service 가 파드 분산을 처리하도록 일원화
호출 baseUrl(http://hr-service, http://collaboration-service)은 그대로 유지 — K8s Service 매니페스트가 port: 80 → targetPort: 8080 매핑이라 포트 명시 불필요
search-service 클래스 javadoc 을 "Eureka 서비스명" → "K8s Service DNS" 기준으로 갱신해 향후 오해 차단
결과
캐시 미스 / 재배포 직후 / 신규 회사 ID 첫 호출 케이스에서도 결재 기안이 즉시 성공 → 사용자 체감 장애 0건
산발적 SERVICE_UNAVAILABLE 알림 사라져 운영팀의 가짜 알람 대응 비용 제거
클라이언트 LB ↔ 서버사이드 LB 이중화 구조를 K8s 단일 경로로 단순화 → 디스커버리 인프라(Eureka 서버) 추가 운영 부담 없이 EKS 표준 패턴으로 정렬