# 프론트엔드 배포 파이프라인 보고서

## 개요

GitHub Actions에 워크플로우를 작성해 다음과 같이 배포가 진행되도록 합니다.

(사전작업: Ubuntu 최신 버전 설치)

1. Checkout 액션을 사용해 코드 내려받기
2. `npm ci` 명령어로 프로젝트 의존성 설치
3. `npm run build` 명령어로 Next.js 프로젝트 빌드
4. AWS 자격 증명 구성
5. 빌드된 파일을 S3 버킷에 동기화
6. CloudFront 캐시 무효화

<br />

## 주요 링크

S3 버킷 웹사이트 엔드포인트: [http://hanghae-s3-song.s3-website-us-east-1.amazonaws.com/](http://hanghae-s3-song.s3-website-us-east-1.amazonaws.com/)

CloudFront 배포 도메인 이름: [https://d1vf9srvbxwqle.cloudfront.net/](https://d1vf9srvbxwqle.cloudfront.net/)

<br />

## 주요 개념

![Image](https://github.com/user-attachments/assets/9b7d7f77-e8fb-4e16-bf03-ec50e160e45d)

### GitHub Actions과 CI/CD 도구

GitHub Actions는 코드 변경이 감지되면 자동으로 배포를 수행하는 CI/CD(Continuous Integration & Continuous Deployment) 도구입니다.

- 현재 `main` 브랜치에 **push** 이벤트가 발생하면 워크플로우가 실행됩니다.
- `npm ci`로 프로젝트 내부 의존성을 설치하고, `npm run build`로 Next.js 프로젝트를 빌드합니다.
- 빌드된 정적 파일을 AWS S3에 업로드하고, CloudFront 캐시를 무효화 하는 작업을 합니다.
- CI/CD를 사용하면 수동 배포 없이 코드의 변경사항이 자동으로 반영되어 편리하고 일관성 있는 배포가 가능합니다.

### S3와 스토리지

AWS S3(Simple Storage Service) 는 정적 웹사이트를 배포하는 데 자주 사용되는 클라우드 스토리지 서비스입니다.

- Next.js 프로젝트를 `next export`하면 /out에 빌드 결과물이 생성되고 해당 폴더 내부의 하위 파일들을 S3에 업로드 합니다.
- S3는 확장성이 뛰어나고, 안정적이며, 사용 한 만큼 과금되는 비용 측면에서 효율적인 정적 파일 저장소로 적합합니다.

### CloudFront와 CDN

AWS CloudFront는 CDN(Content Delivery Network) 역할을 하여 전 세계에 **Edge Server(Location)** 을 두어 사용자에게 가장 가까운 곳에 캐싱해두고 S3의 정적 파일을 응답해주는 역할을 합니다.

- 사용자가 Next.js로 배포한 웹사이트에 접속하면 CloudFront의 엣지 서버에서 가까운 캐시된 파일을 제공하여 로딩 속도를 단축합니다.
- Route 53과 연동하여 도메인을 CloudFront에 연결할 수 있습니다.

### 캐시 무효화(Cache Invalidation)

CloudFront는 성능을 위해 S3에서 가져온 파일을 캐싱 합니다. 하지만 새로운 배포 후에도 기존 캐시가 유지되면, 사용자가 변경된 파일을 보지 못하는 문제가 발생할 수 있습니다.

- CloudFront로 배포되는 파일의 캐시가 유지되는 기본 시간은 24시간입니다.
- 이를 방지하기 위해 CloudFront 캐시를 강제로 무효화 해야 합니다.
- 파이프라인에서 CloudFront 캐시를 무효화하기 위해 invalidation 처리를 하고 있는 걸 확인 할 수 있습니다.

### Repository secret과 환경변수

GitHub Actions에서 배포할 때 AWS 자격 증명(AWS Access Key, Secret Key), S3 버킷 이름, CloudFront 배포 ID 등의 민감한 정보를 사용해야 합니다.
이러한 정보를 코드에 직접 포함하면 보안 문제가 발생할 수 있으므로 GitHub Actions의 secrets와 환경변수를 사용하여 안전하게 관리를 해주어야 합니다.

- GitHub의 secrets 기능을 사용하면, 배포 스크립트 내에서만 환경변수로 사용 가능하며, 저장소에서 직접 확인할 수 없습니다.
- 배포에 필요한 정보를 코드에서 직접 수정할 필요 없이, GitHub Secrets에서 변경할 수 있습니다.
- 개발(Dev), 스테이징(Staging), 운영(Production) 환경마다 다른 AWS 자격 증명, S3_BUCKET_NAME,CLOUDFRONT_DISTRIBUTION_ID등을 사용할 수 있습니다.

<br />

# CDN과 성능최적화

### CDN(CloudFront)을 사용했을 때 이점

- S3는 특정 리전에 파일을 저장하기 때문에 사용자가 해당 리전에 직접 요청해야 합니다. 예를 들어 해당 리전이 미국에 있다면 한국에서 미국까지의 거리만큼 지연시간이 생길 수 있습니다 **CDN(CloudFront)** 을 사용하면 엣지 로케이션을 활용하기 때문에 사용자와 가장 가까운 서버에서 콘텐츠를 제공하여 지연 시간을 감소 시킬 수 있습니다.

- **CDN(CloudFront)** 는 요청 한 정적 파일에 대해 캐싱을 적용하여 반복 요청을 줄여주고 페이지 로딩 속도가 빨라집니다.

- CloudFront는 자동으로 Brotli 압축을 적용하는데 이는 컨텐츠를 작게 압축하여 데이터 전송량이 줄어들게 해주기 때문에 페이지 로딩 속도가 개선됩니다. 현재 프로젝트에도 Brotli 압축 방식이 적용되어 있습니다.
  - Response Header에서 `content-encoding: br` Brotli을 이용한 압축이 적용되어 있는 걸 확인 할 수 있습니다

<br />

### CDN(CloudFront) 적용 하기 전 S3로만 배포 된 웹의 네트워크 탭

<img width="1431" alt="Image" src="https://github.com/user-attachments/assets/ef8c36bb-168e-4212-9fce-a41c0e3d7603" />

### CloudFront

<img width="1403" alt="Image" src="https://github.com/user-attachments/assets/777f8c55-1aa1-4f49-a88c-a6167d1c61f8" />

<br />

### 네트워크 탭에서 확인 한 개선 사항

| **항목**                             | **S3** | **CDN (CloudFront)** | **개선된 부분**                                                         |
| ------------------------------------ | ------ | -------------------- | ----------------------------------------------------------------------- |
| **Transferred (전송된 크기)**        | 1.2MB  | 931KB                | 콘텐츠를 압축하고 캐싱을 사용하여 전송 데이터 크기가 감소했습니다.      |
| **Resource Size (리소스 크기)**      | 1.2MB  | 1.2MB                | 동일 (리소스 크기는 변하지 않습니다.)                                   |
| **Finish (페이지 로드 완료)**        | 8.14s  | 7.13s                | 가까운 엣지 서버 사용으로 페이지 로드 시간이 단축되었습니다. 약 1s 개선 |
| **DOMContentLoaded (DOM 로드 완료)** | 792ms  | 528ms                | 캐시된 콘텐츠 제공으로 DOM 로드 속도가 개선되었습니다. 264ms 개선       |
| **Load (전체 페이지 로드)**          | 2.28s  | 1.09s                | 캐싱과 엣지 로케이션으로 페이지 로드 속도가 단축되었습니다. 1.19s 개선  |
