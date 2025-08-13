알겠습니다. Markdown 형식으로 정리해 드리겠습니다. 아래 내용을 복사하여 `.md` 확장자로 저장하시면 됩니다.

---

### **전체 아키텍처 구조 요약**

이 아키텍처는 AWS 서비스들을 활용하여 클라이언트 요청을 체계적으로 처리하는 구조입니다. 이 구성은 비용 효율성을 위해 단일 **ALB**를 활용합니다.

#### **1. 요청 진입 및 사전 검사 단계**
- **클라이언트 → Route 53 → API Gateway**: 클라이언트 요청은 **Route 53**을 통해 **API Gateway**로 들어옵니다.
- **API Gateway → AWS Lambda**: API Gateway는 모든 요청을 **Lambda 함수**로 전달합니다.
- **AWS Lambda → Amazon ElastiCache (Redis)**: Lambda 함수는 요청의 소스 IP를 추출하여 **ElastiCache(Redis)**에 저장된 블랙리스트를 조회합니다.
  - **결과**:
    - **블랙리스트에 있는 경우**: Lambda가 즉시 `403 Forbidden` 응답을 반환하여 요청을 차단합니다.
    - **블랙리스트에 없는 경우**: Lambda는 요청을 다음 단계인 **ALB**로 전달합니다.

---

#### **2. 트래픽 검사 및 라우팅 단계**
- **AWS Lambda → ALB (단일 ALB)**: Lambda는 요청을 **ALB**로 보냅니다.
- **ALB → Detection 컨테이너**: ALB는 Lambda로부터 받은 요청을 **Detection 컨테이너 대상 그룹**으로 전달합니다.
- **Detection 컨테이너 내부 로직**:
  - Detection 컨테이너는 트래픽의 내용을 상세하게 검사합니다.
  - **정상 트래픽으로 판단**: Detection 컨테이너는 **ALB**의 특정 경로(예: `/final`)를 호출하여 요청을 **EC2 인스턴스**로 보냅니다.
  - **비정상 트래픽으로 판단**: Detection 컨테이너는 **ALB**의 특정 경로(예: `/blacklist`)를 호출하여 요청을 **Response 컨테이너**로 보냅니다.

---

#### **3. 최종 처리 단계**
- **ALB → EC2 인스턴스**: 정상 트래픽은 최종 목적지인 **EC2 인스턴스**에서 처리됩니다.
- **ALB → Response 컨테이너**: 비정상 트래픽은 **Response 컨테이너**에서 처리됩니다.
  - **Response 컨테이너의 역할**: 비정상 트래픽의 소스 IP를 추출하여 **ElastiCache(Redis)**에 블랙리스트로 등록합니다.

---

### **주요 서비스 및 역할**
- **Amazon Route 53**: 도메인 네임 시스템(DNS).
- **Amazon API Gateway**: 모든 API 요청을 받아들이는 프론트엔드.
- **AWS Lambda**: 서버리스 함수로, 요청의 IP를 검사하고 Redis에 쿼리하는 로직을 수행.
- **Amazon ElastiCache (Redis)**: 고성능의 인메모리 데이터 스토어로, IP 블랙리스트를 빠르게 조회하고 업데이트.
- **Amazon ALB**: 단일 인스턴스로 여러 컨테이너 대상 그룹에 트래픽을 분산시키는 로드 밸런서.
- **Amazon ECS (or Fargate)**: Detection 및 Response 컨테이너를 관리하고 실행.
- **Amazon EC2**: 정상 트래픽을 처리하는 최종 백엔드 서버.
