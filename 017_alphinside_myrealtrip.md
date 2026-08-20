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
from agent import BurgerSellerAgent
import click
import logging
from dotenv import load_dotenv
import os

load_dotenv()

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


@click.command()
@click.option("--host", "host", default="0.0.0.0")
@click.option("--port", "port", default=10001)
def main(host, port):
    """Starts the Burger Seller Agent server."""
    try:
        capabilities = AgentCapabilities(pushNotifications=True)
        skill = AgentSkill(
            id="create_burger_order",
            name="Burger Order Creation Tool",
            description="Helps with creating burger orders",
            tags=["burger order creation"],
            examples=["I want to order 2 classic cheeseburgers"],
        )
        agent_card = AgentCard(
            name="burger_seller_agent",
            description="Helps with creating burger orders",
            # The URL provided here is for the sake of demo,
            # in production you should use a proper domain name
            url=f"http://{host}:{port}/",
            version="1.0.0",
            authentication=AgentAuthentication(schemes=["Basic"]),
            defaultInputModes=BurgerSellerAgent.SUPPORTED_CONTENT_TYPES,
            defaultOutputModes=BurgerSellerAgent.SUPPORTED_CONTENT_TYPES,
            capabilities=capabilities,
            skills=[skill],
        )

        notification_sender_auth = PushNotificationSenderAuth()
        notification_sender_auth.generate_jwk()
        server = A2AServer(
            agent_card=agent_card,
            task_manager=AgentTaskManager(
                agent=BurgerSellerAgent(),
                notification_sender_auth=notification_sender_auth,
            ),
            host=host,
            port=port,
            auth_username=os.environ.get("AUTH_USERNAME"),
            auth_password=os.environ.get("AUTH_PASSWORD"),
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

데모 수준의 조립 코드는 깔끔하지만, 필수 인증 설정 검증 부재와 광범위한 예외 삼키기로 실패 원인을 잃어버린 채 서버만 종료될 수 있는 운영 취약형 엔트리포인트다.

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

import logging
import os
import sys
from typing import Final, Tuple

import click
from a2a_server.push_notification_auth import PushNotificationSenderAuth
from a2a_server.server import A2AServer
from a2a_types import (
    AgentAuthentication,
    AgentCapabilities,
    AgentCard,
    AgentSkill,
)
from agent import BurgerSellerAgent
from dotenv import load_dotenv
from task_manager import AgentTaskManager

# 환경 변수 로드
load_dotenv()

# 로깅 포맷 및 레벨 정밀 설정 (운영 환경 가시성 확보)
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    handlers=[logging.StreamHandler(sys.stdout)],
)
logger: Final[logging.Logger] = logging.getLogger(__name__)


def validate_environment() -> Tuple[str, str]:
    """[방어 강화] 서버 구동 전 필수 인증 정보 누락 여부를 사전 차단합니다."""
    auth_username = os.environ.get("AUTH_USERNAME")
    auth_password = os.environ.get("AUTH_PASSWORD")

    if not auth_username or not auth_password:
        logger.critical(
            "Security Error: AUTH_USERNAME or AUTH_PASSWORD is missing in environment variables."
        )
        raise EnvironmentError(
            "Authentication credentials must be provided via environment variables."
        )
    return auth_username, auth_password


@click.command()
@click.option("--host", "host", default="0.0.0.0", help="Server bind host")
@click.option("--port", type=int, default=10001, help="Server bind port")
def main(host: str, port: int) -> None:
    """Starts the Burger Seller Agent server with enterprise-grade hardening."""
    try:
        # 1. 환경 변수 무결성 검증 (Fail-Fast)
        auth_username, auth_password = validate_environment()

        # 2. 에이전트 역량 및 스킬 정의
        capabilities = AgentCapabilities(pushNotifications=True)
        skill = AgentSkill(
            id="create_burger_order",
            name="Burger Order Creation Tool",
            description="Helps with creating burger orders",
            tags=["burger order creation"],
            examples=["I want to order 2 classic cheeseburgers"],
        )

        # 3. 에이전트 카드 메타데이터 생성 (외부 공개 URL과 바인딩 호스트 분리 관점 적용)
        external_host = "localhost" if host == "0.0.0.0" else host
        agent_card = AgentCard(
            name="burger_seller_agent",
            description="Helps with creating burger orders",
            url=f"http://{external_host}:{port}/",
            version="1.0.0",
            authentication=AgentAuthentication(schemes=["Basic"]),
            defaultInputModes=BurgerSellerAgent.SUPPORTED_CONTENT_TYPES,
            defaultOutputModes=BurgerSellerAgent.SUPPORTED_CONTENT_TYPES,
            capabilities=capabilities,
            skills=[skill],
        )

        # 4. 푸시 알림 인증 모듈 초기화 및 JWK 생성
        notification_sender_auth = PushNotificationSenderAuth()
        notification_sender_auth.generate_jwk()

        # 5. A2A 서버 인스턴스 조립
        server = A2AServer(
            agent_card=agent_card,
            task_manager=AgentTaskManager(
                agent=BurgerSellerAgent(),
                notification_sender_auth=notification_sender_auth,
            ),
            host=host,
            port=port,
            auth_username=auth_username,
            auth_password=auth_password,
        )

        # 6. 라우트 등록
        server.app.add_route(
            "/.well-known/jwks.json",
            notification_sender_auth.handle_jwks_endpoint,
            methods=["GET"],
        )

        logger.info("Starting server on %s:%s (Bind: %s)", external_host, port, host)
        server.start()

    except EnvironmentError as env_err:
        logger.critical("Configuration validation failed: %s", env_err)
        sys.exit(1)
    except Exception as e:
        logger.error(
            "An unexpected error occurred during server startup: %s",
            e,
            exc_info=True,
        )
        sys.exit(1)


if __name__ == "__main__":
    main()

최종 개선사항
✅ 인증 환경변수 존재 여부를 서버 내부에서 뒤늦게 검증 → 기동 전 Fail-Fast 검증 → 인증 설정 누락 상태의 서버 실행 차단
✅ 단순 문자열 예외 로그 → logger.exception() 기반 traceback 보존 → startup 장애의 실제 원인 추적성 확보
✅ 0.0.0.0 바인드 주소를 Agent 공개 URL에 그대로 사용 → 바인드 주소와 광고 주소 분리 → 배포 환경별 접근 경로 오인 방지
✅ 문자열 기반 CLI 포트 처리 → int 타입 검증 → 잘못된 포트 입력의 조기 차단
✅ 인증 설정 오류와 일반 startup 오류를 동일 처리 → 설정 오류와 예기치 않은 장애 분리 → 장애 원인 분류 및 운영 대응성 향상
✅ 환경변수에서 가져온 인증 정보를 서버 생성 직전에 검증 → 검증된 설정만 A2AServer에 주입 → 초기화 단계의 인증 계약 무결성 확보

데모 조립 코드에서 실제 운영 장애를 방어하는 엔트리포인트로 승격됐으며, 특히 인증 Fail-Fast와 startup traceback 보존이 핵심적인 개선이다.
