# Cloud Infra 

<img width="958" height="563" alt="image" src="https://github.com/user-attachments/assets/9932d9ee-5fed-4014-96ca-1a25b16b19a8" />

## 유형 
### 1. 컴퓨팅 서비스
- 다양한 성능의 컴퓨터 빌려쓰기 
- Amazon EC2(가상 서버), AWS Lambda(서버리스 컴퓨팅), Amazon ECS/EKS(컨테이너)
### 2. 데이터베이스 서비스
- 데이터 저장하고 관리하기 위한 다양한 유형의 데이터 베이스 서비스 제공
-  Amazon RDS(관계형 데이터베이스), Amazon DynamoDB(NoSQL 데이터베이스 ) ex) JSON 파일 저장 
### 3. 스토리지 서비스
- 다양한 형태의 데이터를 저장할 수 있는 솔루션 제공
- Amazon S3(객체 스토리지), Amazon EBS(블록 스토리지), Amazon EFS(파일 스토리지)
### 4. 네트워킹 및 콘텐츠 전송 서비스
- 네트워크 인프라 구성 관리
- 네트워크 인프라와 콘텐츠 전송 서비스를 제공합니다. Amazon VPC(가상 네트워크), Amazon CloudFront(CDN), Route 53(DNS 서비스), ELB(Load Balancer)
### 5. 보안, 자격 증명 및 규정 준수 서비스 
- 인프라 자원 보호, 접근 권한 관리, 규제 요구사항 충족
### 6. 관리 도구 
- 클라우드 인프라 자원 모니터링, 자동화 

---
# AWS 서버리스 서비스 

| **서비스 유형** | **서비스 이름** | **설명** |
|------------------|------------------|-----------|
| **컴퓨팅** | AWS Lambda | 서버리스의 핵심, 이벤트에 응답하여 코드 실행, **사용한 컴퓨팅 시간에 대해서만 비용 지불** |
| **API 관리** | Amazon API Gateway | 서버리스 **API를 생성**, 게시, 관리 및 모니터링하는 완전 관리형 서비스 |
| **데이터베이스** | Amazon DynamoDB | 완전 관리형 NoSQL 데이터베이스, 빠른 응답 시간과 자동 확장 제공 |
| **스토리지** | Amazon S3 | 확장 가능한 객체 스토리지, 파일 저장 및 정적 웹사이트 호스팅 가능 |
| **메시징** | Amazon SNS | 푸시 기반 알림 서비스, 여러 수신자에게 메시지 동시 전송 |
| **메시징** | Amazon SQS | 완전 관리형 메시지 대기열 서비스, 마이크로서비스 간 비동기 통신 지원 |
| **워크플로우** | AWS Step Functions | 시각적 워크플로우를 통해 분산 애플리케이션 구성 요소를 조정하는 오케스트레이션 서비스 |
| **이벤트 처리** | Amazon EventBridge | 이벤트 기반 애플리케이션을 구축하기 위한 서버리스 이벤트 버스 |
| **인증/권한부여** | Amazon Cognito | 웹 및 모바일 앱을 위한 사용자 인증, 권한 부여 및 사용자 관리 서비스 |
| **API/데이터 동기화** | AWS AppSync | GraphQL API를 생성하고 관리하는 서비스, 실시간 데이터 동기화 지원 |
| **분석** | Amazon Athena | 서버리스 쿼리 서비스, S3에 저장된 데이터를 SQL로 분석 |
| **ETL** | AWS Glue | 서버리스 데이터 통합 서비스, 데이터 발견, 준비, 이동 자동화 |
| **스트리밍** | Amazon Kinesis | 실시간 스트리밍 데이터 수집, 처리, 분석 서비스 |
| **컨테이너** | AWS Fargate | 서버리스 컨테이너 실행 환경, EC2 인스턴스 관리 없이 컨테이너 실행 |
| **애플리케이션 통합** | Amazon EventBridge Pipes | 이벤트 소스와 대상 간의 점대점 통합을 위한 서비스 |
| **비밀 관리** | AWS Secrets Manager | 서버리스 애플리케이션의 비밀(API 키, 비밀번호 등) 관리 및 교체 |
| **모니터링** | Amazon CloudWatch | 서버리스 애플리케이션 모니터링, 로깅, 알람 설정 서비스 |
| **배포** | AWS SAM (Serverless Application Model) | 서버리스 애플리케이션 빌드 및 배포를 간소화하는 프레임워크 |

---
- API Gateway : Restful API와 같은(push ,Put, delete ..)
- DynamoDB : JSON형 DB
- EventBridge : 조건부, 필터링
- Cognito : 사용자 인증
- Athena : 데이터 분석
- Glue : 데이터 통합
- Kensis : 데이터 수집


---
AWS Command URL : https://docs.aws.amazon.com/cli/latest/
