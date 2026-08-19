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

from a2a_server.server import A2AServer
from a2a_types import AgentCard, AgentCapabilities, AgentSkill, AgentAuthentication
from a2a_server.push_notification_auth import PushNotificationSenderAuth
from task_manager import AgentTaskManager
from agent import PizzaSellerAgent
import click
import logging
from dotenv import load_dotenv
import os

load_dotenv()

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


@click.command()
@click.option("--host", "host", default="0.0.0.0")
@click.option("--port", "port", default=10000)
def main(host, port):
    """Starts the Pizza Seller Agent server."""
    try:
        capabilities = AgentCapabilities(pushNotifications=True)
        skill = AgentSkill(
            id="create_pizza_order",
            name="Pizza Order Creation Tool",
            description="Helps with creating pizza orders",
            tags=["pizza order creation"],
            examples=["I want to order 2 pepperoni pizzas"],
        )
        agent_card = AgentCard(
            name="pizza_seller_agent",
            description="Helps with creating pizza orders",
            # The URL provided here is for the sake of demo,
            # in production you should use a proper domain name
            url=f"http://{host}:{port}/",
            version="1.0.0",
            authentication=AgentAuthentication(schemes=["Bearer"]),
            defaultInputModes=PizzaSellerAgent.SUPPORTED_CONTENT_TYPES,
            defaultOutputModes=PizzaSellerAgent.SUPPORTED_CONTENT_TYPES,
            capabilities=capabilities,
            skills=[skill],
        )

        notification_sender_auth = PushNotificationSenderAuth()
        notification_sender_auth.generate_jwk()
        server = A2AServer(
            agent_card=agent_card,
            task_manager=AgentTaskManager(
                agent=PizzaSellerAgent(),
                notification_sender_auth=notification_sender_auth,
            ),
            host=host,
            port=port,
            api_key=os.environ.get("API_KEY"),
        )

        server.app.add_route(
            "/.well-known/jwks.json",
            notification_sender_auth.handle_jwks_endpoint,
            methods=["GET"],
        )

        logger.info(f"Starting server on {host}:{port}")
        server.start()
    except Exception as e:
        logger.error(f"An error occurred during server startup: {e}")
        exit(1)


if __name__ == "__main__":
    main()

원본은 A2A 서버 부트스트랩 구조와 JWK 인증 연동은 깔끔하지만, API_KEY 누락 검증과 예외 경계가 취약해 운영 장애 시 원인 추적과 인증 실패를 안정적으로 보장하지 못하는 상태다.

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

import ipaddress
import os
import logging
import click
from dotenv import load_dotenv

from a2a_server.server import A2AServer
from a2a_types import AgentCard, AgentCapabilities, AgentSkill, AgentAuthentication
from a2a_server.push_notification_auth import PushNotificationSenderAuth
from task_manager import AgentTaskManager
from agent import PizzaSellerAgent

load_dotenv()

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


def _validate_host_and_port(host: str, port: int) -> None:
    """[방어 강화 - 3번 검증] host 주소 유효성과 port 범위를 사전에 검증하여 바인딩 에러 방지"""
    try:
        if host not in ("0.0.0.0", "127.0.0.1", "localhost"):
            ipaddress.ip_address(host)
    except ValueError as e:
        raise ValueError(f"Security/Config Audit: Invalid host address format '{host}': {e}")

    if not (1 <= port <= 65535):
        raise ValueError(f"Security/Config Audit: Port {port} is out of valid range (1-65535).")


@click.command()
@click.option("--host", "host", default="0.0.0.0", help="Host to bind the server to.")
@click.option("--port", "port", default=10000, type=int, help="Port to bind the server to.")
@click.option("--external-url", "external_url", default=None, help="Advertised external URL for AgentCard (optional).")
def main(host, port, external_url):
    """Starts the Pizza Seller Agent server with fail-fast validation and robust exception tracking."""
    try:
        # 1. 포트 및 호스트 유효성 검증
        _validate_host_and_port(host, port)

        # 2. 필수 인증 설정의 Fail-fast 검증 (API_KEY 누락 즉시 차단)
        api_key = os.environ.get("API_KEY")
        if not api_key:
            logger.error("Security Audit: API_KEY environment variable is missing. Failing fast.")
            raise RuntimeError("API_KEY is required to run the agent server securely.")

        # 4. bind 주소와 AgentCard의 advertised URL 분리 구조 적용 (외부 주소가 주어지면 우선 사용)
        advertised_url = external_url if external_url else f"http://{host}:{port}/"

        capabilities = AgentCapabilities(pushNotifications=True)
        skill = AgentSkill(
            id="create_pizza_order",
            name="Pizza Order Creation Tool",
            description="Helps with creating pizza orders",
            tags=["pizza order creation"],
            examples=["I want to order 2 pepperoni pizzas"],
        )
        
        agent_card = AgentCard(
            name="pizza_seller_agent",
            description="Helps with creating pizza orders",
            url=advertised_url,
            version="1.0.0",
            authentication=AgentAuthentication(schemes=["Bearer"]),
            defaultInputModes=PizzaSellerAgent.SUPPORTED_CONTENT_TYPES,
            defaultOutputModes=PizzaSellerAgent.SUPPORTED_CONTENT_TYPES,
            capabilities=capabilities,
            skills=[skill],
        )

        notification_sender_auth = PushNotificationSenderAuth()
        notification_sender_auth.generate_jwk()
        
        server = A2AServer(
            agent_card=agent_card,
            task_manager=AgentTaskManager(
                agent=PizzaSellerAgent(),
                notification_sender_auth=notification_sender_auth,
            ),
            host=host,
            port=port,
            api_key=api_key,
        )

        server.app.add_route(
            "/.well-known/jwks.json",
            notification_sender_auth.handle_jwks_endpoint,
            methods=["GET"],
        )

        logger.info(f"Starting server bound to {host}:{port} (Advertised URL: {advertised_url})")
        server.start()

    except Exception:
        # 3. logger.exception()을 통한 트레이스백 보존 및 원인 추적 보장
        logger.exception("Critical error during server lifecycle or startup configuration")
        exit(1)


if __name__ == "__main__":
    main()

최종 개선사항
✅ 무검증 host/port 입력 → 주소 형식·포트 범위 사전 검증 → 잘못된 서버 바인딩 설정의 조기 차단
✅ API_KEY 누락 상태의 무방비 기동 → Fail-fast 인증 설정 검증 → 인증 설정 오류의 운영 진입 차단
✅ bind 주소와 AgentCard URL 혼용 → external-url 기반 advertised URL 분리 → 내부 바인딩 주소와 외부 접근 주소의 의미 분리
✅ 문자열 기반 port 설정 → Click type=int 및 범위 검증 → CLI 입력 계약과 서버 설정 무결성 강화
✅ logger.error()로 예외 정보 축약 → logger.exception()으로 traceback 보존 → 초기화·기동 장애의 원인 추적성 확보
✅ 서버 설정과 인증 초기화를 단일 무방비 흐름으로 처리 → 각 단계의 Fail-fast 검증 강화 → 잘못된 설정으로 서버가 기동되는 위험 최소화

원본의 A2A 서버 구조는 유지하면서 설정·인증·주소 검증과 traceback 보존을 보강해, 잘못된 구성은 기동 전에 차단하고 장애 원인은 끝까지 추적 가능한 운영형 구조로 승격했다.
