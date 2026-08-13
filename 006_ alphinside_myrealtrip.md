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

from .purchasing_agent import PurchasingAgent
from dotenv import load_dotenv
import os

load_dotenv()

root_agent = PurchasingAgent(
    remote_agent_addresses=[
        os.getenv("PIZZA_SELLER_AGENT_URL", "http://localhost:10000"),
        os.getenv("BURGER_SELLER_AGENT_URL", "http://localhost:10001"),
    ]
).create_agent()

리팩 중요성 6.5~7/10. 기능 자체는 단순하고 현재 코드도 정상 동작하기 쉽지만, 운영환경에서 환경변수 누락을 localhost로 조용히 대체하는 부분이 가장 위험하다. 이 부분과 URL 검증만 보강하면 충분하고, 그 이상 구조를 키울 필요는 없다.

제안패치
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

import os
from urllib.parse import urlparse

from dotenv import load_dotenv

from .purchasing_agent import PurchasingAgent


load_dotenv()

_REQUIRED_AGENT_URLS = (
    "PIZZA_SELLER_AGENT_URL",
    "BURGER_SELLER_AGENT_URL",
)


def _get_agent_url(name: str) -> str:
    url = os.getenv(name)

    if not url:
        raise RuntimeError(
            f"Required environment variable '{name}' is not configured."
        )

    parsed = urlparse(url)

    if parsed.scheme not in {"http", "https"} or not parsed.netloc:
        raise RuntimeError(
            f"Environment variable '{name}' contains an invalid agent URL."
        )

    return url.rstrip("/")


def _build_remote_agent_addresses() -> list[str]:
    return [_get_agent_url(name) for name in _REQUIRED_AGENT_URLS]


root_agent = PurchasingAgent(
    remote_agent_addresses=_build_remote_agent_addresses()
).create_agent()


최종 개선사항
✅ 환경변수 누락 → localhost 자동 대체 → 필수 설정 누락을 즉시 실패시켜 잘못된 원격 연결 방지
✅ 문자열 존재 여부만 사용 → HTTP/HTTPS scheme·host 검증 → 잘못된 Agent URL의 초기 유입 차단
✅ 피자·버거 URL 검증 로직 분산 가능성 → 공통 _get_agent_url() 사용 → 설정 검증 규칙 일관성 확보
✅ URL 끝단의 불필요한 / → 로딩 시 정규화 → 원격 주소 조합 시 경로 중복 위험 감소
✅ root_agent 초기화에 검증되지 않은 환경값 직접 전달 → 검증 완료된 주소만 PurchasingAgent에 전달 → Agent 초기화 무결성 강화
✅ 설정값 접근과 Agent 생성 로직 혼재 → URL 수집·검증을 별도 함수로 분리 → 테스트 및 유지보수성 향상

최종 판정: 이 정도면 동결해도 된다. Config 클래스나 별도 설정 관리 계층까지 만드는 것은 현재 코드 규모에서는 과설계고, 환경변수 누락·잘못된 URL이라는 실제 장애 지점만 차단한 9.5~9.7 수준의 적정 리팩이다.
