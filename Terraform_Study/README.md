# Terraform_Study

# 1. Terraform이란? 
HashiCorp에서 만든 IaC(Infrastructure as Code) 도구. 인프라를 선언형 코드로 정의하고 자동으로 생성/변경/삭제할 수 있다. 
- 특징
 - 선언형 : 어떻게가 아니라 무엇을 원하는지를 기술
 - 멀티 프로바이더 : Aws,GCP,Azure,Kubernetes등 다양한 인프라 자원을 동일 문법으로 관리
 - HCL 언어 사용 : HashiCorp Configuration Language
 - State 기반 : terraform.tfstate로 현재 인프라 상태를 추적

# 2. HCL 문법 핵심 
/* 
  블록타입 "라벨1" "라벨2" {
    인자1 = 값1
    인자2 = 값2 
  }
  */ 

  ex) resource "aws_instance" "web" {
        ami           = "ami-@@@@@"
        instance_type = "t2.micro"
        }
    여기서 resource는 블록 타입 
    aws_instance는 Provider가 정해놓은 리소스 타입(변경이 절대 불가, 이름도 고정) 
    "web" -> 개발자가 붙이는 이름(자유이며 DB의 AS와 같음) 

    파일 구조 관습 
project/
  ├── main.tf        # 핵심 리소스
  ├── variables.tf   # 변수 선언
  ├── outputs.tf     # 출력 선언
  └── providers.tf   # 프로바이더 설정
  
# 3. 동작 흐름 
  코드 작성 -> init -> plan -> apply -> (운영 단계) -> destroy (삭제/운영 중단) 
  명령어                   역할                                             빈도 
  terraform init          Provider 다운로드, 초기화                        처음 한번만 (Provider 변경시에) 
  terraform plan          변경 사항 미리보기(수정이 아닌 단순 미리보기)      매번(수시로 할것) 
  terraform apply          실제로 인프라 생성/변경                          코드 변경시 
  terraform destroy        모든 자원 삭제                                   사실 거의 안함(운영 중단할 떄만 함)

# 4. state 파일 
  terraform.tfstate == 코드와 실제 자원의 매핑표 
  위에 plan 명령어가 동작하는 원리는 
  1. 코드 .tf 읽기 -> 코드와 같은 모습이어야 함
  2. State 읽기 -> 전엔 이런 모습 이었다
  3. 실제 클라우드 조회 -> 지금은 이런 상태임
  4. 셋을 비교 -> 이만큼 바꿔야 함

State 다루는 철칙 
- State는 Git에 절대 커밋 금지(시크릿 평문 노출)
    - gitingnore에 올릴 것
    - *.tfstate
      *.tfstate.backup
      .terraform/
- 직접 인프라 UI로 수동 편집 금지.  tfstate가 꼬여서 해결하기 힘듬

그러면 어떤걸 커밋해야 하는가? 
*.tf, .terraform.lock.hcl 

# 5. Drift (드리프트) 
코드는 그대로인데 누군가 콘솔에서 직접 만져서 코드와 실제 자원이 어긋난 상태 -> 이럴때 terraform은 코드를 정답으로 보고 원래대로 되돌리려고 함 
그렇기 때문에 IaC 도입 후엔 콘솔 수동 작업 금지

---

# 테라폼 실행 방법
- winget install Hashicorp.Terraform
-> terraform version
테라폼 설치 후 aws 인증
-> aws version으로 aws 깔려있는지 확인
- aws configure
- > aws 인증 
  
