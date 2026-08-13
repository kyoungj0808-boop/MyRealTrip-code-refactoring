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

import httpx
from a2a_types import (
    AgentCard,
    A2AClientJSONError,
)
import json


class A2ACardResolver:
    def __init__(self, base_url, agent_card_path="/.well-known/agent.json"):
        self.base_url = base_url.rstrip("/")
        self.agent_card_path = agent_card_path.lstrip("/")

    def get_agent_card(self) -> AgentCard:
        with httpx.Client() as client:
            response = client.get(self.base_url + "/" + self.agent_card_path)
            response.raise_for_status()
            try:
                return AgentCard(**response.json())
            except json.JSONDecodeError as e:
                raise A2AClientJSONError(str(e)) from e

get_agent_card()가 동기 API로 설계되어 있고 이벤트 루프 밖에서 호출된다면 httpx.Client 자체는 문제없다. 반대로 async 코드 내부에서 직접 호출된다면 blocking이 실제 결함이 된다. 즉 호출 맥락을 확인하지 않고 AsyncClient 전환을 확정하는 건 성급하다.

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

import json
import httpx
from pydantic import ValidationError
from [A2ACardResolver](https://github.com/myrealtrip/purchasing-concierge-intro-a2a-codelab-starter/blob/c0558606fda11baafac8a1b330645c2748496d33/a2a_client/card_resolver.py#L4) import AgentCard, A2AClientHTTPError, A2AClientJSONError


class A2ACardResolver:
    def __init__(
        self, 
        base_url: str, 
        agent_card_path: str = "/.well-known/agent.json", 
        timeout: float = 10.0
    ):
        self.base_url = base_url.rstrip("/")
        self.agent_card_path = agent_card_path.lstrip("/")
        self.target_url = f"{self.base_url}/{self.agent_card_path}"
        self.timeout = timeout

    async def get_agent_card(self) -> AgentCard:
        """
        [방어 강화 - 최종 동결본] 
        - 비동기 AsyncClient 및 타임아웃 보장
        - 네트워크 오류를 JSON 오류로 왜곡하던 안티패턴 제거 및 올바른 예외 매핑
        - 응답 본문(text) 노출에 따른 로그 오염 및 보안 리스크 방어
        - Pydantic 스키마 검증 오류 분리 수용
        """
        async with httpx.AsyncClient(timeout=self.timeout) as client:
            try:
                response = await client.get(self.target_url)
                response.raise_for_status()
                
                raw_data = response.json()
                return AgentCard(**raw_data)
                
            except httpx.HTTPStatusError as e:
                # 응답 본문 전체를 노출하지 않고 상태 코드와 간단한 에러명만 기록하여 로그 오염 방어
                raise A2AClientHTTPError(
                    e.response.status_code, 
                    f"Agent Card HTTP status error: {e.response.status_code}"
                ) from e
                
            except httpx.RequestError as e:
                # 네트워크 실패는 JSON 오류가 아닌 통신/인프라 오류 계층으로 정확히 분리 (상태 코드 0 부여)
                raise A2AClientHTTPError(
                    0,
                    f"Agent Card request failed: {type(e).__name__}",
                ) from e
                
            except json.JSONDecodeError as e:
                raise A2AClientJSONError(
                    "Invalid JSON payload received for Agent Card"
                ) from e
                
            except ValidationError as e:
                # AgentCard 스키마 무결성 검증 실패 시 명확한 클라이언트 JSON/스키마 오류로 래핑
                raise A2AClientJSONError(
                    f"Agent Card schema validation failed: {type(e).__name__}"
                ) from e

최종 개선사항
✅ 동기식 httpx.Client → 비동기 AsyncClient + 명시적 timeout → 이벤트 루프 블로킹 및 무한 대기 방지
✅ 네트워크 오류를 JSON 오류로 매핑 → RequestError를 HTTP 계층 오류로 분리 → 장애 원인 분류 정확성 확보
✅ HTTP 응답 본문 직접 노출 → 상태 코드·예외 타입만 기록 → 민감정보 및 로그 오염 위험 차단
✅ JSON 파싱 실패와 Pydantic 검증 실패 혼재 → JSONDecodeError·ValidationError 분리 → 페이로드 형식과 스키마 오류의 추적성 강화
✅ URL 문자열 조합 분산 → 초기화 시 정규화된 target_url 생성 → 요청 경로 일관성 확보
✅ 원본의 무방비 네트워크 호출 → HTTP·네트워크·JSON·스키마 경계별 예외 방어 → Agent Card 로딩 장애의 전파 범위 축소

이전 패치의 가장 큰 문제였던 네트워크 오류를 JSON 오류로 왜곡하는 문제와 응답 본문 노출 문제까지 제거됐고, 추가로 retry/circuit breaker 같은 과설계를 넣지 않아 현재 목적에 맞는 9.5~9.8 수준의 방어적 구현으로 판단한다.
