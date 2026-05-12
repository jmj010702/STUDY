EKS 환경에서 RestClient 사용 시 @LoadBalanced 가 일으키는 문제
문제 사항
전자결재 문서 기안 시 일부 요청에서 SERVICE_UNAVAILABLE + "HR 서비스 연결 실패: 회사 정보를 조회할 수 없습니다." 메시지 노출
동일 사용자가 직전 요청은 성공, 일정 시간 후 또는 신규 회사 ID 첫 진입 시점에만 재현되는 산발적 장애
collaboration-service ↔ hr-service, search-service(AI Copilot) → 양쪽 사내 호출 모두 동일 패턴 잠복
사용자 입장: "방금 됐는데 다시 누르니 에러" → 결재 기안 자체 차단으로 업무 중단
원인 분석
사내 호출 구조 : Redis 캐시 → 미스 시 RestClient 의 2단 구조 → 캐시가 비는 순간(TTL 1시간 만료 · 재배포 직후 · 신규 ID 첫 조회)에만 RestClient 가 실제 호출되며 실패
RestClient.Builder 빈에 @LoadBalanced 부착 → http://hr-service 호출 시 호스트명을 DNS 가 아닌 Spring Cloud LoadBalancer 의 service-id 로 해석
prod 환경은 eureka.client.enabled: false (EKS 사용, Eureka 미배포) → DiscoveryClient 인스턴스 0개 반환 → 네트워크 호출 자체 발생 X
@CircuitBreaker fallback 발동 → BusinessException("HR 서비스 연결 실패…") 로 변환되며 원인이 메시지에서 가려짐
동일 패턴이 3개 모듈(collaboration / hr / search)에 분산 → 외관상 무관해 보이는 산발 장애로 인지
시도 방법
에러 메시지 발신 지점 역추적 → HrServiceClient 의 @CircuitBreaker fallback 도달, RestClient 호출 실패가 메시지로 가려진 구조 확인
application-prod.yml 점검 → 모든 서비스 모듈에서 Eureka 비활성, K8s 디스커버리 의존성도 미도입 상태 확인
RestClientConfig / CollaborationApiConfig / InternalRestClientConfig 3곳에 @LoadBalanced 일관 부착됨을 grep 으로 전수 확인
정상 동작 호출(face-api, redis, kafka) 비교 → 모두 K8s Service DNS + 포트 직접 사용, @LoadBalanced 미사용 → "K8s Service 가 이미 서버사이드 LB 수행, 클라이언트 LB 중복" 결론 도출
해결 방법
3개 모듈 사내 호출용 빈에서 @LoadBalanced 어노테이션 및 미사용 import 제거 → kube-dns 가 호스트명을 직접 해석, K8s Service 가 파드 분산 처리하도록 일원화
호출 baseUrl(http://hr-service, http://collaboration-service) 유지 — K8s Service 매니페스트가 port: 80 → targetPort: 8080 매핑이라 포트 명시 불필요
search-service 클래스 JavaDoc 을 "Eureka 서비스명" → "K8s Service DNS" 기준으로 갱신 → 향후 오해 차단
결과
캐시 미스 · 재배포 직후 · 신규 회사 ID 첫 호출 케이스에서도 결재 기안 즉시 성공 → 사용자 체감 장애 0건
산발적 SERVICE_UNAVAILABLE 알림 제거 → 운영팀 가짜 알람 대응 비용 0
클라이언트 LB ↔ 서버사이드 LB 이중화를 K8s 단일 경로로 단순화 → Eureka 서버 등 추가 디스커버리 인프라 운영 부담 없이 EKS 표준 패턴으로 정렬
