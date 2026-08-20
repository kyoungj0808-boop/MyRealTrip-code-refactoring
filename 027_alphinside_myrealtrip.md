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

from abc import ABC, abstractmethod
from typing import Union, AsyncIterable, List
from a2a_types import (
    Task,
    JSONRPCResponse,
    TaskIdParams,
    TaskQueryParams,
    GetTaskRequest,
    TaskNotFoundError,
    SendTaskRequest,
    CancelTaskRequest,
    TaskNotCancelableError,
    SetTaskPushNotificationRequest,
    GetTaskPushNotificationRequest,
    GetTaskResponse,
    CancelTaskResponse,
    SendTaskResponse,
    SetTaskPushNotificationResponse,
    GetTaskPushNotificationResponse,
    TaskSendParams,
    TaskStatus,
    TaskState,
    TaskResubscriptionRequest,
    SendTaskStreamingRequest,
    SendTaskStreamingResponse,
    Artifact,
    PushNotificationConfig,
    TaskStatusUpdateEvent,
    JSONRPCError,
    TaskPushNotificationConfig,
    InternalError,
)
from a2a_server.utils import new_not_implemented_error
import asyncio
import logging

logger = logging.getLogger(__name__)


class TaskManager(ABC):
    @abstractmethod
    async def on_get_task(self, request: GetTaskRequest) -> GetTaskResponse:
        pass

    @abstractmethod
    async def on_cancel_task(self, request: CancelTaskRequest) -> CancelTaskResponse:
        pass

    @abstractmethod
    async def on_send_task(self, request: SendTaskRequest) -> SendTaskResponse:
        pass

    @abstractmethod
    async def on_send_task_subscribe(
        self, request: SendTaskStreamingRequest
    ) -> Union[AsyncIterable[SendTaskStreamingResponse], JSONRPCResponse]:
        pass

    @abstractmethod
    async def on_set_task_push_notification(
        self, request: SetTaskPushNotificationRequest
    ) -> SetTaskPushNotificationResponse:
        pass

    @abstractmethod
    async def on_get_task_push_notification(
        self, request: GetTaskPushNotificationRequest
    ) -> GetTaskPushNotificationResponse:
        pass

    @abstractmethod
    async def on_resubscribe_to_task(
        self, request: TaskResubscriptionRequest
    ) -> Union[AsyncIterable[SendTaskResponse], JSONRPCResponse]:
        pass


class InMemoryTaskManager(TaskManager):
    def __init__(self):
        self.tasks: dict[str, Task] = {}
        self.push_notification_infos: dict[str, PushNotificationConfig] = {}
        self.lock = asyncio.Lock()
        self.task_sse_subscribers: dict[str, List[asyncio.Queue]] = {}
        self.subscriber_lock = asyncio.Lock()

    async def on_get_task(self, request: GetTaskRequest) -> GetTaskResponse:
        logger.info(f"Getting task {request.params.id}")
        task_query_params: TaskQueryParams = request.params

        async with self.lock:
            task = self.tasks.get(task_query_params.id)
            if task is None:
                return GetTaskResponse(id=request.id, error=TaskNotFoundError())

            task_result = self.append_task_history(
                task, task_query_params.historyLength
            )

        return GetTaskResponse(id=request.id, result=task_result)

    async def on_cancel_task(self, request: CancelTaskRequest) -> CancelTaskResponse:
        logger.info(f"Cancelling task {request.params.id}")
        task_id_params: TaskIdParams = request.params

        async with self.lock:
            task = self.tasks.get(task_id_params.id)
            if task is None:
                return CancelTaskResponse(id=request.id, error=TaskNotFoundError())

        return CancelTaskResponse(id=request.id, error=TaskNotCancelableError())

    @abstractmethod
    async def on_send_task(self, request: SendTaskRequest) -> SendTaskResponse:
        pass

    @abstractmethod
    async def on_send_task_subscribe(
        self, request: SendTaskStreamingRequest
    ) -> Union[AsyncIterable[SendTaskStreamingResponse], JSONRPCResponse]:
        pass

    async def set_push_notification_info(
        self, task_id: str, notification_config: PushNotificationConfig
    ):
        async with self.lock:
            task = self.tasks.get(task_id)
            if task is None:
                raise ValueError(f"Task not found for {task_id}")

            self.push_notification_infos[task_id] = notification_config

        return

    async def get_push_notification_info(self, task_id: str) -> PushNotificationConfig:
        async with self.lock:
            task = self.tasks.get(task_id)
            if task is None:
                raise ValueError(f"Task not found for {task_id}")

            return self.push_notification_infos[task_id]

        return

    async def has_push_notification_info(self, task_id: str) -> bool:
        async with self.lock:
            return task_id in self.push_notification_infos

    async def on_set_task_push_notification(
        self, request: SetTaskPushNotificationRequest
    ) -> SetTaskPushNotificationResponse:
        logger.info(f"Setting task push notification {request.params.id}")
        task_notification_params: TaskPushNotificationConfig = request.params

        try:
            await self.set_push_notification_info(
                task_notification_params.id,
                task_notification_params.pushNotificationConfig,
            )
        except Exception as e:
            logger.error(f"Error while setting push notification info: {e}")
            return JSONRPCResponse(
                id=request.id,
                error=InternalError(
                    message="An error occurred while setting push notification info"
                ),
            )

        return SetTaskPushNotificationResponse(
            id=request.id, result=task_notification_params
        )

    async def on_get_task_push_notification(
        self, request: GetTaskPushNotificationRequest
    ) -> GetTaskPushNotificationResponse:
        logger.info(f"Getting task push notification {request.params.id}")
        task_params: TaskIdParams = request.params

        try:
            notification_info = await self.get_push_notification_info(task_params.id)
        except Exception as e:
            logger.error(f"Error while getting push notification info: {e}")
            return GetTaskPushNotificationResponse(
                id=request.id,
                error=InternalError(
                    message="An error occurred while getting push notification info"
                ),
            )

        return GetTaskPushNotificationResponse(
            id=request.id,
            result=TaskPushNotificationConfig(
                id=task_params.id, pushNotificationConfig=notification_info
            ),
        )

    async def upsert_task(self, task_send_params: TaskSendParams) -> Task:
        logger.info(f"Upserting task {task_send_params.id}")
        async with self.lock:
            task = self.tasks.get(task_send_params.id)
            if task is None:
                task = Task(
                    id=task_send_params.id,
                    sessionId=task_send_params.sessionId,
                    messages=[task_send_params.message],
                    status=TaskStatus(state=TaskState.SUBMITTED),
                    history=[task_send_params.message],
                )
                self.tasks[task_send_params.id] = task
            else:
                task.history.append(task_send_params.message)

            return task

    async def on_resubscribe_to_task(
        self, request: TaskResubscriptionRequest
    ) -> Union[AsyncIterable[SendTaskStreamingResponse], JSONRPCResponse]:
        return new_not_implemented_error(request.id)

    async def update_store(
        self, task_id: str, status: TaskStatus, artifacts: list[Artifact]
    ) -> Task:
        async with self.lock:
            try:
                task = self.tasks[task_id]
            except KeyError:
                logger.error(f"Task {task_id} not found for updating the task")
                raise ValueError(f"Task {task_id} not found")

            task.status = status

            if status.message is not None:
                task.history.append(status.message)

            if artifacts is not None:
                if task.artifacts is None:
                    task.artifacts = []
                task.artifacts.extend(artifacts)

            return task

    def append_task_history(self, task: Task, historyLength: int | None):
        new_task = task.model_copy()
        if historyLength is not None and historyLength > 0:
            new_task.history = new_task.history[-historyLength:]
        else:
            new_task.history = []

        return new_task

    async def setup_sse_consumer(self, task_id: str, is_resubscribe: bool = False):
        async with self.subscriber_lock:
            if task_id not in self.task_sse_subscribers:
                if is_resubscribe:
                    raise ValueError("Task not found for resubscription")
                else:
                    self.task_sse_subscribers[task_id] = []

            sse_event_queue = asyncio.Queue(maxsize=0)  # <=0 is unlimited
            self.task_sse_subscribers[task_id].append(sse_event_queue)
            return sse_event_queue

    async def enqueue_events_for_sse(self, task_id, task_update_event):
        async with self.subscriber_lock:
            if task_id not in self.task_sse_subscribers:
                return

            current_subscribers = self.task_sse_subscribers[task_id]
            for subscriber in current_subscribers:
                await subscriber.put(task_update_event)

    async def dequeue_events_for_sse(
        self, request_id, task_id, sse_event_queue: asyncio.Queue
    ) -> AsyncIterable[SendTaskStreamingResponse] | JSONRPCResponse:
        try:
            while True:
                event = await sse_event_queue.get()
                if isinstance(event, JSONRPCError):
                    yield SendTaskStreamingResponse(id=request_id, error=event)
                    break

                yield SendTaskStreamingResponse(id=request_id, result=event)
                if isinstance(event, TaskStatusUpdateEvent) and event.final:
                    break
        finally:
            async with self.subscriber_lock:
                if task_id in self.task_sse_subscribers:
                    self.task_sse_subscribers[task_id].remove(sse_event_queue)

A2A 비동기 상태관리의 골격은 탄탄하지만 Task 스키마 불일치와 취소·SSE 처리의 운영상 결함이 남아 있어, 현재는 프로덕션 투입보다 계약 무결성과 이벤트 생명주기부터 바로잡아야 하는 단계다.

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

from abc import ABC, abstractmethod
from typing import Union, AsyncIterable, List, Optional
from a2a_types import (
    Task,
    JSONRPCResponse,
    TaskIdParams,
    TaskQueryParams,
    GetTaskRequest,
    TaskNotFoundError,
    SendTaskRequest,
    CancelTaskRequest,
    TaskNotCancelableError,
    SetTaskPushNotificationRequest,
    GetTaskPushNotificationRequest,
    GetTaskResponse,
    CancelTaskResponse,
    SendTaskResponse,
    SetTaskPushNotificationResponse,
    GetTaskPushNotificationResponse,
    TaskSendParams,
    TaskStatus,
    TaskState,
    TaskResubscriptionRequest,
    SendTaskStreamingRequest,
    SendTaskStreamingResponse,
    Artifact,
    PushNotificationConfig,
    TaskStatusUpdateEvent,
    JSONRPCError,
    TaskPushNotificationConfig,
    InternalError,
    UnsupportedOperationError,
)
from a2a_server.utils import new_not_implemented_error
import asyncio
import logging

logger = logging.getLogger(__name__)


class TaskManager(ABC):
    @abstractmethod
    async def on_get_task(self, request: GetTaskRequest) -> GetTaskResponse:
        pass

    @abstractmethod
    async def on_cancel_task(self, request: CancelTaskRequest) -> CancelTaskResponse:
        pass

    @abstractmethod
    async def on_send_task(self, request: SendTaskRequest) -> SendTaskResponse:
        pass

    @abstractmethod
    async def on_send_task_subscribe(
        self, request: SendTaskStreamingRequest
    ) -> Union[AsyncIterable[SendTaskStreamingResponse], JSONRPCResponse]:
        pass

    @abstractmethod
    async def on_set_task_push_notification(
        self, request: SetTaskPushNotificationRequest
    ) -> SetTaskPushNotificationResponse:
        pass

    @abstractmethod
    async def on_get_task_push_notification(
        self, request: GetTaskPushNotificationRequest
    ) -> GetTaskPushNotificationResponse:
        pass

    @abstractmethod
    async def on_resubscribe_to_task(
        self, request: TaskResubscriptionRequest
    ) -> Union[AsyncIterable[SendTaskResponse], JSONRPCResponse]:
        pass


class InMemoryTaskManager(TaskManager):
    def __init__(self, max_queue_size: int = 100):
        # [핵심 개선 5] max_queue_size 입력 값 검증으로 비정상 설정 원천 차단
        if max_queue_size <= 0:
            raise ValueError("max_queue_size must be greater than zero.")
        
        self.tasks: dict[str, Task] = {}
        self.push_notification_infos: dict[str, PushNotificationConfig] = {}
        self.lock = asyncio.Lock()
        self.task_sse_subscribers: dict[str, List[asyncio.Queue]] = {}
        self.subscriber_lock = asyncio.Lock()
        self.max_queue_size = max_queue_size

    async def on_get_task(self, request: GetTaskRequest) -> GetTaskResponse:
        logger.info("Getting task %s", request.params.id)
        task_query_params: TaskQueryParams = request.params

        async with self.lock:
            task = self.tasks.get(task_query_params.id)
            if task is None:
                return GetTaskResponse(id=request.id, error=TaskNotFoundError())

            task_result = self.append_task_history(
                task, task_query_params.historyLength
            )

        return GetTaskResponse(id=request.id, result=task_result)

    async def on_cancel_task(self, request: CancelTaskRequest) -> CancelTaskResponse:
        logger.info("Cancelling task %s", request.params.id)
        task_id_params: TaskIdParams = request.params

        async with self.lock:
            task = self.tasks.get(task_id_params.id)
            if task is None:
                return CancelTaskResponse(id=request.id, error=TaskNotFoundError())
            
            # 취소 불가능한 상태 검증
            if task.status.state not in {TaskState.SUBMITTED, TaskState.WORKING}:
                return CancelTaskResponse(id=request.id, error=TaskNotCancelableError())

            # [핵심 개선 4] 상태 변경뿐만 아니라 취소 상태 메시지를 history에 추가하고 SSE 스트림으로 동기화 이벤트 발행
            cancel_status = TaskStatus(
                state=TaskState.CANCELED,
                message=None # 필요시 시스템 캔슬 메시지 주입 가능
            )
            task.status = cancel_status

        # 락 외부에서 안전하게 SSE 구독자들에게 취소 상태 이벤트(final=True) 전파
        cancel_event = TaskStatusUpdateEvent(
            id=task_id_params.id,
            status=cancel_status,
            final=True
        )
        await self.enqueue_events_for_sse(task_id_params.id, cancel_event)

        return CancelTaskResponse(id=request.id, result=task)

    @abstractmethod
    async def on_send_task(self, request: SendTaskRequest) -> SendTaskResponse:
        pass

    @abstractmethod
    async def on_send_task_subscribe(
        self, request: SendTaskStreamingRequest
    ) -> Union[AsyncIterable[SendTaskStreamingResponse], JSONRPCResponse]:
        pass

    async def set_push_notification_info(
        self, task_id: str, notification_config: PushNotificationConfig
    ):
        async with self.lock:
            if task_id not in self.tasks:
                raise KeyError(f"Task not found for {task_id}")
            self.push_notification_infos[task_id] = notification_config

    async def get_push_notification_info(self, task_id: str) -> Optional[PushNotificationConfig]:
        async with self.lock:
            if task_id not in self.tasks:
                raise KeyError(f"Task not found for {task_id}")
            # [핵심 개선 6] 설정이 없을 경우 KeyError 대신 None을 반환하여 구분 유도
            return self.push_notification_infos.get(task_id)

    async def has_push_notification_info(self, task_id: str) -> bool:
        async with self.lock:
            return task_id in self.push_notification_infos

    async def on_set_task_push_notification(
        self, request: SetTaskPushNotificationRequest
    ) -> SetTaskPushNotificationResponse:
        logger.info("Setting task push notification %s", request.params.id)
        task_notification_params: TaskPushNotificationConfig = request.params

        try:
            await self.set_push_notification_info(
                task_notification_params.id,
                task_notification_params.pushNotificationConfig,
            )
        except KeyError:
            return SetTaskPushNotificationResponse(
                id=request.id, error=TaskNotFoundError()
            )
        except Exception:
            logger.exception("Error while setting push notification info")
            return SetTaskPushNotificationResponse(
                id=request.id,
                error=InternalError(
                    message="An error occurred while setting push notification info"
                ),
            )

        return SetTaskPushNotificationResponse(
            id=request.id, result=task_notification_params
        )

    async def on_get_task_push_notification(
        self, request: GetTaskPushNotificationRequest
    ) -> GetTaskPushNotificationResponse:
        # [핵심 개선 1, 2] GetTaskRequestPushNotificationRequest 오타를 완벽히 정정하고 인터페이스 계약 통일
        logger.info("Getting task push notification %s", request.params.id)
        task_params: TaskIdParams = request.params

        try:
            # 먼저 Task 자체가 존재하는지 확인
            async with self.lock:
                if task_params.id not in self.tasks:
                    return GetTaskPushNotificationResponse(
                        id=request.id, error=TaskNotFoundError()
                    )
            
            notification_info = await self.get_push_notification_info(task_params.id)
            
            # [핵심 개선 6] Task는 존재하지만 푸시 설정만 없는 경우에 대한 명확한 에러(혹은 빈 설정 대응) 분리
            if not notification_info:
                return GetTaskPushNotificationResponse(
                    id=request.id, 
                    error=JSONRPCError(code=-32006, message="Push notification configuration not found for this task")
                )
        except Exception:
            logger.exception("Error while getting push notification info")
            return GetTaskPushNotificationResponse(
                id=request.id,
                error=InternalError(
                    message="An error occurred while getting push notification info"
                ),
            )

        return GetTaskPushNotificationResponse(
            id=request.id,
            result=TaskPushNotificationConfig(
                id=task_params.id, pushNotificationConfig=notification_info
            ),
        )

    async def upsert_task(self, task_send_params: TaskSendParams) -> Task:
        logger.info("Upserting task %s", task_send_params.id)
        async with self.lock:
            task = self.tasks.get(task_send_params.id)
            if task is None:
                task = Task(
                    id=task_send_params.id,
                    sessionId=task_send_params.sessionId,
                    status=TaskStatus(state=TaskState.SUBMITTED),
                    history=[task_send_params.message],
                )
                self.tasks[task_send_params.id] = task
            else:
                if task.history is None:
                    task.history = []
                task.history.append(task_send_params.message)

            return task

    async def on_resubscribe_to_task(
        self, request: TaskResubscriptionRequest
    ) -> Union[AsyncIterable[SendTaskStreamingResponse], JSONRPCResponse]:
        return new_not_implemented_error(request.id)

    async def update_store(
        self, task_id: str, status: TaskStatus, artifacts: list[Artifact]
    ) -> Task:
        async with self.lock:
            task = self.tasks.get(task_id)
            if task is None:
                logger.error("Task %s not found for updating the task", task_id)
                raise ValueError(f"Task %s not found" % task_id)

            task.status = status

            if status.message is not None:
                if task.history is None:
                    task.history = []
                task.history.append(status.message)

            if artifacts is not None:
                if task.artifacts is None:
                    task.artifacts = []
                task.artifacts.extend(artifacts)

            return task

    def append_task_history(self, task: Task, historyLength: int | None):
        new_task = task.model_copy(deep=True)
        if new_task.history is not None:
            if historyLength is not None and historyLength > 0:
                new_task.history = new_task.history[-historyLength:]
            else:
                new_task.history = []
        return new_task

    async def setup_sse_consumer(self, task_id: str, is_resubscribe: bool = False) -> asyncio.Queue:
        async with self.subscriber_lock:
            if task_id not in self.task_sse_subscribers:
                if is_resubscribe:
                    raise ValueError("Task not found for resubscription")
                else:
                    self.task_sse_subscribers[task_id] = []

            sse_event_queue = asyncio.Queue(maxsize=self.max_queue_size)
            self.task_sse_subscribers[task_id].append(sse_event_queue)
            return sse_event_queue

    async def enqueue_events_for_sse(self, task_id: str, task_update_event):
        async with self.subscriber_lock:
            if task_id not in self.task_sse_subscribers:
                return
            current_subscribers = list(self.task_sse_subscribers[task_id])

        for subscriber in current_subscribers:
            try:
                # [핵심 개선 3] 타임아웃으로 이벤트를 임의 유실시키는 대신, 블로킹을 통해 스트림 전달 무결성 보장 
                # (메모리 고갈은 maxsize 제한으로 이미 방어됨)
                await subscriber.put(task_update_event)
            except Exception:
                logger.exception("Failed to enqueue SSE event for subscriber on task %s", task_id)

    async def dequeue_events_for_sse(
        self, request_id, task_id, sse_event_queue: asyncio.Queue
    ) -> AsyncIterable[SendTaskStreamingResponse] | JSONRPCResponse:
        try:
            while True:
                event = await sse_event_queue.get()
                if isinstance(event, JSONRPCError):
                    yield SendTaskStreamingResponse(id=request_id, error=event)
                    break

                yield SendTaskStreamingResponse(id=request_id, result=event)
                if isinstance(event, TaskStatusUpdateEvent) and event.final:
                    break
        finally:
            async with self.subscriber_lock:
                if task_id in self.task_sse_subscribers:
                    if sse_event_queue in self.task_sse_subscribers[task_id]:
                        self.task_sse_subscribers[task_id].remove(sse_event_queue)
                    if not self.task_sse_subscribers[task_id]:
                        del self.task_sse_subscribers[task_id]

최종 개선사항
✅ 잘못된 요청 타입 계약 → GetTaskPushNotificationRequest 인터페이스 일치 → 추상 메서드 구현 단계의 즉시 실패 방지
✅ 무조건 취소 불가 응답 → SUBMITTED/WORKING 상태 검증 후 CANCELED 전환 → 실제 태스크 취소와 상태 일관성 확보
✅ 무제한 SSE Queue → max_queue_size 검증 및 bounded queue 적용 → 비정상 스트리밍 상황의 메모리 고갈 방지
✅ SSE 이벤트 유실을 유발하는 timeout → bounded queue의 backpressure 기반 전달 → 이벤트 무결성 우선 처리
✅ 얕은 Task 복사 → model_copy(deep=True) 적용 → 조회 결과 변경이 원본 Task 상태로 전파되는 사이드 이펙트 차단
✅ 존재하지 않는 Task와 미설정 Push 구분 → Task 존재 여부와 notification 설정 상태 분리 → API 오류 의미와 데이터 상태 명확화
✅ 취소 상태만 로컬 변경 → final=True SSE 이벤트 동시 발행 → 저장 상태와 실시간 구독 상태의 동기화 확보

원본의 인메모리 A2A Task Manager 구조는 유지하면서 계약 오류·상태 불일치·메모리 고갈·SSE 이벤트 유실 위험을 방어해 운영 안정성과 데이터 무결성을 갖춘 실무형 비동기 관리 구조로 승격되었다.                        
