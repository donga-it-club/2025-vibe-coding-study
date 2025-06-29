## AWS Lambda와 FFmpeg를 활용한 비용 효율적 영상 처리 파이프라인

---

## 프로젝트 배경

온라인 강의 플랫폼을 개발하면서 비디오 트랜스코딩과 DRM 적용이 필요했습니다. 기존 솔루션들을 검토한 결과 다음과 같은 문제점들을 발견했습니다.

**요구사항**

- 업로드된 영상을 다양한 해상도(480p, 720p, 1080p)로 변환
- 불법 다운로드 방지를 위한 DRM 적용
- 강사 1명 기준 주 2-3회 업로드 (평균 1-2시간 분량)
- 월 운영비 최소화

**기존 솔루션 검토**

- AWS MediaConvert: 기본 요금 + 분당 처리 비용
- 타사 SaaS: 월 구독료 + 트래픽 과금
- 자체 EC2 서버: 24시간 운영 비용 + 관리 오버헤드

## 기술 스택 선택 과정

### 컴퓨팅 플랫폼 비교

**AWS Lambda vs ECS Fargate vs EC2**

각 옵션의 특징을 분석했습니다

**AWS Lambda**

- 장점: 실행 시간만 과금, 자동 스케일링, 운영 부담 없음
- 단점: 15분 실행 시간 제한
- 적용 가능성: 긴 영상의 경우 세그먼트 분할 처리 필요

**ECS Fargate**

- 장점: 시간 제한 없음, 컨테이너 기반으로 FFmpeg 구성 용이
- 단점: 최소 1분 과금, 상대적으로 높은 비용 부담
- 적용 가능성: 긴 영상이나 복잡한 처리에 적합

**EC2 Spot Instance**

- 장점: 가장 높은 성능, GPU 가속 가능
- 단점: 인스턴스 관리 필요, 중단 위험성
- 적용 가능성: 대용량 배치 처리에 적합

### 메시징 시스템 SQS 선택 이유

**고려사항**

- 영상 업로드와 처리 프로세스 분리 필요
- 처리 실패시 재시도 메커니즘 필요
- 처리 상태 추적 및 모니터링 필요

**SQS 선택 근거**

- 메시지 내구성 보장 (처리 중 실패해도 메시지 보존)
- Dead Letter Queue를 통한 실패 분석 가능
- Lambda와의 기본 통합 지원
- 비용: 월 100만 요청까지 무료

### DRM 솔루션 설계

**기술 조사 결과**

- FFmpeg: 기본 AES-128 암호화 지원하지만 DRM 표준 미준수
- Bento4: CENC(Common Encryption) 표준 지원, Multi-DRM 가능
- Shaka Player: Google 오픈소스, 주요 DRM 지원

**선택한 방식**

1.  FFmpeg로 트랜스코딩 (DRM 적용 전)
2.  Bento4로 CENC 암호화 및 DASH/HLS 패키징
3.  Shaka Player로 클라이언트 재생

---

## 최종 아키텍처

### 전체 구조

```
S3 Upload → SQS Queue → Lambda Router → Processing Engine → S3 Output
```

**핵심 컴포넌트**

1.  **업로드 트리거**: S3 이벤트가 SQS 메시지 생성
2.  **라우터 Lambda**: 영상 메타데이터 분석 후 처리 방법 결정
3.  **처리 엔진**: Lambda 병렬 처리 또는 Fargate 실행
4.  **DRM 적용**: Bento4를 통한 암호화 및 패키징

### Lambda 병렬 처리 구현

15분 제한을 극복하기 위한 세그먼트 분할 방식

```
func processVideoWithSegmentation(videoPath string, duration int) error {
    // 12분 단위로 세그먼트 분할 (여유시간 고려)
    segmentDuration := 12 * 60 // 720초
    segmentCount := int(math.Ceil(float64(duration) / float64(segmentDuration)))

    // Step Functions로 병렬 처리 오케스트레이션
    segments := createSegments(videoPath, segmentCount, segmentDuration)
    return processSegmentsConcurrently(segments)
}
```

**병렬 처리 장점**

- 60분 영상을 이론적으로 12분 내 처리 가능
- 일부 세그먼트 실패시 해당 부분만 재처리
- 처리량에 따른 자동 스케일링

---

## 비용 분석

### AWS 요금 계산 (2025년 서울 기준)

**Lambda 비용 구조**

- 요청당: $0.0000002
- 실행시간: $0.0000166667 per GB-second
- 메모리 3,072MB 사용시: $0.0000000500 per ms

**예상 사용량 (주 3시간 업로드 기준)**

- 월 총 처리시간: 약 12시간 (원본 기준)
- Lambda 실행시간: 약 4시간 (병렬 처리 효과)
- 메모리: 3072MB
- 요청 수: 약 100회/월

**월 비용 계산**

```
Lambda 비용:
100회, 15분, 3GB 메모리 = $4.50 (프리티어 포함 시 $0, lambda의 경우 12개월 제한이 아니라 계속 포함되기에 항상 무료로 비용이 안 들 것으로 예상)
만약, 500회로 늘어날 시 $22.5 (프리티어 포함 시 $15.85)


Fargate 비용
100회, 15분, vCPU 1, 3GB 메모리 = $1.54
500회, ... = $7.74

위를 비교해봤을 때 프리티어 사용량까지 쓴다면 람다가 유리하고 그 이상 사용한다면 Fargate가 유리한 것을 알 수 있었다.
다만, Fargate 사용 시 15분의 제약이 사라지는 장점이 있긴 해서 만약 트랜스코딩에 대한 수요가 어느정도 있고 프로덕션 서비스의 경우 Fargate나 EC2 spot 인스턴스를 사용하는 것이 나아보인다.


S3 저장 비용
100GB × $0.023 = $2.30

SQS 비용
100 메시지 (프리 티어 내)

총 예상 비용: 약 $2.30/월
```

**기존 솔루션 대비**

- AWS MediaConvert: 분당 $0.015 (HD) × 180분 = $2.7/월 + 기본 요금 (다만, 커스텀이 불가능하기에 애초에 사용 고려 X)
- 자체 EC2 t3.medium: $24.34/월 (24시간 운영시)

### 비용 최적화 전략

1.  **S3 Lifecycle 정책**: 30일 후 IA, 90일 후 Glacier 이동
2.  **Lambda 메모리 최적화**: 영상 복잡도에 따라 동적 할당
3.  **Reserved Capacity**: 사용량 증가시 예약 인스턴스 활용

---

## 구현 상세

### 지능형 라우터 구현

```
func analyzeAndRoute(ctx context.Context, videoPath string) error {
    metadata, err := getVideoMetadata(videoPath)
    if err != nil {
        return err
    }

    // 처리 시간 예측
    estimatedTime := estimateProcessingTime(metadata)

    if estimatedTime <= 12*60 { // 12분 이하
        return processWithSingleLambda(ctx, videoPath)
    } else if estimatedTime <= 30*60 { // 30분 이하
        return processWithLambdaSegments(ctx, videoPath)
    } else {
        return processWithFargate(ctx, videoPath)
    }
}
```

### DRM 파이프라인

```
func applyDRM(inputFiles []string, outputDir string) error {
    // Bento4 mp4dash를 사용한 Multi-DRM 적용
    kid := generateKeyID()
    key := generateContentKey()

    args := []string{
        "--force",
        "--output-dir=" + outputDir,
        "--encryption-key=" + kid + ":" + key,
        "--widevine",
        "--playready",
        "--hls", // FairPlay 호환을 위한 HLS 출력
    }
    args = append(args, inputFiles...)

    cmd := exec.Command("mp4dash", args...)
    return cmd.Run()
}
```

---

## 모니터링 및 운영

### CloudWatch 메트릭

핵심 모니터링 지표

- Lambda 실행 시간 및 오류율
- SQS 메시지 대기 시간
- S3 저장 용량 및 비용
- 처리 성공률

### 알람 설정

```
alarms:
  - name: "Lambda_High_Error_Rate"
    metric: "AWS/Lambda/Errors"
    threshold: 3
    period: 300

  - name: "Processing_Queue_Backup"
    metric: "AWS/SQS/ApproximateNumberOfVisibleMessages"
    threshold: 10
    period: 600
```

---

## 프로젝트 성과 및 한계

### 달성한 성과

1.  **비용 효율성**: 기존 EC2 상시 운영 대비 월 $20 절약
2.  **운영 단순화**: 서버 관리 불필요, 자동 스케일링
3.  **처리 안정성**: SQS 기반 재시도 메커니즘으로 안정성 확보

### 기술적 제약사항

1.  **Lambda 제한**: 15분, 3008MB 메모리 제한
2.  **세그먼트 품질**: 분할 지점에서 미세한 품질 차이 가능성
3.  **콜드 스타트**: 첫 실행시 지연 시간 발생

### 개선 계획

1.  **품질 검증**: 세그먼트 병합 후 자동 품질 검사 추가
2.  **모니터링 강화**: 상세 성능 메트릭 수집
3.  **확장성**: 다중 테넌트 지원을 위한 아키텍처 개선

---

## 배운 점

### 기술적 학습

1.  **제약 조건의 활용**: Lambda 15분 제한을 병렬 처리 기회로 전환
2.  **비용 모델링**: 실제 사용 패턴 기반 정확한 비용 예측의 중요성
3.  **점진적 최적화**: MVP부터 시작하여 단계적 개선의 효과

### 아키텍처 설계 원칙

1.  **단순성 우선**: 복잡한 구조보다 이해하기 쉬운 설계
2.  **관찰 가능성**: 각 단계별 모니터링 포인트 설계
3.  **비용 의식**: 기술 선택시 비용 영향도 고려

---

## 참고 자료

### 기술 스택

- **언어**: Go (서버리스 함수), Terraform (인프라)
- **AWS 서비스**: Lambda, S3, SQS, CloudWatch
- **도구**: FFmpeg, Bento4, Shaka Player

### 관련 AWS 문서

- [AWS Lambda Pricing](https://aws.amazon.com/lambda/pricing/)
- [Processing user-generated content using AWS Lambda and FFmpeg](https://aws.amazon.com/blogs/media/processing-user-generated-content-using-aws-lambda-and-ffmpeg/)
- [Serverless Media Solution based on FFmpeg](https://www.amazonaws.cn/en/solutions/serverless-media-solution-based-on-ffmpeg/)

---

이 프로젝트를 통해 서버리스 아키텍처의 실용적 활용법과 AWS 서비스를 조합한 비용 효율적 솔루션 설계 경험을 얻을 수 있었습니다. 특히 제약 조건을 극복하는 방안을 생각해보며 조금 더 식견을 넓히는 데 중요한 경험이 되었다 생각합니다.
