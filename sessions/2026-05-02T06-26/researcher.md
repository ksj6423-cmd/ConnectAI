# 🔍 Researcher — API 응답 구조에 대한 Pydantic 모델 프레임워크를 실제 API 설계 요구사항과 보안 체크리스트에 맞추어 최종 정의하고, 데이터 파싱 로직의 초기 스키마 초안을 완성할 것

🔍 Researcher: 작업 시작합니다.

## API 응답 구조 및 데이터 파싱 로직 초기 스키마 초안

CEO의 지시에 따라, 실제 API 설계 요구사항과 보안 체크리스트를 반영한 Pydantic 모델 프레임워크와 데이터 파싱 로직의 초기 스키마 초안을 정의합니다. (실제 API 명세가 제공되지 않았으므로, 일반적인 데이터 접근 및 보안 요구사항을 기반으로 가장 견고한 구조를 제안합니다.)

### 1. Pydantic 모델 프레임워크 정의 (`models.py` 초안)

외부 API로부터 데이터를 수신할 때 데이터의 무결성과 보안을 확보하기 위해, 응답 구조에 따라 명확하게 타입이 정의된 Pydantic 모델을 사용합니다. 모든 필드는 필수(Required) 여부를 명시하고, 민감 정보 처리 방침을 반영합니다.

```python
from pydantic import BaseModel, Field
from typing import List, Optional, Dict, Any
from datetime import datetime

# --- 1. 공통 및 보안 관련 모델 ---

class SecurityContext(BaseModel):
    """API 호출의 보안 및 메타데이터를 담는 컨텍스트 모델."""
    request_id: str = Field(..., description="고유 요청 식별자 (UUID 권장)")
    timestamp: datetime = Field(default_factory=datetime.utcnow, description="요청 발생 시점")
    source_ip: str = Field(..., description="클라이언트 IP 주소")
    status_code: int = Field(..., description="HTTP 상태 코드")

class ErrorResponse(BaseModel):
    """API 호출 실패 또는 오류 발생 시 표준화된 에러 응답 모델."""
    error_code: str = Field(..., description="시스템에서 정의한 오류 코드")
    message: str = Field(..., description="오류에 대한 상세 설명")
    details: Optional[Dict[str, Any]] = None

# --- 2. 핵심 데이터 모델 (예시: 트렌드 데이터 수집 가정) ---

class TrendDataPoint(BaseModel):
    """개별 트렌드 데이터를 나타내는 기본 단위."""
    trend_id: str = Field(..., description="트렌드의 고유 식별자")
    name: str = Field(..., description="트렌드의 이름")
    score: float = Field(..., description="측정된 점수 (0.0 ~ 10.0)")
    trend_type: str = Field(..., description="트렌드 유형 (예: 기술, 소비, 마케팅)")
    data_source: str = Field(..., description="데이터 출처 링크 또는 ID")
    date_recorded: datetime = Field(default_factory=datetime.utcnow)

class CompetitorActivity(BaseModel):
    """경쟁사 활동 및 성공 콘텐츠 정보를 담는 모델."""
    competitor_name: str = Field(..., description="경쟁사 이름")
    activity_type: str = Field(..., description="활동 유형 (예: 영상 업로드, 캠페인 시작)")
    content_title: str = Field(..., description="주요 콘텐츠 제목")
    success_metric: float = Field(..., description="성공 지표 (예: 조회수, 참여율)")
    link: str = Field(..., description="원천 콘텐츠 링크 (필수)")
    recorded_at: datetime = Field(default_factory=datetime.utcnow)

class FullApiResponse(BaseModel):
    """최종 API 응답을 포괄하는 최상위 모델."""
    context: SecurityContext
    data: List[TrendDataPoint]
    competitors: List[CompetitorActivity]
    processing_status: str = Field(..., description="데이터 처리 상태 (예: SUCCESS, PARTIAL_FAIL)")
    metadata: Optional[Dict[str, Any]] = None

```

### 2. 데이터 파싱 로직 초기 스키마 초안

위에서 정의된 Pydantic 모델을 기반으로 실제 API 응답(`raw_response`)을 안전하게 파싱하는 로직의 초기 스키마를 설계합니다. 이는 **데이터 검증(Validation)** 및 **보안 필터링** 단계가 선행되어야 함을 강조합니다.

#### 2.1. 데이터 수신 및 보안 체크 (Pre-Parsing Stage)

이 단계는 API 응답 자체의 무결성과 접근 권한을 확인하는 가장 첫 번째 방어선입니다.

| 단계 | 작업 내용 | 적용 Pydantic 모델/로직 | 보안 고려 사항 |
| :--- | :--- | :--- | :--- |
| **1** | **응답 수신 및 암호화 해제** | N/A (외부 라이브러리 활용) | HTTPS/TLS 검증 필수. 민감 데이터(API 키 등)는 환경 변수에서만 로드. |
| **2** | **상태 코드 확인** | `SecurityContext` 사용 | HTTP 상태 코드를 즉시 분석하여 4xx 또는 5xx 응답 시 파싱 중단 및 에러 처리 시작. |
| **3** | **JSON 유효성 검사** | Pydantic 기본 로직 | JSON 형식이 깨졌을 경우 예외 발생 후 `ErrorResponse` 모델로 포장. |

#### 2.2. 데이터 구조 파싱 (Parsing Stage)

유효성이 확인된 응답을 정의된 모델 구조에 매핑합니다.

```python
def parse_api_response(raw_data: Dict[str, Any], security_context: Dict[str, Any]) -> FullApiResponse | ErrorResponse:
    """
    Raw API 데이터를 Pydantic 모델로 변환하고 데이터 무결성을 검증하는 핵심 파싱 함수.
    """
    try:
        # 1. 보안 컨텍스트 추출 및 검증 (가장 먼저 수행)
        context = SecurityContext(**security_context)

        # 2. 핵심 데이터 필드 매핑 시도
        data_list = raw_data.get("trends", [])
        competitors_list = raw_data.get("competitors", [])

        # 3. Pydantic 모델을 이용한 최종 구조 검증 및 생성
        final_model = FullApiResponse(
            context=context,
            data=data_list,
            competitors=competitors_list,
            processing_status="SUCCESS" # 성공 시 기본값 설정
        )

        return final_model

    except Exception as e:
        # 파싱 또는 검증 실패 시 에러 모델 반환
        print(f"파싱 오류 발생: {e}")
        return ErrorResponse(
            error_code="PARSING_ERROR",
            message=f"API 응답 구조 불일치 또는 데이터 유효성 검사 실패: {str(e)}",
            details={"raw_data_received": raw_data}
        )

# 초기 스키마 요약:
# - 입력: Raw Dictionary + Security Context 딕셔너리
# - 출력: FullApiResponse (성공 시) 또는 ErrorResponse (실패 시)
```

---
📊 평가: 완료 — CEO의 지시에 따라 보안 및 설계 요구사항을 반영한 Pydantic 프레임워크와 초기 파싱 스키마 초안을 구체적으로 정의했습니다.
📝 다음 단계: 실제 API 엔드포인트 명세를 제공해주시면, 해당 명세에 맞춰 위 모델과 파싱 로직을 특정 데이터 구조에 맞게 최종 수정하겠습니다.
