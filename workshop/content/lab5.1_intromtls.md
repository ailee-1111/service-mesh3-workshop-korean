# 상호 TLS (Mutual TLS)
서비스 메시(Service Mesh)의 핵심 기능 중 하나는 애플리케이션에 추가적인 보안 레이어를 제공하는 것입니다. 이는 다음 몇 개의 실습에서 살펴볼 다양한 방식을 통해 수행됩니다. 그중 첫 번째는 "상호 TLS(Mutual TLS)" 혹은 줄여서 **mTLS**라고 불리는 개념입니다.

마이크로서비스 애플리케이션을 배포하고 서비스 간에 많은 양의 개인정보(PII)가 전송되는 시나리오를 상상해 보십시오. 이 시나리오에서는 준수해야 할 온갖 종류의 보안 요구사항이 있을 것입니다. 전형적이면서도 (개발 및 운영 팀에게 큰 골칫거리를 안겨줄 수 있는) 대표적인 요구사항 중 하나는 서비스 간의 모든 통신을 암호화하는 것입니다. 이는 SSL 키 관리 및 교환, 유효성 검사 및 인증, 네트워크 트래픽 암호화 및 복호화, 그리고 각 애플리케이션 스택(Node.js, Java, Go 등)에서 이를 구현하는 방식을 고민해야 함을 의미합니다.

## 암호화되지 않은 현재 상태
현재, 이전 실습에서 배포한 서비스들은 표준 OpenShift 네트워킹/라우팅을 통해 외부 환경으로부터 보호받고 있습니다. 본질적으로 인그레스 게이트웨이(ingress gateway)를 통해 제어되므로 대부분의 서비스로 직접 들어오는 인그레스 라우트가 존재하지 않습니다. 하지만 프로젝트 내에서 실행 중인 악성 포드(Pod)가 존재한다면 우리의 데이터를 가로채거나 다른 서비스로 직접 HTTP 요청을 보낼 수도 있습니다.

<p><i class="fa fa-info-circle"></i> 아직 공유 보드에 항목을 추가하지 않았다면 먼저 몇 가지 항목을 추가해 보시기 바랍니다. </p>

<blockquote>
<i class="fa fa-terminal"></i> 이를 확인하기 위해 CLI를 사용하여 직접 요청을 보내보겠습니다:
</blockquote>

```execute
curl http://boards.%username%:8080/shareditems | jq
```

이 명령은 현재 공유 보드 목록을 조회하기 위해 직접 HTTP curl 요청을 실행합니다. 다음과 유사한 출력이 인쇄됩니다:

```json
[
  {
    "_id": "5e5d75b33396fe0043f63e5c",
    "owner": "anonymous",
    "type": "string",
    "raw": "something goes here",
    "name": "",
    "id": "MNCkr3mK",
    "created_at": "2020-03-02T21:08:03+00:00"
  },
  {
    "_id": "5e5d75b63396fe0043f63e5d",
    "owner": "anonymous",
    "type": "string",
    "raw": "another item",
    "name": "",
    "id": "x5ORoJu8",
    "created_at": "2020-03-02T21:08:06+00:00"
  }
]
```

<br>

## 기존 서비스에 mTLS 추가하기
이제 서비스 메시가 코드 변경 없이, 복잡한 네트워킹 업데이트 없이, 그리고 키 생성 도구(예: ssh-keygen)나 서버를 설치하거나 사용할 필요 없이 어떻게 모든 트래픽을 암호화할 수 있는지 보여드리겠습니다. 당사 애플리케이션의 모든 서비스에 적용되는 정책을 사용해 이를 구성하겠습니다.
<p>
<i class="fa fa-info-circle"></i>
서비스 메시는 개별 서비스, 네임스페이스 및 전체 메시 레벨에서 정책을 설정할 수 있습니다 (적용 및 시행은 이 순서대로 우선순위를 가집니다).
</p>

이 작업을 수행하기 위한 YAML 설정이 이미 작성되어 있습니다. 다음과 같습니다:
```yaml
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "default"
spec:
  mtls:
    mode: STRICT
```

<blockquote>
<i class="fa fa-terminal"></i> 먼저 다음 명령어로 정책을 적용합니다:
</blockquote>

```execute
oc create -f ./config/istio/peer-authentication-mtls.yaml
```

이제 사이드카(sidecar)가 서로 통신할 때 mTLS를 사용하도록 각 서비스에 대한 데스티네이션 룰(DestinationRules)도 설정해야 합니다. 이에 대한 YAML은 다음과 같습니다. 와일드카드가 사용된 것에 유의해 주세요:
```yaml
apiVersion: "networking.istio.io/v1alpha3"
kind: "DestinationRule"
metadata:
  name: "destinationrule-mtls-istio-mutual"
spec:
  host: "*.svc.cluster.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```

<blockquote>
<i class="fa fa-terminal"></i> 기존 데스티네이션 룰(destination rules)을 모두 삭제합니다:
</blockquote>

```execute
oc delete dr --all
```
<br>

<blockquote>
<i class="fa fa-terminal"></i> 다음 명령을 실행하여 이를 적용합니다:
</blockquote>

```execute
oc create -f ./config/istio/destinationrule-mtls.yaml
```

## mTLS를 활성화한 상태에서 애플리케이션 작동 확인하기
정책이 적용되었으므로 이제 서비스 간(피어 간)의 모든 통신은 mTLS를 **필수(STRICT)**로 요구합니다.

<blockquote>
<i class="fa fa-desktop"></i> 웹 애플리케이션으로 이동하여 웹사이트를 몇 번 새로고침하면서 마이크로서비스로 트래픽을 전송해 봅니다.
</blockquote>
이전과 모든 것이 동일하게 보일 것입니다. 보이지 않는 백그라운드 영역에서 추가적인 보안 암호화가 작동하고 있습니다.

<img src="images/app-boardslist.png" width="1024" class="screenshot"><br/>

<br>
다음으로 방금 활성화한 mTLS 상태를 검증해 보겠습니다.
