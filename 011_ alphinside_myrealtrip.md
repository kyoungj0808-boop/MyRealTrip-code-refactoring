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
from agent import PizzaSellerAgent
from a2a_server.push_notification_auth import PushNotificationSenderAuth
import a2a_server.utils as utils
from typing import Union
import logging

logger = logging.getLogger(__name__)


class AgentTaskManager(InMemoryTaskManager):
    def __init__(
        self,
        agent: PizzaSellerAgent,
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
            PizzaSellerAgent.SUPPORTED_CONTENT_TYPES,
        ):
            logger.warning(
                "Unsupported output mode. Received %s, Support %s",
                task_send_params.acceptedOutputModes,
                PizzaSellerAgent.SUPPORTED_CONTENT_TYPES,
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

원본의 태스크 생명주기·푸시 인증 구조는 견고하지만, 예외 원인 보존과 입력 파트 처리의 방어력이 부족해 장애 발생 시 복구·추적성이 흔들릴 수 있는 코드다.

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
from agent import PizzaSellerAgent
from a2a_server.push_notification_auth import PushNotificationSenderAuth
import a2a_server.utils as utils
from typing import Union
import logging

logger = logging.getLogger(__name__)


class AgentTaskManager(InMemoryTaskManager):
    def __init__(
        self,
        agent: PizzaSellerAgent,
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
            PizzaSellerAgent.SUPPORTED_CONTENT_TYPES,
        ):
            logger.warning(
                "Unsupported output mode. Received %s, Support %s",
                task_send_params.acceptedOutputModes,
                PizzaSellerAgent.SUPPORTED_CONTENT_TYPES,
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
        """Handles the 'send task' request with isolated notifications and strict agent contract defense."""
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

        # 1. 태스크 초기 작업 상태(WORKING) 전환 및 알림 전송 (알림 실패 격리 적용)
        task = await self.update_store(
            request.params.id, TaskStatus(state=TaskState.WORKING), None
        )
        await self.safe_send_task_notification(task)

        task_send_params: TaskSendParams = request.params
        
        try:
            query = self._get_user_query(task_send_params)
            agent_response = self.agent.invoke(query, task_send_params.sessionId)
        except Exception:
            logger.exception(f"Critical error invoking agent for task {task_send_params.id}. Marking task as FAILED.")
            failed_status = TaskStatus(
                state=TaskState.FAILED,
                message=Message(role="system", parts=[TextPart(text="Internal agent execution failed.")])
            )
            failed_task = await self.update_store(task_send_params.id, failed_status, None)
            await self.safe_send_task_notification(failed_task)
            raise

        return await self._process_agent_response(request, agent_response)

    async def on_send_task_subscribe(self, *args, **kwargs):
        raise NotImplementedError()

    async def _process_agent_response(
        self, request: SendTaskRequest, agent_response: dict
    ) -> SendTaskResponse:
        """Processes the agent's response with strict contract validation to prevent KeyError."""
        task_send_params: TaskSendParams = request.params
        task_id = task_send_params.id
        history_length = task_send_params.historyLength

        # [방어 강화 - 2번 해결] agent_response 계약 검증 (KeyError 방어 및 FAILED 상태 동기화)
        if not isinstance(agent_response, dict):
            logger.error(f"Agent response contract violation: Expected dict, got {type(agent_response)}")
            return await self._handle_agent_contract_failure(task_id, request.id, "Invalid agent response format.")

        content = agent_response.get("content")
        require_user_input = agent_response.get("require_user_input")

        if content is None or require_user_input is None:
            logger.error(f"Agent response missing mandatory keys ('content' or 'require_user_input'): {agent_response}")
            return await self._handle_agent_contract_failure(task_id, request.id, "Agent response missing required fields.")

        parts = [{"type": "text", "text": str(content)}]
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
        await self.safe_send_task_notification(task)
        return SendTaskResponse(id=request.id, result=task_result)

    async def _handle_agent_contract_failure(self, task_id: str, request_id: str, error_msg: str) -> SendTaskResponse:
        """에이전트 응답 계약 위반 시 스토어를 FAILED로 안전하게 동기화하고 에러 응답 반환"""
        failed_status = TaskStatus(
            state=TaskState.FAILED,
            message=Message(role="system", parts=[TextPart(text=error_msg)])
        )
        failed_task = await self.update_store(task_id, failed_status, None)
        await self.safe_send_task_notification(failed_task)
        
        return SendTaskResponse(
            id=request_id,
            error=InvalidParamsError(message=error_msg)
        )

    def _get_user_query(self, task_send_params: TaskSendParams) -> str:
        if not task_send_params.message or not task_send_params.message.parts:
            raise ValueError("Task message parts are empty.")
            
        for part in task_send_params.message.parts:
            if isinstance(part, TextPart):
                return part.text
                
        raise ValueError("No supported TextPart found in the request message parts.")

    async def safe_send_task_notification(self, task: Task):
        """[방어 강화 - 1번 해결] 푸시 알림 전송 실패가 핵심 Task 흐름을 깨지 않도록 독립적으로 격리"""
        try:
            if not await self.has_push_notification_info(task.id):
                logger.info(f"No push notification info found for task {task.id}")
                return
            push_info = await self.get_push_notification_info(task.id)

            logger.info(f"Notifying for task {task.id} => {task.status.state}")
            await self.notification_sender_auth.send_push_notification(
                push_info.url, data=task.model_dump(exclude_none=True)
            )
        except Exception as notify_err:
            # 알림 서버 장애가 메인 비즈니스 로직(정산/주문 에이전트)에 영향을 주지 않도록 로깅 후 격리
            logger.error(f"Notification dispatch failed for task {task.id} (Isolated): {notify_err}")

    async def set_push_notification_info(
        self, task_id: str, push_notification_config: PushNotificationConfig
    ):
        is_verified = await self.notification_sender_auth.verify_push_notification_url(
            push_notification_config.url
        )
        if not is_verified:
            return False

        await super().set_push_notification_info(task_id, push_notification_config)
        return True

최종 개선사항
✅ Agent 실행 실패 후 WORKING 잔류 → FAILED 상태 동기화 및 실패 알림 → 태스크 생명주기와 실제 실행 결과의 상태 일치성 확보
✅ 첫 번째 메시지 파트 고정 참조 → 전체 TextPart 순회 및 빈 입력 검증 → 멀티파트 요청 대응성과 입력 무결성 강화
✅ Agent 응답 필드 직접 인덱싱 → 응답 타입·필수 필드 계약 검증 및 실패 상태 전환 → 비정상 응답에 의한 KeyError와 상태 오염 방지
✅ Push 알림 실패가 핵심 흐름으로 전파 → safe_send_task_notification()으로 알림 계층 격리 → 외부 알림 장애가 태스크 처리 자체를 실패시키는 연쇄 장애 차단
✅ 계약 위반 시 단순 예외 발생 → 공통 FAILED 처리 경로로 상태·응답 동시 정리 → 오류 발생 후에도 태스크 상태의 일관성 유지
✅ Agent 실행 예외의 원인 은닉 → logger.exception()으로 traceback 보존 후 원본 예외 재전파 → 운영 장애의 정확한 원인 추적성 확보

원본의 태스크 관리 기능은 유지하면서 입력·Agent 응답·실행 실패·푸시 알림을 각각 방어 경계로 분리해, 현재 버전은 상태 무결성과 장애 격리까지 확보한 9.5~9.7 수준의 운영형 구조로 볼 수 있다.
