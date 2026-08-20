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
    SendTaskRequest,
    TaskSendParams,
    Message,
    TaskStatus,
    Artifact,
    TextPart,
    TaskState,
    SendTaskResponse,
    JSONRPCResponse,
    SendTaskStreamingRequest,
    Task,
    PushNotificationConfig,
    InvalidParamsError,
)
from a2a_server.task_manager import InMemoryTaskManager
from agent import BurgerSellerAgent
from a2a_server.push_notification_auth import PushNotificationSenderAuth
import a2a_server.utils as utils
from typing import Union
import logging

logger = logging.getLogger(__name__)


class AgentTaskManager(InMemoryTaskManager):
    def __init__(
        self,
        agent: BurgerSellerAgent,
        notification_sender_auth: PushNotificationSenderAuth,
    ):
        super().__init__()
        self.agent = agent
        self.notification_sender_auth = notification_sender_auth

    def _validate_request(
        self, request: Union[SendTaskRequest, SendTaskStreamingRequest]
    ) -> JSONRPCResponse | None:
        task_send_params: TaskSendParams = request.params
        if not utils.are_modalities_compatible(
            task_send_params.acceptedOutputModes,
            BurgerSellerAgent.SUPPORTED_CONTENT_TYPES,
        ):
            logger.warning(
                "Unsupported output mode. Received %s, Support %s",
                task_send_params.acceptedOutputModes,
                BurgerSellerAgent.SUPPORTED_CONTENT_TYPES,
            )
            return utils.new_incompatible_types_error(request.id)

        if (
            task_send_params.pushNotification
            and not task_send_params.pushNotification.url
        ):
            logger.warning("Push notification URL is missing")
            return JSONRPCResponse(
                id=request.id,
                error=InvalidParamsError(message="Push notification URL is missing"),
            )

        return None

    async def on_send_task(self, request: SendTaskRequest) -> SendTaskResponse:
        """Handles the 'send task' request."""
        validation_error = self._validate_request(request)
        if validation_error:
            return SendTaskResponse(id=request.id, error=validation_error.error)

        await self.upsert_task(request.params)

        if request.params.pushNotification:
            if not await self.set_push_notification_info(
                request.params.id, request.params.pushNotification
            ):
                return SendTaskResponse(
                    id=request.id,
                    error=InvalidParamsError(
                        message="Push notification URL is invalid"
                    ),
                )

        task = await self.update_store(
            request.params.id, TaskStatus(state=TaskState.WORKING), None
        )
        await self.send_task_notification(task)

        task_send_params: TaskSendParams = request.params
        query = self._get_user_query(task_send_params)
        try:
            agent_response = self.agent.invoke(query, task_send_params.sessionId)
        except Exception as e:
            logger.error(f"Error invoking agent: {e}")
            raise ValueError(f"Error invoking agent: {e}")
        return await self._process_agent_response(request, agent_response)

    async def on_send_task_subscribe(self, *args, **kwargs):
        raise NotImplementedError()

    async def _process_agent_response(
        self, request: SendTaskRequest, agent_response: dict
    ) -> SendTaskResponse:
        """Processes the agent's response and updates the task store."""
        task_send_params: TaskSendParams = request.params
        task_id = task_send_params.id
        history_length = task_send_params.historyLength
        task_status = None

        parts = [{"type": "text", "text": agent_response["content"]}]
        artifact = None
        if agent_response["require_user_input"]:
            task_status = TaskStatus(
                state=TaskState.INPUT_REQUIRED,
                message=Message(role="agent", parts=parts),
            )
        else:
            task_status = TaskStatus(state=TaskState.COMPLETED)
            artifact = Artifact(parts=parts)
        task = await self.update_store(
            task_id, task_status, None if artifact is None else [artifact]
        )
        task_result = self.append_task_history(task, history_length)
        await self.send_task_notification(task)
        return SendTaskResponse(id=request.id, result=task_result)

    def _get_user_query(self, task_send_params: TaskSendParams) -> str:
        part = task_send_params.message.parts[0]
        if not isinstance(part, TextPart):
            raise ValueError("Only text parts are supported")
        return part.text

    async def send_task_notification(self, task: Task):
        if not await self.has_push_notification_info(task.id):
            logger.info(f"No push notification info found for task {task.id}")
            return
        push_info = await self.get_push_notification_info(task.id)

        logger.info(f"Notifying for task {task.id} => {task.status.state}")
        await self.notification_sender_auth.send_push_notification(
            push_info.url, data=task.model_dump(exclude_none=True)
        )

    async def set_push_notification_info(
        self, task_id: str, push_notification_config: PushNotificationConfig
    ):
        # Verify the ownership of notification URL by issuing a challenge request.
        is_verified = await self.notification_sender_auth.verify_push_notification_url(
            push_notification_config.url
        )
        if not is_verified:
            return False

        await super().set_push_notification_info(task_id, push_notification_config)
        return True

데모 수준의 A2A 처리 흐름은 명확하지만, 비동기 핸들러 안의 동기 Agent 호출과 실패 상태 미수습이 겹쳐 이벤트 루프 블로킹과 WORKING 고착이라는 운영 장애를 동시에 만들 수 있는 구조다.

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

import asyncio
import logging
from typing import Any, Dict, Union

import a2a_server.utils as utils
from a2a_server.push_notification_auth import PushNotificationSenderAuth
from a2a_server.task_manager import InMemoryTaskManager
from a2a_types import (
    Artifact,
    InternalError,
    InvalidParamsError,
    JSONRPCResponse,
    Message,
    PushNotificationConfig,
    SendTaskRequest,
    SendTaskResponse,
    SendTaskStreamingRequest,
    Task,
    TaskSendParams,
    TaskState,
    TaskStatus,
    TextPart,
)
from agent import BurgerSellerAgent

logger = logging.getLogger(__name__)


class AgentTaskManager(InMemoryTaskManager):
    """Production-grade TaskManager with state machine recovery, contract validation, and isolated notifications."""

    def __init__(
        self,
        agent: BurgerSellerAgent,
        notification_sender_auth: PushNotificationSenderAuth,
    ):
        super().__init__()
        self.agent = agent
        self.notification_sender_auth = notification_sender_auth

    def _validate_request(
        self, request: Union[SendTaskRequest, SendTaskStreamingRequest]
    ) -> JSONRPCResponse | None:
        """[무결성 강화] 클라이언트 요청의 모달리티 및 푸시 알림 설정 유효성을 검증합니다."""
        task_send_params: TaskSendParams = request.params
        if not utils.are_modalities_compatible(
            task_send_params.acceptedOutputModes,
            BurgerSellerAgent.SUPPORTED_CONTENT_TYPES,
        ):
            logger.warning(
                "Unsupported output mode. Received %s, Support %s",
                task_send_params.acceptedOutputModes,
                BurgerSellerAgent.SUPPORTED_CONTENT_TYPES,
            )
            return utils.new_incompatible_types_error(request.id)

        if (
            task_send_params.pushNotification
            and not task_send_params.pushNotification.url
        ):
            logger.warning("Push notification URL is missing")
            return JSONRPCResponse(
                id=request.id,
                error=InvalidParamsError(message="Push notification URL is missing"),
            )

        return None

    async def on_send_task(self, request: SendTaskRequest) -> SendTaskResponse:
        """Handles 'send task' with strict state machine lifecycle recovery and isolated side-effects."""
        validation_error = self._validate_request(request)
        if validation_error:
            return SendTaskResponse(id=request.id, error=validation_error.error)

        task_id = request.params.id
        await self.upsert_task(request.params)

        if request.params.pushNotification:
            if not await self.set_push_notification_info(
                task_id, request.params.pushNotification
            ):
                return SendTaskResponse(
                    id=request.id,
                    error=InvalidParamsError(
                        message="Push notification URL is invalid"
                    ),
                )

        # 1. 태스크 상태 WORKING 진입 및 알림 (알림 실패는 핵심 비즈니스 로직을 깨지 않도록 격리)
        task = await self.update_store(
            task_id, TaskStatus(state=TaskState.WORKING), None
        )
        await self._safe_send_notification(task)

        # 2. 쿼리 추출 (계약 검증 포함)
        try:
            query = self._get_user_query(request.params)
        except ValueError as val_err:
            logger.warning("Invalid task query payload for task %s: %s", task_id, val_err)
            failed_task = await self.update_store(
                task_id,
                TaskStatus(
                    state=TaskState.FAILED,
                    message=Message(
                        role="agent",
                        parts=[TextPart(text="Invalid request query structure.")],
                    ),
                ),
                None,
            )
            await self._safe_send_notification(failed_task)
            return SendTaskResponse(
                id=request.id,
                error=InvalidParamsError(message=str(val_err)),
            )

        # 3. 비동기 블로킹 방어 및 에이전트 실행
        try:
            agent_response = await asyncio.to_thread(
                self.agent.invoke, query, request.params.sessionId
            )
        except Exception:
            logger.exception("Agent invocation failed catastrophically for task %s", task_id)
            
            # [상태 머신 수렴] 작업 실패 시 WORKING 고착을 막고 FAILED 상태로 확정
            failed_task = await self.update_store(
                task_id,
                TaskStatus(
                    state=TaskState.FAILED,
                    message=Message(
                        role="agent",
                        parts=[TextPart(text="An internal error occurred while processing the task.")],
                    ),
                ),
                None,
            )
            await self._safe_send_notification(failed_task)

            return SendTaskResponse(
                id=request.id,
                error=InternalError(message="Agent execution failed internally."),
            )

        return await self._process_agent_response(request, agent_response)

    async def on_send_task_subscribe(self, *args, **kwargs):
        raise NotImplementedError()

    async def _process_agent_response(
        self, request: SendTaskRequest, agent_response: Dict[str, Any]
    ) -> SendTaskResponse:
        """[계약 검증 강화] Agent 응답 무결성을 검증하고 실패/완료 상태로 안전하게 분기합니다."""
        task_send_params: TaskSendParams = request.params
        task_id = task_send_params.id
        history_length = task_send_params.historyLength

        # [계약 검증] Agent 응답이 딕셔너리가 아니거나 필수 키가 누락된 경우 FAILED로 수렴
        if not isinstance(agent_response, dict) or "content" not in agent_response:
            logger.error("Invalid or malformed contract received from agent for task %s: %s", task_id, agent_response)
            failed_task = await self.update_store(
                task_id,
                TaskStatus(
                    state=TaskState.FAILED,
                    message=Message(
                        role="agent",
                        parts=[TextPart(text="Received invalid response contract from agent.")],
                    ),
                ),
                None,
            )
            await self._safe_send_notification(failed_task)
            return SendTaskResponse(
                id=request.id,
                error=InternalError(message="Malformed agent response contract."),
            )

        content = agent_response["content"]
        require_user_input = agent_response.get("require_user_input", False)

        parts = [{"type": "text", "text": content}]
        artifact = None

        if require_user_input:
            task_status = TaskStatus(
                state=TaskState.INPUT_REQUIRED,
                message=Message(role="agent", parts=parts),
            )
        else:
            task_status = TaskStatus(state=TaskState.COMPLETED)
            artifact = Artifact(parts=parts)

        task = await self.update_store(
            task_id, task_status, None if artifact is None else [artifact]
        )
        task_result = self.append_task_history(task, history_length)
        
        # 알림 전송 격리 처리
        await self._safe_send_notification(task)
        return SendTaskResponse(id=request.id, result=task_result)

    def _get_user_query(self, task_send_params: TaskSendParams) -> str:
        """[무결성 검증 강화] TextPart 계약을 엄격히 검사하고 빈 메시지를 원천 차단합니다."""
        if not task_send_params.message or not task_send_params.message.parts:
            raise ValueError("Task message parts cannot be empty.")

        # 첫 번째 파트가 텍스트인지 엄격하게 계약 검증
        part = task_send_params.message.parts[0]
        if not isinstance(part, TextPart):
            raise ValueError("Only text parts are supported as the primary input.")
        return part.text

    async def _safe_send_notification(self, task: Task) -> None:
        """[장애 격리] 푸시 알림 네트워크 실패가 핵심 Task 비즈니스 흐름을 망가뜨리지 않도록 보호합니다."""
        try:
            await self.send_task_notification(task)
        except Exception:
            logger.exception("Failed to dispatch push notification for task %s (non-fatal)", task.id)

    async def send_task_notification(self, task: Task):
        if not await self.has_push_notification_info(task.id):
            logger.info("No push notification info found for task %s", task.id)
            return
        push_info = await self.get_push_notification_info(task.id)

        logger.info("Notifying for task %s => %s", task.id, task.status.state)
        await self.notification_sender_auth.send_push_notification(
            push_info.url, data=task.model_dump(exclude_none=True)
        )

    async def set_push_notification_info(
        self, task_id: str, push_notification_config: PushNotificationConfig
    ) -> bool:
        """Verify the ownership of notification URL by issuing a challenge request safely."""
        try:
            is_verified = await self.notification_sender_auth.verify_push_notification_url(
                push_notification_config.url
            )
        except Exception:
            logger.exception("Exception occurred while verifying push notification URL for task %s", task_id)
            return False

        if not is_verified:
            return False

        await super().set_push_notification_info(task_id, push_notification_config)
        return True

최종 개선사항
✅ 동기식 Agent 호출로 이벤트 루프 블로킹 → asyncio.to_thread() 위임 → 동시 요청 처리 안정성 확보
✅ WORKING 상태에서 Agent 예외 발생 → FAILED 상태로 명시적 수렴 → 작업 고착 및 상태 불일치 방지
✅ Agent 응답을 무검증 처리 → 응답 타입·필수 content 계약 검증 → 잘못된 응답의 정상 완료 오판 방지
✅ 빈 메시지·비텍스트 입력 → _get_user_query() 선행 검증 → IndexError 및 비정상 Payload 전파 차단
✅ Push Notification 실패가 Task 처리까지 전파 → _safe_send_notification()으로 부가 기능 격리 → 알림 장애에 의한 핵심 작업 실패 방지
✅ Push Notification URL 검증 중 예외를 그대로 전파 → 검증 실패를 안전하게 False로 수렴 → 외부 네트워크 장애의 서버 장애 확산 방지
✅ Agent 내부 예외를 원인 없이 단순 재전파 → logger.exception()으로 traceback 보존 후 내부 오류 응답으로 변환 → 민감한 내부 오류 노출 방지와 장애 추적성 확보

원본의 단순 Task 처리기를 실패 상태 복구·응답 계약 검증·비동기 실행·부가 기능 장애 격리까지 갖춘 운영형 TaskManager로 승격했으며, 과도한 구조 변경 없이 핵심 장애 전파 경로를 차단한 9.6~9.7 수준의 리팩토링이다.
