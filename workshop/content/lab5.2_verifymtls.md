# mTLS 상태 확인 및 검증
좋습니다. 이제 모든 서비스가 트래픽을 암호화하고 있으므로 Kiali에서 암호화 설정 상태를 확인해 보겠습니다.

<blockquote>
<i class="fa fa-desktop"></i>
Kiali 대시보드를 열고 (아직 열려있지 않은 경우) Graph 뷰로 이동합니다.
<br>
첫 번째 드롭다운에서 "Service graph"를 선택하고 "Display" 드롭다운에서 "Security" 체크박스가 활성화되어 있는지 확인합니다.
</blockquote>

서비스 간의 통신에 mTLS가 작동 중임을 나타내는 작은 자물쇠 모양 아이콘이 표시된 다음 스크린샷과 같은 화면을 확인할 수 있습니다.

<img src="images/kiali-mtls.png" width="1024" class="screenshot"><br/>
<br/>
<br/>

<blockquote>
<i class="fa fa-desktop"></i>
app-ui 서비스와 boards 서비스를 연결하는 선을 클릭해 봅니다.
</blockquote>

연결 세부 정보(connection details) 뷰에서도 mTLS 작동 여부가 표시되는 것을 확인할 수 있습니다:

<img src="images/kiali-mtls-connection.png" width="300"><br/>
<br/>
<br/>

<blockquote>
<i class="fa fa-terminal"></i>
그럼, 트래픽을 가로채기 위해 이전과 동일한 명령을 다시 실행해 보겠습니다:
</blockquote>

```execute
curl boards.%username%:8080/shareditems | jq
```

데이터 조회를 가져오지 못했다는 오류 출력이 발생합니다. 이는 해당 트래픽이 메시에서 확인 가능하고 신뢰할 수 있는 서비스로부터 발송된 것이 아니기 때문에 보안 mTLS 토큰 핸드셰이크를 완료할 수 없었기 때문입니다. 다음과 유사한 출력이 표시됩니다:

```
curl: (56) Recv failure: Connection reset by peer
```

## Strict 모드 끄기
다음 실습들에서 의도적으로 비보안적인 작업을 몇 가지 테스트할 예정이므로, STRICT 모드를 다시 비활성화하겠습니다.

<blockquote>
<i class="fa fa-terminal"></i>
CLI에 다음을 입력하여 지워줍니다:
</blockquote>

```execute
oc delete peerauthentication/default
oc delete dr --all
```


## mTLS 옵션에 대한 추가 정보
서비스 메시의 mTLS는 단순한 On/Off 토글 기능만이 아닙니다. 실습을 위해 단순하게 구성했지만, 특정 서비스들만 지정하여 보안을 강제하도록 설정하는 것도 쉽게 가능합니다. 특히 **PERMISSIVE** 모드는 기존의 평문(plaintext) 트래픽을 처리하던 서비스를 실시간 무중단 상태에서 순차적으로 mTLS 트래픽으로 안전하게 마이그레이션해야 할 때 매우 유용합니다. 이 모드를 사용하면 서비스가 평문 트래픽과 mTLS 트래픽을 동시에 모두 수락할 수 있습니다.

기존에 보유한 [루트 인증서, 서명 인증서 및 키][2]를 사용하여 서비스 메시를 구성하는 것도 가능합니다. 추가적인 보안 구성에 대해서는 [여기의 보안 개념 개요][1]와 [인증 정책(Authentication Policy) 섹션][3]을 참고하세요.


## 요약 및 아키텍처 리뷰
생각보다 간편하게 활용할 수 있지 않나요? 작동 원리에 대해 조금 더 자세히 살펴보겠습니다.

먼저, 사이드카 및 경계(perimeter) 프록시는 PEP(Policy Enforcement Points, 정책 시행 지점)로 작동하여 메시 외부로부터의 통신뿐만 아니라 메시 내의 서비스 간 통신(모든 클라이언트/서버 연결)을 보호합니다. 그리고 서비스 메시의 컨트롤 플레인(Control Plane)은 설정, 인증서 및 키를 통합 관리합니다. 이를 단순화한 아키텍처 다이어그램은 다음과 같습니다:

<img src="images/architecture-security.svg" width="1024"><br/>

서비스 메시 데이터 플레인(Data Plane)은 각 Envoy 사이드카 컨테이너에 구현된 PEP를 통해 서비스 간 통신을 터널링합니다. 특정 워크로드가 상호 TLS 인증을 사용하여 다른 워크로드로 요청을 보낼 때, 요청은 다음과 같이 처리됩니다:

* 아웃바운드 트래픽은 서비스 A에서 동일한 포드 내에 실행 중인 로컬 사이드카 Envoy로 경로가 재지정됩니다.
* 클라이언트 측 Envoy가 서비스 B의 서버 측 Envoy와 상호 TLS 핸드셰이크를 개시합니다. 이 핸드셰이크 과정에서 클라이언트 측 Envoy는 서버 인증서에 명시된 서비스 계정(Service Account)이 대상 서비스를 실행할 권한이 있는지 확인하는 안전한 네이밍 검증(secure naming check)을 함께 수행합니다.
* 클라이언트 측 Envoy와 서버 측 Envoy 간에 상호 TLS 연결이 수립되고, 트래픽이 클라이언트 측 Envoy에서 서버 측 Envoy로 안전하게 전송됩니다.
* 인가(Authorization) 처리가 정상적으로 끝나면 서버 측 Envoy는 로컬 TCP 연결을 통해 트래픽을 최종 서비스 B로 전달합니다.

<p>
<i class="fa fa-info-circle"></i>
이 모든 과정은 YAML을 통해 설정 가능하며(컨트롤 플레인이 각 Envoy 사이드카의 설정을 자동으로 업데이트함), 기존 서비스들을 다시 빌드하거나 재배포할 필요가 전혀 없습니다.
</p>


## mTLS 요약
* 각 서비스에 강력한 신원/ID(Identity) 제공
* 키 및 인증서의 생성, 배포, 순환(rotation)을 자동화하는 키 관리 시스템 제공
* 서비스 간(service-to-service) 보안 무결성 확보

이러한 모든 작동을 유기적으로 연결하는 아키텍처는 다소 정교하고 복잡합니다. 더 상세한 개념을 공부하고 싶다면 [보안 개요 페이지][1]에서 시작하는 것을 추천합니다.

보안에 관한 자주 묻는 질문(FAQ)은 [이 페이지][4]를 확인해 보시기 바랍니다.


[1]: https://istio.io/docs/concepts/security/
[2]: https://istio.io/docs/tasks/security/plugin-ca-cert/
[3]: https://istio.io/docs/concepts/security/#authentication-policies
[4]: https://istio.io/faq/security/
