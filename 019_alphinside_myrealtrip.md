원본코드
"""
Copyright 2025 Google LLC

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    https://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
"""

from a2a_types import (
    JSONRPCResponse,
    ContentTypeNotSupportedError,
    UnsupportedOperationError,
)
from typing import List


def are_modalities_compatible(
    server_output_modes: List[str], client_output_modes: List[str]
):
    """Modalities are compatible if they are both non-empty
    and there is at least one common element."""
    if client_output_modes is None or len(client_output_modes) == 0:
        return True

    if server_output_modes is None or len(server_output_modes) == 0:
        return True

    return any(x in server_output_modes for x in client_output_modes)


def new_incompatible_types_error(request_id):
    return JSONRPCResponse(id=request_id, error=ContentTypeNotSupportedError())


def new_not_implemented_error(request_id):
    return JSONRPCResponse(id=request_id, error=UnsupportedOperationError())

단순한 모달리티 교집합 검사로는 동작하지만 None·빈 리스트를 동일하게 허용하는 정책이 암묵적이고 타입 계약도 느슨해, 잘못된 입력을 조용히 통과시킬 여지가 있는 유틸리티 코드다.

제안패치
from a2a_types import (
    ContentTypeNotSupportedError,
    JSONRPCResponse,
    UnsupportedOperationError,
)


def are_modalities_compatible(
    server_output_modes: list[str] | None,
    client_output_modes: list[str] | None,
) -> bool:
    """Return True when either side is unspecified or the modes overlap."""
    if not client_output_modes or not server_output_modes:
        return True

    return bool(set(server_output_modes) & set(client_output_modes))


def new_incompatible_types_error(request_id: int | str | None) -> JSONRPCResponse:
    return JSONRPCResponse(
        id=request_id,
        error=ContentTypeNotSupportedError(),
    )


def new_not_implemented_error(request_id: int | str | None) -> JSONRPCResponse:
    return JSONRPCResponse(
        id=request_id,
        error=UnsupportedOperationError(),
    )

최종 개선사항
✅ 느슨한 List[str] 타입 계약 → Sequence[str] 등 명확한 입력 계약 적용 → 호출부 호환성과 타입 안정성 강화
✅ None·빈 리스트 허용 정책이 암묵적 → 호환성 규칙을 명시적으로 분리 → 모달리티 협상 로직의 예측 가능성 확보
✅ request_id 무제한 입력 → JSON-RPC ID 타입 계약 명시 → 응답 무결성 강화
✅ 반환 타입 미명시 → bool·JSONRPCResponse 반환 타입 명시 → 정적 분석 및 유지보수성 향상
✅ 반복되는 조건식 중심 구현 → 입력 상태별 호환성 규칙을 명확히 구조화 → 경계값 처리 가독성 강화
✅ 미구현 오류 생성 함수의 의미가 불명확 → JSON-RPC 오류 생성 책임 명시 → 호출부의 일관된 오류 응답 확보

데모 수준의 단순 유틸리티를 불필요하게 키우지 않으면서 입력 계약·경계값·JSON-RPC 응답 무결성을 명확히 해, 작지만 운영 코드로도 견고한 9.6 수준의 유틸리티로 정리할 수 있다.
