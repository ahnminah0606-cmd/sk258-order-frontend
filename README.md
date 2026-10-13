# sk258 주문 관리 프론트엔드

SKALA DevOps 교재 마지막 실습. 교재 03.frontend 샘플을 기반으로 구성했습니다.

- 프론트엔드: HTML/CSS/JavaScript + nginx
- 백엔드: myfirst-api-server의 Spring Boot REST API
- Jenkins: BuildKit 이미지 빌드 및 Harbor 업로드
- GitOps: myfirst-api-server 저장소 gitops 브랜치 capstone/frontend 갱신
- Argo CD: GitOps 상태를 class-8에 자동 배포

주소: https://sk258-orders.class-8.skala-ai.com/

상품·사용자·주문 관리 화면에서 같은 주소의 /api 경로로 백엔드를 호출합니다.
