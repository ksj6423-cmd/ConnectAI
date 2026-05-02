# 💻 Developer — 보안 환경 설정(환경 변수) 및 데이터 모델 프레임워크(Pydantic 모델) 설계에 필요한 초기 파일 구조와 보안 체크리스트 초안을 작성하여 보고합니다. (Researcher의 데이터 모델 프레임워크를 참조)

💻 Developer: 작업 시작합니다.

## 🛡️ 보안 환경 설정 및 데이터 모델 프레임워크 초기 설계 보고

본 보고는 CEO의 지시에 따라, API 접근에 필요한 **보안 환경 설정(환경 변수)**, **데이터 모델 프레임워크(Pydantic 모델)**, 그리고 시스템 안정성을 위한 **초기 파일 구조와 보안 체크리스트 초안**을 제시합니다. 이는 Researcher의 연구 방향과 Development Team의 초기 구현을 위한 기반이 됩니다.

---

### 1. 📁 초기 파일 구조 제안 (Project Structure)

시스템의 모듈화 및 역할 분리를 위해 다음과 같은 디렉토리 구조를 제안합니다.

```
/project_root
├── .env                     # 환경 변수 설정 파일 (절대 Git에 커밋 금지)
├── requirements.txt         # Python 의존성 목록
├── main.py                  # 메인 실행 로직 및 API 호출 진입점
├── data_access.py           # 외부 API 접근 및 데이터 파싱 로직 모듈
├── models/                  # Pydantic 모델 정의 폴더 (데이터 구조 정의)
│   ├── __init__.py
│   └── api_schemas.py       # 모든 API 응답에 대한 Pydantic 스키마 정의
├── security/                # 보안 관련 설정 및 스크립트
│   └── config_loader.py     # 환경 변수 로드 및 검증 로직
└── README.md                # 프로젝트 개요 및 사용법
```

**설명:**
*   `data_access.py`: 실제 데이터 통신 로직을 분리하여 비즈니스 로직과 외부 통신을 분리합니다.
*   `models/api_schemas.py`: API 응답 구조를 엄격하게 정의하여 데이터의 안정성을 확보하고, 향후 데이터 파이프라인 구축 시 모델 불일치 오류를 방지합니다.
*   `security/config_loader.py`: 환경 변수 로드 및 보안 검증을 전담하는 모듈로 분리하여 보안 책임(Operations Team)을 명확히 합니다.

---

### 2. 🔒 데이터 모델 프레임워크 (Pydantic Schema) 설계 초안

데이터 접근의 안정성을 위해 외부 API 응답에 대한 Pydantic 모델 정의를 우선적으로 수행합니다. 이는 Researcher가 데이터 파싱 로직 설계에 즉시 활용할 수 있는 기반이 됩니다.

**파일 위치:** `models/api_schemas.py`

```python
# models/api_schemas.py

from pydantic import BaseModel, Field
from typing import List, Optional
from datetime import datetime

# --- 1. 기본 데이터 구조 정의 (API 응답에 따른 스키마) ---

class BaseApiResponse(BaseModel):
    """모든 API 응답의 기본 공통 필드"""
    status: str = Field(..., description="요청 성공 여부 (success/error)")
    timestamp: datetime = Field(default_factory=datetime.utcnow)
    data: Optional[dict] = Field(None, description="실제 반환 데이터")

class UserDataSchema(BaseModel):
    """사용자 정보 API 응답 스키마 예시"""
    user_id: int = Field(..., description="고유 사용자 ID")
    username: str = Field(..., description="사용자 이름")
    email: str = Field(..., description="사용자 이메일")
    created_at: datetime = Field(..., description="계정 생성 일시")

class TransactionSchema(BaseModel):
    """거래 기록 API 응답 스키마 예시"""
    transaction_id: str = Field(..., description="거래 고유 ID")
    amount: float = Field(..., gt=0, description="거래 금액 (양수여야 함)")
    currency: str = Field(..., description="통화 코드 (예: KRW)")
    transaction_date: datetime = Field(..., description="거래 일시")

# --- 2. 복합 응답 구조 예시 (실제 파이프라인을 위해 필요) ---

class DataPipelineResult(BaseModel):
    """데이터 파이프라인 작업 전체 결과"""
    success: bool = Field(False, description="파이프라인 성공 여부")
    processed_records: int = Field(0, description="성공적으로 처리된 레코드 수")
    errors: List[str] = Field(default_factory=list, description="발생한 오류 목록")
    raw_data: Optional[List[dict]] = Field(None, description="원시 데이터 (디버깅용)")

```

---

### 3. ✅ 보안 체크리스트 초안 (Security Checklist Draft)

Operations Team의 요구사항을 반영하여 시스템 전체에 적용해야 할 필수 보안 항목을 정의합니다.

**파일 위치:** `security/security_checklist.md`

#### A. 환경 변수 관리 및 접근 통제
1.  **🔑 민감 정보 분리:** 모든 API 키, 비밀번호 등 민감 정보는 코드 내부에 하드코딩을 엄격히 금지한다. (✅ **환경 변수 사용 의무화**)
2.  **🔒 로드 방식 검증:** 환경 변수는 `os.getenv()` 등을 통해 로드하며, 로드 실패 시 즉시 시스템을 종료하거나 에러를 발생시켜 데이터 접근을 차단해야 한다.
3.  **🚫 Git 제외:** `.env` 파일은 반드시 `.gitignore`에 포함하여 버전 관리 시스템에 노출되지 않도록 한다.

#### B. 데이터 접근 및 파싱 보안 (Researcher Focus)
1.  **🛡️ 입력 유효성 검사:** 모든 외부로부터 수신된 데이터(API 응답)는 Pydantic 모델을 통해 **엄격하게 타입 검증 및 구조적 유효성 검사**를 거쳐야 한다.
2.  **🛑 인젝션 방지:** 데이터 파싱 과정에서 발생하는 문자열 처리 시, SQL/NoSQL 인젝션 공격에 대비한 입력값 필터링 로직을 적용한다.

#### C. 시스템 안정성 및 모니터링 (Operations Focus)
1.  **🚨 오류 로깅:** 모든 API 호출 실패, 데이터 유효성 검사 실패(Pydantic 에러 포함), 환경 변수 로드 실패는 중앙 집중식 로그 시스템에 기록되어야 한다.
2.  **🔍 접근 감사:** 데이터 파이프라인의 각 단계별 성공/실패 상태를 추적할 수 있는 메타데이터를 `DataPipelineResult` 모델에 반드시 포함한다.

---

📝 다음 단계: Researcher는 제시된 Pydantic 스키마(`api_schemas.py`)를 기반으로 실제 API 응답 구조(예: AIzaSyCFUM)를 분석하고, 데이터 파싱 로직 설계의 초기 프레임워크를 구체화합니다.
