# 결함 주입(Fault Injection)을 통한 회복탄력성 테스트 (Testing Resiliency with Fault Injection)

새로운 버전의 사용자 프로필 서비스 덕분에 현재 애플리케이션은 훌륭하게 작동하고 있습니다. 하지만 이전 버전은 성능 문제를 야기했으며, 향후 업데이트 시 애플리케이션의 다른 영역에서 문제가 발생할 가능성도 있습니다. 장애가 발생했을 때 애플리케이션이 어떻게 작동하는지 어떻게 테스트할 수 있을까요?

이를 해결하기 위해 서비스 메시 내에서 오류를 인위적으로 시뮬레이션하는 방법이 필요합니다. 그렇게 함으로써 기능이 저하된 상태(degraded state)에서도 애플리케이션이 올바르게 작동하는지 검증할 수 있습니다. 이 개념은 일반적으로 [카오스 엔지니어링(Chaos Engineering)][1]으로 잘 알려져 있습니다. 카오스 엔지니어링에서는 운영 환경에서 인위적으로 장애를 일으켜 소프트웨어가 잘 견디는지 테스트합니다.

Istio는 Virtual Service를 사용하여 애플리케이션 계층에서 [지연 결함(Delay Faults)][2]과 [중단 결함(Abort Faults)][3]을 주입할 수 있는 방법을 제공합니다. 페이스트 보드 애플리케이션을 통해 이를 직접 시도해 봅시다.

## 중단 결함 (Abort Faults)

<blockquote>
<i class="fa fa-terminal"></i>
가장 선호하는 에디터나 bash를 통해 Virtual Service 설정을 확인합니다:
</blockquote>

```execute
cat ./config/istio/virtual-service-userprofile-503.yaml
```

출력 결과 (일부):
```yaml
...
  http:
  - fault:
      abort:
        httpStatus: 503
        percent: 50
    route:
    - destination:
        host: userprofile
        subset: v3
...
```

이 구성은 사용자 프로필 서비스로 전송되는 트래픽의 50%에 인위적으로 503 오류를 주입하도록 Istio에 지시합니다.

<blockquote>
<i class="fa fa-terminal"></i>
라우팅 규칙을 배포합니다:
</blockquote>

```execute
oc apply -f ./config/istio/virtual-service-userprofile-503.yaml
```

<blockquote>
<i class="fa fa-terminal"></i>
(이전 실습에서 진행한 상태가 아닌 경우) 사용자 프로필 서비스에 부하를 전송합니다:
</blockquote>

```execute
while true; do curl -s -o /dev/null $GATEWAY_URL/profile; done
```

<br>

Kiali에서 변경 사항을 검사합니다. 
<blockquote>
<i class="fa fa-desktop"></i>
왼쪽 내비게이션 바에서 'Graph'로 이동합니다. 
</blockquote>

<blockquote>
<i class="fa fa-desktop"></i>
'Versioned app graph' 뷰로 전환하고 'Last 1m'으로 변경합니다.
</blockquote>
<blockquote>
<i class="fa fa-desktop"></i>
'No edge labels' 드롭다운을 'Request Distribution'으로 변경합니다.
</blockquote>

<blockquote>
<i class="fa fa-desktop"></i>
'app-ui' (정사각형)와 'userprofile' 서비스 (삼각형) 사이의 빨간색 연결선을 클릭합니다.  
</blockquote>

대략 50%의 요청이 HTTP 오류를 반환하고 있는 것을 볼 수 있습니다.

<img src="images/kiali-userprofile-503.png" width="1024"><br/>
*중단 결함(Abort Faults)이 적용된 Kiali 그래프*

브라우저에서 애플리케이션을 테스트해 보겠습니다.

<blockquote>
<i class="fa fa-desktop"></i>
브라우저에서 헤더의 'Profile' 섹션으로 이동합니다.  
</blockquote>

<p><i class="fa fa-info-circle"></i> URL을 분실한 경우 다음 명령으로 확인할 수 있습니다:</p>

```execute
echo $GATEWAY_URL
```

<blockquote>
<i class="fa fa-desktop"></i>
브라우저를 몇 번 새로고침해 봅니다.  
</blockquote>

어떤 때는 프로필 페이지가 정상적으로 로드되지만, 다른 때는 'Unknown User'가 표시됩니다.

중단 결함 주입은 애플리케이션이 오류를 어떻게 처리하는지 검증할 수 있는 탁월한 메커니즘입니다. 보다 복잡한 서비스 메시 환경에서는 오류가 발생한 단일 서비스가 다른 여러 서비스들까지 연쇄적으로 실패하게 만드는 '연쇄 장애(cascading failures)' 현상을 식별하고 사전에 예방하는 데 유용하게 활용할 수 있습니다.

<br>

## 지연 결함 (Delay Faults)

사용자 프로필 서비스 버전 2에는 성능 문제가 있었습니다. Istio의 지연 결함 기능을 사용하여 이 시나리오를 가상으로 시뮬레이션할 수 있습니다.

<blockquote>
<i class="fa fa-terminal"></i>
가장 선호하는 에디터나 bash를 통해 Virtual Service 설정을 확인합니다:
</blockquote>

```execute
cat ./config/istio/virtual-service-userprofile-delay.yaml
```

출력 결과 (일부):
```yaml
...
  http:
  - fault:
      delay:
        fixedDelay: 5s
        percent: 50
    route:
    - destination:
        host: userprofile
        subset: v3
...
```

이 구성은 사용자 프로필 서비스로 전송되는 트래픽의 50%에 인위적으로 5초의 지연(delay)을 주입하도록 Istio에 지시합니다.

<blockquote>
<i class="fa fa-terminal"></i>
라우팅 규칙을 배포합니다:
</blockquote>

```execute
oc apply -f ./config/istio/virtual-service-userprofile-delay.yaml
```

<blockquote>
<i class="fa fa-terminal"></i>
아직 부하를 전송하고 있지 않은 경우, 사용자 프로필 서비스에 부하를 전송합니다:
</blockquote>

```execute
while true; do curl -s -o /dev/null $GATEWAY_URL/profile; done
```

<br>

<blockquote>
<i class="fa fa-desktop"></i>
Jaeger에서 추적 내역을 검사해 보겠습니다.
</blockquote>

Jaeger 탭을 열고, **Service**를 `userprofile.%username%`, 그리고 **Find Traces**를 클릭합니다.

어떤 추적은 지연 시간이 5초 정도인 반면, 다른 추적은 밀리초(ms) 단위인 것을 알 수 있습니다.

지연 결함 주입은 외부 서비스를 호출할 때 발생하는 느린 반응에 애플리케이션이 어떻게 대처하는지 검증할 수 있는 좋은 방법입니다.


<img src="images/kiali-userprofile-faultdelaytraces.png" width="1024"><br/>
*지연 결함이 적용된 사용자 프로필 서비스의 추적 내역*

## 정리 작업 (Clean up)

<blockquote>
<i class="fa fa-terminal"></i>
이 실습을 마치기 전에 변경했던 설정들을 원상복구합니다:
</blockquote>

```execute
oc apply -f ./config/istio/virtual-service-userprofile-v3.yaml
```

<br>

## 요약 (Summary)

축하합니다, Istio를 사용한 결함 주입 구성을 완료했습니다!

주요 내용은 다음과 같습니다:

* 지연 결함(Delay Faults)과 중단 결함(Abort Faults)을 활용해 서비스 메시의 회복탄력성을 손쉽게 테스트할 수 있습니다.
* 중단 결함은 서비스 호출에 응답할 때 인위적으로 50x 에러를 주입합니다.
* 지연 결함은 서비스 호출 응답에 인위적으로 지연(latency)을 추가합니다.
* Jaeger 분산 추적 기능을 통해 이러한 성능 지연을 시각적으로 포착할 수 있습니다.

[1]: https://en.wikipedia.org/wiki/Chaos_engineering
[2]: https://istio.io/docs/tasks/traffic-management/fault-injection/#injecting-an-http-delay-fault
[3]: https://istio.io/docs/tasks/traffic-management/fault-injection/#injecting-an-http-abort-fault
