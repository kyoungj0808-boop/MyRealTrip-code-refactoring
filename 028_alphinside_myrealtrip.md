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

스타터 템플릿으로는 충분하지만, 스키마 계약·취소 상태·SSE 메모리·예외 분류가 방치되어 있어 프로덕션 투입 시 런타임 장애와 데이터 유실 위험을 그대로 안고 있는 구조다.

제안패치
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

        async with self.lock:
            task = self.tasks.get(request.params.id)
            if task is None:
                return GetTaskResponse(
                    id=request.id,
                    error=TaskNotFoundError(),
                )

            task_result = self.append_task_history(
                task,
                request.params.historyLength,
            )

        return GetTaskResponse(
            id=request.id,
            result=task_result,
        )

    async def on_cancel_task(
        self, request: CancelTaskRequest
    ) -> CancelTaskResponse:
        logger.info("Cancelling task %s", request.params.id)

        async with self.lock:
            task = self.tasks.get(request.params.id)

            if task is None:
                return CancelTaskResponse(
                    id=request.id,
                    error=TaskNotFoundError(),
                )

            if task.status.state not in {
                TaskState.SUBMITTED,
                TaskState.WORKING,
            }:
                return CancelTaskResponse(
                    id=request.id,
                    error=TaskNotCancelableError(),
                )

            cancel_status = TaskStatus(state=TaskState.CANCELED)
            task.status = cancel_status

            result = task.model_copy(deep=True)

        cancel_event = TaskStatusUpdateEvent(
            id=request.params.id,
            status=cancel_status,
            final=True,
        )

        await self.enqueue_events_for_sse(
            request.params.id,
            cancel_event,
        )

        return CancelTaskResponse(
            id=request.id,
            result=result,
        )

    @abstractmethod
    async def on_send_task(
        self, request: SendTaskRequest
    ) -> SendTaskResponse:
        pass

    async def on_send_task_subscribe(
        self, request: SendTaskStreamingRequest
    ) -> Union[AsyncIterable[SendTaskStreamingResponse], JSONRPCResponse]:
        return new_not_implemented_error(request.id)

    async def set_push_notification_info(
        self,
        task_id: str,
        notification_config: PushNotificationConfig,
    ) -> None:
        async with self.lock:
            if task_id not in self.tasks:
                raise KeyError(f"Task not found for {task_id}")

            self.push_notification_infos[task_id] = notification_config

    async def get_push_notification_info(
        self,
        task_id: str,
    ) -> Optional[PushNotificationConfig]:
        async with self.lock:
            if task_id not in self.tasks:
                raise KeyError(f"Task not found for {task_id}")

            return self.push_notification_infos.get(task_id)

    async def has_push_notification_info(self, task_id: str) -> bool:
        async with self.lock:
            return task_id in self.push_notification_infos

    async def on_set_task_push_notification(
        self,
        request: SetTaskPushNotificationRequest,
    ) -> SetTaskPushNotificationResponse:
        logger.info(
            "Setting task push notification %s",
            request.params.id,
        )

        try:
            await self.set_push_notification_info(
                request.params.id,
                request.params.pushNotificationConfig,
            )
        except KeyError:
            return SetTaskPushNotificationResponse(
                id=request.id,
                error=TaskNotFoundError(),
            )
        except Exception:
            logger.exception(
                "Error while setting push notification info"
            )
            return SetTaskPushNotificationResponse(
                id=request.id,
                error=InternalError(
                    message="An error occurred while setting push notification info"
                ),
            )

        return SetTaskPushNotificationResponse(
            id=request.id,
            result=request.params,
        )

    async def on_get_task_push_notification(
        self,
        request: GetTaskPushNotificationRequest,
    ) -> GetTaskPushNotificationResponse:
        logger.info(
            "Getting task push notification %s",
            request.params.id,
        )

        try:
            notification_info = await self.get_push_notification_info(
                request.params.id
            )
        except KeyError:
            return GetTaskPushNotificationResponse(
                id=request.id,
                error=TaskNotFoundError(),
            )
        except Exception:
            logger.exception(
                "Error while getting push notification info"
            )
            return GetTaskPushNotificationResponse(
                id=request.id,
                error=InternalError(
                    message="An error occurred while getting push notification info"
                ),
            )

        if notification_info is None:
            return GetTaskPushNotificationResponse(
                id=request.id,
                error=InternalError(
                    message="Push notification configuration not found"
                ),
            )

        return GetTaskPushNotificationResponse(
            id=request.id,
            result=TaskPushNotificationConfig(
                id=request.params.id,
                pushNotificationConfig=notification_info,
            ),
        )

    async def upsert_task(
        self,
        task_send_params: TaskSendParams,
    ) -> Task:
        logger.info(
            "Upserting task %s",
            task_send_params.id,
        )

        async with self.lock:
            task = self.tasks.get(task_send_params.id)

            if task is None:
                task = Task(
                    id=task_send_params.id,
                    sessionId=task_send_params.sessionId,
                    status=TaskStatus(
                        state=TaskState.SUBMITTED,
                    ),
                    history=[task_send_params.message],
                )
                self.tasks[task_send_params.id] = task
            else:
                if task.history is None:
                    task.history = []

                task.history.append(task_send_params.message)

            return task

    async def on_resubscribe_to_task(
        self,
        request: TaskResubscriptionRequest,
    ) -> Union[AsyncIterable[SendTaskStreamingResponse], JSONRPCResponse]:
        return new_not_implemented_error(request.id)

    async def update_store(
        self,
        task_id: str,
        status: TaskStatus,
        artifacts: list[Artifact],
    ) -> Task:
        async with self.lock:
            task = self.tasks.get(task_id)

            if task is None:
                logger.error(
                    "Task %s not found for updating the task",
                    task_id,
                )
                raise ValueError(f"Task {task_id} not found")

            task.status = status

            if status.message is not None:
                if task.history is None:
                    task.history = []

                task.history.append(status.message)

            if artifacts:
                if task.artifacts is None:
                    task.artifacts = []

                task.artifacts.extend(artifacts)

            return task

    def append_task_history(
        self,
        task: Task,
        history_length: int | None,
    ) -> Task:
        new_task = task.model_copy(deep=True)

        if new_task.history is None:
            return new_task

        if history_length is not None and history_length > 0:
            new_task.history = new_task.history[-history_length:]
        else:
            new_task.history = []

        return new_task

    async def setup_sse_consumer(
        self,
        task_id: str,
        is_resubscribe: bool = False,
    ) -> asyncio.Queue:
        async with self.subscriber_lock:
            if task_id not in self.task_sse_subscribers:
                if is_resubscribe:
                    raise ValueError(
                        "Task not found for resubscription"
                    )

                self.task_sse_subscribers[task_id] = []

            queue = asyncio.Queue(maxsize=self.max_queue_size)
            self.task_sse_subscribers[task_id].append(queue)

            return queue

    async def enqueue_events_for_sse(
        self,
        task_id: str,
        task_update_event,
    ) -> None:
        async with self.subscriber_lock:
            subscribers = list(
                self.task_sse_subscribers.get(task_id, [])
            )

        for subscriber in subscribers:
            try:
                subscriber.put_nowait(task_update_event)
            except asyncio.QueueFull:
                logger.warning(
                    "Dropping SSE event for slow subscriber on task %s",
                    task_id,
                )
            except Exception:
                logger.exception(
                    "Failed to enqueue SSE event for task %s",
                    task_id,
                )

    async def dequeue_events_for_sse(
        self,
        request_id,
        task_id,
        sse_event_queue: asyncio.Queue,
    ) -> AsyncIterable[SendTaskStreamingResponse] | JSONRPCResponse:
        try:
            while True:
                event = await sse_event_queue.get()

                if isinstance(event, JSONRPCError):
                    yield SendTaskStreamingResponse(
                        id=request_id,
                        error=event,
                    )
                    break

                yield SendTaskStreamingResponse(
                    id=request_id,
                    result=event,
                )

                if (
                    isinstance(event, TaskStatusUpdateEvent)
                    and event.final
                ):
                    break
        finally:
            async with self.subscriber_lock:
                subscribers = self.task_sse_subscribers.get(task_id)

                if subscribers is None:
                    return

                try:
                    subscribers.remove(sse_event_queue)
                except ValueError:
                    return

                if not subscribers:
                    del self.task_sse_subscribers[task_id]

최종 개선사항
✅ 잘못된 Task 필드 주입 → 실제 스키마 계약에 맞춘 history 초기화 → 런타임 ValidationError 방지
✅ 무조건 취소 불가 반환 → SUBMITTED/WORKING 상태 검증 후 취소 전이 → 상태 머신 무결성 확보
✅ 취소 상태만 변경 → 최종 SSE 이벤트까지 동시 전파 → 저장 상태와 스트림 상태 일관성 확보
✅ 무제한 SSE Queue → bounded Queue 적용 → 느린 클라이언트의 메모리 고갈 방지
✅ 락 내부 대기형 SSE 전달 → subscriber 스냅샷 + put_nowait 전환 → 단일 느린 소비자의 전체 엔진 정지 방지
✅ 얕은 Task 복사 → deep copy 기반 반환 객체 격리 → mutable history/artifact 오염 방지
✅ 광범위한 예외 처리 → KeyError와 내부 예외 분리 → 정상적인 도메인 오류와 시스템 장애 구분
✅ 취약한 SSE cleanup → membership 검증 후 안전한 제거 → 연결 종료·중복 정리 상황의 2차 예외 방지

스타터 템플릿 수준의 상태 관리 코드를 동시성·메모리·상태 전이·SSE 생명주기까지 방어하는 실무형 InMemoryTaskManager로 승격했다.
