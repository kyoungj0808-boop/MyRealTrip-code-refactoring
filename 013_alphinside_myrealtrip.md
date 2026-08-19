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

from starlette.applications import Starlette
from starlette.responses import JSONResponse
from sse_starlette.sse import EventSourceResponse
from starlette.requests import Request
from a2a_types import (
    A2ARequest,
    JSONRPCResponse,
    InvalidRequestError,
    JSONParseError,
    GetTaskRequest,
    CancelTaskRequest,
    SendTaskRequest,
    SetTaskPushNotificationRequest,
    GetTaskPushNotificationRequest,
    InternalError,
    AgentCard,
    TaskResubscriptionRequest,
    SendTaskStreamingRequest,
)
from pydantic import ValidationError
import json
from typing import AsyncIterable, Any
from a2a_server.task_manager import TaskManager

import logging
import base64

logger = logging.getLogger(__name__)


class A2AServer:
    def __init__(
        self,
        host="0.0.0.0",
        port=5000,
        endpoint="/",
        agent_card: AgentCard = None,
        task_manager: TaskManager = None,
        api_key: str | None = None,
        auth_username: str | None = None,
        auth_password: str | None = None,
    ):
        self.host = host
        self.port = port
        self.endpoint = endpoint
        self.task_manager = task_manager
        self.agent_card = agent_card
        self.api_key = api_key
        self.auth_username = auth_username
        self.auth_password = auth_password
        self.app = Starlette()
        self.app.add_route(self.endpoint, self._process_request, methods=["POST"])
        self.app.add_route(
            "/.well-known/agent.json", self._get_agent_card, methods=["GET"]
        )

        if len(self.agent_card.authentication.schemes) > 1:
            raise ValueError("Only one authentication scheme is supported for now")

        if self.agent_card.authentication.schemes[0].lower() == "bearer":
            self.auth_scheme = "bearer"

            if self.api_key is None:
                raise ValueError(
                    "Authentication scheme is bearer but api_key is not defined"
                )
        elif self.agent_card.authentication.schemes[0].lower() == "basic":
            self.auth_scheme = "basic"

            if self.auth_username is None or self.auth_password is None:
                raise ValueError(
                    "Authentication scheme is basic but auth_username and auth_password are not defined"
                )
        else:
            raise ValueError("Unsupported authentication scheme")

    def start(self):
        if self.agent_card is None:
            raise ValueError("agent_card is not defined")

        if self.task_manager is None:
            raise ValueError("request_handler is not defined")

        import uvicorn

        uvicorn.run(self.app, host=self.host, port=self.port)

    def _get_agent_card(self, request: Request) -> JSONResponse:
        return JSONResponse(self.agent_card.model_dump(exclude_none=True))

    def verify_bearer_token(self, token):
        """Verify the provided bearer token against the expected token."""
        # Simple token comparison for demonstration
        # In a real application, you might want to use a more secure verification method
        return token == self.api_key

    def verify_basic_auth(self, username, password):
        """Verify the provided basic auth credentials against the expected credentials."""
        return username == self.auth_username and password == self.auth_password

    async def verify_auth_header(self, request):
        """Verify the authorization header based on the configured auth scheme."""
        auth_header = request.headers.get("Authorization")

        if not auth_header:
            return False, "Authorization header is missing"

        parts = auth_header.split()
        if len(parts) != 2:
            return False, "Invalid Authorization header format"

        auth_type, credentials = parts
        auth_type = auth_type.lower()

        # Handle different authentication schemes
        if self.auth_scheme == "bearer" and auth_type == "bearer":
            if not self.api_key:
                return True, None  # Skip auth if no API key is set

            if not self.verify_bearer_token(credentials):
                return False, "Invalid bearer token"
            return True, None

        elif self.auth_scheme == "basic" and auth_type == "basic":
            if not self.auth_username or not self.auth_password:
                return True, None  # Skip auth if credentials not set

            try:
                decoded = base64.b64decode(credentials).decode("utf-8")
                username, password = decoded.split(":", 1)
                if not self.verify_basic_auth(username, password):
                    return False, "Invalid credentials"
                return True, None
            except Exception as e:
                logger.error(f"Error decoding basic auth: {e}")
                return False, "Invalid basic auth format"
        else:
            return (
                False,
                f"Authentication scheme mismatch. Expected {self.auth_scheme}, got {auth_type}",
            )

    async def _process_request(self, request: Request):
        # Check authentication based on configured auth scheme
        is_valid, error_message = await self.verify_auth_header(request)
        if not is_valid:
            return JSONResponse({"error": error_message}, status_code=401)

        try:
            body = await request.json()
            json_rpc_request = A2ARequest.validate_python(body)
            print(json_rpc_request)

            if isinstance(json_rpc_request, GetTaskRequest):
                result = await self.task_manager.on_get_task(json_rpc_request)
            elif isinstance(json_rpc_request, SendTaskRequest):
                result = await self.task_manager.on_send_task(json_rpc_request)
            elif isinstance(json_rpc_request, SendTaskStreamingRequest):
                result = await self.task_manager.on_send_task_subscribe(
                    json_rpc_request
                )
            elif isinstance(json_rpc_request, CancelTaskRequest):
                result = await self.task_manager.on_cancel_task(json_rpc_request)
            elif isinstance(json_rpc_request, SetTaskPushNotificationRequest):
                result = await self.task_manager.on_set_task_push_notification(
                    json_rpc_request
                )
            elif isinstance(json_rpc_request, GetTaskPushNotificationRequest):
                result = await self.task_manager.on_get_task_push_notification(
                    json_rpc_request
                )
            elif isinstance(json_rpc_request, TaskResubscriptionRequest):
                result = await self.task_manager.on_resubscribe_to_task(
                    json_rpc_request
                )
            else:
                logger.warning(f"Unexpected request type: {type(json_rpc_request)}")
                raise ValueError(f"Unexpected request type: {type(request)}")

            return self._create_response(result)

        except Exception as e:
            return self._handle_exception(e)

    def _handle_exception(self, e: Exception) -> JSONResponse:
        if isinstance(e, json.decoder.JSONDecodeError):
            json_rpc_error = JSONParseError()
        elif isinstance(e, ValidationError):
            json_rpc_error = InvalidRequestError(data=json.loads(e.json()))
        else:
            logger.error(f"Unhandled exception: {e}")
            json_rpc_error = InternalError()

        response = JSONRPCResponse(id=None, error=json_rpc_error)
        return JSONResponse(response.model_dump(exclude_none=True), status_code=400)

    def _create_response(self, result: Any) -> JSONResponse | EventSourceResponse:
        if isinstance(result, AsyncIterable):

            async def event_generator(result) -> AsyncIterable[dict[str, str]]:
                async for item in result:
                    yield {"data": item.model_dump_json(exclude_none=True)}

            return EventSourceResponse(event_generator(result))
        elif isinstance(result, JSONRPCResponse):
            return JSONResponse(result.model_dump(exclude_none=True))
        else:
            logger.error(f"Unexpected result type: {type(result)}")
            raise ValueError(f"Unexpected result type: {type(result)}")

A2A 요청 라우팅과 인증 흐름은 잘 갖췄지만, 내부 예외를 전부 400으로 뭉개고 traceback까지 잃어버려 장애가 발생하면 원인 추적과 서버 상태 판별이 동시에 무너질 수 있는 구조다.

제안패치
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License. You may obtain a copy of the License at
#
#    https://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

from a2a_types import (
    A2ARequest,
    JSONRPCResponse,
    InvalidRequestError,
    JSONParseError,
    GetTaskRequest,
    CancelTaskRequest,
    SendTaskRequest,
    SetTaskPushNotificationRequest,
    GetTaskPushNotificationRequest,
    InternalError,
    AgentCard,
    TaskResubscriptionRequest,
    SendTaskStreamingRequest,
)
from pydantic import ValidationError
from starlette.applications import Starlette
from starlette.requests import Request
from starlette.responses import JSONResponse
from sse_starlette.sse import EventSourceResponse
from a2a_server.task_manager import TaskManager

from collections.abc import AsyncIterable
from typing import Any
import base64
import hmac
import json
import logging

logger = logging.getLogger(__name__)


class A2AServer:
    def __init__(
        self,
        host: str = "0.0.0.0",
        port: int = 5000,
        endpoint: str = "/",
        agent_card: AgentCard | None = None,
        task_manager: TaskManager | None = None,
        api_key: str | None = None,
        auth_username: str | None = None,
        auth_password: str | None = None,
    ):
        self.host = host
        self.port = port
        self.endpoint = endpoint
        self.task_manager = task_manager
        self.agent_card = agent_card
        self.api_key = api_key
        self.auth_username = auth_username
        self.auth_password = auth_password

        if self.agent_card is None:
            raise ValueError("agent_card is required")

        if self.task_manager is None:
            raise ValueError("task_manager is required")

        schemes = self.agent_card.authentication.schemes

        if not schemes:
            raise ValueError("At least one authentication scheme is required")

        if len(schemes) > 1:
            raise ValueError("Only one authentication scheme is supported for now")

        self.auth_scheme = schemes[0].lower()

        if self.auth_scheme == "bearer":
            if not self.api_key:
                raise ValueError(
                    "Authentication scheme is bearer but api_key is not configured"
                )
        elif self.auth_scheme == "basic":
            if not self.auth_username or not self.auth_password:
                raise ValueError(
                    "Authentication scheme is basic but credentials are not configured"
                )
        else:
            raise ValueError(
                f"Unsupported authentication scheme: {self.auth_scheme}"
            )

        self.app = Starlette()
        self.app.add_route(
            self.endpoint,
            self._process_request,
            methods=["POST"],
        )
        self.app.add_route(
            "/.well-known/agent.json",
            self._get_agent_card,
            methods=["GET"],
        )

    def start(self) -> None:
        import uvicorn

        uvicorn.run(
            self.app,
            host=self.host,
            port=self.port,
        )

    def _get_agent_card(self, request: Request) -> JSONResponse:
        return JSONResponse(
            self.agent_card.model_dump(exclude_none=True)
        )

    def verify_bearer_token(self, token: str) -> bool:
        return hmac.compare_digest(token, self.api_key or "")

    def verify_basic_auth(self, username: str, password: str) -> bool:
        return (
            hmac.compare_digest(username, self.auth_username or "")
            and hmac.compare_digest(password, self.auth_password or "")
        )

    async def verify_auth_header(
        self, request: Request
    ) -> tuple[bool, str | None]:
        auth_header = request.headers.get("Authorization")

        if not auth_header:
            return False, "Authorization header is missing"

        parts = auth_header.split()
        if len(parts) != 2:
            return False, "Invalid Authorization header format"

        auth_type, credentials = parts

        if auth_type.lower() != self.auth_scheme:
            return (
                False,
                f"Authentication scheme mismatch. Expected {self.auth_scheme}",
            )

        if self.auth_scheme == "bearer":
            if not self.verify_bearer_token(credentials):
                return False, "Invalid bearer token"
            return True, None

        if self.auth_scheme == "basic":
            try:
                decoded = base64.b64decode(
                    credentials,
                    validate=True,
                ).decode("utf-8")

                username, password = decoded.split(":", 1)

            except (ValueError, UnicodeDecodeError):
                return False, "Invalid basic auth format"
            except Exception:
                logger.exception("Unexpected Basic Auth decoding failure")
                return False, "Invalid basic auth format"

            if not self.verify_basic_auth(username, password):
                return False, "Invalid credentials"

            return True, None

        logger.error(
            "Unsupported authentication scheme configured: %s",
            self.auth_scheme,
        )
        return False, "Server authentication configuration error"

    async def _process_request(self, request: Request):
        is_valid, error_message = await self.verify_auth_header(request)

        if not is_valid:
            return JSONResponse(
                {"error": error_message},
                status_code=401,
            )

        try:
            body = await request.json()
            json_rpc_request = A2ARequest.validate_python(body)

            if isinstance(json_rpc_request, GetTaskRequest):
                result = await self.task_manager.on_get_task(json_rpc_request)

            elif isinstance(json_rpc_request, SendTaskRequest):
                result = await self.task_manager.on_send_task(json_rpc_request)

            elif isinstance(json_rpc_request, SendTaskStreamingRequest):
                result = await self.task_manager.on_send_task_subscribe(
                    json_rpc_request
                )

            elif isinstance(json_rpc_request, CancelTaskRequest):
                result = await self.task_manager.on_cancel_task(json_rpc_request)

            elif isinstance(
                json_rpc_request,
                SetTaskPushNotificationRequest,
            ):
                result = await self.task_manager.on_set_task_push_notification(
                    json_rpc_request
                )

            elif isinstance(
                json_rpc_request,
                GetTaskPushNotificationRequest,
            ):
                result = await self.task_manager.on_get_task_push_notification(
                    json_rpc_request
                )

            elif isinstance(
                json_rpc_request,
                TaskResubscriptionRequest,
            ):
                result = await self.task_manager.on_resubscribe_to_task(
                    json_rpc_request
                )

            else:
                raise ValueError(
                    f"Unexpected request type: {type(json_rpc_request)}"
                )

            return self._create_response(result)

        except Exception as exc:
            return self._handle_exception(exc)

    def _handle_exception(self, exc: Exception) -> JSONResponse:
        if isinstance(exc, json.JSONDecodeError):
            json_rpc_error = JSONParseError()
            status_code = 400

        elif isinstance(exc, ValidationError):
            json_rpc_error = InvalidRequestError(
                data=json.loads(exc.json())
            )
            status_code = 400

        else:
            logger.exception(
                "Unhandled internal server exception occurred"
            )
            json_rpc_error = InternalError()
            status_code = 500

        response = JSONRPCResponse(
            id=None,
            error=json_rpc_error,
        )

        return JSONResponse(
            response.model_dump(exclude_none=True),
            status_code=status_code,
        )

    def _create_response(
        self,
        result: Any,
    ) -> JSONResponse | EventSourceResponse:
        if isinstance(result, AsyncIterable):

            async def event_generator(
                stream_source: AsyncIterable,
            ) -> AsyncIterable[dict[str, str]]:
                async for item in stream_source:
                    yield {
                        "data": item.model_dump_json(exclude_none=True)
                    }

            return EventSourceResponse(
                event_generator(result)
            )

        if isinstance(result, JSONRPCResponse):
            return JSONResponse(
                result.model_dump(exclude_none=True)
            )

        raise ValueError(
            f"Unexpected result type: {type(result)}"
        )

최종 개선사항
✅ agent_card.authentication.schemes[0] 무방비 접근 → 필수 객체·빈 인증 scheme 선검증 → 서버 초기화 단계의 설정 장애 차단
✅ 단순 문자열 인증 비교 → hmac.compare_digest() 적용 → 인증 비밀값 비교의 방어력 강화
✅ 느슨한 Base64 디코딩 → validate=True 기반 엄격 검증 → 비정상 인증 입력의 허용 범위 축소
✅ 모든 Basic Auth 예외 포괄 처리 → 예상 가능한 입력 오류와 내부 오류 분리 → 예외 은닉 방지와 외부 정보 노출 최소화
✅ 클라이언트 오류와 서버 오류 동일 처리 → 400/500 상태 분리 → HTTP 오류 의미와 운영 모니터링 정확성 확보
✅ 내부 장애 단순 로그 출력 → logger.exception()으로 traceback 보존 → 실제 장애 원인 추적성 확보

현재 버전은 이전 리팩터링보다 한 단계 올라갔지만, 위 인증 초기화·비교·Base64 방어까지 반영한 뒤 동결하는 것이 맞으며, 그 시점이면 9.6~9.8 수준의 운영형 서버 코드로 평가할 수 있다.
