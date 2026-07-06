# API 명세서 (최신, 합본)

> 기준일: 2026-07-06
> 

> 이 문서는 기존 `API 명세서 (Fluter ↔ Spring)`와 `API 명세서 - AI 기능 (Fluter ↔ Spring ↔ FastAPI)`를 통합하고, 현재 프로젝트 코드 기준으로 최신화한 마이피티 API 명세서입니다. GitHub 공개 업로드를 고려해 실제 운영 도메인, 토큰, OAuth 클라이언트 값, S3 버킷명, 환경변수 값 등 민감하거나 환경 의존적인 정보는 포함하지 않습니다.
>
 
<br>

## 목차

1. 문서 범위
2. 시스템 구성과 API 경계
3. 공통 규약
4. Flutter ↔ Spring API
5. Flutter ↔ Spring ↔ FastAPI AI 리포트 API
6. 관리자 API
7. 에러 및 운영 정책

<br>

## 1. 문서 범위

- 사용자 앱 API: Flutter 앱에서 Spring Boot API 서버로 호출하는 인증, 회원, 루틴, 운동 기록, 분석, 커뮤니티, 신고, 이미지, 앱 버전 API를 포함합니다.
- AI 리포트 API: Flutter가 운동/자세 분석 결과를 Spring에 전달하고, Spring이 운동 기록과 사용자 맥락을 보강해 FastAPI AI 서버에 분석을 요청하는 흐름을 포함합니다.
- 관리자 API: React 관리자 웹에서 호출하는 카테고리, 운동 종목, 공식 루틴, 사용자, 게시글/댓글, 신고, AI quota, 앱 버전 정책 API를 포함합니다.
- 제외 범위: 실제 배포 도메인, API 키, 토큰, OAuth 클라이언트 ID, S3 버킷명, 내부 서버 IP, `.env` 값은 문서에 포함하지 않습니다.

## 2. 시스템 구성과 API 경계

| 구간 | 호출 방향 | 역할 |
| --- | --- | --- |
| 사용자 앱 | Flutter → Spring Boot | 인증, 회원/설정, 루틴/운동 기록, 분석, 커뮤니티, 이미지, 앱 버전 확인 |
| AI 리포트 | Flutter → Spring Boot → FastAPI | Flutter가 운동별 자세 분석 요약을 전달하고, Spring이 저장된 운동 기록/이전 기록/사용자 맥락을 합쳐 FastAPI에 요청 |
| 자세 분석 | Flutter 내부 로컬 엔진 | 실시간 자세 분석, 반복 횟수, 정확도, 자세 이슈 집계. 현재 사용자 앱에서는 HTTP FastAPI 호출이 아니라 로컬 엔진으로 처리 |
| 관리자 웹 | React → Spring Boot | 운영 데이터, AI quota, 앱 버전, 신고/커뮤니티 관리 |

## 3. 공통 규약

### 3.1 Base URL

- Spring API: `{SPRING_API_BASE_URL}`
- FastAPI AI Server: `{FASTAPI_BASE_URL}`
- 로컬 개발, 운영, 테스트 환경의 실제 URL은 환경변수 또는 배포 설정에서 관리합니다.

### 3.2 Header

```
Content-Type: application/json
Authorization: Bearer {ACCESS_TOKEN}
```

- 인증이 필요 없는 API: 회원가입, 로그인, 구글 로그인, 토큰 재발급, Android 앱 버전 확인
- 관리자 API: `ROLE_ADMIN` 권한을 가진 Access Token 필요
- 이미지 업로드 API: `multipart/form-data` 사용

### 3.3 Date Format

- 날짜: `yyyy-MM-dd`
- 일시: `yyyy-MM-dd'T'HH:mm:ss`
- 앱 버전 비교: Android `versionCode` 정수 기준

### 3.4 공통 응답 Wrapper

성공 응답은 Spring `ApiResponse<T>` 구조로 반환합니다.

```json
{
  "success": true,
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": {}
}
```

실패 응답은 비즈니스 에러 코드와 메시지를 포함합니다. Validation 실패 시 `errors`가 추가될 수 있습니다.

```json
{
  "success": false,
  "code": "C001",
  "message": "입력값 유효성 실패",
  "errors": [
    {
      "field": "email",
      "value": "wrong-email",
      "reason": "올바른 이메일 형식이 아닙니다."
    }
  ]
}
```

### 3.5 Paging

Spring `Page<T>` 응답은 `data.content`에 실제 목록이 들어가며, `page`, `size`, `sort` 쿼리 파라미터를 사용할 수 있습니다.

```json
{
  "success": true,
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": {
    "content": [],
    "pageable": {},
    "totalElements": 0,
    "totalPages": 0,
    "size": 10,
    "number": 0
  }
}
```

## 4. Flutter ↔ Spring API

### 4.1 Auth

| Method | URI | Auth | 설명 |
| --- | --- | --- | --- |
| POST | `/auth/signup` | X | 이메일, 비밀번호, 닉네임으로 회원가입 |
| POST | `/auth/login` | X | 이메일/비밀번호 로그인 |
| POST | `/auth/login/google` | X | Google ID Token 검증 후 JWT 발급 |
| POST | `/auth/reissue` | X | Refresh Token 기반 Access/Refresh Token 재발급 |
| POST | `/auth/logout` | X | 서버 저장 Refresh Token 삭제 |
| POST | `/auth/withdraw` | O | 비밀번호 확인 후 회원 탈퇴 처리 |

회원가입 요청:

```json
{
  "email": "user@example.com",
  "password": "Password123!",
  "nickname": "마이피티"
}
```

로그인 응답:

```json
{
  "accessToken": "{ACCESS_TOKEN}",
  "refreshToken": "{REFRESH_TOKEN}",
  "tokenType": "Bearer"
}
```

### 4.2 Member, Settings, Friendship

| Method | URI | Auth | 설명 |
| --- | --- | --- | --- |
| GET | `/members/me` | O | 프로필 메인 화면용 계정/신체/활동 요약 조회 |
| GET | `/members/me/details` | O | 프로필 수정 화면 데이터 조회 |
| PUT | `/members/me/details` | O | 닉네임, 프로필 이미지, 신체 정보, 활동 수준, 공개 범위 수정 |
| GET | `/members/me/settings` | O | 앱 설정 조회 |
| PATCH | `/members/me/settings` | O | 푸시, 마케팅 수신, 단위, 테마 설정 수정 |
| POST | `/members/me/community-rules-agreement` | O | 커뮤니티 이용 규정 동의 버전 저장 |
| GET | `/members/me/achievements` | O | 목표 달성도 및 배지 조회 |
| GET | `/friendships?status={status}` | O | 친구/요청/차단 목록 조회 |
| POST | `/friendships` | O | 이메일 기반 친구 요청 |
| PATCH | `/friendships/{friendshipId}` | O | 친구 요청 수락/거절/차단 |
| DELETE | `/friendships/{friendshipId}` | O | 친구 삭제 또는 차단 해제 |
| POST | `/users/{targetUserId}/block` | O | 특정 사용자 차단 |

프로필 수정 요청:

```json
{
  "nickname": "홍길동",
  "profileImage": "https://example.com/profile.jpg",
  "gender": "MALE",
  "birthDate": "1998-01-01",
  "height": 175.0,
  "weight": 70.0,
  "targetWeight": 68.0,
  "activityLevel": "NORMAL",
  "experienceLevel": "BEGINNER",
  "privacyLevel": "PUBLIC"
}
```

앱 설정 응답:

```json
{
  "pushNotification": true,
  "marketingAgree": false,
  "weightUnit": "KG",
  "distanceUnit": "KM",
  "theme": "SYSTEM",
  "communityRulesVersion": 1,
  "communityRulesAgreedAt": "2026-07-06T12:00:00"
}
```

### 4.3 Program, Exercise

| Method | URI | Auth | 설명 |
| --- | --- | --- | --- |
| GET | `/programs/me?type=ALL&page=0&size=10` | O | 내 루틴 목록 조회. `ALL`, `CREATED`, `SAVED` 지원 |
| GET | `/programs/explore?difficulty=ALL&page=0&size=10` | O | 공식/공개 루틴 탐색 |
| POST | `/programs` | O | 새 루틴 생성 |
| GET | `/programs/{programId}` | O | 루틴 상세 조회 |
| PUT | `/programs/{programId}` | O | 루틴 전체 수정 |
| DELETE | `/programs/{programId}` | O | 루틴 삭제 |
| POST | `/programs/{programId}/copy` | O | 공개/공식 루틴을 내 루틴으로 복제 |
| GET | `/categories` | O | 운동 카테고리 목록 조회 |
| GET | `/exercises?categoryId={id}&keyword={keyword}` | O | 운동 종목 목록 조회 |
| GET | `/exercises/{exerciseId}` | O | 운동 종목 상세 조회 |

루틴 생성/수정 요청:

```json
{
  "title": "하체 루틴",
  "description": "스쿼트 중심 하체 루틴",
  "difficulty": "INTERMEDIATE",
  "isPublic": true,
  "exercises": [
    {
      "exerciseId": 50,
      "orderIndex": 1,
      "targetSets": 4,
      "targetReps": 12,
      "restSeconds": 90
    }
  ]
}
```

루틴 목록 주요 응답 필드:

```json
{
  "id": 1,
  "title": "초보자 전신 루틴",
  "difficulty": "BEGINNER",
  "description": "전신 기초 루틴",
  "creatorName": "시스템 루틴",
  "estimatedMinutes": 30,
  "estimatedCalories": 200,
  "tags": ["하체", "가슴"],
  "isMine": false,
  "isOfficial": true,
  "isPublic": true,
  "copiedCount": 10,
  "lastPerformedAt": null
}
```

### 4.4 Workout, Analysis

| Method | URI | Auth | 설명 |
| --- | --- | --- | --- |
| POST | `/workouts/sessions` | O | 운동 완료 후 세션/세트 저장 |
| GET | `/workouts/sessions/calendar?startDate={date}&endDate={date}` | O | 기간별 운동 기록 유무 조회 |
| GET | `/workouts/sessions/attachable` | O | 커뮤니티 글에 첨부 가능한 최근 운동 기록 조회 |
| GET | `/workouts/sessions?date={date}` | O | 특정 날짜의 운동 기록 목록 조회 |
| GET | `/workouts/sessions?page=0&size=10` | O | 전체 운동 히스토리 페이징 조회 |
| GET | `/workouts/sessions/{sessionId}` | O | 운동 기록 상세 조회 |
| DELETE | `/workouts/sessions/{sessionId}` | O | 운동 기록 삭제 |
| GET | `/analysis/dashboard?today={date}` | O | 분석 대시보드 조회 |
| GET | `/analysis/weekly-stats?startDate={date}&endDate={date}` | O | 주간 통계 상세 조회 |

운동 기록 저장 요청:

```json
{
  "programId": 5,
  "status": "COMPLETED",
  "scheduledAt": "2026-07-06T18:00:00",
  "startTime": "2026-07-06T18:00:00",
  "endTime": "2026-07-06T19:00:00",
  "totalVolume": 8500,
  "totalTimeSeconds": 3600,
  "moodMemo": "컨디션 좋았음",
  "exercises": [
    {
      "exerciseId": 50,
      "sets": [
        {
          "setNumber": 1,
          "weight": 60.0,
          "loadStatus": "NORMAL",
          "reps": 12,
          "rpe": 7,
          "restSeconds": 60,
          "isWarmup": false,
          "isCompleted": true
        }
      ]
    }
  ]
}
```

분석 대시보드 응답 주요 필드:

```json
{
  "summary": {
    "totalWorkoutCount": 156,
    "weeklyTotalCalories": 2450
  },
  "balance": {
    "score": 82,
    "radarChart": [
      { "label": "하체", "value": 90 }
    ]
  },
  "weeklyStats": {
    "startDate": "2026-07-06",
    "endDate": "2026-07-12",
    "totalCalories": 2450,
    "dailyData": [
      {
        "dayOfWeek": "MON",
        "date": "2026-07-06",
        "calories": 500,
        "isWorkoutDay": true
      }
    ]
  }
}
```

### 4.5 Community, Report, Image

| Method | URI | Auth | 설명 |
| --- | --- | --- | --- |
| GET | `/posts?type=ALL&period=WEEKLY&page=0&size=10` | O | 게시글 목록 조회. `ALL`, `FRIENDS`, `POPULAR`, `QUESTION` 지원 |
| POST | `/posts` | O | 게시글 작성. 운동 기록 첨부 가능 |
| GET | `/posts/{postId}` | O | 게시글 상세 및 댓글 조회 |
| PATCH | `/posts/{postId}` | O | 게시글 수정 |
| DELETE | `/posts/{postId}` | O | 게시글 삭제 |
| POST | `/posts/{postId}/likes` | O | 좋아요 토글 |
| POST | `/posts/{postId}/comments` | O | 댓글 작성 |
| PATCH | `/comments/{commentId}` | O | 댓글 수정 |
| DELETE | `/comments/{commentId}` | O | 댓글 삭제 |
| POST | `/reports` | O | 게시글/댓글/회원 신고 접수 |
| POST | `/images/upload` | O | 단건 이미지 업로드 |
| POST | `/images/upload/multiple` | O | 다건 이미지 업로드 |
| DELETE | `/images?imageUrl={url}` | O | 이미지 삭제 |

게시글 작성 요청:

```json
{
  "content": "오늘 하체 운동 완료",
  "imageUrl": "https://example.com/image.jpg",
  "sessionId": 123,
  "type": "WORKOUT"
}
```

게시글 응답 주요 필드:

```json
{
  "postId": 1,
  "type": "WORKOUT",
  "user": {
    "userId": 10,
    "nickname": "마이피티",
    "profileImage": "https://example.com/profile.jpg"
  },
  "content": "오늘 하체 운동 완료",
  "imageUrl": "https://example.com/image.jpg",
  "createdAt": "2026-07-06T12:00:00",
  "likeCount": 3,
  "commentCount": 1,
  "isLiked": true,
  "workoutSummary": {
    "sessionId": 123,
    "title": "하체 루틴",
    "totalTimeSeconds": 3600,
    "totalVolume": 8500,
    "burnedCalories": 450,
    "mainBodyParts": ["하체"]
  }
}
```

신고 요청:

```json
{
  "targetType": "POST",
  "targetId": 1,
  "reason": "SPAM",
  "details": "반복 홍보성 게시글입니다."
}
```

### 4.6 App Version

| Method | URI | Auth | 설명 |
| --- | --- | --- | --- |
| GET | `/public/app-version/android?currentVersionCode={code}` | X | 서버 정책 기준 Android 업데이트 필요 여부 확인 |

응답:

```json
{
  "platform": "ANDROID",
  "currentVersionCode": 9,
  "latestVersionName": "1.0.3",
  "latestVersionCode": 12,
  "minimumVersionCode": 10,
  "updateRequired": true,
  "forceUpdate": true,
  "updateType": "FORCE",
  "message": "최신 버전으로 업데이트해 주세요.",
  "storeUrl": "https://example.com/store"
}
```

## 5. Flutter ↔ Spring ↔ FastAPI AI 리포트 API

### 5.1 처리 흐름

1. Flutter 앱은 운동 중 로컬 자세 분석 엔진으로 운동별 정확도, 총 프레임 수, 자세 이슈를 집계합니다.
2. 운동 완료 시 `POST /workouts/sessions`로 운동 기록을 저장합니다.
3. 사용자가 AI 리포트 생성을 요청하면 Flutter가 `POST /workouts/{sessionId}/ai-report`를 호출합니다.
4. Spring은 사용자 권한, 운동 기록 소유권, AI 기능 활성화 여부, 유저별/전체 일일 quota를 검증합니다.
5. Spring은 저장된 운동 세트, 이전 운동 기록, 사용자 목표/경험 수준, Flutter 자세 분석 결과를 결합해 FastAPI 요청을 구성합니다.
6. FastAPI는 Pydantic 스키마로 요청을 검증하고 Gemini 기반 피드백을 생성합니다.
7. Spring은 FastAPI 응답을 내부 응답 모델로 변환해 DB에 저장하고 Flutter에 반환합니다.

### 5.2 Flutter → Spring

| Method | URI | Auth | 설명 |
| --- | --- | --- | --- |
| POST | `/workouts/{sessionId}/ai-report` | O | AI 운동 분석 리포트 생성 |
| GET | `/workouts/{sessionId}/ai-report` | O | 저장된 AI 운동 분석 리포트 조회 |

생성 요청:

```json
{
  "routines": [
    {
      "exerciseId": 50,
      "exerciseType": "스쿼트",
      "accuracy": 84.5,
      "totalFrames": 1200,
      "postureIssues": [
        { "code": "knees_caving", "count": 3 },
        { "code": "leaning_forward", "count": 2 }
      ]
    }
  ]
}
```

Spring 응답:

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "workoutSessionId": 123,
  "status": "COMPLETED",
  "executionTimeMs": 1250,
  "delayTimeMs": 0,
  "totalScore": 84.5,
  "createdAt": "2026-07-06T12:00:00",
  "coachingMode": "POSE_AND_RECORD",
  "overallFeedback": {
    "summaryLine": "전체적으로 안정적인 운동을 수행했습니다.",
    "strength": "반복 수행의 안정성이 좋습니다.",
    "priority": "무릎 정렬을 우선 개선하세요.",
    "nextPlan": "다음 운동에서는 하강 구간을 천천히 수행하세요."
  },
  "recordBasedAnalysis": {
    "routineType": "LOWER_BODY",
    "summary": "이전 기록 대비 운동량이 증가했습니다.",
    "findings": ["총 볼륨이 증가했습니다."],
    "nextActions": ["다음 운동에서는 동일 중량으로 반복 수를 유지하세요."]
  },
  "generationSource": "AI",
  "modelName": "{MODEL_NAME}",
  "promptVersion": "{PROMPT_VERSION}",
  "fallbackReason": null,
  "analysisDetails": [
    {
      "detailId": 1,
      "exerciseId": 50,
      "exerciseType": "스쿼트",
      "feedbackType": "POSE",
      "accuracy": 84.5,
      "postureErrorLog": "무릎이 안쪽으로 모임",
      "summaryLine": "스쿼트 자세에서 무릎 정렬 개선이 필요합니다.",
      "detailedAnalysis": [
        "무릎과 발끝 방향을 맞춰 하강하세요.",
        "코어를 고정하고 상체 기울어짐을 줄이세요."
      ]
    }
  ]
}
```

### 5.3 Spring → FastAPI

| Method | URI | Auth | 설명 |
| --- | --- | --- | --- |
| POST | `/analyze/routine` | 내부 호출 | 운동 세션 기반 AI 피드백 생성 |

Spring이 FastAPI로 전달하는 요청:

```json
{
  "routines": [
    {
      "exercise_id": 50,
      "exercise_type": "스쿼트",
      "category": "하체",
      "order_index": 1,
      "sets": [
        {
          "set_number": 1,
          "weight": 60.0,
          "load_status": "NORMAL",
          "reps": 12,
          "rpe": 7,
          "rest_seconds": 60,
          "is_completed": true
        }
      ],
      "previous_record": {
        "performed_at": "2026-07-01T18:00:00",
        "sets": []
      },
      "pose_available": true,
      "accuracy": 84.5,
      "total_frames": 1200,
      "posture_issues": [
        { "code": "knees_caving", "count": 3 }
      ]
    }
  ],
  "totalTimeSeconds": 3600,
  "userContext": {
    "goal": "MUSCLE_GAIN",
    "experience_level": "BEGINNER",
    "weekly_frequency": 3
  }
}
```

FastAPI 응답:

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "COMPLETED",
  "executionTime": 1250,
  "delayTime": 0,
  "total_score": 84.5,
  "output": {
    "coaching_mode": "POSE_AND_RECORD",
    "overall_feedback": {
      "summary_line": "전체적으로 안정적인 운동을 수행했습니다.",
      "strength": "반복 수행의 안정성이 좋습니다.",
      "priority": "무릎 정렬을 우선 개선하세요.",
      "next_plan": "다음 운동에서는 하강 구간을 천천히 수행하세요."
    },
    "record_based_analysis": {
      "routine_type": "LOWER_BODY",
      "summary": "이전 기록 대비 운동량이 증가했습니다.",
      "findings": ["총 볼륨이 증가했습니다."],
      "next_actions": ["동일 중량에서 반복 수를 유지하세요."]
    },
    "pose_feedbacks": [],
    "progress_feedbacks": [],
    "generation_source": "AI",
    "model_name": "{MODEL_NAME}",
    "prompt_version": "{PROMPT_VERSION}",
    "fallback_reason": null
  }
}
```

### 5.4 AI 리포트 운영 정책

AI 리포트 생성은 외부 AI API 비용과 지연 시간이 발생하므로 Spring에서 서버 주도 정책으로 제한합니다.

| 정책 | 설명 |
| --- | --- |
| 기능 활성화 | `reportEnabled=false`이면 사용자 리포트 생성을 차단 |
| 유저별 일일 quota | 사용자 1명 기준 하루 생성 가능 횟수 제한 |
| 서비스 전체 일일 quota | 전체 서비스 기준 하루 생성 가능 횟수 제한 |
| 비활성화 사유 | 관리자 웹에서 사용자 안내 문구로 사용할 수 있는 사유 저장 |

## 6. 관리자 API

모든 관리자 API는 관리자 권한 Access Token이 필요합니다.

### 6.1 운영 설정

| Method | URI | 설명 |
| --- | --- | --- |
| POST | `/auth/admin/login` | 관리자 로그인 |
| GET | `/admin/ai-settings` | AI 리포트 생성 정책 조회 |
| PUT | `/admin/ai-settings` | AI 리포트 생성 정책 수정 |
| GET | `/admin/app-version/android` | Android 앱 버전 정책 조회 |
| PUT | `/admin/app-version/android` | Android 앱 버전 정책 수정 |

AI 설정 수정 요청:

```json
{
  "reportEnabled": true,
  "userDailyLimit": 3,
  "globalDailyLimit": 100,
  "disabledReason": null
}
```

앱 버전 정책 수정 요청:

```json
{
  "latestVersionName": "1.0.3",
  "latestVersionCode": 12,
  "minimumVersionCode": 10,
  "message": "최신 버전으로 업데이트해 주세요.",
  "storeUrl": "https://example.com/store"
}
```

### 6.2 카테고리, 운동 종목, 공식 루틴

| Method | URI | 설명 |
| --- | --- | --- |
| GET | `/admin/categories` | 전체 카테고리 조회 |
| POST | `/admin/categories` | 카테고리 생성 |
| PATCH | `/admin/categories/order` | 카테고리 노출 순서 변경 |
| PUT | `/admin/categories/{categoryId}` | 카테고리 수정 |
| DELETE | `/admin/categories/{categoryId}` | 카테고리 삭제 |
| GET | `/admin/exercises` | 운동 종목 목록 조회. `categoryId`, `isActive`, `keyword`, paging 지원 |
| POST | `/admin/exercises` | 운동 종목 단건 등록 |
| POST | `/admin/exercises/bulk` | 운동 종목 다건 등록 |
| PUT | `/admin/exercises/{exerciseId}` | 운동 종목 수정 |
| PATCH | `/admin/exercises/{exerciseId}/status` | 운동 종목 활성/비활성 변경 |
| GET | `/admin/programs` | 공식 루틴 목록 조회 |
| POST | `/admin/programs` | 공식 루틴 생성 |
| POST | `/admin/programs/bulk` | 공식 루틴 다건 생성 |
| PUT | `/admin/programs/{programId}` | 공식 루틴 수정 |
| DELETE | `/admin/programs/{programId}` | 공식 루틴 삭제 |

### 6.3 사용자, 커뮤니티, 신고, 파일

| Method | URI | 설명 |
| --- | --- | --- |
| GET | `/admin/users` | 유저 목록 조회 |
| PATCH | `/admin/users/{userId}/status` | 유저 상태 변경 |
| PATCH | `/admin/users/{userId}/withdraw` | 관리자 권한 유저 탈퇴 처리 |
| GET | `/admin/posts` | 전체 게시글 목록 조회 |
| DELETE | `/admin/posts/{postId}` | 게시글 강제 삭제 |
| DELETE | `/admin/comments/{commentId}` | 댓글 강제 삭제 |
| GET | `/admin/reports?status={status}` | 신고 목록 조회 |
| PATCH | `/admin/reports/{reportId}/resolve` | 신고 처리 |
| POST | `/admin/files/upload` | 관리자 전용 파일 업로드 |
| DELETE | `/admin/files?fileUrl={url}` | 관리자 전용 파일 삭제 |

## 7. 에러 및 운영 정책

### 7.1 주요 에러 코드

| 코드 | 의미 |
| --- | --- |
| `C001` | 입력값 유효성 실패 |
| `C002` | 인증 실패 |
| `C003` | 서버 내부 오류 |
| `A001` | 유효하지 않은 Refresh Token |
| `U001` | 이메일 중복 |
| `U002` | 사용자 없음 |
| `W001` | 프로그램 없음 |
| `W003` | 운동 종목 없음 |
| `W004` | 운동 기록 없음 |
| `W005` | AI 리포트 없음 |
| `W006` | 유저별 AI 리포트 일일 제한 초과 |
| `W007` | AI 리포트 기능 비활성화 |
| `W008` | 서비스 전체 AI 리포트 일일 제한 초과 |
| `W009` | AI 분석 가능한 운동 데이터 없음 |
| `W010` | AI 제공자 연결 실패 |
| `W011` | AI 응답 시간 초과 |
| `W012` | AI 응답 구조 오류 |
| `W013` | AI 리포트 생성 실패 |
| `CO001` | 게시글 없음 |
| `CO002` | 댓글 없음 |
| `CO003` | 커뮤니티 이용 규정 동의 필요 |
| `I001` | 파일 용량 초과 |
| `I002` | 빈 파일 |
| `I003` | 지원하지 않는 파일 확장자 |
| `I004` | S3 업로드 실패 |

### 7.2 FastAPI 에러 매핑

| FastAPI detail.code | Spring ErrorCode | HTTP 의미 |
| --- | --- | --- |
| `AI_INVALID_WORKOUT_DATA` | `W009` | 분석 가능한 운동 데이터 없음 |
| `AI_GENERATION_TIMEOUT` | `W011` | AI 생성 시간 초과 |
| `AI_INVALID_RESPONSE` | `W012` | AI 응답 구조 오류 |
| `AI_PROVIDER_UNAVAILABLE` | `W010` | 외부 AI 제공자 연결 실패 |

### 7.3 공개 문서 작성 원칙

- 실제 운영 도메인 대신 `{SPRING_API_BASE_URL}`, `{FASTAPI_BASE_URL}`을 사용합니다.
- Access Token, Refresh Token, OAuth Client ID, API Key, S3 Bucket, DB 접속 정보는 포함하지 않습니다.
- 예시 URL은 `https://example.com/...` 형태로만 작성합니다.
- AI 모델명과 프롬프트 버전은 필요 시 `{MODEL_NAME}`, `{PROMPT_VERSION}` 플레이스홀더로 표기합니다.
