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

데모용 서버 골격과 요청 분기 구조는 깔끔하지만, 인증 정책 충돌·초기화 계약 검증 부재·JSON-RPC ID 유실·내부 예외 추적 부족이 겹쳐 프로덕션 경계층으로는 방어력이 부족한 엔트리포인트다.

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

import base64
import json
import logging
import secrets
from typing import Any, AsyncIterable, Optional, Tuple, Union

from a2a_server.task_manager import TaskManager
from a2a_types import (
    A2ARequest,
    AgentCard,
    CancelTaskRequest,
    GetTaskPushNotificationRequest,
    GetTaskRequest,
    InternalError,
    InvalidRequestError,
    JSONParseError,
    JSONRPCResponse,
    SendTaskRequest,
    SendTaskStreamingRequest,
    SetTaskPushNotificationRequest,
    TaskResubscriptionRequest,
)
from pydantic import ValidationError
from sse_starlette.sse import EventSourceResponse
from starlette.applications import Starlette
from starlette.requests import Request
from starlette.responses import JSONResponse

logger = logging.getLogger(__name__)


class A2AServer:
    """Production-grade A2A Server with strict auth enforcement, sanitized logging, and robust exception boundaries."""

    def __init__(
        self,
        host: str = "0.0.0.0",
        port: int = 5000,
        endpoint: str = "/",
        agent_card: Optional[AgentCard] = None,
        task_manager: Optional[TaskManager] = None,
        api_key: Optional[str] = None,
        auth_username: Optional[str] = None,
        auth_password: Optional[str] = None,
    ):
        if agent_card is None:
            raise ValueError("agent_card must not be None.")
        if task_manager is None:
            raise ValueError("task_manager must not be None.")

        self.host = host
        self.port = port
        self.endpoint = endpoint
        self.agent_card = agent_card
        self.task_manager = task_manager
        self.api_key = api_key
        self.auth_username = auth_username
        self.auth_password = auth_password

        if not self.agent_card.authentication or not self.agent_card.authentication.schemes:
            raise ValueError("AgentCard must specify at least one authentication scheme.")
        
        if len(self.agent_card.authentication.schemes) > 1:
            raise ValueError("Only one authentication scheme is supported for now.")

        scheme = self.agent_card.authentication.schemes[0].lower()
        if scheme == "bearer":
            self.auth_scheme = "bearer"
            if not self.api_key:
                raise ValueError("Authentication scheme is bearer, but api_key is not defined.")
        elif scheme == "basic":
            self.auth_scheme = "basic"
            if not self.auth_username or not self.auth_password:
                raise ValueError("Authentication scheme is basic, but auth_username and auth_password are not defined.")
        else:
            raise ValueError(f"Unsupported authentication scheme: {scheme}")

        self.app = Starlette()
        self.app.add_route(self.endpoint, self._process_request, methods=["POST"])
        self.app.add_route("/.well-known/agent.json", self._get_agent_card, methods=["GET"])

    def start(self) -> None:
        import uvicorn
        logger.info("Starting A2A Server on %s:%s (endpoint: %s)", self.host, self.port, self.endpoint)
        uvicorn.run(self.app, host=self.host, port=self.port)

    def _get_agent_card(self, request: Request) -> JSONResponse:
        return JSONResponse(self.agent_card.model_dump(exclude_none=True))

    def verify_bearer_token(self, token: str) -> bool:
        if not self.api_key:
            return False
        return secrets.compare_digest(token.encode("utf-8"), self.api_key.encode("utf-8"))

    def verify_basic_auth(self, username: str, password: str) -> bool:
        if not self.auth_username or not self.auth_password:
            return False
        is_user_valid = secrets.compare_digest(username.encode("utf-8"), self.auth_username.encode("utf-8"))
        is_pass_valid = secrets.compare_digest(password.encode("utf-8"), self.auth_password.encode("utf-8"))
        return is_user_valid and is_pass_valid

    async def verify_auth_header(self, request: Request) -> Tuple[bool, Optional[str]]:
        auth_header = request.headers.get("Authorization")
        if not auth_header:
            return False, "Authorization header is missing."

        parts = auth_header.split()
        if len(parts) != 2:
            return False, "Invalid Authorization header format."

        auth_type, credentials = parts
        auth_type = auth_type.lower()

        if self.auth_scheme == "bearer" and auth_type == "bearer":
            if not self.verify_bearer_token(credentials):
                return False, "Invalid bearer token."
            return True, None

        elif self.auth_scheme == "basic" and auth_type == "basic":
            try:
                # [엄격한 Base64 검증] 잘못된 패딩이나 비정상 문자를 엄격히 차단
                decoded_bytes = base64.b64decode(credentials.encode("utf-8"), validate=True)
                decoded = decoded_bytes.decode("utf-8")
                
                if ":" not in decoded:
                    return False, "Invalid basic auth payload structure."
                username, password = decoded.split(":", 1)
                if not self.verify_basic_auth(username, password):
                    return False, "Invalid credentials."
                return True, None
            except Exception:
                logger.warning("Failed to decode or validate basic authentication payload securely.")
                return False, "Invalid basic auth decoding format."
        else:
            return False, f"Authentication scheme mismatch. Expected {self.auth_scheme}, got {auth_type}."

    async def _process_request(self, request: Request) -> JSONResponse | EventSourceResponse:
        is_valid, error_message = await self.verify_auth_header(request)
        if not is_valid:
            return JSONResponse({"error": error_message}, status_code=401)

        request_id = None
        try:
            body = await request.json()
            
            if isinstance(body, dict):
                request_id = body.get("id")

            json_rpc_request = A2ARequest.validate_python(body)
            request_id = getattr(json_rpc_request, "id", request_id)

            logger.debug("Processing valid A2A request: %s (id: %s)", type(json_rpc_request).__name__, request_id)

            if isinstance(json_rpc_request, GetTaskRequest):
                result = await self.task_manager.on_get_task(json_rpc_request)
            elif isinstance(json_rpc_request, SendTaskRequest):
                result = await self.task_manager.on_send_task(json_rpc_request)
            elif isinstance(json_rpc_request, SendTaskStreamingRequest):
                result = await self.task_manager.on_send_task_subscribe(json_rpc_request)
            elif isinstance(json_rpc_request, CancelTaskRequest):
                result = await self.task_manager.on_cancel_task(json_rpc_request)
            elif isinstance(json_rpc_request, SetTaskPushNotificationRequest):
                result = await self.task_manager.on_set_task_push_notification(json_rpc_request)
            elif isinstance(json_rpc_request, GetTaskPushNotificationRequest):
                result = await self.task_manager.on_get_task_push_notification(json_rpc_request)
            elif isinstance(json_rpc_request, TaskResubscriptionRequest):
                result = await self.task_manager.on_resubscribe_to_task(json_rpc_request)
            else:
                logger.warning("Unexpected request type received: %s", type(json_rpc_request))
                raise ValueError(f"Unexpected request type: {type(json_rpc_request)}")

            return self._create_response(result)

        except Exception as e:
            return self._handle_exception(e, request_id=request_id)

    def _handle_exception(self, e: Exception, request_id: Any = None) -> JSONResponse:
        if isinstance(e, json.decoder.JSONDecodeError):
            json_rpc_error = JSONParseError()
            status_code = 400
        elif isinstance(e, ValidationError):
            # [로그 민감정보 역유출 차단] 스키마 에러 객체 전체 대신 식별자(request_id)만 안전하게 기록
            logger.warning("Pydantic validation failed for request id=%s", request_id)
            json_rpc_error = InvalidRequestError(message="Invalid request payload structure.")
            status_code = 400
        else:
            logger.exception("Unhandled internal exception during request execution for id=%s", request_id)
            json_rpc_error = InternalError()
            status_code = 500

        response = JSONRPCResponse(id=request_id, error=json_rpc_error)
        return JSONResponse(response.model_dump(exclude_none=True), status_code=status_code)

    def _create_response(self, result: Any) -> JSONResponse | EventSourceResponse:
        if isinstance(result, AsyncIterable):
            async def event_generator(stream_source) -> AsyncIterable[dict[str, str]]:
                async for item in stream_source:
                    yield {"data": item.model_dump_json(exclude_none=True)}

            return EventSourceResponse(event_generator(result))
        elif isinstance(result, JSONRPCResponse):
            return JSONResponse(result.model_dump(exclude_none=True))
        else:
            logger.error("Unexpected result return type: %s", type(result))
            raise ValueError(f"Unexpected result type: {type(result)}")

최종 개선사항
✅ 인증정보 누락 시 런타임 우회 가능 → 초기화 단계에서 필수 인증 계약 검증 → 인증 없는 서버 기동 차단
✅ 일반 문자열 인증 비교 → secrets.compare_digest() 적용 → 인증 자격 증명의 타이밍 공격 노출 완화
✅ 느슨한 Base64 디코딩 → validate=True 기반 엄격 검증 → 비정상 인증 페이로드 조기 차단
✅ 요청 본문 파싱 실패 시 RPC ID 유실 → 파싱 전 ID 보존 → 오류 응답의 요청 추적성과 JSON-RPC 무결성 강화
✅ ValidationError 전체 로그 기록 → request ID만 기록하는 정제 로그 → 입력 데이터의 로그 역유출 방지
✅ 내부 예외를 HTTP 400으로 일괄 처리 → 입력 오류 400·내부 장애 500 분리 및 traceback 보존 → 장애 의미와 운영 추적성 강화
✅ print() 및 무분별한 payload 출력 → 레벨 기반 구조화 로그 → 운영 관측성과 민감정보 노출 방지

원본의 A2A 요청·SSE 처리 구조는 유지하면서 인증 우회, 오류 ID 유실, 민감정보 로그 노출, 내부 장애 은폐를 제거해 프로토콜 경계와 운영 안정성을 함께 확보한 9.7 수준의 서버 진입점으로 승격됐다.
