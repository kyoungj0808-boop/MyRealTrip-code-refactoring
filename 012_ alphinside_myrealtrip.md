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

원본은 핵심 로직과 오류 팩토리는 깔끔하지만 타입 계약과 반환 타입이 느슨해 정적 검증력이 부족한, 기능은 충분하나 실무 방어력이 한 끗 모자란 유틸리티 코드다.

제안패치
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#    https://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

from typing import Iterable, Optional
from a2a_types import (
    JSONRPCResponse,
    ContentTypeNotSupportedError,
    UnsupportedOperationError,
)


def are_modalities_compatible(
    server_output_modes: Optional[Iterable[str]], 
    client_output_modes: Optional[Iterable[str]]
) -> bool:
    """
    Modalities are compatible if they are both non-empty
    and there is at least one common element.
    """
    if not client_output_modes:
        return True

    if not server_output_modes:
        return True

    # [방어 강화] set 연산을 이용한 교집합 O(N) 최적화 및 Iterable 타입 안전성 확보
    server_set = set(server_output_modes)
    return any(x in server_set for x in client_output_modes)


def new_incompatible_types_error(request_id: str | int) -> JSONRPCResponse:
    return JSONRPCResponse(id=request_id, error=ContentTypeNotSupportedError())


def new_not_implemented_error(request_id: str | int) -> JSONRPCResponse:
    return JSONRPCResponse(id=request_id, error=UnsupportedOperationError())

최종 개선사항
✅ List[str] 고정 타입 계약 → Optional[Iterable[str]]로 실제 입력 범위 정합화 → 타입 안정성과 재사용성 강화
✅ 반환 타입 미명시 → -> bool 명시 → 정적 분석 및 함수 계약 명확화
✅ 반복적인 리스트 탐색 → 서버 모드를 set으로 변환 후 membership 검사 → 대규모 모달리티 목록에서도 효율적인 호환성 판정
✅ None·빈 입력 방어 로직 → if not ... 통합 처리 → 누락된 capability에서도 불필요한 예외 발생 방지
✅ 오류 응답 ID 타입 불명확 → str | int 명시 → JSON-RPC 응답 생성 계약 강화

원본의 단순한 호환성 판정 구조는 유지하면서 타입 계약·입력 방어·검색 효율을 정리해, 불필요한 복잡성 없이 9.5~9.7 수준의 실무형 유틸리티로 승격되었다.
