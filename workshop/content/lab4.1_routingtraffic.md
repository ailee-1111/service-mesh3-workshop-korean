# 서비스 메시를 통한 트래픽 라우팅 기초 (Basics on Routing Traffic Through the Mesh)

Istio의 핵심 기능 중 하나는 애플리케이션 코드를 수정하지 않고 서비스 간의 통신 방식을 동적으로 제어하는 능력입니다. 이 전반적인 개념을 [트래픽 관리(Traffic Management)][1]라고 부릅니다. 이를 통해 A/B 테스트, 카나리(canary) 릴리스, 롤백(rollback) 등을 수행할 수 있습니다.

트래픽 관리를 위한 두 가지 핵심 API 개체는 [Virtual Service][2]와 [Destination Rule][3]입니다. Destination Rule은 마이크로서비스 제공자 관점의 설정으로, "내가 어떤 버전을 외부에 노출하고 트래픽이 서비스에 도달하기 전에 어떤 일이 일어날 것인가?"를 다룹니다. Virtual Service는 마이크로서비스 소비자(클라이언트) 관점의 설정으로, "마이크로서비스로 트래픽을 어떻게 라우팅하고 싶은가?"를 정의합니다.

예를 들어, Destination Rule은 서비스의 두 가지 버전(예: 'v1', 'v2')을 노출하고 각 버전에 다른 로드 밸런싱 정책을 명시할 수 있습니다. 그런 다음 Virtual Service는 클라이언트가 마이크로서비스를 호출할 때 각 버전으로 트래픽을 나눌(split) 수 있습니다.

## 트래픽 라우팅 (Traffic Routing)

트래픽 라우팅 규칙이 이미 구성되어 있습니다.

<blockquote>
<i class="fa fa-terminal"></i>
가장 선호하는 에디터나 bash를 통해 Destination Rule 설정을 확인합니다:
</blockquote>

```execute
cat ./config/istio/destinationrules-all.yaml
```

출력 결과 (일부):
```
...
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: userprofile
spec:
  host: userprofile
  subsets:
  	- name: v1
  	  labels:
  	  	version: 1.0
      trafficPolicy:
        loadBalancer:
          simple: ROUND_ROBIN        
    - name: v2
      labels:
        version: 2.0
      trafficPolicy:
        loadBalancer:
          simple: RANDOM
...
```

대부분의 Destination Rule은 나중에 실습에서 설명할 TLS 설정을 제외하고는 특별한 구성을 포함하고 있지 않습니다. 하지만 'userprofile' Destination Rule은 호출 가능한 서비스 버전인 'subsets'를 노출하고 있습니다. 버전 '1.0'과 '2.0'에 대해 다른 로드 밸런서 정책이 적용된 것을 확인할 수 있습니다. 기본적으로 Istio는 'ROUND_ROBIN' 로드 밸런싱 방식을 사용합니다.

<blockquote>
<i class="fa fa-terminal"></i>
가장 선호하는 에디터나 bash를 통해 Virtual Service 설정을 확인합니다:
</blockquote>

```execute
cat ./config/istio/virtual-services-all-v2.yaml
```

출력 결과 (일부):
```
...
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: userprofile
spec:
  hosts:
  - userprofile
  http:
  - route:
    - destination:
        host: userprofile
        subset: v2
---
...
```

대부분의 Virtual Service는 특별한 구성을 포함하고 있지 않습니다. 하지만 'userprofile' Virtual Service는 특별히 'userprofile' 마이크로서비스의 'v2' 버전으로 라우팅하도록 되어 있습니다.

<blockquote>
<i class="fa fa-terminal"></i>
이 라우팅 규칙들을 배포합니다:
</blockquote>

```execute
oc apply -f ./config/istio/destinationrules-all.yaml 
oc apply -f ./config/istio/virtual-services-all-v2.yaml
```

<blockquote>
<i class="fa fa-terminal"></i>
Destination Rule을 확인합니다:
</blockquote>

```execute
oc get dr
```

출력 결과:
```
NAME                     HOST                     AGE
app-ui                   app-ui                   24m
boards                   boards                   24m
boards-mongodb           boards-mongodb           24m
userprofile              userprofile              20m
userprofile-postgresql   userprofile-postgresql   16m
```

<blockquote>
<i class="fa fa-terminal"></i>
Virtual Service를 확인합니다:
</blockquote>

```execute
oc get virtualservice
```

출력 결과 (예시):
```
NAME                     GATEWAYS                       HOSTS                      AGE
app-ui                                                  [app-ui]                   25m
boards                                                  [boards]                   25m
boards-mongodb                                          [boards-mongodb]           25m
context-scraper                                         [context-scraper]          25m
ingressgateway          ["ingressgateway"]              [*]                        44m
userprofile                                             [userprofile]              25m
userprofile-postgresql                                  [userprofile-postgresql]   25m
```

브라우저에서 애플리케이션 UI를 테스트해 보겠습니다.

<blockquote>
<i class="fa fa-desktop"></i>
브라우저에서 헤더의 'Profile' 섹션으로 이동합니다.  
</blockquote>

<p><i class="fa fa-info-circle"></i> URL을 분실한 경우 다음 명령으로 확인할 수 있습니다:</p>

```execute
echo $GATEWAY_URL
```

이 시점에서는 아무런 변화가 없어야 합니다. 이전 실습에서 배포한 매우 느린 버전 2 사용자 프로필 서비스로 여전히 라우팅되고 있기 때문입니다.

<br>

이것이 Kiali에서 어떻게 보이는지 확인해 보겠습니다.

<blockquote>
<i class="fa fa-terminal"></i>
사용자 프로필 서비스에 부하를 전송합니다:
</blockquote>

```execute
for ((i=1;i<=5;i++)); do curl -s -o /dev/null $GATEWAY_URL/profile; done
```

<br>

<blockquote>
<i class="fa fa-desktop"></i>
Kiali 탭에서 왼쪽 내비게이션 바의 'Graph'로 이동합니다.
</blockquote>

<br>

<blockquote>
<i class="fa fa-desktop"></i>
'Versioned app graph' 뷰로 전환하고 'Last 1m'으로 변경합니다.  
</blockquote>

사용자 프로필 버전 '2.0'으로 트래픽이 흐르고 있는 것을 확인할 수 있습니다.

트래픽 흐름은 그래프에서 녹색 하이라이트로 표시됩니다.

<img src="images/kiali-userprofile-v2.png" width="1024"><br/>
*v2 라우팅이 적용된 Kiali 그래프*

<br>

## 트래픽 라우팅 변경하기 (Change Traffic Routing)

Istio의 장점 중 하나는 애플리케이션 코드를 수정하지 않고 트래픽 라우팅을 바꿀 수 있다는 점입니다. 프로필 서비스의 버전 2에 성능 문제가 있으므로 버전 1로 롤백해 보겠습니다. Virtual Service 설정만 변경하면 모든 작업이 완료됩니다.

<blockquote>
<i class="fa fa-terminal"></i>
수정된 Virtual Service 설정을 가장 선호하는 에디터나 bash를 통해 확인합니다:
</blockquote>

```execute
cat ./config/istio/virtual-service-userprofile-v1.yaml
```

출력 결과:
```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: userprofile
spec:
  hosts:
  - userprofile
  http:
  - route:
    - destination:
        host: userprofile
        subset: v1
```

이 설정에서는 Virtual Service의 'v1' 서브셋으로 라우팅하게 됩니다.

<blockquote>
<i class="fa fa-terminal"></i>
변경 사항을 배포합니다:
</blockquote>

```execute
oc apply -f ./config/istio/virtual-service-userprofile-v1.yaml
```

<blockquote>
<i class="fa fa-terminal"></i>
변경 사항을 검증합니다:
</blockquote>

```execute
oc describe virtualservice userprofile
```

출력 결과 (일부):
```
...
Spec:
  Hosts:
    userprofile
  Http:
    Route:
      Destination:
        Host:    userprofile
        Subset:  v1
Events:          <none>
```

<br>

<blockquote>
<i class="fa fa-desktop"></i>
애플리케이션 UI에서 헤더의 'Profile' 섹션으로 이동합니다.  
</blockquote>

페이지가 신속하게 로드되며 다시 사용자 프로필 서비스 버전 1로 라우팅되는 것을 볼 수 있습니다.

<br>

<blockquote>
<i class="fa fa-terminal"></i>
트래픽을 발생시키기 위해 사용자 프로필 요청 100개를 전송해 봅시다:
</blockquote>

```execute
for ((i=1;i<=100;i++)); do curl -s -o /dev/null $GATEWAY_URL/profile; done
```

<blockquote>
<i class="fa fa-desktop"></i>
이제 Kiali에서 왼쪽 내비게이션 바의 'Graph'로 이동합니다.
</blockquote>

<blockquote>
<i class="fa fa-desktop"></i>
'Versioned app graph' 뷰로 전환하고 'Last 1m'으로 변경합니다.  
</blockquote>

사용자 프로필 버전 '1.0'으로 트래픽이 흐르는 것을 확인할 수 있습니다.

트래픽 흐름은 그래프에서 녹색 하이라이트로 표시됩니다.

<img src="images/kiali-userprofile-v1.png" width="1024"><br/>
*v1 라우팅이 적용된 Kiali 그래프*

<br>

## 요약 (Summary)

축하합니다, Istio를 사용한 트래픽 라우팅 구성을 마쳤습니다!

주요 내용은 다음과 같습니다:

* 트래픽 관리를 위한 두 가지 핵심 API 개체는 Virtual Service와 Destination Rule입니다.
* Virtual Service를 수정하여 우리가 호출하는 서비스의 버전을 유연하게 변경할 수 있습니다.
* Kiali는 서비스 메시 내부에서 트래픽이 흐르는 동안 서비스 그래프 뷰를 직관적으로 제공합니다.

[1]: https://istio.io/docs/concepts/traffic-management
[2]: https://istio.io/docs/concepts/traffic-management/#virtual-services
[3]: https://istio.io/docs/concepts/traffic-management/#destination-rules
