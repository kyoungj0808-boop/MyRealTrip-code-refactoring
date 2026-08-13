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
import base64
from typing import Any, AsyncIterable
from a2a_types import (
    AgentCard,
    SendTaskRequest,
    SendTaskResponse,
    JSONRPCRequest,
    A2AClientHTTPError,
    A2AClientJSONError,
    SendTaskStreamingResponse,
)
import json


class A2AClient:
    def __init__(self, agent_card: AgentCard, auth: str, agent_url: str):
        # The URL accessed here should be the same as the one provided in the agent card
        # However, in this demo we are using the URL provided in the key arguments
        self.url = agent_url
        # self.url = agent_card.url
        self.auth_header = None

        if agent_card.authentication:
            if len(agent_card.authentication.schemes) > 1:
                raise ValueError(
                    "Only one A2A client authentication scheme is supported for now"
                )
            elif len(agent_card.authentication.schemes) == 1:
                if agent_card.authentication.schemes[0].lower() == "bearer":
                    self.auth_header = f"Bearer {auth}"
                elif agent_card.authentication.schemes[0].lower() == "basic":
                    # Encode auth string to base64 for Basic authentication
                    encoded_auth = base64.b64encode(auth.encode()).decode()
                    self.auth_header = f"Basic {encoded_auth}"
                else:
                    raise ValueError("Unsupported authentication scheme")

    async def send_task(self, payload: dict[str, Any]) -> SendTaskResponse:
        request = SendTaskRequest(params=payload)
        return SendTaskResponse(**await self._send_request(request))

    async def send_task_streaming(
        self, payload: dict[str, Any]
    ) -> AsyncIterable[SendTaskStreamingResponse]:
        raise NotImplementedError("Streaming is not supported for now")

    async def _send_request(self, request: JSONRPCRequest) -> dict[str, Any]:
        async with httpx.AsyncClient() as client:
            try:
                # Image generation could take time, adding timeout
                print(f"Send Remote Agent Task Request: {request.model_dump()}")
                print("=" * 100)
                request_kwargs = {
                    "url": self.url,
                    "json": request.model_dump(),
                    "timeout": 30,
                }
                if self.auth_header:
                    request_kwargs["headers"] = {"Authorization": self.auth_header}

                response = await client.post(**request_kwargs)
                response.raise_for_status()
                print(f"Send Remote Agent Task Response: {response.json()}")
                print("=" * 100)
                return response.json()
            except httpx.HTTPStatusError as e:
                raise A2AClientHTTPError(e.response.status_code, str(e)) from e
            except json.JSONDecodeError as e:
                raise A2AClientJSONError(str(e)) from e

A2A 호출의 정상 경로는 간결하지만, 운영 장애의 핵심인 네트워크 예외와 타임아웃을 제대로 경계 처리하지 않고 요청·응답 전체를 출력하며, 응답 JSON을 실제 프로토콜 모델까지 안전하게 검증하는 방어층도 약하다.

제안패치
import base64
from typing import Any, AsyncIterable

import httpx

from a2a_types import (
    A2AClientHTTPError,
    A2AClientJSONError,
    AgentCard,
    JSONRPCRequest,
    SendTaskRequest,
    SendTaskResponse,
    SendTaskStreamingResponse,
)


class A2AClient:
    def __init__(
        self,
        agent_card: AgentCard,
        auth: str,
        agent_url: str,
        timeout: float = 30.0,
    ):
        self.url = agent_url
        self.timeout = timeout
        self.auth_header: str | None = None

        self._configure_auth(agent_card, auth)

    def _configure_auth(self, agent_card: AgentCard, auth: str) -> None:
        authentication = agent_card.authentication

        if not authentication:
            return

        schemes = authentication.schemes

        if len(schemes) > 1:
            raise ValueError(
                "Only one A2A client authentication scheme is supported"
            )

        if not schemes:
            return

        scheme = schemes[0].lower()

        if scheme == "bearer":
            self.auth_header = f"Bearer {auth}"
        elif scheme == "basic":
            encoded_auth = base64.b64encode(
                auth.encode("utf-8")
            ).decode("ascii")
            self.auth_header = f"Basic {encoded_auth}"
        else:
            raise ValueError(f"Unsupported authentication scheme: {schemes[0]}")

    async def send_task(self, payload: dict[str, Any]) -> SendTaskResponse:
        request = SendTaskRequest(params=payload)
        response_data = await self._send_request(request)
        return SendTaskResponse.model_validate(response_data)

    async def send_task_streaming(
        self,
        payload: dict[str, Any],
    ) -> AsyncIterable[SendTaskStreamingResponse]:
        raise NotImplementedError("Streaming is not supported for now")

    async def _send_request(
        self,
        request: JSONRPCRequest,
    ) -> dict[str, Any]:
        headers = (
            {"Authorization": self.auth_header}
            if self.auth_header
            else None
        )

        try:
            async with httpx.AsyncClient(
                timeout=self.timeout,
                headers=headers,
            ) as client:
                response = await client.post(
                    self.url,
                    json=request.model_dump(mode="json"),
                )
                response.raise_for_status()

                try:
                    return response.json()
                except ValueError as exc:
                    raise A2AClientJSONError(
                        "Remote agent returned invalid JSON"
                    ) from exc

        except httpx.HTTPStatusError as exc:
            raise A2AClientHTTPError(
                exc.response.status_code,
                f"Remote agent returned HTTP {exc.response.status_code}",
            ) from exc

        except httpx.RequestError as exc:
            raise A2AClientHTTPError(
                0,
                f"Remote agent request failed: {exc}",
            ) from exc

최종 개선사항
✅ HTTP 상태 코드만 처리 → httpx.RequestError까지 명시 처리 → 연결·타임아웃 장애의 예외 누수 방지
✅ 요청·응답 전체 print → 민감 payload 출력 제거 → 운영 환경 정보 노출 위험 차단
✅ 원격 응답 즉시 생성 → model_validate()로 응답 계약 검증 → HTTP 200 이후의 데이터 무결성 확보
✅ 전역 JSON 예외 처리 → response.json() 경계에서 명시 처리 → 실제 파싱 실패 지점의 방어력 강화
✅ 하드코딩 timeout → 생성자 설정값으로 분리 → 호출 환경별 timeout 정책 대응
✅ 인증 분기와 HTTP 요청 로직 혼재 → _configure_auth() 분리 → 인증 정책 변경 시 영향 범위 축소

최종 판정: 리팩 가치 높음. 원본은 데모 클라이언트로는 충분하지만 운영 코드 기준으로는 네트워크 실패·정보 노출·응답 계약이라는 세 경계가 약하다. 이 세 곳을 보강한 뒤 retry/circuit breaker 같은 추가 계층은 넣지 않고 동결하는 것이 가장 적절하다.
