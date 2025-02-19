# 🚀 프론트엔드 배포 파이프라인


## 📌 배포 프로세스 흐름
<img width="747" alt="image" src="https://github.com/user-attachments/assets/a343ed98-315d-4345-be19-1173fbad7881" />

1. 개발자가 GitHub의 main 브랜치에 코드를 push
2. GitHub Actions가 실행되어 프로젝트 빌드 및 배포 자동화 진행
3. CI/CD 파이프라인이 아래의 작업을 수행
	- 저장소 체크아웃 → 최신 코드 가져오기
	- 의존성 설치 (npm ci) → 패키지 캐싱 및 최적화
	- 프로젝트 빌드 (npm run build) → 정적 파일 생성
4. 빌드 완료 후 AWS 인증
5. 빌드된 정적 파일을 Amazon S3에 업로드 및 동기화
7. CloudFront 캐시를 무효화하여 최신 파일 제공
8. 사용자가 CloudFront 도메인으로 접속하면 S3의 최신 정적 파일을 빠르게 서빙


## ⚡ GitHub Actions 배포 과정
**1️⃣ 워크플로우 실행 트리거**

```
on:
  push:
    branches:
      - main # 또는 master, 프로젝트의 기본 브랜치 이름에 맞게 조정
  workflow_dispatch:
```
- main 브랜치에 코드가 푸시될 때 실행됩니다.
- workflow_dispatch: GitHub Actions에서 수동 실행 가능하도록 합니다.

**2️⃣ 배포 작업 시작**
```
jobs:
  deploy:
    runs-on: ubuntu-latest
```
- ubuntu-latest → 최신 Ubuntu 환경에서 실행합니다.

**3️⃣ 코드 체크아웃**
```
- name: Checkout repository
  uses: actions/checkout@v4
```
- GitHub 저장소에서 최신 코드 가져와 CI 환경으로 다운로드합니다.

**4️⃣ 의존성 설치**
```
 - name: Install dependencies
   run: npm ci
```
- npm ci 명령어를 실행하여 package-lock.json 기준으로 패키지를 설치합니다.

**5️⃣ 프로젝트 빌드**
```
- name: Build
  run: npm run build
```
- Next.js 프로젝트를 정적 파일로 변환하여 S3에 배포할 준비를 합니다.
- npm run build 실행 후, out/ 폴더에 빌드 결과물이 생성됩니다.

**6️⃣ AWS 자격 증명 설정**
```
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v1
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ${{ secrets.AWS_REGION }}
```
- GitHub Secrets에 저장된 AWS 인증 정보를 사용하여 AWS CLI를 구성합니다.

**7️⃣ S3에 배포**
```
 - name: Deploy to S3
   run: |
     aws s3 sync out/ s3://${{ secrets.S3_BUCKET_NAME }} --delete
```
- aws s3 sync 명령어를 사용해 out/ 폴더 내의 빌드 파일을 S3 버킷에 업로드합니다.
- --delete 옵션을 사용해 S3에 남아있는 불필요한 파일을 자동으로 삭제하여 최신 상태 유지합니다.

**8️⃣ CloudFront 캐시 무효화**
```
- name: Invalidate CloudFront cache
  run: |
    aws cloudfront create-invalidation --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} --paths "/*"
```
- aws cloudfront create-invalidation 명령어로 CloudFront의 캐시를 무효화합니다.
- 최신 빌드 파일이 사용자에게 바로 적용될 수 있도록 모든 경로(/*)에 대해 무효화 처리합니다.


## 🔗 주요 링크
- S3 버킷 웹사이트 엔드포인트: http://performance-inspection-practice.s3-website-ap-southeast-2.amazonaws.com
- CloudFrount 배포 도메인 이름: https://d2bs5sb8hgr7py.cloudfront.net


## 💫 주요 개념
#### 1. GitHub Actions과 CI/CD 도구
- GitHub Actions는 GitHub에서 제공하는 자동화 및 지속적 통합/지속적 배포(CI/CD) 서비스입니다. GitHub Actions을 통해 코드를 푸시하거나 풀 리퀘스트를 생성하는 등의 이벤트에 반응하여 자동화된 작업을 실행할 수 있습니다.
- CI/CD (지속적 통합/지속적 배포)
> - **CI (지속적 통합, Continuous Integration):** 개발자들이 개별적으로 작업한 코드를 주기적으로 통합하여, 자동화된 빌드 및 테스트를 거침으로써 코드 품질을 유지하고 통합 과정에서 발생할 수 있는 문제를 빠르게 식별하는 프로세스입니다.
> - **CD (지속적 배포, Continuous Deployment):** CI 과정을 거친 코드를 자동으로 배포 환경에 배포하는 과정을 의미합니다. 코드가 GitHub Actions 등의 CI 툴을 통해 자동으로 테스트된 후 배포 서버에 배포되도록 하는 과정입니다.

#### 2. S3와 스토리지
- S3 (Simple Storage Service)는 Amazon Web Services(AWS)에서 제공하는 객체 스토리지 서비스입니다. 텍스트, 이미지, 비디오 등 다양한 형태의 데이터를 저장하고 관리할 수 있으며, 높은 확장성과 안정성을 제공합니다.
- S3는 버전 관리, 데이터 암호화, 데이터 접근 제어 등을 제공하여 클라우드 환경에서 유연하게 데이터를 관리할 수 있습니다.
- 스토리지는 데이터를 저장하는 장치나 시스템을 의미합니다. S3는 클라우드 기반의 스토리지 서비스로, 필요에 따라 용량을 늘리거나 줄일 수 있는 확장성과 고가용성을 제공합니다.

#### 3. CloudFront와 CDN
- CloudFront는 AWS에서 제공하는 CDN (Content Delivery Network) 서비스입니다. 웹 콘텐츠를 전 세계에 분산된 서버에 캐싱하여 사용자에게 더 빠르고 효율적으로 콘텐츠를 전달할 수 있습니다. CloudFront는 자동으로 트래픽을 최적화하여 로딩 속도를 개선하고, 글로벌 배포와 데이터 처리 성능을 강화합니다.
- CDN은 웹 콘텐츠를 사용자에게 더 가까운 서버에서 전송하여 전송 속도를 높이는 기술입니다. CloudFront는 대표적인 CDN 서비스 중 하나로, 서버 간의 콘텐츠 전송 지연을 최소화하고 웹사이트의 전반적인 성능을 향상시킬 수 있습니다.

#### 4. 캐시 무효화(Cache Invalidation)
- 캐시 무효화는 CDN에 저장된 콘텐츠가 변경되었을 때, CDN 서버에 저장된 캐시를 삭제하고 최신 콘텐츠를 가져오도록 하는 과정입니다.
- CloudFront와 같은 CDN 서비스에서는 캐시 무효화 기능을 제공하여, 콘텐츠 업데이트를 빠르게 반영할 수 있습니다.
- 캐시 무효화가 발생하지 않으면, 오래된 캐시가 사용자에게 전달되어 최신 정보가 반영되지 않는 문제가 발생할 수 있습니다. 따라서 적절한 캐시 무효화 전략이 필요합니다.

#### 5. Repository secret과 환경변수
- Repository secret은 GitHub Repository에 저장하는 민감한 정보를 안전하게 관리하기 위한 기능입니다.
- Repository secret은 암호화되어 저장되며, GitHub Actions 워크플로우에서 안전하게 사용할 수 있습니다.
- GitHub Actions에서는 secrets를 환경변수처럼 다룰 수 있어 보안에 중요한 정보를 외부에 노출하지 않고도 자동화된 작업을 실행할 수 있습니다.
- 환경변수는 프로그램 실행 환경에 영향을 주는 변수로, GitHub Actions 워크플로우에서 다양한 설정을 지정할 수 있습니다.
- 환경변수는 워크플로우 실행 시, 컨테이너나 가상 환경에서 사용됩니다. Repository secret도 환경변수 형태로 워크플로우에서 사용할 수 있으며, 워크플로우 내에서 다른 작업들을 수행할 때 이러한 변수를 활용하여 민감한 정보를 안전하게 처리할 수 있습니다.

#### 6. 웹 성능 측정 지표
- **TTFB (Time to First Byte)** : 사용자의 요청이 서버에 도달한 후, 첫 번째 바이트가 브라우저로 전달되는 시간입니다. 서버 응답 속도를 측정하는 중요한 지표입니다.
- **FCP (First Contentful Paint)** : 사용자가 화면에서 첫 번째 콘텐츠(텍스트, 이미지 등)를 볼 수 있는 시점을 의미합니다. 사용자 경험과 직결되는 지표입니다.
- **LCP (Largest Contentful Paint)** : 페이지에서 가장 큰 콘텐츠(주로 이미지, 제목 등)가 렌더링되는 시간입니다. 페이지 로딩 성능을 평가하는 데 중요한 요소입니다.
- **DOMContentLoaded** : 브라우저가 HTML을 파싱하고 DOM 생성을 완료한 시점입니다. CSS, 이미지, 스크립트 등 다른 리소스의 로딩은 완료되지 않아도 상관없습니다.
- **총 로드 시간** : 웹 페이지의 모든 리소스 (HTML, CSS, JavaScript, 이미지 등)가 완전히 로딩되는 데 걸리는 시간입니다.  



# 📍 CDN 도입 전과 도입 후의 성능 개선 보고서
### 1. 개요
- 목적 :  CDN 성능 분석 및 개선 결과 보고
- 배경 :  기존 S3 단독 사용 시 성능 문제 및 개선 필요성
- 테스트 URL
  - 개선 전 : http://performance-inspection-practice.s3-website-ap-southeast-2.amazonaws.com  
  - 개선 후 : https://d2bs5sb8hgr7py.cloudfront.net/
- 테스트 도구 : Chrome DevTools (네트워크 탭)  
- 테스트 환경 : Chrome 브라우저 (PC)  

### 2. 개선 방안
#### 1. CloudFront CDN 적용
- 기존 S3 단독 배포 방식에서 CloudFront CDN을 적용하여 정적 콘텐츠를 캐싱하고, 사용자와 가까운 엣지 로케이션에서 제공하도록 설정했습니다.

#### 2. 캐싱 최적화
- CloudFront 기본 캐시 정책을 통해 반복 요청 시 S3 부하를 줄이고 응답 속도를 개선했습니다.

#### 3. HTTPS 및 압축 적용
- CloudFront를 통해 HTTPS와 Gzip 압축을 적용하여 네트워크 트래픽을 최적화하고, 콘텐츠 로딩 속도를 개선했습니다.

### 3. CloudFront CDN 적용 후 성능 비교
<img width="815" alt="image" src="https://github.com/user-attachments/assets/f84a1b01-5a1f-4a13-9e88-7fdfc3f8c799" />
<img width="892" alt="image" src="https://github.com/user-attachments/assets/5203ce23-c8a8-4c04-a0e1-097c61cb11e3" />
<img width="957" alt="image" src="https://github.com/user-attachments/assets/76970973-41c9-4e2c-bf28-6568fa577b8d" />
<img width="1002" alt="image" src="https://github.com/user-attachments/assets/71d4dec3-5625-4b57-83f0-063baf5d03c0" />


| 측정 지표                 | S3 단독 사용 | CloudFront CDN 적용 | 개선율 (%) |
|--------------------------|-------------|-------------------|-----------|
| FCP  | 750.8 ms | 122.1 ms | 84% |
| LCP  | 750.8 ms | 122.1 ms | 84% |
| TTFB | 165.65 ms | 6.85 ms | 95% |
| DOMContentLoaded | 360 ms |48 ms | 87%|
| 웹사이트 로드 시간 | 685 ms | 291 ms | 58% |

### 4. 분석 및 결론

- CDN 도입 후 전반적인 성능 지표가 크게 개선되었습니다. 
- 특히 서버 응답 속도를 나타내는 TTFB가 165.65ms에서 6.85ms로 개선되어 가장 큰 성능 향상을 보였습니다.
- 사용자가 실제로 체감하는 속도를 나타내는 FCP와 LCP는 각각 750.8ms에서 122.1ms로 84% 감소하였으며, 전체 웹사이트 로드 시간은 685ms에서 291ms로 58% 단축되었습니다.

### 5. 추가 개선 가능성
- 이미지 최적화
  - WebP 포맷 사용 및 이미지 크기 조정
  - 이미지 레이지 로딩(lazy loading) 적용
- 브라우저 캐싱 정책 최적화
  - CloudFront 캐싱 설정 세밀화
  - 적절한 TTL(Time to Live) 설정
- 코드 최적화
  - JavaScript 및 CSS 파일 최소화
  - JavaScript의 lazy loading 적용

