# API 명세서 (Github 업로드용)

문서 유형: 설계 및 명세
분류: 💻 Dev
작성자: seen
진행 상태: 작성 중
최종 수정일: 2026년 3월 28일

# 1. 기본 설정 및 공통 규약 (Global Standards)

개발 단계에서 사용할 기본 호스트와 데이터 교환 규칙입니다.

### 1.1 Base URL

- **Dev:** `https://dev-api.fitnessapp.com/api/v1`
- **Prod:** `https://api.fitnessapp.com/api/v1`

### 1.2 공통 사항

- Header: 모든 요청/응답은 JSON 형식을 따릅니다.
    - `Content-Type`: `application/json`
    - `Authorization`: `Bearer {ACCESS_TOKEN}` (로그인 필요 API 시)
- Date Format: `yyyy-MM-dd` (날짜), `yyyy-MM-dd'T'HH:mm:ss` (일시)
- Paging**:** 목록 조회 시 `page` (0부터 시작), `size` (기본 10) 파라미터 사용 권장.
- **Authentication:** Header에 `Authorization: Bearer {Access_Token}` 포함

### 1.3 공통 응답 포맷 (Response Wrapper)

성공과 실패의 구조를 통일하여 프론트엔드에서 일관되게 파싱할 수 있도록 합니다.

### ✅ 성공 응답 (HTTP 200/201)

JSON

```json
{
  "success": true,
  "code": "SUCCESS",
  "message": "요청이 성공적으로 처리되었습니다.",
  "data": {
    // 실제 데이터 (Object or Array)
  }
}
```

### ❌ 실패 응답 (HTTP 4xx/5xx)

JSON

```json
{
  "success": false,
  "code": "U002",
  "message": "존재하지 않는 사용자입니다.",
  "errors": [  // 필드 유효성 검사 실패 시 상세 내용 (Optional)
    {
      "field": "email",
      "value": "wrong-email",
      "reason": "이메일 형식이 올바르지 않습니다."
    }
  ]
}
```

---

# 2. 도메인별 API 명세

## 2.1 인증 (Auth)

> 
> 

| **Method** | **URI** | **설명** | **Auth** |
| --- | --- | --- | --- |
| `POST` | `/auth/signup` | 소셜/이메일 회원가입 (토큰 발급) | X |
| `POST` | `/auth/login` | 소셜/이메일 로그인 | X |
| `GET` | `/users/me` | 내 프로필 정보 조회 | O |
| `GET` | `/users/me/settings` | 설정 페이지 진입 시 현재 설정값 반환 | O |
| `PATCH` | `/users/me/settings` | 사용자 설정 수정 (푸시, 공개여부) | O |

---

## 2.2 운동 프로그램 및 종목 (Programs & Exercises)

> 관련 Flow: 운동 페이지 -> 프로그램 탭 / 운동 탭
> 

| **Method** | **URI** | **설명** | **Auth** |
| --- | --- | --- | --- |
| `GET` | `/programs/me` | 내 루틴 목록 조회 | O |
| `GET` | `/programs/explore` | 공개/공식 루틴 목록 조회 | O |
| `POST` | `/programs` | 새 루틴 생성 | O |
| `GET` | `/programs/{programid}` | 루틴 상세 조회 | O |
| `PUT` | `/programs/{programid}` | 루틴 수정 | O |
| `DELETE` | `/programs/{programid}` | 루틴 삭제 | O |
| `POST` | `/programs/{programid}/copy` | 다른 루틴을 내 루틴으로 복제 | O |
| `GET` | `/categories` | 운동 카테고리 목록 조회 | O |
| `GET` | `/exercises` | 운동 종목 목록 조회 (필터/검색) | O |
| `GET` | `/exercises/{exerciseId}` | 운동 종목 상세 정보 | O |

### 2.2.1 내 루틴 목록 조회

- **Endpoint:** `GET /programs/me`
- **설명:** 내 보관함에 있는 모든 루틴을 조회합니다. 파라미터를 통해 '내가 만든 것'과 '저장(스크랩)한 것'을 필터링할 수 있습니다.
- **Query Parameters:**
    - `type` (Optional, Default: `ALL`)
        - `ALL`: 전체 조회 (생성 + 스크랩)
        - `CREATED`: 내가 직접 생성 루틴만 조회 (`creator_id` = 나)
        - `SAVED`: 다른 사람의 루틴을 복제(copy)한 루틴만 조회 (`PROGRAM_SAVES` 테이블 참조)
    - `page`: 페이지 번호 (0부터 시작)
    - `size`: 페이지 크기 (기본 10)
    - 내 루틴 목록 조회를 할 때 기본 정렬이 최근 수행한 순으로 하고 싶은데 Program 엔티티에 lastPerformedAt을 추가 안 하고 구현하는 방법이 없을까?
- **Response Body**
    
    ```json
    {
      "page": 0,
      "size": 10,
      "totalElements": 15,
      "programs": [
        {
          "id": 101,
          "title": "아침 홈트레이닝",
          "difficulty": "BEGINNER",
          "estimatedMinutes": 20,
          "estimatedCalories": 180,
          "tags": ["푸시업", "스쿼트"],
          "creatorName": "유저 1",        // 작성자 merberId로 닉네임 조회
          "isMine": true,                 // true: 내가 만든 루틴, false: 복제한 루틴
          "lastPerformedAt": "2026-02-14T07:00:00"
        },
        {
          "id": 5,
          "title": "초보자 전신 운동",
          "difficulty": "BEGINNER",
          "estimatedMinutes": 25,
          "estimatedCalories": 200,
          "tags": ["전신", "기초체력"],
          "creatorName": "관리자",        // 시스템 생성 루틴
          "isMine": false,              // true: 내가 만든 루틴, false: 복제한 루틴
          "lastPerformedAt": null
        }
      ]
    }
    ```
    

### 2.2.2 공개/공식 루틴 목록 조회

- **Endpoint:** `GET /programs/explore`
- **설명**
    - `creator_id`가 NULL(시스템)이거나, `is_public`이 TRUE(다른 유저)인 루틴을 조회합니다.
    - `source_program_id` 가 NULL이 아닌 복제된 루틴은 조회되지 않습니다.
- **Query Param:**
    - `sort`: `POPULAR` (인기순/스크랩+복사수), `LATEST` (최신순)
    - `difficulty`: `BEGINNER`, `INTERMEDIATE`, `ADVANCED`
- **Response Body**
    
    ```json
    {
      "programs": [
        {
          "id": 1,
          "title": "초보자 전신 운동",
          "creatorName": "피트니스 앱",   // creator_id가 NULL일 경우 시스템명 표시
          "isOfficial": true,            // 공식 인증 마크용
          "difficulty": "BEGINNER",
          "copiedCount": 1250,           // 인기 지표
          "estimatedMinutes": 25,
          "estimatedCalories": 200,
          "tags": ["전신", "기초체력"]
        },
        {
          "id": 200,
          "title": "복근 2주 챌린지",
          "creatorName": "헬스왕",
          "isOfficial": false,
          "difficulty": "INTERMEDIATE",
          "copiedCount": 54,
          "estimatedMinutes": 15,
          "estimatedCalories": 120,
          "tags": ["복근", "코어"]
        }
      ]
    }
    ```
    

### 2.2.3 새 루틴 생성

- **Endpoint:** `POST /programs`
- **설명:** '루틴 만들기' 화면에서 사용자가 직접 구성한 루틴을 저장합니다.
- **Request Body**
    
    ```json
    {
      "title": "나만의 하체 루틴",
      "description": "불타는 허벅지를 위한 루틴",
      "difficulty": "INTERMEDIATE",
      "isPublic": true,                 // 다른 사람에게 공개 여부
      "exercises": [
        {
          "exerciseId": 50,             // 스쿼트 ID
          "orderIndex": 1,
          "targetSets": 4,
          "targetReps": 12,
          "restSeconds": 90
        },
        {
          "exerciseId": 52,             // 런지 ID
          "orderIndex": 2,
          "targetSets": 3,
          "targetReps": 15,
          "restSeconds": 60
        }
      ]
    }
    ```
    
- **Response Body**
    
    ```json
    {
      "programId": 301,
      "message": "루틴이 생성되었습니다."
    }
    ```
    

### 2.2.4 루틴 상세 조회

- **Endpoint:** `GET /programs/{programId}`
- **설명:** 루틴 클릭 시 상세 운동 목록(세트, 횟수 등)을 반환합니다. 내 루틴이라면 수정 화면 데이터로도 사용됩니다.
- **Response Body**
    
    ```json
    {
      "id": 301,
      "title": "나만의 하체 루틴",
      "description": "불타는 허벅지를 위한 루틴",
      "difficulty": "INTERMEDIATE",
      "isPublic": true,
      "creatorId": 101,                  // 작성자 ID (본인 여부 확인용)
      "isMine": true,                    // 클라이언트 편의를 위한 플래그
      "exercises": [
        {
          "exerciseId": 50,
          "name": "스쿼트",
          "imageUrl": "https://...",
          "targetSets": 4,
          "targetReps": 12,
          "restSeconds": 90,
          "orderIndex": 1
        },
        // ...
      ]
    }
    ```
    

### 2.2.5 루틴 수정

- **Endpoint:** `PUT /programs/{programId}`
- **설명:** 내가 만든 루틴의 제목, 설명, 공개 여부, **포함된 운동 목록 전체**를 수정합니다.
    - **중요 로직:** 기존에 연결된 운동(`PROGRAM_EXERCISES`)을 모두 제거하고, 요청받은 운동 목록으로 새로 덮어씁니다(Replace).
    - 수정 후 `estimatedMinutes`, `estimatedCalories`, `mainTargets`(태그)가 서버에서 재계산되어 저장됩니다.
- **Request Body**
    
    ```json
    {
      "title": "수정된 가슴 루틴",
      "description": "더 강력해진 루틴입니다.",
      "difficulty": "ADVANCED",
      "isPublic": true,
      "exercises": [ // 수정된 전체 운동 리스트 전송
        {
          "exerciseId": 50,
          "orderIndex": 1,
          "targetSets": 5, // 4세트 -> 5세트 변경
          "targetReps": 10,
          "restSeconds": 90
        },
        {
          "exerciseId": 55, // 새로 추가된 운동
          "orderIndex": 2,
          "targetSets": 3,
          "targetReps": 12,
          "restSeconds": 60
        }
      ]
    }
    ```
    
- **Response Body**
    
    ```json
    {
      "programId": 101,
      "updatedAt": "2026-02-16T14:30:00",
      "message": "루틴이 수정되었습니다."
    }
    ```
    

### 2.2.6 루틴 삭제

- **Endpoint:** `DELETE /programs/{programId}`
- **설명:** 내가 만든 루틴을 영구 삭제합니다.
    - **주의:** 루틴을 삭제하면 해당 루틴에 연결된 운동 목록(`PROGRAM_EXERCISES`)도 함께 삭제됩니다. (Cascade Delete)
- **Response Body**
    
    ```json
    {
      "programId": 101,
      "message": "루틴이 삭제되었습니다."
    }
    ```
    

### 2.2.7 다른 루틴을 내 루틴으로 복제

- **Endpoint:** `POST /programs/{programId}/copy`
- **설명:** 다른 사람의 루틴이나 시스템 루틴을 내 루틴으로 복제(Deep Copy)합니다.
    - `PROGRAMS` 테이블에 새로운 row가 생성되며 `creator_id`는 내 ID가 됩니다.
    - 복제 이후 수정(무게/횟수 변경)이 가능해집니다.
    - 원본 루틴의 `copied_count`가 1 증가합니다.
- **Response Body**
    
    ```json
    {
      "originalProgramId": 1,
      "newProgramId": 302,              // 새로 생성된 내 루틴 ID
      "message": "내 루틴에 추가되었습니다."
    }
    ```
    

### 2.2.8 운동 카테고리 목록 조회

- **Endpoint:** `GET /categories`
- **Response Body**
    
    ```json
    [
      { "id": 1, "orderIndex": 1, "name": "가슴" },
      { "id": 2, "orderIndex": 3, "name": "등" },
      { "id": 3, "orderIndex": 4, "name": "하체" },
      { "id": 4, "orderIndex": 2, "name": "어깨" },
      { "id": 5, "orderIndex": 6, "name": "팔" },
      { "id": 6, "orderIndex": 5, "name": "유산소/복근" }
    ]
    ```
    

### 2.2.9 운동 종목 목록 조회 (필터/검색)

- **Endpoint:** `GET /exercises`
- **Query Param:**
    - `categoryId`: 특정 부위 필터링 (Optional)
    - `keyword`: 검색어 (Optional)
- **Response Body**
    
    ```json
    [
      {
        "id": 10,
        "name": "벤치프레스",
        "categoryId": 1,
        "categoryName": "가슴",
        "imageUrl": "https://...",
        "description": "가슴 운동의 대표적인..." // 목록에서는 짧게(말줄임) 또는 제외
      }
    ]
    ```
    

### 2.2.10 운동 종목 상세 정보

- **Endpoint:** `GET /exercises/{exerciseId}`
- **설명:** 운동 가이드 영상이나 자세한 설명을 보여줍니다.
- **Response Body**
    
    ```json
    {
      "id": 10,
      "name": "벤치프레스",
      "categoryName": "가슴",
      "imageUrl": "https://...",
      "description": "벤치에 누워 바벨을 가슴 위로 들어 올리는 동작입니다...",
    }
    ```
    

---

## 2.3 운동 기록 (Workout Sessions)

> 관련 Flow: 운동 시작하기 -> 운동 수행 -> 완료
> 

| **Method** | **URI** | **설명** | **Auth** |
| --- | --- | --- | --- |
| `POST` | `/workouts/sessions` | 운동 완료 및 기록 저장 | O |
| `GET` | `/workouts/sessions/calendar` | 캘린더 운동 유무 표시 | O |
| `GET` | `/workouts/sessions` | 선택된 날짜의 운동 목록 | O |
| `GET` | `/workouts/sessions/{sessionId}` | 특정 운동 기록 상세 조회 | O |
| `DELETE` | `/workouts/sessions/{sessionId}` | 운동 기록 삭제 | O |

### 2.3.1 운동 기록 저장

- **Endpoint:** `POST /workouts/sessions`
- **설명:** 운동 완료 후 로컬에 저장된 세트 정보를 서버로 전송합니다.
- **Request Body**
    
    ```json
    {
      "programId": 5, // (Optional) 루틴을 수행한 경우. 자율운동이면 null
      "status": "COMPLETED",
      "scheduledAt": "2024-05-21T18:00:00",
      "startTime": "2024-05-21T18:00:00", // (Optional)
      "endTime": "2024-05-21T19:30:00", // (Optional)
      "totalVolume": 8500, // 프론트에서 계산해서 전달 (검증 용도)
      "totalTimeSeconds": 5400,
      "moodMemo": "컨디션 좋았음",
      "exercises": [ // 수행한 운동 목록
        {
          "exerciseId": 50, // 스쿼트
          "sets": [
            {
              "setNumber": 1,
              "weight": 60,
              "reps": 12,
              "rpe": 7,
              "restSeconds": 60,
              "isWarmup": true,
              "isCompleted": true
            },
            {
              "setNumber": 2,
              "weight": 100,
              "reps": 5,
              "rpe": 9,
              "restSeconds": 120,
              "isWarmup": false,
              "isCompleted": true
            }
          ]
        }
      ]
    }
    ```
    
- **Response Body**
    
    ```json
    {
      "sessionId": 12345, // 생성된 세션 ID (공유하기 등에 사용)
      "message": "운동이 저장되었습니다."
    }
    ```
    

### 2.3.2 캘린더 운동 유무 표시

- **Endpoint: `GET** /workouts/sessions/calendar`
- **설명:** 특정 기간을 입력값으로 받고 그 기간의 운동 기록 유무를 전달합니다.
- **Query Param:** `startDate`, `endDate`
- **Response Body**
    
    ```json
    {
      "startDate": "2026-02-08",
      "endDate": "2026-02-14",
      "dates": [
        { "date": "2026-02-08", "hasWorkout": true },
        { "date": "2026-02-10", "hasWorkout": true },
        { "date": "2026-02-12", "hasWorkout": true }
      ]
    }
    ```
    

### 2.3.3 선택된 날짜의 운동 목록

- **Endpoint: `GET** /workouts/sessions`
- **설명:** 특정 날짜의 운동 목록을 간략화해서 받아옵니다.
- **Query Param:** `date` (2026-02-12)
- **Response Body**
    
    ```json
    [
      {
        "sessionId": 12345,
        "title": "아침 홈트레이닝",
        "status": "COMPLETED", 
        "totalTimeSeconds": 1200,
        "burnedCalories": 180,
        "exerciseCount": 3,
        "workoutTags": ["푸시업", "스쿼트", "플랭크"] 
      }
    ]
    ```
    

### 2.3.4 운동 기록 상세 조회

- **Endpoint:** `GET /workouts/sessions/{sessionId}`
- **설명:** 캘린더나 피드에서 특정 운동 기록을 클릭했을 때 상세 내용을 보여줍니다.
- **Response Body**
    
    ```json
    [
      {
        "sessionId": 12345,
        "status": "COMPLETED",
    	  "scheduledAt": "2024-05-21T18:00:00",
    	  "startTime": "2024-05-21T18:00:00", // (Optional)
    	  "endTime": "2024-05-21T19:30:00", // (Optional)
    	  "totalVolume": 8500, // 프론트에서 계산해서 전달 (검증 용도)
    	  "totalTimeSeconds": 5400,
    	  "moodMemo": "컨디션 좋았음",
    	  "exercises": [ // 수행한 운동 목록
    	    {
    	      "exerciseId": 50, // 스쿼트
    	      "sets": [
    	        {
    	          "setNumber": 1,
    	          "weight": 60,
    	          "reps": 12,
    	          "rpe": 7,
    	          "restSeconds": 60,
    	          "isWarmup": true,
    	          "isCompleted": true
    	        },
    	        {
    	          "setNumber": 2,
    	          "weight": 100,
    	          "reps": 5,
    	          "rpe": 9,
    	          "restSeconds": 120,
    	          "isWarmup": false,
    	          "isCompleted": true
    	        }
    	      ]
    	    }
    	  ]
      }
    ]
    ```
    

### 2.3.5 운동 기록 삭제

- **Endpoint:** `DELETE /workouts/sessions/{sessionId}`
- **설명:** 잘못 저장된 기록을 삭제합니다. (Soft Delete 권장)

---

## 2.4 분석 (Analysis)

> 관련 Flow: 분석 페이지 -> 기간 선택 -> 캘린더 이동
> 

| **Method** | **URI** | **설명** | **Auth** |
| --- | --- | --- | --- |
| `GET` | `/analysis/dashboard` | 분석 탭 대시보드 통합 조회 | O |
| `GET` | `/analysis/weekly-stats` | 주간 통계 상세 조회 (스와이프용) | O |

### 2.4.1 분석 탭 대시보드 통합 조회

- **Endpoint:** `GET /analysis/dashboard`
- **설명:**
    - 분석 탭에 처음 들어왔을 때 호출합니다.
    - 상단 카드(총 횟수, 칼로리), 레이더 차트, 그리고 오늘이 포함된 이번 주의 그래프 데이터를 한 번에 내려줍니다.
- **Query Parameters:**
    - `today`: 기준 날짜 (Optional, Default: 서버 기준 오늘).
- **Response Body**
    
    ```json
    {
      "summary": {
        "totalWorkoutCount": 156,     // [UI 상단] 총 운동 횟수 (누적)
        "weeklyTotalCalories": 2450   // [UI 상단] 이번 주 소모 칼로리 합계
      },
      "balance": {                    // [UI 중앙] 운동 균형 (레이더 차트)
        "score": 85,                  // 균형 점수
        "radarChart": [
          { "label": "상체", "value": 80 },
          { "label": "하체", "value": 60 },
          { "label": "코어", "value": 40 },
          { "label": "유산소", "value": 90 },
          { "label": "스트레칭", "value": 30 },
          { "label": "전신", "value": 70 }
        ]
      },
      "weeklyStats": {                // [UI 하단] 이번 주 그래프 데이터 (초기값)
        "startDate": "2026-02-09",
        "endDate": "2026-02-15",
        "totalCalories": 2450,
        "dailyData": [
          { "dayOfWeek": "MON", "date": "2026-02-09", "calories": 300 },
          { "dayOfWeek": "TUE", "date": "2026-02-10", "calories": 280 },
          { "dayOfWeek": "WED", "date": "2026-02-11", "calories": 450 },
          { "dayOfWeek": "THU", "date": "2026-02-12", "calories": 380 },
          { "dayOfWeek": "FRI", "date": "2026-02-13", "calories": 420 },
          { "dayOfWeek": "SAT", "date": "2026-02-14", "calories": 350 },
          { "dayOfWeek": "SUN", "date": "2026-02-15", "calories": 270 }
        ]
      }
    }
    ```
    

### 2.4.2 주간 통계 상세 조회 (스와이프용)

- **Endpoint:** `GET /analysis/weekly-stats`
- **설명:**
    - **그래프 영역을 스와이프**하여 지난주나 다음 주로 이동할 때 호출합니다.
    - 요청한 `startDate`와 `endDate` 사이의 일별 데이터를 반환합니다.
- **Query Parameters:**
    - `startDate`: 조회 시작일 (예: "2026-02-01")
    - `endDate`: 조회 종료일 (예: "2026-02-07")
- **Response Body**
    
    ```json
    {
      "startDate": "2026-02-01",
      "endDate": "2026-02-07",
      "totalCalories": 2100, // 해당 기간의 총합 (UI 갱신용)
      "dailyData": [
        {
          "dayOfWeek": "MON",
          "date": "2026-02-01",
          "calories": 200,
          "isWorkoutDay": true // (Optional) 그래프 색상 강조 등을 위함
        },
        {
          "dayOfWeek": "TUE",
          "date": "2026-02-02",
          "calories": 0,       // 운동 안 한 날은 0으로 채워서 리턴
          "isWorkoutDay": false
        },
        // ... (수, 목, 금, 토)
        {
          "dayOfWeek": "SUN",
          "date": "2026-02-07",
          "calories": 400,
          "isWorkoutDay": true
        }
      ]
    }
    ```
    

---

## 2.5 커뮤니티 (Community)

> 관련 Flow: 커뮤니티 페이지 -> 탭 전환(전체/친구) -> 글쓰기
> 

| **Method** | **URI** | **설명** | **Auth** |
| --- | --- | --- | --- |
| `GET` | `/posts` | 게시글 목록 조회 (피드) | O |
| `POST` | `/posts` | 게시글 작성 (공유) | O |
| `GET` | `/posts/{postId}` | 게시글 상세 조회 | O |
| `GET` | `/workouts/sessions/attachable` | 작성 가능한 운동 기록 조회 | O |
| `POST` | `/posts/{postId}/likes` | 좋아요 토글 | O |
| `POST` | `/posts/{postId}/comments` | 댓글 작성 | O |
| `PATCH` | `/posts/{postId}` | 게시글 수정 | O |
| `DELETE` | `/posts/{postId}` | 게시글 삭제 | O |
| `DELETE` | `/comments/{commentId}` | 댓글 삭제 | O |
| `PATCH` | `/comments/{commentId}` | 댓글 수정 | O |
| `POST` | `/reports` | 유저/게시글/댓글 신고 | O |
| `POST` | `/users/{targetUserId}/block` | 유저 차단 | O |

### 2.5.1 게시글 목록 조회 (피드)

- **Endpoint:** `GET /posts`
- **설명:** 커뮤니티 탭 진입 시 게시글 목록을 불러옵니다. 운동 기록이 연동된 경우 요약 정보를 포함하여 프론트엔드에서 태그(#상체 #푸시업)를 표시할 수 있게 합니다. 차단한 유저의 게시글은 조회되지 않도록 백엔드에서 처리합니다.
- **Query Param:** `type` (ALL, FRIENDS), `page`, `size`
- **Response Body**
    
    ```json
    [
    	{
        "postId": 500,
        "user": {
          "userId": 102,
          "nickname": "김운동",
          "profileImage": "https://cdn.fitnessapp.com/profiles/u102.jpg"
        },
        "content": "오늘도 열심히 운동 완료! 💪",
        "imageUrl": "https://cdn.fitnessapp.com/posts/img_500.jpg",
        "createdAt": "2026-02-11T20:12:50",
        "timeAgo": "2시간 전",
        "likeCount": 42,
        "commentCount": 8,
        "isLiked": true,
        
        // 운동 기록 연동 시 포함되는 정보 (Null 가능)
        "workoutSummary": {
          "sessionId": 12345,
          "title": "상체 가슴 루틴",
          "totalTimeSeconds": 5400,
          "totalVolume": 8500,
          "burnedCalories": 450,
          "mainBodyParts": ["상체", "가슴"] // 태그 표시용
        }
      }
    ]
    ```
    

### 2.5.2 게시글 작성 (공유)

- **Endpoint:** `POST /posts`
- **설명:** 글쓰기 화면에서 작성한 내용과 (선택적으로) 운동 기록 ID를 받아 게시글을 생성합니다.
- **Request Body**
    
    ```json
    {
      "content": "운동 기록을 공유해보세요...", // 필수 (최대 2000자)
      "imageUrl": "https://cdn.fitnessapp.com/uploads/...", // 선택 (업로드 API로 받은 URL)
      "sessionId": 12345 // 선택 (운동 기록 공유 시 linkedSessionId)
    }
    ```
    
- **Response Body**
    
    ```json
    {
      "postId": 500,
      "user": {
        "userId": 102,
        "nickname": "김운동",
        "profileImage": "https://cdn.fitnessapp.com/profiles/u102.jpg"
      },
      "content": "오늘도 열심히 운동 완료! 💪",
      "imageUrl": "https://cdn.fitnessapp.com/posts/img_500.jpg",
      "createdAt": "2026-02-11T20:12:50",
      "timeAgo": "2시간 전",
      "likeCount": 0,
      "commentCount": 0,
      "isLiked": false,
      
      // 운동 기록 연동 시 포함되는 정보 (Null 가능)
      "workoutSummary": {
        "sessionId": 12345,
        "title": "상체 가슴 루틴",
        "totalTimeSeconds": 5400,
        "totalVolume": 8500,
        "burnedCalories": 450,
        "mainBodyParts": ["상체", "가슴"] // 태그 표시용
      }
    }
    ```
    

### 2.5.3 게시글 상세 조회

- **Endpoint:** `GET /posts/{postId}`
- **설명:** 피드에서 특정 글을 클릭했을 때, 상세 내용과 댓글 목록을 함께 조회합니다.
- **Response Body**
    
    ```json
    {
      "post": {
        "postId": 500,
        "user": {
          "userId": 102,
          "nickname": "김운동",
          "profileImage": "https://cdn.fitnessapp.com/profiles/u102.jpg"
        },
        "content": "오늘도 열심히 운동 완료! 💪",
        "imageUrl": "https://cdn.fitnessapp.com/posts/img_500.jpg",
        "createdAt": "2026-02-11T20:12:50",
        "likeCount": 42,
        "commentCount": 8,
        "isLiked": true,
        "workoutSummary": {
          "sessionId": 12345,
          "title": "상체 가슴 루틴",
          "totalTimeSeconds": 5400,
          "totalVolume": 8500,
          "mainBodyParts": ["상체", "가슴"]
        }
      },
      "comments": [
        {
          "commentId": 10,
          "user": {
            "userId": 200,
            "nickname": "헬린이",
            "profileImage": "https://cdn.fitnessapp.com/profiles/u200.jpg"
          },
          "content": "오운완! 멋지십니다.",
          "createdAt": "2026-02-11T20:20:00",
          "isMyComment": false
        },
        {
          "commentId": 11,
          "user": {
            "userId": 102,
            "nickname": "김운동",
            "profileImage": "https://cdn.fitnessapp.com/profiles/u102.jpg"
          },
          "content": "감사합니다!",
          "createdAt": "2026-02-11T20:22:00",
          "isMyComment": true
        }
      ]
    }
    ```
    

### 2.5.4 작성 가능한 운동 기록 조회 (Attachable Sessions)

- **Endpoint:** `GET /workouts/sessions/attachable`
- **설명:** 글쓰기 화면에서 '운동 기록 첨부'를 눌렀을 때, 최근 내 운동 중 아직 게시글로 작성되지 않은 목록을 불러옵니다.
- **Response Body**
    
    ```json
    [
      {
        "sessionId": 12345,
        "title": "하체 운동",
        "startTime": "2026-02-11T19:00:00",
        "endTime": "2026-02-11T20:30:00",
        "volumeSummary": "8,500kg",
        "category": "하체"
      },
      {
        "sessionId": 12340,
        "title": "아침 유산소",
        "startTime": "2026-02-10T07:00:00",
        "endTime": "2026-02-10T07:30:00",
        "volumeSummary": "3km",
        "category": "유산소"
      }
    ]
    ```
    

### 2.5.5 좋아요 토글

- **Endpoint:** `POST /posts/{postId}/likes`
- **설명:** 게시글에 좋아요를 누르거나 취소합니다. (Toggle 방식)
- **Response Body**
    
    ```json
    {
      "isLiked": true,
      "currentLikeCount": 43
    }
    ```
    

### 2.5.6 댓글 작성

- **Endpoint:** `POST /posts/{postId}/comments`
- **설명:** 게시글에 새로운 댓글을 등록합니다.
- **Request Body**
    
    ```json
    {
      "content": "루틴 공유 좀 부탁드려요!" // 1자 이상 500자 이내
    }
    ```
    
- **Response Body**
    
    ```json
    {
      "commentId": 12,
      "content": "루틴 공유 좀 부탁드려요!",
      "createdAt": "2026-02-11T20:30:00",
      "user": {
        "userId": 300,
        "nickname": "질문1",
        "profileImage": "https://cdn.fitnessapp.com/profiles/u300.jpg"
      }
    }
    ```
    

### 2.5.7 게시글 수정

- **Endpoint:** `PATCH /posts/{postId}`
- **설명:** 본인이 작성한 게시글의 내용을 수정합니다. (운동 기록 ID는 수정 불가)
- **Request Body**
    
    ```json
    {
      "content": "수정된 내용입니다...", 
      "imageUrl": "https://..." 
    }
    ```
    
- **Response Body**
    
    ```json
    {
      "message": "게시글이 수정되었습니다."
    }
    ```
    

### 2.5.8 게시글 삭제

- **Endpoint:** `DELETE /posts/{postId}`
- **설명:** 본인이 작성한 게시글을 삭제합니다.
- **Response Body:**
    
    ```json
    {
      "message": "게시글이 삭제되었습니다."
    }
    ```
    

### 2.5.9 댓글 삭제

- **Endpoint:** `DELETE /comments/{commentId}`
- **설명:** 본인이 작성한 댓글을 삭제합니다.
- **Response Body:**
    
    ```json
    {
      "message": "댓글이 삭제되었습니다."
    }
    ```
    

### 2.5.10 댓글 수정

- **Endpoint:** `PATCH /comments/{commentId}`
- **설명:** 본인이 작성한 댓글의 내용을 수정합니다.
- **Request Body**
    
    ```json
    {
      "content": "수정된 댓글 내용입니다." // 필수
    }
    ```
    
- **Response Body**
    
    ```json
    {
      "commentId": 12,
      "content": "수정된 댓글 내용입니다.",
      "updatedAt": "2026-02-12T10:00:00" // 수정됨 표시
    }
    ```
    

### 2.5.11 유저/게시글/댓글 신고

- **Endpoint:** `POST /reports`
- **설명:** 사용자로부터 신고 유형, 대상 ID, 사유 등을 받아 새로운 신고 객체를 생성합니다. 인증된 신고자 정보(`reporter`)는 서버 내부에서 `memberId`를 통해 조회하여 설정됩니다.
- **Request Body**
    
    ```jsx
    {
      "targetType": "POST", // 필수. 신고 대상 유형 (POST, USER, COMMENT)
      "targetId": 12345,    // 필수. 신고 대상의 ID (Long)
      "reason": "SPAM",     // 필수. 신고 사유 (SPAM, ABUSE, PORNOGRAPHY, HATE_SPEECH, OTHER)
      "details": "부적절한 광고성 게시글입니다." // 선택. 상세 내용
    }
    ```
    
- **Response Body**

### 2.5.12 유저 차단

- **Endpoint:** `POST /users/{targetUserId}/block`
- **설명:** 특정 유저를 차단합니다.
- **Request Body**
- **Response Body**

---

## 2.6 프로필 및 설정 (Profile & Settings)

> 관련 Flow: 프로필 -> 친구 목록 화면
> 

| **Method** | **URI** | **설명** | **Auth** |
| --- | --- | --- | --- |
| `GET` | `/members/me` | 내 계정 기본 정보 및 요약 통계 조회 | O |
| `GET` | `/members/me/details` | 프로필 수정 화면 조회 | O |
| `PUT` | `/members/me/details` | 프로필 정보 통합 수정 | O |
| `GET` | `/members/me/settings` | 유저 앱 설정 조회 | O |
| `PATCH` | `/members/me/settings` | 유저 앱 설정 변경 | O |
| `GET` | `/friendships` | 친구 목록/요청 목록 조회 | O |
| `POST` | `/friendships` | 친구 요청 보내기 | O |
| `PATCH` | `/friendships/{friendshipId}` | 친구 요청 수락/거절/차단 | O |
| `DELETE` | `/friendships/{friendshipId}` | 차단 해제 | O |
| `GET` | `/members/me/achievements` | 목표 달성도 및 업적 조회 | O |

### 2.6.1 내 계정 기본 정보 및 요약 통계 조회

- Endpoint: `GET /members/me`
- 설명: 프로필 메인 화면에 필요한 사용자 기본 정보와 누적 통계 데이터를 반환합니다.
- **Response Body**
    
    ```json
    {
      "id": 101,
      "email": "gildong@test.com",
      "nickname": "홍길동",
      "profileImage": "https://s3.aws.com/profile/default.png",
      "bodySummary": { // 프로필 카드 하단 표시용 (계산된 값)
        "height": 175.0,
        "weight": 68.0,
        "bmi": 22.2 // 서버에서 계산해서 전달 (Weight / Height^2)
      },
      "stats": { // 프로필 메인 중간 통계 영역
        "totalWorkoutHours": 78,    // "78 시간"
        "totalBurnedCalories": 15200, // "15.2K kcal"
        "achievedGoalCount": 24     // "24 개"
      }
    }
    ```
    

### 2.6.2 프로필 수정 화면 조회

- Endpoint: `GET /members/me/details`
- 설명: 프로필 수정 화면 진입 시, 폼(Form)에 채워 넣을 ****모든 정보를 조회합니다.
- **Response Body**
    
    ```json
    {
      "nickname": "홍길동",            // [Members] 테이블 데이터
      "profileImage": "https://...",   // [Members] 테이블 데이터
      "gender": "MALE",                // [Profiles] 테이블 데이터 (MALE, FEMALE)
      "birthDate": "1990-01-01",       // [Profiles]
      "height": 175.0,                 // [Profiles]
      "weight": 68.0,                  // [Profiles]
      "targetWeight": 65.0,            // [Profiles]
      "activityLevel": "HIGH",         // [Profiles] (슬라이더 값 매핑)
      "experienceLevel": "INTERMEDIATE"// [Profiles]
    }
    ```
    

### 2.6.3 프로필 정보 통합 수정

- Endpoint: `PUT /members/me/details`
- 설명: 프로필 수정 화면에서 ‘저장’ 버튼을 눌렀을 때 호출됩니다. 계정 정보와 신체 정보를 일괄 수정합니다.
- **Request Body**
    
    ```json
    {
      "nickname": "홍길동",            // (Optional) 수정 안 했으면 기존 값
      "profileImage": "https://...",   // (Optional) 변경된 이미지 URL
      "gender": "MALE",
      "birthDate": "1990-01-01",
      "height": 175.0,
      "weight": 68.0,
      "targetWeight": 65.0,
      "activityLevel": "HIGH",
      "experienceLevel": "INTERMEDIATE"
    }
    ```
    

### 2.1.4 유저 앱 설정 조회

- Endpoint: `GET /members/me/settings`
- 설명: 앱 설정 화면 진입 시 현재 설정 상태를 반환합니다.
- **Response Body**
    
    ```json
    {
      "pushNotification": true,   // 푸시 알림
      "marketingAgree": false,    // 마케팅 정보 수신
      "weightUnit": "KG",         // "KG", "LBS"
      "distanceUnit": "KM",       // "KM", "MILE"
      "theme": "SYSTEM"           // "SYSTEM", "LIGHT", "DARK"
    }
    ```
    

### 2.1.5 유저 앱 설정 수정

- Endpoint: `PATCH /members/me/settings`
- 설명: 설정 화면에서 토글이나 옵션을 변경하고 '저장' 또는 뒤로가기 시 호출됩니다.
- **Request Body**
    
    ```json
    {
      "pushNotification": true,
      "marketingAgree": false,
      "weightUnit": "KG",
      "distanceUnit": "KM",
      "theme": "DARK"
    }
    ```
    
- **Response Data:** (수정된 설정 객체 반환)

### 2.6.6 친구 목록/요청 목록 조회

- Endpoint: `GET /friendships`
- **설명:** 나에게 온 요청, 내 친구 목록, 내가 보낸 요청, 차단한 목록을 확인합니다.
- **Query Param:** `status` (`PENDING`, `ACCEPTED`, `SENT`, `BLOCKED`)
- **Response Body**
    
    ```json
    {
      "totalCount": 5,
      "friends": [
        {
          "friendshipId": 77,
          "friendUser": {
            "id": 200,
            "nickname": "친구A",
            "profileImage": "..."
          },
          "workoutCount": 45, // "운동 45회" 표시용
          "status": "ONLINE" // (옵션) 초록색 점 표시용 (ONLINE, OFFLINE)
        },
        {
          "friendshipId": 78,
          "friendUser": {
            "id": 201,
            "nickname": "친구B",
            "profileImage": "..."
          },
          "workoutCount": 32,
          "status": "OFFLINE" // 회색 점
        }
      ]
    }
    ```
    

### 2.6.7 친구 요청 보내기

- **Endpoint:** `POST /friendships`
- **설명:** 친구 요청을 보냅니다.
- **Request Body**
    
    ```json
    {
      "targetUserEmail": "target@email.com" // 이메일 검색 또는 targetUserId
    }
    ```
    
- **Response Body**
    
    ```json
    {
      "friendshipId": 77,
      "status": "ACCEPTED", // 변경된 최종 상태 ("ACCEPTED", "REJECTED", "BLOCKED")
      "message": "친구 요청을 수락했습니다." // 클라이언트 토스트 메시지용
    }
    ```
    

### 2.6.8 친구 상태 변경 (수락/거절/차단)

- **Endpoint:** `PATCH /friendships/{friendshipId}`
- **설명:** 나에게 온 친구 요청을 처리하거나, 기존 친구 관계를 차단합니다.
- **Request Body**
    
    ```json
    {
      "action": "ACCEPT" 
      // "ACCEPT": 친구 요청 수락
      // "REJECT": 친구 요청 거절 (요청 목록에서 삭제됨)
      // "BLOCK": 차단 (상대방이 나를 검색하거나 요청 불가)
    }
    ```
    

### 2.6.9 차단 해제

- **Endpoint**: `DELETE /friendships/{friendshipId}`
- **설명:** 특정 사용자에 대한 차단을 해제합니다. (관계 데이터를 삭제하여, 다시 친구 요청을 보낼 수 있는 상태인 '남'으로 돌아갑니다.)
- **Response Body**
    
    ```json
    {
      "friendshipId": 99,
      "message": "차단이 해제되었습니다."
    }
    ```
    

### 2.6.10 목표 달성도 및 업적 조회

- Endpoint: `GET /members/me/achievements`
- **설명:** 내 업적 화면의 목표 진행률과 배지 목록을 조회합니다.
- **Response Body**
    
    ```json
    {
      "goals": {
        "weeklyWorkout": { // 주 5회 운동
          "current": 4,
          "target": 5,
          "unit": "회"
        },
        "weightLoss": { // 체중 감량
          "current": 68.0,
          "target": 65.0,
          "unit": "kg",
          "startWeight": 70.0 // (옵션) 진행률 바 계산용
        },
        "monthlyCalories": { // 월 2000kcal 소모
          "current": 1250,
          "target": 2000,
          "unit": "kcal"
        }
      },
      "badges": [
        {
          "id": 1,
          "name": "7일 연속 운동",
          "description": "일주일 동안 매일 운동하기",
          "iconUrl": "...",
          "isAcquired": false,
          "progress": 4, // 4/7
          "maxProgress": 7
        },
        {
          "id": 2,
          "name": "하루 300kcal 소모",
          "description": "하루에 300kcal 이상 소모",
          "iconUrl": "...",
          "isAcquired": true, // 완료 표시 (체크)
          "acquiredAt": "2024-02-01"
        }
        // ... 기타 배지
      ]
    }
    ```
    

---

## 2.7 이미지 (Images)

> 관련 Flow: 프로필 수정 -> 사진 변경, 커뮤니티 -> 게시글 작성 (사진 첨부)
> 

| **Method** | **URI** | **설명** | **Auth** |
| --- | --- | --- | --- |
| `POST` | `/images/upload` | 단건 이미지 업로드 | O |
| `POST` | `/images/upload/multiple` | 다건 이미지 업로드 (최대 용량 주의) | O |
| `DELETE` | `/images` | 업로드된 이미지 삭제 | O |

### 2.7.1 단건 이미지 업로드

- **Endpoint:** `POST /images/upload`
- **설명:** 단일 파일을 S3에 업로드하고 접근 가능한 공개 URL을 반환받습니다. 프로필 이미지 업데이트 등에 사용됩니다.
- **Request (multipart/form-data)**
    - `file`: 업로드할 이미지 파일 (필수)
    - `dir`: S3 내 저장될 폴더명 (`profiles`(프로필 이미지)와 `posts`(게시글 이미지) 중 선택, 기본값: `profiles`)
- **Response Body**
    
    ```json
    {
      "imageUrl": "https://myapp-bucket.s3.ap-northeast-2.amazonaws.com/profiles/uuid-1234.jpg"
    }
    
    ```
    

### 2.7.2 다건 이미지 업로드

- **Endpoint:** `POST /images/upload/multiple`
- **설명:** 여러 장의 이미지를 동시에 업로드합니다. 커뮤니티 게시글 사진 첨부 등에 사용됩니다.
- **Request (multipart/form-data)**
    - `files`: 업로드할 이미지 파일 배열 (필수)
    - `dir`: S3 내 저장될 폴더명 (선택, 기본값: `posts`)
- **Response Body**
    
    ```json
    {
      "imageUrls": [
        "https://myapp-bucket.s3.ap-northeast-2.amazonaws.com/posts/uuid-5678.jpg",
        "https://myapp-bucket.s3.ap-northeast-2.amazonaws.com/posts/uuid-9012.png"
      ]
    }
    ```
    

### 2.7.3 이미지 삭제

- **Endpoint:** `DELETE /images`
- **설명:** 업로드된 이미지 URL을 기반으로 S3 객체를 삭제합니다. 게시글 작성 취소나 사진 변경 시 기존 파일을 정리하는 데 사용됩니다.
- **Query Parameters:**
    - `imageUrl`: 삭제할 이미지의 전체 S3 URL (필수)
- **Response Body**
    
    ```json
    {
      "success": true,
      "code": "SUCCESS",
      "message": "이미지가 성공적으로 삭제되었습니다.",
      "data": null
    }
    ```
    

---
