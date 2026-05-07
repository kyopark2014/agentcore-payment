# Amazon Bedrock AgentCore Payments

Amazon Bedrock AgentCore Payments는 AI agent가 자율적으로 결제(microtransaction)를 수행할 수 있도록 하는 AWS의 관리형 결제 서비스입니다.

- Coinbase 및 Stripe를 이용해 Payment를 구현할 수 있습니다.
- Agent가 HTTP 402 "Payment Required" 상태 코드를 받을때 활용할 수 있습니다. 
- Agent가 유료 API, MCP 서버, 웹 콘텐츠에 사람 개입 없이 결제하여 접근 가능합니다.

> 업계는 agent 커머스가 2030년까지 글로벌 커머스의 3~5조 달러를 중개할 것으로 전망하고 있습니다. 

> "Our research estimates that by 2030, agentic commerce could orchestrate 3-5 trillion globally, as AI agents increasingly influence discovery, decision-making, and transactions across categories.", [McKinsey](https://www.linkedin.com/posts/mckinsey_our-research-estimates-that-by-2030-agentic-activity-7444669942127374336-MQl5/)

### 사용 가능 리전 (Preview)
`us-east-1`, `us-west-2`, `eu-central-1`, `ap-southeast-2`

---

## 핵심 구성 요소 (Architecture)

| 구성 요소 | 역할 |
|---|---|
| PaymentManager | AWS 계정 내 결제 작업을 조율하는 최상위 리소스. `AWS_IAM` 또는 `CUSTOM_JWT` 인증 방식 사용 |
| PaymentConnector | 외부 결제 제공자(Coinbase CDP, Stripe Privy)와 연결하는 커넥터 |
| PaymentCredentialProvider | AgentCore Identity를 통해 AWS Secrets Manager에 자격 증명을 안전하게 저장 |
| PaymentSession | 단일 agent 상호작용에 대한 결제 컨텍스트 (시간·예산 제한 적용) |
| PaymentInstrument | 사용자를 대신해 결제하는 임베디드 크립토 월렛 (예: USDC 지갑) |

### 리소스 라이프사이클 상태
`CREATING` → `READY` → `UPDATING` / `CREATE_FAILED` / `UPDATE_FAILED`

---

## 지원 결제 제공자

| 제공자 | 설명 |
|---|---|
| CoinbaseCDP | Coinbase Developer Platform 기반 크립토 지갑 (GA) |
| StripePrivy | Stripe + Privy 임베디드 지갑 인프라 (Preview) |

---

## 동작 방식 (How it works)

```
┌────────────────────────────────────────────────────────────┐
│  [개발자] PaymentManager + PaymentConnector 생성 (1회성 설정)   │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│  [사용자] 지갑 충전 (USDC, 직불카드, Apple Pay, Google Pay, ACH) │
└────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌───────────────────────────────────────────────────────────┐
│  [agent] PaymentSession 생성 (예: "$1.00, 5분 만료")         │
└───────────────────────────────────────────────────────────┘
                          │
                          ▼
┌───────────────────────────────────────────────────────────┐
│  [agent] 유료 엔드포인트 호출 → HTTP 402 응답 수신               │
└───────────────────────────────────────────────────────────┘
                          │
                          ▼
┌───────────────────────────────────────────────────────────┐
│  ProcessPayment API 호출 → 예산 확인 → 서명 → 결제 프루프 생성    │
└───────────────────────────────────────────────────────────┘
                          │
                          ▼
┌───────────────────────────────────────────────────────────┐
│  Base 네트워크 USDC 정산 (~200ms, 트랜잭션당 1센트 미만)           │
└───────────────────────────────────────────────────────────┘
```


## 보안 및 거버넌스

### 4-Role IAM 모델

| 역할 | 용도 |
|---|---|
| ControlPlaneRole | PaymentManager, Connector, Credential Provider 관리 (관리자) |
| ManagementRole | agent 개발자용 관리 권한 |
| ProcessPaymentRole | 결제 실행 전용 권한 |
| ResourceRetrievalRole | 서비스 리소스 조회 권한 |

핵심 보안 기능은 아래와 같습니다.

- 개인 키 미노출: agent는 private key에 직접 접근 불가
- 예산 가드레일: 세션별 `maxSpendAmount` 및 만료 시간 설정
- 명시적 사용자 승인: 사용자가 지갑 사용을 승인해야 agent가 결제 가능
- 컴플라이언스 내장: Coinbase CDP Facilitator가 제재 및 불법 자금 리스크 관리
- 자동 롤백: 결제 서명 실패 시 차감된 금액 자동 복구
- 관찰성: CloudWatch 연동 로그, 메트릭, 대시보드 제공


## 통합 방식

- AgentCore SDK 또는 Strands SDK를 통한 간편한 통합
- Browser Tool (Playwright) 연동: agent가 브라우저를 통해 x402 유료 콘텐츠 접근
- 표준 HTTP 호출 연동 지원
- AgentCore Gateway와 연결된 Coinbase MCP를 통해 Exa, Messari, Browserbase 등 10,000+ x402 서비스에 접근

---

## 가격 (Pricing)

- Consumption-based pricing (사용량 기반)
- 최소 약정 및 선불 비용 없음
- AgentCore Runtime 참고 가격:
  - CPU: `$0.0895` per vCPU-hour
  - Memory: `$0.00945` per GB-hour (최소 128 MB)
- AgentCore Gateway: `$0.005` per 1,000 API 호출

> 자세한 내용은 [AgentCore Pricing 페이지](https://aws.amazon.com/bedrock/agentcore/pricing/) 참고하세요.


## 사용 사례

- Warner Bros. Discovery: 라이브 스포츠 및 프리미엄 콘텐츠 agent 결제 실험
- Cox Automotive: 자동차 커머스 agent
- Thomson Reuters: 법률/뉴스 데이터 agent
- PGA TOUR: 스포츠 콘텐츠 agent


## 샘플 코드 

아래와 같이 필요한 페키지를 설치합니다.

```bash
# Python 3.10+ 필요
pip install boto3
pip install 'bedrock-agentcore[strands-agents]'
```

아래는 Strands SDK 예제입니다. HTTP 402 응답을 자동으로 감지하고 처리합니다.

```python
from strands import Agent
from strands_tools import http_request
from bedrock_agentcore.payments.integrations.config import AgentCorePaymentsPluginConfig
from bedrock_agentcore.payments.integrations.strands.plugin import AgentCorePaymentsPlugin

# 1. Payments 플러그인 설정
config = AgentCorePaymentsPluginConfig(
    payment_manager_arn="arn:aws:bedrock-agentcore:us-west-2:123456789012:payment-manager/pm-abc123",
    user_id="test-user-123",
    payment_instrument_id="payment-instrument-xyz789",
    payment_session_id="payment-session-def456",
    region="us-west-2",
)

# 2. 플러그인 생성
plugin = AgentCorePaymentsPlugin(config=config)

# 3. 자동 결제 처리가 가능한 agent 생성
agent = Agent(
    system_prompt="You are a helpful assistant that can access paid APIs.",
    tools=[http_request],
    plugins=[plugin],
)

# 4. agent가 402 응답을 자동으로 처리
agent("Access the premium endpoint at https://example.com/paid-api")
```

아래는 AgentCore SDK의 PaymentManager를 이용해 직접 사용하는 예제입니다. 프레임워크 제약 없이 커스텀 agent에서 사용할 수 있는 방법입니다.

```python
import uuid
from bedrock_agentcore.payments import PaymentManager

# PaymentManager 인스턴스 생성
manager = PaymentManager(
    payment_manager_arn="arn:aws:bedrock-agentcore:us-west-2:123456789012:payment-manager/my-manager",
    region_name="us-west-2"
)

# 402 응답을 수신했을 때 결제 프루프 생성
payment_required_request = {
    "statusCode": 402,
    "headers": payment_required["headers"],
    "body": payment_required["body"],
}

payment_proof_headers = manager.generate_payment_header(
    user_id="test-user-123",
    payment_instrument_id="payment-instrument-xyz789",
    payment_session_id="payment-session-def456",
    payment_required_request=payment_required_request,
    client_token=str(uuid.uuid4()),
)

# 생성된 헤더를 유료 엔드포인트 재요청에 포함하여 전송
# (예: requests.get(url, headers=payment_proof_headers))
```

아래는 AWS SDK (Boto3)을 이용하여 ProcessPayment 직접 호출하는 예제입니다. API를 직접 호출하여 세밀한 제어가 필요한 경우 사용합니다.

```python
import uuid
import boto3

dp_client = boto3.client("bedrock-agentcore", region_name="us-west-2")

payment = dp_client.process_payment(
    userId="test-user-123",
    paymentManagerArn="arn:aws:bedrock-agentcore:us-west-2:123456789012:payment-manager/my-manager",
    paymentSessionId="payment-session-abc123",
    paymentInstrumentId="payment-instrument-xyz789",
    paymentType="CRYPTO_X402",
    paymentInput={
        "cryptoX402": {
            "version": "2",
            "payload": {
                "scheme": "exact",
                "network": "eip155:84532",          # Base Sepolia
                "amount": "100000",                  # 0.1 USDC (6 decimals)
                "asset": "0x036CbD53842c5426634e7929541eC2318f3dCF7e",  # USDC
                "payTo": "0x99935f281d3ED1E804bF1413b76E0B03e1fed4F9",
                "maxTimeoutSeconds": 300,
                "extra": {"name": "USDC", "version": "2"},
            },
        }
    },
    clientToken=str(uuid.uuid4()),
)

print(payment["status"])  # PROOF_GENERATED
print(payment["paymentOutput"])
```

이때의 응답 예시입니다.
```json
{
    "processPaymentId": "12345678-1234-1234-1234-123456789012",
    "paymentManagerArn": "arn:aws:bedrock-agentcore:us-west-2:123456789012:payment-manager/my-manager",
    "paymentSessionId": "payment-session-abc123def4567",
    "paymentInstrumentId": "payment-instrument-xyz789abc1234",
    "paymentType": "CRYPTO_X402",
    "status": "PROOF_GENERATED",
    "paymentOutput": {
        "cryptoX402": {
            "version": "2",
            "payload": {
                "...signed transaction proof..."
            }
        }
    },
    "createdAt": "2025-07-15T10:35:00Z",
    "updatedAt": "2025-07-15T10:35:02Z"
}
```

아래는 AWS CLI를 이용하는 예제입니다.

```bash
aws bedrock-agentcore process-payment \
    --payment-manager-arn "arn:aws:bedrock-agentcore:us-west-2:123456789012:payment-manager/my-manager" \
    --payment-session-id "payment-session-abc123" \
    --payment-instrument-id "payment-instrument-xyz789" \
    --payment-type "CRYPTO_X402" \
    --payment-input '{
        "cryptoX402": {
            "version": "2",
            "payload": {
                "scheme": "exact",
                "network": "eip155:84532",
                "amount": "100000",
                "asset": "0x036CbD53842c5426634e7929541eC2318f3dCF7e",
                "payTo": "0x99935f281d3ED1E804bF1413b76E0B03e1fed4F9",
                "maxTimeoutSeconds": 300,
                "extra": {"name": "USDC", "version": "2"}
            }
        }
    }' \
    --client-token "$(uuidgen)" \
    --region us-west-2
```

아래는 Strands SDK에서 결제 실패 시 인터럽트 처리 예제입니다.

```python
result = agent("Access the premium endpoint at https://api.example.com/premium")

while result.stop_reason == "interrupt":
    responses = []
    for interrupt in result.interrupts:
        if interrupt.name.startswith("payment-failure-"):
            reason = interrupt.reason
            exception_type = reason.get("exceptionType")

            if exception_type == "PaymentInstrumentConfigurationRequired":
                # 새로운 payment instrument로 업데이트
                plugin.config.update_payment_instrument_id("payment-instrument-new123")
                responses.append({
                    "interruptResponse": {
                        "interruptId": interrupt.id,
                        "response": "Payment instrument configured. Please retry.",
                    }
                })
            elif exception_type == "PaymentSessionConfigurationRequired":
                # 새로운 payment session으로 업데이트
                plugin.config.update_payment_session_id("payment-session-new456")
                responses.append({
                    "interruptResponse": {
                        "interruptId": interrupt.id,
                        "response": "Payment session configured. Please retry.",
                    }
                })
            else:
                responses.append({
                    "interruptResponse": {
                        "interruptId": interrupt.id,
                        "response": f"Payment failed: {reason.get('exceptionMessage')}",
                    }
                })

    result = agent(responses)
```

### End-to-End 워크플로우

실전 사용 시 전체 설정 → 결제 처리 흐름 개요입니다.

```python
import boto3
import uuid

cp_client = boto3.client("bedrock-agentcore-control", region_name="us-west-2")
dp_client = boto3.client("bedrock-agentcore", region_name="us-west-2")

# ───────────── [1회성] 컨트롤 플레인 설정 ─────────────

# 1. PaymentManager 생성
mgr = cp_client.create_payment_manager(
    name="my-payment-manager",
    authorizerConfiguration={"customJWTAuthorizer": {...}},  # 또는 AWS_IAM
    roleArn="arn:aws:iam::123456789012:role/AgentCorePaymentsServiceRole",
)

# 2. PaymentConnector 생성 (Coinbase CDP 예시)
conn = cp_client.create_payment_connector(
    paymentManagerArn=mgr["paymentManagerArn"],
    name="coinbase-connector",
    connectorType="CoinbaseCDP",
    credentialProviderArn="arn:aws:bedrock-agentcore:...:credential-provider/...",
)

# ───────────── [런타임] 데이터 플레인 호출 ─────────────

# 3. PaymentInstrument(지갑) 생성
instrument = dp_client.create_payment_instrument(
    userId="test-user-123",
    paymentManagerArn=mgr["paymentManagerArn"],
    instrumentType="EMBEDDED_CRYPTO_WALLET",
    network="eip155:84532",  # Base Sepolia
)

# 4. 사용자를 Coinbase WalletHub로 리다이렉트하여 자금 충전
print("Fund wallet at:", instrument["paymentInstrumentDetails"]["redirectUrl"])

# 5. PaymentSession 생성 (예산 및 만료 시간 설정)
session = dp_client.create_payment_session(
    userId="test-user-123",
    paymentManagerArn=mgr["paymentManagerArn"],
    expiryInSeconds=300,  # 5분
    paymentLimit={"maxSpendAmount": "1.00", "currency": "USD"},
)

# 6. ProcessPayment 호출 ("AWS SDK (Boto3) - ProcessPayment 직접 호출" 예시 참조)
```

---

## Reference

### AWS 공식 문서
- [Amazon Bedrock AgentCore 홈](https://aws.amazon.com/bedrock/agentcore/)
- [AgentCore Payments - How it works](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/payments-how-it-works.html)
- [Core concepts for AgentCore payments](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/payments-concepts.html)
- [Get started with AgentCore payments](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/payments-getting-started.html)
- [Process a payment](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/payments-process-payment.html)
- [Create a Payment Manager and Connector](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/payments-create-manager.html)
- [Create a payment instrument](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/payments-create-instrument.html)
- [Create a payment session](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/payments-create-session.html)
- [IAM roles for AgentCore payments](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/payments-iam-roles.html)
- [Integrating with Browser Tool](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/payments-browser.html)
- [AgentCore Pricing](https://aws.amazon.com/bedrock/agentcore/pricing/)

### 블로그 및 뉴스
- [AWS ML Blog: Agents that transact - Introducing Amazon Bedrock AgentCore payments](https://aws.amazon.com/blogs/machine-learning/agents-that-transact-introducing-amazon-bedrock-agentcore-payments-built-with-coinbase-and-stripe/)
- [Coinbase Blog: Introducing AgentCore Payments powered by x402 and Coinbase](https://www.coinbase.com/blog/introducing-amazon-bedrock-agentcore-payments-powered-by-x402-and-coinbase)
- [x402 Protocol 공식 사이트](https://www.x402.org/)

### 관련 SDK 및 도구
- [bedrock-agentcore Python SDK (PyPI)](https://pypi.org/project/bedrock-agentcore/)
- [Strands Agents 프레임워크](https://github.com/strands-agents)
- [Coinbase Developer Platform](https://docs.cdp.coinbase.com/)
- [Privy Embedded Wallets](https://docs.privy.io/)

