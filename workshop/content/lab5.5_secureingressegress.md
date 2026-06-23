# 인그레스 및 이그레스 보안
엄격한 보안 기준이 요구되는 환경에서는 인그레스(Ingress, 외부 유입) 및 이그레스(Egress, 내부 유출) 트래픽 제어에 관한 구체적인 보안 설정을 구성해야 합니다. 특히 이그레스 보안은 내부 네트워크망 외부의 잠재적으로 유해한 리소스로 접근하는 경로를 차단하고 통제할 때 유용하게 쓰입니다. 또한 클러스터 내부에서 시작될 수 있는 악의적인 활동들이 외부로 뻗어나가지 못하도록 방지하는 것도 보안 상 모범 사례(Best Practice)입니다.

기본적인 인그레스 보안 개념은 이미 잘 알고 계실 것입니다. 본질적으로 클러스터 외부에서 특정 서비스만 접근할 수 있도록 노출하고, 기본적인 TLS/SSL 암호화를 적용하는 것입니다. 서비스 메시에는 독립형 Envoy 프록시로 실행되는 인그레스 라우터가 기본 탑재되어 있으며, 우리는 "서비스 메시에 애플리케이션 배포하기" 실습 진행 중에 이미 이를 구성한 바 있습니다.

우리의 마이크로서비스로 유입되는 트래픽을 한층 더 고도화하여 추적하고 차단하는 더 나은 방법은, 모든 API 서비스의 전면에 전문적인 API 관리 솔루션을 배치하는 것입니다. 이는 3scale 서비스 메시 플러그인을 사용하여 구현할 수 있습니다. 본 실습에서는 이 과정을 다루지 않으나, 관심이 있으시다면 [여기][7] 및 [여기][8]에서 관련 문서를 자세히 읽어보시기 바랍니다.

지금부터는 이그레스(Egress) 트래픽을 통제해 보겠습니다.

## 이그레스 트래픽 제어하기 (Lock Down)
이 예제에서는 사전에 승인된 호스트 명으로만 외부 엔드포인트에 접속할 수 있도록 외부 접근 권한을 엄격히 제한하겠습니다. 서비스 메시에는 독립형 Envoy 프록시로 동작하는 이그레스 라우터가 기본 실행되고 있어, 이를 알맞게 구성해 주기만 하면 됩니다.

Istio를 설치하기 위해 사용했던 `ServiceMeshControlPlane` 커스텀 리소스에는 이그레스 보안의 기본 동작 방식을 제어할 수 있는 `global` 설정이 포함되어 있습니다. 다음과 같은 형태입니다:

```
# Set the default behavior of the sidecar for handling outbound traffic from the application:
# ALLOW_ANY - outbound traffic to unknown destinations will be allowed, in case there are no
#   services or ServiceEntries for the destination port
# REGISTRY_ONLY - restrict outbound traffic to services defined in the service registry as well
#   as those defined through ServiceEntries  
outboundTrafficPolicy:
  mode: REGISTRY_ONLY
```

현재 실습에서 사용 중인 서비스 메시는 이미 해당 모드가 `REGISTRY_ONLY`로 지정되어 있습니다. 자동 생성된 컨피그맵(ConfigMap)에서 이 설정 상태를 직접 검증해 보겠습니다:

<blockquote>
<i class="fa fa-terminal"></i>
다음 명령을 실행하고 `outboundTrafficPolicy` 설정을 찾아보세요:
</blockquote>

```execute
oc describe cm/istio-workshop-install -n %username%-istio | grep outboundTrafficPolicy -A 1
```

Output:

```
outboundTrafficPolicy:
  mode: REGISTRY_ONLY
```

<blockquote>
<i class="fa fa-terminal"></i>
동작이 잘 되고 있는지 검증해 보겠습니다. 외부 데이터를 스크랩해 오는 다음 명령을 호출해 봅니다:
</blockquote>

```execute
curl context-scraper.%username%:8080/scrape/custom_search?term==skynet | jq
```

아래 출력 결과와 같이 `ECONNRESET` 오류를 동반한 응답이 표시되는 것을 볼 수 있습니다:

```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   682  100   682    0     0  42590      0 --:--:-- --:--:-- --:--:-- 45466
{
  "oops": {
    "name": "RequestError",
    "message": "Error: read ECONNRESET",
    "cause": {
      "errno": "ECONNRESET",
      "code": "ECONNRESET",
      "syscall": "read"
    },
    "error": {
      "errno": "ECONNRESET",
      "code": "ECONNRESET",
      "syscall": "read"
    },
    "options": {
      "method": "GET",
      "uri": "https://www.googleapis.com/customsearch/v1/siterestrict",
      "qs": {
        "key": "AIzaSyDRdgirA2Pakl4PMi7t-8LFfnnjEFHnbY4",
        "cx": "005627457786250373845:lwanzyzfwji",
        "q": "=skynet"
      },
      "headers": {
        "user-agent": "curl/7.29.0",
        "x-request-id": "07ce0cef-5b07-9ee7-8565-a9705bfd90b1",
        "x-b3-traceid": "663d4c9ff4336678433857d48c7a8b5b",
        "x-b3-spanid": "433857d48c7a8b5b",
        "x-b3-sampled": "1"
      },
      "json": true,
      "simple": true,
      "resolveWithFullResponse": false,
      "transform2xxOnly": false
    }
  }
}
```

이는 curl 명령어가 메시 내부에 속한 `context-scraper` 마이크로서비스와 정상적으로 통신하려고 한 결과입니다. 오류 메시지의 세부 정보를 살펴보면 해당 서비스가 외부 주소인 `googleapis.com`으로 아웃바운드 접근을 수행하려고 했으나, 서비스 메시 보안 정책에 의해 접속이 강력히 차단되었음을 알 수 있습니다.

<br>

<blockquote>
<i class="fa fa-desktop"></i>
(선택 사항) 만약 Kiali를 살펴보면 요청이 블랙홀(blackhole) 상태로 흘러 들어가는 모습을 직접 시각적으로 확인할 수 있습니다:
</blockquote>

<img src="images/kiali-egress-blackhole.png" width="1024" class="screenshot"><br/>

<br>

## 승인된 호스트에만 이그레스 통신 허용하기
이제 통제된 방식의 이그레스 아웃바운드 접속을 허용하기 위해, Istio의 API 오브젝트 종류 중 하나인 [서비스 엔트리(ServiceEntry)][5]를 구성하겠습니다. 이 ServiceEntry를 추가해 주면, 메시 내부의 마이크로서비스(와 이에 탑재된 Envoy 사이드카)가 지정된 외부 서비스 주소를 마치 메시 내부 서비스 중 하나인 것처럼 인식하고 성공적으로 트래픽을 전송할 수 있게 됩니다.

작성된 ServiceEntry의 형태는 다음과 같습니다:
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: context-scraper-egress
spec:
  exportTo:
  - "."
  hosts:
  - www.googleapis.com
  ports:
  - number: 443
    name: https
    protocol: HTTPS
  resolution: DNS
  location: MESH_EXTERNAL
```

<br>

<blockquote>
<i class="fa fa-terminal"></i>
다음 명령어로 이 설정을 배포 및 적용합니다:
</blockquote>

```execute
oc apply -f ./config/istio/serviceentry-googleapis.yaml
```

<br>

<blockquote>
<i class="fa fa-terminal"></i>
외부 데이터 스크래핑 명령을 다시 호출해 봅니다:
</blockquote>

```execute
curl context-scraper.%username%:8080/scrape/custom_search?term==skynet | jq
```

이제 차단이 정상적으로 해제되어 다음과 같이 외부 스크랩 결과가 응답 결과 목록으로 잘 출력되는 것을 볼 수 있습니다:

```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  5197  100  5197    0     0    865      0  0:00:06  0:00:06 --:--:--  1157
[
  {
    "link": "https://www.reddit.com/r/DeepBrainChain/comments/8rar5b/deepbrain_chain_launches_skynet_project/",
    "thumbnail": [
      {
        "src": "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcQrwBIFnOZygv1EKJ1z7TnNeWIfom8Kp0Zgbs6YvM4
DXYP0zvra6GNCnh0",
        "width": "225",
        "height": "225"
      }
    ],
    "title": "DeepBrain Chain Launches ''Skynet Project''— Recruiting AI ...",
    "snippet": "Nov 29, 2017 ... To achieve that, DeepBrain Chain Foundation will activate the ''Skynet Project
'' \non June 15th Beijing time and, recruit AI computing power from ..."
  },
  {
    "link": "https://www.reddit.com/r/Terminator/comments/eo6qgf/what_would_happen_if_skynet_wins_would_they_bu
ild/",
    "thumbnail": [
      {
        "src": "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcRq6RYcGTf2e6W8pH3h962NB0_0DNTi93EeqEdtktl
mLoALKvdA1rUY_AU",
        "width": "132",
        "height": "92"
      }
    ],
    "title": "What would happen if Skynet wins? Would they build a world for ...",
    "snippet": "Skynet is an accident and many ways an unstable AI, which makes it different \nfrom the termina
tors it created. We get a hint of this in TSCC of a second machine\n ..."
  },
  ...
```

<br>

더 나아가 세분화된 통제가 필요하다면, 이 서비스 엔트리(ServiceEntry) 주소에 대해서도 가상 서비스(VirtualService) 및 데스티네이션 룰(DestinationRule)을 추가적으로 설정하여 트래픽 전송 규칙을 정교하게 다듬는 것 역시 메시 내부의 일반 서비스를 다룰 때와 완벽히 동일하게 가능합니다.

Kiali 역시 이 서비스 엔트리의 접속 여부를 완벽히 모니터링하여 추적합니다. Kiali의 Graph 화면에서 접속 패턴이 어떻게 시각화되어 표시되는지 확인해 보세요.

<br>

<blockquote>
<i class="fa fa-desktop"></i>
(선택 사항) 이제 Kiali의 Graph 뷰를 열어보면 승인된 외부 통신 대상 서비스가 노드로 명확히 식별되어 표기되는 것을 볼 수 있습니다.
<br>
</blockquote>

<img src="images/kiali-egress.png" width="1024" class="screenshot"><br/>

<br>

# 요약
축하합니다! 특정 외부 호스트에 대한 이그레스 아웃바운드 차단을 성공적으로 테스트하고, 서비스 메시를 통한 접속 상태 추적 구성을 정상적으로 마쳤습니다. 이는 서비스 메시의 고급 기능 중 하나로, 더 자세한 보안 설정에 대해 관심이 있다면 [인그레스(Ingress) 설정법][1] 및 [이그레스(Egress) 설정법][2] 관련 문서를 참고해 보세요.

주요 요약 내용은 다음과 같습니다:

* 사전에 승인되고 정의된 외부 엔드포인트 혹은 서비스 목록으로만 이그레스 트래픽이 통과되도록 차단할 수 있습니다.
* 클러스터 외부의 승인된 원격지 혹은 경로로만 서비스가 외부에 노출되도록 인그레스 트래픽을 통제할 수 있습니다.
* 일반 표준 TLS 및 상호 mTLS 인증 등을 연계하여 유입 경로의 암호화 수준을 보장할 수 있습니다.
* 3scale API 관리 시스템을 활용하여 공개된 API 게이트웨이의 접근 대상자와 통제 규칙을 유연하게 제어할 수 있습니다.


[1]: https://istio.io/docs/tasks/traffic-management/ingress/
[2]: https://istio.io/docs/tasks/traffic-management/egress/
[3]: https://istio.io/docs/ops/best-practices/security/
[4]: https://istio.io/docs/tasks/traffic-management/ingress/secure-ingress-mount/
[5]: https://istio.io/docs/concepts/traffic-management/#service-entries
[6]: https://archive.istio.io/v1.4/docs/tasks/traffic-management/egress/egress-control/
[7]: https://docs.openshift.com/container-platform/4.3/service_mesh/threescale_adapter/threescale-adapter.html
[8]: https://www.redhat.com/en/technologies/jboss-middleware/3scale
