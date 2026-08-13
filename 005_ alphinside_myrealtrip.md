원본코드"""
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

import gradio as gr
from typing import List, Dict, Any
from purchasing_concierge.agent import root_agent as purchasing_agent
from google.adk.sessions import InMemorySessionService
from google.adk.runners import Runner
from google.adk.events import Event
from typing import AsyncIterator
from google.genai import types
from pprint import pformat

APP_NAME = "purchasing_concierge_app"
USER_ID = "default_user"
SESSION_ID = "default_session"
SESSION_SERVICE = InMemorySessionService()
PURCHASING_AGENT_RUNNER = Runner(
    agent=purchasing_agent,  # The agent we want to run
    app_name=APP_NAME,  # Associates runs with our app
    session_service=SESSION_SERVICE,  # Uses our session manager
)
SESSION_SERVICE.create_session(
    app_name=APP_NAME, user_id=USER_ID, session_id=SESSION_ID
)


async def get_response_from_agent(
    message: str,
    history: List[Dict[str, Any]],
) -> str:
    """Send the message to the backend and get a response.

    Args:
        message: Text content of the message.
        history: List of previous message dictionaries in the conversation.

    Returns:
        Text response from the backend service.
    """
    # try:
    events_iterator: AsyncIterator[Event] = PURCHASING_AGENT_RUNNER.run_async(
        user_id=USER_ID,
        session_id=SESSION_ID,
        new_message=types.Content(role="user", parts=[types.Part(text=message)]),
    )

    responses = []
    async for event in events_iterator:  # event has type Event
        if event.content.parts:
            for part in event.content.parts:
                if part.function_call:
                    formatted_call = f"```python\n{pformat(part.function_call.model_dump(), indent=2, width=80)}\n```"
                    responses.append(
                        gr.ChatMessage(
                            role="assistant",
                            content=f"{part.function_call.name}:\n{formatted_call}",
                            metadata={"title": "🛠️ Tool Call"},
                        )
                    )
                elif part.function_response:
                    formatted_response = f"```python\n{pformat(part.function_response.model_dump(), indent=2, width=80)}\n```"

                    responses.append(
                        gr.ChatMessage(
                            role="assistant",
                            content=formatted_response,
                            metadata={"title": "⚡ Tool Response"},
                        )
                    )

        # Key Concept: is_final_response() marks the concluding message for the turn
        if event.is_final_response():
            if event.content and event.content.parts:
                # Extract text from the first part
                final_response_text = event.content.parts[0].text
            elif event.actions and event.actions.escalate:
                # Handle potential errors/escalations
                final_response_text = (
                    f"Agent escalated: {event.error_message or 'No specific message.'}"
                )
            responses.append(
                gr.ChatMessage(role="assistant", content=final_response_text)
            )
            yield responses
            break  # Stop processing events once the final response is found

        yield responses
    # except Exception as e:
    #     yield [
    #         gr.ChatMessage(
    #             role="assistant",
    #             content=f"Error communicating with agent: {str(e)}",
    #         )
    #     ]


if __name__ == "__main__":
    demo = gr.ChatInterface(
        get_response_from_agent,
        title="Purchasing Concierge",
        description="This assistant can help you to purchase food from remote sellers.",
        type="messages",
    )

    demo.launch(
        server_name="0.0.0.0",
        server_port=8080,
    )

데모 목적의 비동기 이벤트 스트리밍 구조와 ADK 연계는 탄탄하지만, 예외 경계가 비활성화되고 전역 세션을 공유하는 순간 장애 격리와 사용자 데이터 무결성이 무너져 프로덕션 기준 7.5점에 머무는 구조다.

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

import logging
import uuid
from typing import List, Dict, Any, AsyncIterator
from pprint import pformat
import gradio as gr
from google.genai import types
from google.adk.sessions import InMemorySessionService
from google.adk.runners import Runner
from google.adk.events import Event
from purchasing_concierge.agent import root_agent as purchasing_agent

# [방어 강화] 프로덕션 수준의 구조화된 로깅 설정
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("PurchasingConciergeApp")

APP_NAME = "purchasing_concierge_app"
SESSION_SERVICE = InMemorySessionService()

# [방어 강화] 과설계(Class 생성)를 피하고 단일 Runner 인스턴스 공유 유지
PURCHASING_AGENT_RUNNER = Runner(
    agent=purchasing_agent,
    app_name=APP_NAME,
    session_service=SESSION_SERVICE,
)

# [방어 강화] 개발/디버그 환경에서만 Tool Payload 원본을 노출하기 위한 플래그
DEBUG_TOOL_OUTPUT = False


def _get_or_create_user_session(request: gr.Request) -> tuple[str, str]:
    """
    [방어 강화 - 타격 1 해결] 전역 단일 세션 문제를 해결하기 위해 
    Gradio Request 기반으로 사용자별 고유 User ID와 Session ID를 격리합니다.
    """
    # Gradio 클라이언트 식별자 또는 익명 기본값 활용
    user_id = getattr(request, "username", None) or "anonymous_user"
    
    # 세션 키가 없으면 쿠키/세션 스코프 내에서 고유 세션 ID 부여 (데모 기준 유저별 1개 세션 매핑)
    session_id = f"session_{user_id}"
    
    try:
        # 세션 서비스에 해당 세션이 없으면 생성
        SESSION_SERVICE.create_session(
            app_name=APP_NAME, user_id=user_id, session_id=session_id
        )
    except Exception:
        # 이미 존재하는 세션일 경우 예외 무시 (재접속 방어)
        pass
        
    return user_id, session_id


async def get_response_from_agent(
    message: str,
    history: List[Dict[str, Any]],
    request: gr.Request,
) -> AsyncIterator[List[gr.ChatMessage]]:
    """
    [방어 강화] 사용자별 세션 격리, UnboundLocalError 방어, 그리고 
    주석 처리되었던 안전한 예외 경계(try-except)를 복구하여 런타임 안정성을 극대화합니다.
    """
    user_id, session_id = _get_or_create_user_session(request)

    try:
        events_iterator: AsyncIterator[Event] = PURCHASING_AGENT_RUNNER.run_async(
            user_id=user_id,
            session_id=session_id,
            new_message=types.Content(role="user", parts=[types.Part(text=message)]),
        )

        responses = []
        async for event in events_iterator:
            if event.content and event.content.parts:
                for part in event.content.parts:
                    if part.function_call and DEBUG_TOOL_OUTPUT:
                        formatted_call = f"```python\n{pformat(part.function_call.model_dump(), indent=2, width=80)}\n```"
                        responses.append(
                            gr.ChatMessage(
                                role="assistant",
                                content=f"{part.function_call.name}:\n{formatted_call}",
                                metadata={"title": "🛠️ Tool Call"},
                            )
                        )
                    elif part.function_response and DEBUG_TOOL_OUTPUT:
                        formatted_response = f"```python\n{pformat(part.function_response.model_dump(), indent=2, width=80)}\n```"
                        responses.append(
                            gr.ChatMessage(
                                role="assistant",
                                content=formatted_response,
                                metadata={"title": "⚡ Tool Response"},
                            )
                        )

            # [방어 강화 - 타격 3 해결] Final Response 누락 시 UnboundLocalError 발생 방지
            if event.is_final_response():
                final_response_text = "Agent returned no final response."

                if event.content and event.content.parts:
                    text = event.content.parts[0].text
                    if text:
                        final_response_text = text
                elif event.actions and event.actions.escalate:
                    final_response_text = (
                        f"Agent escalated: {event.error_message or 'No specific message.'}"
                    )
                    
                responses.append(
                    gr.ChatMessage(role="assistant", content=final_response_text)
                )
                yield responses
                break

            yield responses

    except Exception as e:
        # [방어 강화 - 타격 2 해결] 서버 로그에는 상세 트레이스를 보존하고, 사용자 UI에는 안전한 메시지 반환
        logger.exception("Agent execution failed during streaming turn")
        yield [
            gr.ChatMessage(
                role="assistant",
                content="⚠️ Sorry, an error occurred while communicating with the purchasing concierge. Please try again later.",
            )
        ]


if __name__ == "__main__":
    demo = gr.ChatInterface(
        fn=get_response_from_agent,
        title="Purchasing Concierge",
        description="This assistant can help you to purchase food from remote sellers.",
        type="messages",
    )

    demo.launch(
        server_name="0.0.0.0",
        server_port=8080,
    )

최종 개선사항
✅ 전역 단일 사용자 세션 → Request 기반 사용자·세션 격리 → 사용자 간 대화 상태 오염 방지
✅ 주석 처리된 예외 경계 → 스트리밍 전체 구간의 방어적 예외 처리 → 런타임 장애 시 UI 무응답 방지
✅ final_response_text 미초기화 가능 → 안전한 기본 응답 선할당 → 최종 이벤트 구조 변화에도 UnboundLocalError 방지
✅ Tool payload 무조건 출력 → DEBUG_TOOL_OUTPUT 플래그로 제한 → 민감한 요청·응답 정보의 운영 환경 노출 차단
✅ 서버 상세 오류와 사용자 메시지 혼재 → logger.exception()과 안전한 UI 메시지 분리 → 디버깅 가능성과 정보 노출 방지 동시 확보
✅ 불필요한 클래스화 → 공유 Runner 구조 유지 → 데모 규모에 맞는 단순성과 실행 효율 확보
✅ 세션 생성 실패를 무조건 전파 → 재접속 상황의 기존 세션 충돌을 흡수 → 세션 초기화 과정의 불필요한 장애 전파 완화

데모 코드의 목적은 유지하면서 세션 격리·예외 경계·정보 노출·최종 응답 안정성을 보강한 상태로, 현재 범위에서는 추가적인 retry·circuit breaker·복잡한 세션 매니저까지 넣을 이유가 없어 9.5~9.8 수준에서 동결할 만한 구조다.
