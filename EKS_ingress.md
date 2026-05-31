eks의 ingress자원을 띄울 때 항상 고민이 되는 부분이 있다. 웹소켓,sse를 사용할 때  eks를 사용한다는건 고가용성, 로드밸런싱이 되는데 sticky세션을 사용하는건 맞지 않다. 그러면 웹소켓은 같은 파드에 계속 할당이 되어야 하는거 아닌가? 라는 의문이 듬 그랬기에 아래 방식을 썼었다. 하지만 이건 eks의 사용 원인에 안 맞다고 판단함. 

nginx.ingress.kubernetes.io/affinity: "cookie"
nginx.ingress.kubernetes.io/affinity-mode: "persistent"
nginx.ingress.kubernetes.io/session-cookie-name: "SERVERID"
nginx.ingress.kubernetes.io/session-cookie-max-age: "3600"


그렇기에 아래 방식도 고려해봤다.
nginx.ingress.kubernetes.io/upstream-hash-by: "$binary_remote_addr"

IP해시 방식으로서 : 같은 클라이언트 IP에서 온 요청은 항상 같은 백엔드 파드로 보낸다. - 쿠키 없이 클라이언트 IP를 해시해서 파드에 배치

위에 말을 보면 같은 방식 아닌가? 라고 생각할 수 있지만 조금 다르다. 

먼저 기본(affinity)도 없을 때 nginx의 기본 로드밸런싱은 round-dobin(라운드로빈) 방식인데 {least-connection과 가중치 알고리즘} 해당 방식은 매 요청마다 어느 파드로 갈지 달라짐 (least-connection 때문에) 

쿠키 affinity 방식 
첫 요청때 nginx가 serverID 쿠키 발급 -> 클라이언트가 매번 그 쿠키 보내면 같은 파드로 
장점: 쿠키값으로 정확히 파드 식별 
단점 : 쿠키가 만료됐을 때 또는 쿠키를 안 보내는 클라이언트(모바일,api클라이언트) 는 sticky가 안되고 다른 파드로 튄다. 

IP해시 방식  
nginx가 클라이언트 IP자체를 받아서 해시함수로 -> 백엔드 파드 인덱스 계산 -> 같은 IP면 항상 같은 결과로 
그러면 한 파드에만 몰릴 수 있는 것이 아니냐, 같은 IP를 쓰는 사람이 한 공간에 많다면 이런 걱정이 나올 수 있는데 맞다. B2C서비스에서는 잘 맞을 거 같지만 B2B에서는 맞지 않는 방식인 거 같다. 

# IP 해시방식 
$binary_remote_addr 가 뭔가 . nginx의 내장  변수. 
왜 binary를 쓰는가-> 해시 게산시 더 빠르고 IPV4,6를 같이 처리할 때 일관성이 있다 nginx 공식 문서도 hash key로는 binary를 권장하고 있다. 
consistent hash라는 점에 중점을  둬야 하는데 upstream-hash-by는 단순 modulo가 아닌 consistent hashing을 쓴다.(ketama 알고리즘)
-> 왜 중요한가. 단순 해시 방식이면 파드 수가 바뀔때 (오토스케일링) 모든 클라이언트의 매핑이 어긋남. 하지만 consistent hash는 링  구조를 파드에 배치해서 파드1개 추가시 1/n의 클라이언트만 재배치 되도록 한다. 

다만 한계가 있는데  
1. NAT 뒤 다수 클라이언트일 때 -> 한 파드에 쏠림 '
    - 회사, 학교처럼 수백명이 같은 공인 IP로 나오는 환경에서는 그 IP의 모든 사용자가 같은 파드로 간다. 로드 분산이 약해질 수 있음
2. 클라이언트 IP 보존 필수 
   - 앞서 말했듯이 NLB가 클라이언트 IP를 보존해야 의미가 있다. SNAT되면 모든 IP가 LB노드 IP로 보여서 1~2개 파드로 다 쏠린다. 이 경우에는 externalTrafficPolicy: Local 로 해결 가능 
3. IP가 바뀌는 클라이언트
   - 모바일 사용자가 wifi에서 5g같이 전환되면 ip가 바뀐다. 이 경우에 다른 파드로 라우팅 -> 따로 공부해서 이런 경우까지 해결해볼 것 
   - websocket은 어차피 tcp가 끊겨 재연결해야 하므로 영향이 크다. 
   - 백엔드에 메세지 브로커가 있다면 이 방식으로 해결 가능할 것 같다. 


---

# 전체 정리 

방식	                                        장점	                                                                단점
affinity 없음	                            가장 단순, 균등 분산	                                        sticky 필요한 시나리오 처리 안 됨
cookie affinity	                            쿠키 단위로 정밀 sticky                                     	API 클라이언트 효과 없음, 쿠키 의존
IP hash (현재)	                    쿠키 없이도 sticky, consistent hash 로 scale 친화적	            NAT 환경에서 분산 약화, IP 변경 시 깨짐
upstream-hash-by: "$http_x_forwarded_for"	    다층 프록시 환경에서 IP 정확도 ↑	                                XFF 헤더 신뢰 가능해야 함

---

웹소켓은 TCP연결이 수립되면 한번 맺은 TCP는 LB가 끊지 않는한 한 파드에 계속 유지되기 때문에 괜찮다는 생각이 들었다. 웹소켓은 stompHandler를 통해서 보내고 메세지 브로커를 통해 다시 할당해주기때문에 사실 어느 파드에 있어도 상관이 없는 것 같다. 적으면서 계속 찾아보고 공부를 해보니 필자의 지식이 많이 부족하다는 걸 느끼고 있다. 

웹소켓은 TCP연결이 맺히면 그 자체로 sticky다.
연결 끊김 후 재연결을 할 때 만일 affinity가 없다면 재 연결시 다른 파드로 가서 메모리에 들고있던 sse구독자 목록이 비어있어 알림 누락이 생길 수 있다. 이 문제는 redis나 kafka를 이용해 fan-out한다면 괜챃을 것 같다 



fan-out이란 : 한곳에서 발생한 이벤트를 여러곳에 동시에  뿌리는 패턴(redis pub/sub이나 kafka에 다중 리스너 groupId)
fan-out은 폭증이 올 수있다 이를 해결하기 위해서는 별도  워커기 배치로 처리하거나, 사용자 그룹 단위로 쪼개서 publosh를 하는 방법이 있다. 
