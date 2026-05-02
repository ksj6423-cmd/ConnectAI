# 💻 Developer — Researcher와 이전 논의된 보안 프레임워크 및 Pydantic 모델 정의(`SecurityContext`, `ErrorResponse`, `TrendDataPoint` 등)를 기반으로, 환경 변수를 통해 API 키를 안전하게 로드하고 데이터 파싱 로직을 구현하는 Python 파일(`models.py`, `data_access.py`)을 즉시 작성해 주세요.

💻 Developer: 작업 시작합니다.

요청하신 대로 보안 프레임워크와 Pydantic 모델을 기반으로 환경 변수를 통해 API 키를 안전하게 로드하고 데이터 파싱 로직을 구현하는 Python 파일(`models.py`, `data_access.py`)을 작성합니다.

이 코드는 외부 API 접근 시 필요한 데이터 구조의 안정성을 확보하고, 민감한 정보(API 키)가 코드 내에 직접 노출되지 않도록 환경 변수를 통해 접근하는 보안 원칙을 준수합니다.

### 1. `models.py` (데이터 모델 정의)

이 파일은 데이터 파싱 및 응답 구조의 안정성을 보장하기 위해 Pydantic 모델을 정의합니다.

```python
# models.py

from pydantic import BaseModel, Field
from typing import Dict, Any, Optional
from datetime import datetime

# 1. 보안 컨텍스트 모델: API 접근 권한 및 환경 설정 관리
class SecurityContext(BaseModel):
    """API 접근에 필요한 보안 컨텍스트를 정의합니다."""
    api_key_name: str = Field(..., description="접근하려는 API의 이름 (예: 'ExternalDataAPI')")
    access_level: str = Field("read", description="요청하는 접근 수준 (read, write, admin)")
    timeout_seconds: int = Field(30, description="API 호출에 대한 최대 타임아웃 시간")

# 2. 에러 응답 모델: API 호출 실패 시 일관된 오류 구조 제공
class ErrorResponse(BaseModel):
    """API 호출 중 발생한 오류를 표준화하여 반환합니다."""
    status_code: int = Field(400, description="HTTP 상태 코드")
    error_type: str = Field("API_ERROR", description="오류 유형 (예: AUTH_FAILED, TIMEOUT, PARSING_ERROR)")
    message: str = Field(..., description="상세 오류 메시지")
    details: Optional[Dict[str, Any]] = None

# 3. 데이터 포인트 모델: 실제 파싱된 핵심 데이터 구조
class TrendDataPoint(BaseModel):
    """데이터 파이프라인에서 처리될 핵심 트렌드 데이터를 정의합니다."""
    timestamp: datetime = Field(..., description="데이터가 수집된 시각")
    metric_name: str = Field(..., description="측정된 지표의 이름")
    value: float = Field(..., description="측정된 실제 값")
    source: str = Field(..., description="데이터 출처")

# 4. 통합 응답 모델 (선택 사항, 필요시 확장)
class ApiResponse(BaseModel):
    """API 호출의 전체 성공 또는 실패 결과를 담는 컨테이너."""
    success: bool
    data: Optional[Dict[str, Any]] = None
    error: Optional[ErrorResponse] = None

```

### 2. `data_access.py` (데이터 접근 및 파싱 로직)

이 파일은 환경 변수에서 API 키를 안전하게 로드하고, 정의된 모델을 사용하여 외부 데이터에 접근하는 핵심 로직을 구현합니다.

```python
# data_access.py

import os
from typing import Dict, Any, List
from datetime import datetime

# Pydantic 모델 임포트
from models import SecurityContext, ErrorResponse, TrendDataPoint, ApiResponse

class DataAccessError(Exception):
    """데이터 접근 및 파싱 중 발생하는 사용자 정의 예외."""
    pass

class APIDataAccess:
    """
    환경 변수를 사용하여 API 키를 안전하게 관리하며 외부 데이터에 접근하는 클래스.
    보안 및 데이터 유효성 검사를 책임집니다.
    """
    def __init__(self):
        # 환경 변수에서 필요한 모든 키를 로드합니다.
        self._api_keys: Dict[str, str] = {}
        self._security_context: Optional[SecurityContext] = None

    def load_security_context(self) -> SecurityContext:
        """환경 변수에서 API 키와 보안 컨텍스트를 로드합니다."""
        try:
            # 예시: 실제 환경변수 이름에 따라 키를 로드한다고 가정합니다.
            api_key = os.environ.get("API_KEY_EXTERNAL_DATA")
            if not api_key:
                raise KeyError("API_KEY_EXTERNAL_DATA 환경 변수가 설정되지 않았습니다.")

            # 실제 보안 컨텍스트 정보도 환경 변수에서 로드할 수 있습니다. (여기서는 예시로 하드코딩)
            context = SecurityContext(
                api_key_name="ExternalDataAPI",
                access_level="read",
                timeout_seconds=30
            )

            self._security_context = context
            print("✅ 보안 컨텍스트 로드 성공.")
            return context

        except KeyError as e:
            raise DataAccessError(f"보안 설정 오류: {e}") from e
        except Exception as e:
            raise DataAccessError(f"API 키 로딩 중 예상치 못한 오류 발생: {e}") from e

    def fetch_and_parse_data(self, endpoint: str) -> ApiResponse:
        """
        지정된 엔드포인트에서 데이터를 가져오고 Pydantic 모델로 파싱합니다.
        실제 API 호출은 Mock 처리됩니다.
        """
        if not self._security_context:
            raise DataAccessError("보안 컨텍스트가 로드되지 않았습니다. 먼저 load_security_context()를 호출하세요.")

        api_key = os.environ.get("API_KEY_EXTERNAL_DATA")
        if not api_key:
             raise DataAccessError("API 키가 환경 변수에 설정되어 있지 않아 접근할 수 없습니다.")

        print(f"🔑 API 키를 사용하여 {endpoint} 호출 시도...")
        
        # --- [실제 API 호출 로직 대체 부분] ---
        try:
            # 실제 HTTP 요청 및 응답 파싱이 여기에 위치합니다. (예: requests.get(...))
            
            # Mock 데이터 생성 (테스트를 위한 예시)
            mock_raw_data = {
                "timestamp": datetime.now().isoformat(),
                "metric_name": "Latency",
                "value": 45.7,
                "source": "Simulated_API_Endpoint"
            }

            # 데이터 모델로 파싱
            data_point = TrendDataPoint(
                timestamp=datetime.fromisoformat(mock_raw_data["timestamp"]),
                metric_name=mock_raw_data["metric_name"],
                value=float(mock_raw_data["value"]),
                source=mock_raw_data["source"]
            )

            return ApiResponse(
                success=True,
                data={"trend_data": [data_point]},
                error=None
            )

        except Exception as e:
            # 오류 발생 시 표준화된 에러 응답 반환
            print(f"❌ 데이터 파싱 중 오류 발생: {e}")
            error_details = {
                "code": "PARSING_ERROR",
                "message": f"데이터 파싱 실패: {str(e)}",
                "trace": str(e)
            }
            return ApiResponse(
                success=False,
                data=None,
                error=ErrorResponse(
                    status_code=500,
                    error_type="PARSING_ERROR",
                    message="데이터 구조 파싱에 실패했습니다.",
                    details=error_details
                )
            )

# --- 사용 예시 (테스트 코드) ---
if __name__ == '__main__':
    print("--- Data Access Module 테스트 시작 ---")
    
    access_manager = APIDataAccess()
    
    try:
        # 1. 보안 컨텍스트 로드 (환경 변수 필요)
        security = access_manager.load_security_context()
        print(f"Loaded Context: {security.api_key_name}, Level: {security.access_level}")

        # 2. 데이터 가져오기 및 파싱 시도
        result = access_manager.fetch_and_parse_data("/v1/trends")
        
        print("\n--- 최종 결과 ---")
        if result.success:
            for item in result.data.get("trend_data", []):
                print(f"성공적으로 파싱된 데이터: {item.metric_name} = {item.value}, 시간: {item.timestamp}")
        else:
            print("API 호출 결과 실패:", result.error.model_dump_json())

    except DataAccessError as e:
        print(f"\n🚨 치명적인 데이터 접근 오류 발생: {e}")
    except Exception as e:
        print(f"\n🚨 예상치 못한 시스템 오류: {e}")

```

📊 평가: 완료 — 요청된 보안 프레임워크와 Pydantic 모델을 기반으로 환경 변수 로딩 및 데이터 파싱 로직이 구현되었습니다.

📝 다음 단계: 실제 API 호출 라이브러리(`requests`)를 통합하여 `fetch_and_parse_data` 메서드의 Mock 부분을 실제 외부 통신으로 대체하고, 테스트 환경에서 완전한 End-to-End 흐름을 검증합니다.
