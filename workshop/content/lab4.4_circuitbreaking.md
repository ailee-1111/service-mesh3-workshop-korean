# 서비스의 서킷 브레이킹 (Circuit Breaking)

결함 주입(Fault Injection)을 사용하면 특정 서비스에 대한 네트워크 호출에 장애가 발생했을 때 서비스 메시가 어떻게 작동하는지 확인할 수 있습니다. 하지만 트래픽을 처리하는 인스턴스에 과부하가 걸리거나 오류가 발생할 때 서비스를 어떻게 보호할 수 있을까요? 가장 이상적인 방법은 오류가 발생하는 인스턴스를 식별하고, 특정 임계값(threshold)에 도달하면 클라이언트가 해당 인스턴스에 연결하지 못하도록 차단하는 것입니다.

OpenShift에서 인스턴스는 마이크로서비스를 실행 중인 쿠버네티스 포드(Pod)에 해당합니다.

이 개념을 **서킷 브레이킹(Circuit Breaking)**이라고 합니다. 마이크로서비스를 실행하는 인스턴스에 임계값 제한을 설정하고, 이 임계값에 도달하면 서킷 브레이커가 "트립(trips, 차단)"되어 Istio가 해당 인스턴스로의 추가 연결을 방지합니다. 서킷 브레이킹은 서비스 메시 내에서 회복 탄력성(resiliency)이 뛰어난 서비스를 구축하는 또 다른 방법입니다.

Istio에서는 데스티네이션 룰(Destination Rule)을 사용하여 서킷 브레이킹 제한을 정의할 수 있습니다.

## 임계값 제한 정의하기

사용자 프로필 서비스(user profile service)를 위한 서킷 브레이킹 규칙이 이미 작성되어 있습니다.

<blockquote>
<i class="fa fa-terminal"></i>
가장 선호하는 에디터나 bash를 통해 데스티네이션 룰(destination rule)을 확인해 보세요:
</blockquote>

```execute
cat ./config/istio/destinationrule-circuitbreaking.yaml
```

Output (snippet):
```
...
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
    connectionPool:
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      consecutiveErrors: 1
      interval: 1s
      baseEjectionTime: 10m
      maxEjectionPercent: 100
...
```

이 서킷 브레이킹 규칙은 사용자 프로필 서비스의 v3에만 적용됩니다. 연결 풀(connection pool) 설정은 각 인스턴스에 대한 최대 요청 수를 1개로 제한합니다 (데모 목적으로 서킷 브레이커를 쉽게 트리거할 수 있도록 하기 위함입니다). 아웃라이어 감지(outlier detection) 설정은 임계값을 정의합니다. 50x 에러가 한 번 발생하는 인스턴스는 10분 동안 메시에서 축출(ejection)됩니다. 다양한 설정에 대해서는 Istio [공식 문서][1]를 참조해 보세요.

<blockquote>
<i class="fa fa-terminal"></i>
이 서킷 브레이킹 규칙을 배포합니다:
</blockquote>

```execute
oc apply -f ./config/istio/destinationrule-circuitbreaking.yaml
```

<br>

## 서킷 브레이커 작동시키기 (Trip)

<blockquote>
<i class="fa fa-terminal"></i>
먼저, 사용자 프로필 서비스의 v1과 v3 사이에 트래픽을 50:50으로 균등하게 라우팅합니다.
</blockquote>

```execute
oc apply -f ./config/istio/virtual-service-userprofile-50-50.yaml
```

<blockquote>
<i class="fa fa-terminal"></i>
사용자 프로필 서비스에 부하를 전달합니다:
</blockquote>

```execute
while true; do curl -s -o /dev/null $GATEWAY_URL/profile; done
```

<blockquote>
<i class="fa fa-terminal"></i>
터미널의 다른 탭에서, v3 버전의 사용자 프로필 서비스를 실행 중인 서버 프로세스를 종료(kill)합니다:
</blockquote>

```execute-2
USERPROFILE_POD=$(oc get pod -l deploymentconfig=userprofile,version=3.0 -o jsonpath='{.items[0].metadata.name}')
oc exec $USERPROFILE_POD -- kill 1
```

<br>

Kiali에서 변경 사항을 검사합니다.  
<blockquote>
<i class="fa fa-desktop"></i>
왼쪽 내비게이션 바에서 'Graph'로 이동합니다.
</blockquote>

<blockquote>
<i class="fa fa-desktop"></i>
'Versioned app graph' 뷰로 전환하고 기간을 'Last 1m'으로 변경합니다. 'No edge labels' 드롭다운을 'Request Distribution'으로 변경합니다.  
</blockquote>

<img src="images/kiali-circuitbreaking.png" width="1024"><br/>
*결함 지연이 포함된 사용자 프로필 서비스의 추적(Traces)*

트래픽 비율이 점차 v3에서 v1으로 우회되는 것을 확인할 수 있습니다. 번개 모양 아이콘은 서킷 브레이킹 규칙이 설정되었음을 나타내며, 서킷 브레이커가 작동(tripped)되어 트래픽이 v1으로 라우팅되었습니다.

OpenShift는 헬스 체크가 실패하면 서버를 자동으로 되살리려고(revive) 시도합니다. 트래픽이 다시 분산되기 시작하면 서버 프로세스를 종료하는 명령을 다시 실행해 보세요.

<br>

## 정리하기

<blockquote>
<i class="fa fa-terminal"></i>
이 실습을 종료하기 전에 구성 변경 사항을 원래대로 되돌립니다.
</blockquote>

```execute
oc apply -f ./config/istio/destinationrules-all.yaml
oc apply -f ./config/istio/virtual-services-default.yaml
```

<br>

## 요약

축하합니다! Istio에서 서킷 브레이킹 설정을 완료하셨습니다.

주요 요약 내용은 다음과 같습니다:

* 서킷 브레이킹은 비정상적인 서비스 인스턴스에 대한 연결을 트립(차단)함으로써 서비스 메시 내에서 회복 탄력성(resiliency)을 높일 수 있습니다.
* 서킷을 작동시키기 위한 임계값 제한은 데스티네이션 룰(Destination Rule)에서 설정할 수 있습니다.

[1]: https://istio.io/docs/reference/config/networking/destination-rule/#OutlierDetection
