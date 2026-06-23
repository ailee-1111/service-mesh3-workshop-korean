# 정책을 통한 접근 인증 및 인가
이전 실습에서는 서비스 간(service-to-service) 통신을 안전하게 보호하고 이를 검증했습니다. 그렇다면 사용자-서비스 간(user-to-service) 통신(즉, 기원 인증/Origin Authentication)은 어떻게 보호할까요? 서비스 메시는 이 부분 역시 지원할 수 있습니다. 이를 구현하려면 로그인한 사용자에게 JSON Web Token(JWT)을 발급해 줄 ID 제공자(Identity Provider)를 연동해야 합니다. JWT는 신원 정보를 공유하기 위한 개방형 산업 표준(RFC 7519) 규격입니다. app-ui 서비스는 API 요청 시 이 JWT를 함께 실어서 전달합니다. 이를 통해 서비스 메시의 사이드카 프록시가 토큰을 검증하고, 역할 데이터를 조회하며, 접근 권한을 강제할 수 있습니다.

인증(Authentication) 및 인가(Authorization) 정책을 사용하여 서비스에 대한 접근을 제한하는 기본적인 예제를 단계별로 살펴보겠습니다.

## 인증된 사용자만 접근 허용하기
서비스에 대한 접근을 완전히 차단하고, 정상적으로 인증 과정을 거친 사용자만 접근할 수 있도록 차단할 수 있습니다.

<blockquote>
<i class="fa fa-terminal"></i> 다음 명령을 실행하여 새로운 인증 정책 및 서비스 엔트리(service entry)를 적용합니다:
</blockquote>

```execute
 sed "s|%SSO_SVC%|$SSO_SVC|" ./config/istio/request-authentication-boards-jwt.yaml | oc apply -f -
 sed "s|%SSO_SVC%|$SSO_SVC|" ./config/istio/serviceentry-keycloak.yaml | oc apply -f -
```


이 정책은 boards 서비스로 들어오는 트래픽이 특정 속성을 가진 JWT를 가져야 한다는 요구사항을 지정합니다. 다음과 같은 형태입니다:
```yaml
apiVersion: "security.istio.io/v1beta1"
kind: RequestAuthentication
metadata:
  name: "boards-jwt"
spec:
  selector:
    matchLabels:
      app: boards
      deploymentconfig: boards
  jwtRules:
  - issuer: "https://keycloak-sso-shared.apps.cluster.domain.com/auth/realms/microservices-demo"
    jwksUri: "https://keycloak-sso-shared.apps.cluster.domain.com/auth/realms/microservices-demo/protocol/openid-connect/certs"

```
<p>
<i class="fa fa-info-circle"></i>
앞서 언급했듯이 JWT는 신원 정보를 공유하는 용도로 사용됩니다. JWKS 엔드포인트는 JWT의 서명이 우리가 신뢰하는 기원으로부터 발급된 올바른 서명인지 검증할 수 있는 키(keys) 정보를 제공해 줍니다.
</p>

<p>
<i class="fa fa-info-circle"></i>
서비스 엔트리에 대해서는 지금 당장 걱정하지 않으셔도 됩니다. 다른 실습 과정에서 이를 설명해 드립니다.
</p>

<br>

<blockquote>
<i class="fa fa-desktop"></i> 웹 애플리케이션으로 이동하여 내비게이션 바에서 'Shared' 버튼을 클릭합니다.
</blockquote>
<blockquote>
<i class="fa fa-desktop"></i> 페이지를 몇 번 새로고침한 다음, 공유 보드(shared board)에 새 항목을 추가해 봅니다.
</blockquote>

(정책이 완전히 적용되면) 다음 스크린샷과 같이 오류가 발생하며 실패하게 됩니다.

<img src="images/app-boardsshared-failedget.png" width="1024" class="screenshot"><br/>

<br>

<blockquote>
<i class="fa fa-desktop"></i> 사용자 아이디 "demo", 비밀번호 "demo"로 로그인합니다.
</blockquote>

<blockquote>
<i class="fa fa-desktop"></i> 공유 보드 페이지에 다시 접속해 봅니다.
</blockquote>

이제 공유 보드 목록이 정상적으로 표시되는 것을 확인할 수 있습니다.

<img src="images/app-boardsshared-successget.png" width="1024" class="screenshot"><br/>


<p><i class="fa fa-info-circle"></i>
정책이 완전히 배포되어 반영될 때까지 페이지를 몇 번 더 새로고침해야 할 수 있습니다.
</p>


<br>
<br>

## 특별한 권한(Cool Kids)을 가진 사용자만 허용하기
이번 시나리오에서는 공유 보드 목록에 대한 쓰기 권한을 더욱 강화하여, 특정 그룹의 사용자만 게시물을 작성(POST)할 수 있도록 차단해 보겠습니다. 오직 서비스 메시 설정을 변경하는 것만으로 이 작업을 매우 간단하게 처리할 수 있습니다.

<blockquote>
<i class="fa fa-terminal"></i> 다음 명령어를 사용하여 인가 정책(AuthorizationPolicy)을 적용합니다:
</blockquote>

```execute
oc create -f ./config/istio/authorization-boards-shared-lockdown.yaml
```

해당 구성 파일의 내용은 다음과 같습니다:
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: boards-shared-lockdown
spec:
  selector:
    matchLabels:
      app: boards
      deploymentconfig: boards
  rules:
  - from:
    - source:
        requestPrincipals: ["*"]
    to:
    - operation:
        methods: ["POST"]
        paths: ["*/shareditems"]
    when:
    - key: request.auth.claims[realm_access_roles]
      values: ["cool-kids"]
  - from:
    - source:
        requestPrincipals: ["*"]
    to:
    - operation:
        methods: ["GET"]
    when:
    - key: request.auth.claims[scope]
      values: ["openid"]
```

인가 정책(AuthorizationPolicy)은 셀렉터(selector)와 규칙(rules) 목록으로 구성됩니다. 셀렉터는 이 정책이 적용될 **대상(target)**을 지정합니다 (이 경우 boards 마이크로서비스). 규칙은 **누가(who/from)**, **무엇을(what/to)**, **어떤 조건(which/when)** 하에 수행할 수 있는지 정의합니다. 이 예제에서는 전달받은 JWT 페이로드에 "cool-kids" 역할이 포함되어 있는 소스라면 누구나 POST 요청을 보낼 수 있도록 지정하고 있습니다. 

이제 해당 규칙이 배포되었으므로, 권한이 없는 계정 상태에서 보드 서비스에 접근했을 때 어떻게 제한되는지 테스트해 보겠습니다.

<br>

<blockquote>
<i class="fa fa-desktop"></i> 웹 애플리케이션의 공유 보드 페이지('Shared')로 이동합니다.
</blockquote>
<blockquote>
<i class="fa fa-desktop"></i> 공유 보드에 새 글을 추가해 봅니다.
</blockquote>

이전에 작성된 항목들은 정상적으로 표시되지만, 새로운 항목을 게시(POST)하려고 하면 다음과 같이 에러가 발생하며 차단됩니다.

<blockquote>
<i class="fa fa-desktop"></i> 만약 여전히 게시가 가능하다면, 규칙이 서비스 메시에 전파될 때까지 1분 정도 기다렸다가 다시 시도해 주세요.
</blockquote>

<img src="images/app-boardsshared-failedpost.png" width="1024" class="screenshot"><br/>

<br>

<blockquote>
<i class="fa fa-desktop"></i> 이제 사용자 아이디 "theterminator", 비밀번호 "illbeback"으로 다시 로그인합니다.
</blockquote>

<blockquote>
<i class="fa fa-desktop"></i> 공유 보드에 다시 한번 글을 등록해 봅니다.
</blockquote>

이제 새 항목이 공유 보드 목록에 성공적으로 등록되어 나타나는 것을 볼 수 있습니다.

<img src="images/app-boardsshared-successpost.png" width="1024" class="screenshot"><br/>

<br>

<blockquote>
<i class="fa fa-terminal"></i> 모든 설정을 원래대로 되돌려 놓겠습니다:
</blockquote>

```execute
oc delete authorizationpolicy/boards-shared-lockdown
```

```execute
oc delete requestauthentication/boards-jwt
```

```execute
oc delete serviceentry/keycloak-egress
```


## 작동 원리
사용자가 로그인하면 app-ui 마이크로서비스가 Keycloak SSO로부터 JWT를 발급받습니다. 로그인 상태에서는 app-ui가 호출하는 모든 서비스의 요청 헤더(request header)에 이 JWT가 항상 동반되어 전송됩니다. 우리의 모든 서비스에는 Envoy 사이드카 프록시가 작동하고 있어서, 요청 헤더의 JWT를 포함해 드나드는 모든 트래픽을 실시간으로 감시할 수 있습니다. 우리가 적용한 인가 정책 설정들은 이러한 사이드카 프록시들이 특정한 정책 규칙들을 따르도록 통제합니다. 로그인하지 않았을 때는 보낼 JWT 자체가 존재하지 않았기 때문에 요청이 차단되었습니다. 유효한 사용자로 로그인했을 때 비로소 정상적인 JWT가 전송되었습니다. 또한 사용자별로 발급되는 JWT의 세부 내용이 다르기 때문에, *when* 조건(cool-kids 역할 요구)은 오직 theterminator 사용자 계정의 JWT에만 매칭되었고, 이로 인해 해당 계정에서만 공유 보드로의 쓰기 작업(POST)이 수락된 것입니다.

시간이 여유롭고 깊이 있게 알아보고 싶다면, [Keycloak 구성 방식][4] 및 [JWT 규격][3]에 대해 자세히 알아보세요.

방금 테스트한 요청 흐름의 상위 다이어그램을 확인해 보세요:

<br>
*요청 실패 시나리오*
<img src="images/architecture-jwtfail.png" width="600" class="architecture"><br/>
<br>
*요청 성공 시나리오*
<img src="images/architecture-jwtsuccess.png" width="800" class="architecture"><br/>
<p><i class="fa fa-info-circle"></i>
참고: 이 다이어그램들은 개념 설명을 위해 간소화되었습니다.
</p>


## 인가 정책(Authorization Policy) 요약
* 동적으로 변경 가능한 정책을 통해 인가 제어 수행
* 사용자-서비스(user-to-service) 통신 영역의 무결성 확보
* 업계 표준인 JSON Web Tokens(JWT) 규격 활용
* HTTP, HTTPS, HTTP2뿐만 아니라 일반적인 순수 TCP 프로토콜도 네이티브로 지원
* 정책을 통해 대상(target), 대상이 할 수 있는 액션(who/what), 작동 조건(condition)을 구체적으로 지정 가능 - [다양한 예제는 여기를 참고하세요][1]
* 개별 마이크로서비스가 공통적으로 개발해야 했던 보안 관련 중복 로직 부담을 인프라 플랫폼 레이어로 끌어올려, 코드가 아닌 환경 설정 수준에서 일관성 있게 처리 가능

<i class="fa fa-info-circle"></i>
Istio 1.4 이전 버전의 RBAC을 사용해 본 경험이 있다면, 보안 제어가 이전보다 훨씬 직관적이고 쉬워졌다는 점을 바로 느끼셨을 것입니다. 다음 블로그 포스트에서 이 둘의 차이점을 잘 다루고 있습니다: https://istio.io/blog/2019/v1beta1-authorization-policy/

[1]: https://archive.istio.io/v1.4/docs/concepts/security/#authorization
[2]: https://www.keycloak.org/docs/latest/server_admin/#_clients
[3]: https://en.wikipedia.org/wiki/JSON_Web_Token
[4]: https://www.keycloak.org/docs/latest/securing_apps/
[5]: https://istio.io/docs/reference/config/policy-and-telemetry/templates/authorization/
