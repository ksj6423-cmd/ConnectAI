# 💻 Developer — 연구 결과를 바탕으로 환경 변수 설정을 포함한 안전한 API 접근 권한 확보 계획과 Pydantic 모델의 초기 코드를 작성하여 기반 환경을 구축할 것

💻 Developer: 작업 시작합니다.

CEO의 지시에 따라, 환경 변수를 이용한 안전한 API 접근 권한 확보 계획과 Pydantic 모델을 기반으로 한 데이터 파싱 및 접근 모듈을 구축하겠습니다. 이는 보안과 데이터 무결성을 최우선으로 합니다.

### 1. 안전한 환경 설정 및 보안 계획 (Security & Setup Plan)

API 키와 같은 민감 정보는 코드 내에 절대 하드코딩하지 않고, 운영체제의 환경 변수를 통해 접근하도록 설계합니다.

**A. 환경 변수 설정 계획:**
모든 API 키 및 민감 데이터는 `.env` 파일 또는 시스템 환경 변수로 관리되어야 합니다.

1.  **파일 생성:** 프로젝트 루트에 `.env` 파일을 생성하여 환경 변수를 정의합니다. (이 파일은 Git에 커밋되지 않도록 `.gitignore`에 추가해야 합니다.)
2.  **변수 정의 예시 (`.env`):**
    ```ini
    # .env 파일 내용 (실제 키는 여기에 입력)
    API_KEY="YOUR_SECURE_API_KEY_HERE"
    API_BASE_URL="https://api.example.com/v1"
    ```
3.  **접근 방식:** Python 코드 내에서는 `os.getenv("VARIABLE_NAME")`을 사용하여 이 값들을 동적으로 로드합니다.

**B. 보안 체크리스트 (Security Checklist):**
*   [X] API 키는 환경 변수로만 접근한다. (하드코딩 금지)
*   [X] 데이터 파싱은 Pydantic 모델을 통해 응답 구조를 엄격하게 검증한다.
*   [ ] API 통신 시 HTTPS 사용을 강제한다. (추가 검토 필요)
*   [ ] 모든 외부 호출은 최소 권한 원칙을 따른다.

---

### 2. Pydantic 모델 및 데이터 접근 모듈 구현 (Code Implementation)

연구 결과를 바탕으로 `models.py`와 `data_access.py` 파일을 구성하여 기반 환경을 구축합니다.

#### A. `models.py` (데이터 모델 정의)

이 파일은 API 응답의 구조적 안정성을 보장하기 위해 Pydantic 모델을 정의합니다. Researcher의 제안과 보안 요구사항을 반영하여 확장했습니다.

```python
# sessions/models.py

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
    trend_type: str = Field(..., description="트렌드 유형 (예: Tech, Finance)")

class TrendResponse(BaseModel):
    """API로부터 수신될 전체 트렌드 응답 구조."""
    status: str = Field("success", description="전체 요청 상태")
    total_count: int = Field(..., description="총 데이터 포인트 수")
    data: List[TrendDataPoint] = Field(..., description="수집된 트렌드 목록")
    metadata: Optional[Dict[str, Any]] = None

```

#### B. `data_access.py` (API 통신 및 파싱 로직)

이 파일은 환경 변수를 통해 키를 로드하고, Pydantic 모델을 사용하여 API 응답을 안전하게 검증하는 핵심 로직을 담고 있습니다.

```python
# sessions/data_access.py

import os
import requests
from typing import Dict, Any
from datetime import datetime
from pydantic import ValidationError

from .models import TrendResponse, ErrorResponse, SecurityContext

class APIClientError(Exception):
    """API 통신 중 발생한 오류를 위한 커스텀 예외."""
    pass

class SecureAPIClient:
    """
    환경 변수를 사용하여 안전하게 API에 접근하고 Pydantic 모델로 데이터를 검증하는 클라이언트.
    """
    def __init__(self, base_url: str, api_key: str):
        if not all([base_url, api_key]):
            raise ValueError("Base URL과 API Key는 필수 설정입니다.")
            
        self.base_url = base_url
        self.headers = {
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json"
        }

    def _validate_response(self, response: requests.Response) -> Dict[str, Any]:
        """API 응답을 Pydantic 모델로 파싱하고 유효성을 검증합니다."""
        try:
            data = response.json()
            # API가 성공적인 구조를 반환한다고 가정하고 TrendResponse로 파싱 시도
            return TrendResponse(**data)
        except requests.exceptions.RequestException as e:
            raise APIClientError(f"HTTP 요청 오류 발생: {e}")
        except ValidationError as e:
            # Pydantic 유효성 검사 실패 시 에러 응답 구조를 확인하거나 상세 에러 반환
            error_details = e.errors()
            raise APIClientError(f"데이터 파싱/유효성 검증 오류: {e}") from e

    def fetch_trend_data(self, endpoint: str) -> TrendResponse:
        """
        지정된 엔드포인트에서 트렌드 데이터를 안전하게 가져옵니다.
        """
        url = f"{self.base_url}/{endpoint}"
        print(f"Fetching data from: {url}")

        try:
            response = requests.get(url, headers=self.headers, timeout=10)
            response.raise_for_status()  # 4xx/5xx 에러 발생 시 예외 발생
            
            validated_data = self._validate_response(response)
            return validated_data

        except requests.exceptions.HTTPError as e:
            # HTTP 상태 코드가 4xx 또는 5xx일 때 구체적인 오류 메시지 반환 시도
            try:
                error_response = ErrorResponse(**response.json())
                raise APIClientError(f"API 호출 실패 (HTTP {response.status_code}): {error_response.message}") from e
            except Exception:
                 # JSON 파싱이 불가능하거나 예상치 못한 응답일 경우
                raise APIClientError(f"API 호출 실패 (HTTP {response.status_code}): 응답 형식 오류.") from e
        except APIClientError as e:
             # Pydantic 또는 자체 정의 에러 발생 시
            print(f"Data Validation Error: {e}")
            raise
        except Exception as e:
            # 기타 모든 예외 처리
            raise APIClientError(f"알 수 없는 통신 오류: {e}") from e


# --- 사용 예시 (실행 방법) ---

if __name__ == '__main__':
    print("--- Secure API Client Test ---")
    
    # 1. 환경 변수 로드 (실제 실행 시 .env 파일이 필요함)
    API_KEY = os.getenv("API_KEY")
    BASE_URL = os.getenv("API_BASE_URL")

    if not API_KEY or not BASE_URL:
        print("\n[경고] 환경 변수(API_KEY, API_BASE_URL)가 설정되지 않아 테스트를 건너뜁니다.")
    else:
        try:
            client = SecureAPIClient(base_url=BASE_URL, api_key=API_KEY)
            
            # 실제 데이터 요청 예시 (이 부분은 실제 API 엔드포인트로 대체해야 함)
            endpoint = "trends" 
            print(f"\nAttempting to fetch data from {endpoint}...")
            
            # 실제 테스트를 위해서는 유효한 API와 엔드포인트가 필요함.
            # 이 코드는 보안 및 구조적 검증 프레임워크의 작동을 시연합니다.
            result = client.fetch_trend_data(endpoint)
            
            print("\n✅ 데이터 성공적으로 수신 및 검증 완료!")
            print("--- 최종 결과 ---")
            print(f"Status: {result.status}")
            print(f"Total Count: {result.total_count}")
            print(f"First Trend Name: {result.data[0].name if result.data else 'N/A'}")

        except APIClientError as e:
            print(f"\n❌ 치명적인 API 오류 발생: {e}")
        except ValueError as e:
             print(f"\n❌ 초기 설정 오류: {e}")
        except Exception as e:
            print(f"\n❌ 예상치 못한 시스템 오류: {e}")

```

📊 평가: 진행중 — 환경 변수 및 Pydantic 기반의 안전한 접근 프레임워크는 성공적으로 구축되었으나, 실제 API 연동을 위한 구체적인 엔드포인트와 키가 필요하여 최종 테스트 단계로 넘어갈 수 없습니다.
📝 다음 단계: `.env` 파일 생성 및 `data_access.py`를 사용하여 실제 API 호출을 시도하고 오류 처리 로직을 검증합니다.
