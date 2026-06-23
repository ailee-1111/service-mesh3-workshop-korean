# Grafana를 이용한 서비스 메시 메트릭(Metrics) 분석 (Service Mesh Metrics with Grafana)

[Grafana][1]는 메트릭 관찰을 위해 Istio와 통합할 수 있는 모니터링 도구입니다. Grafana를 사용하면 서비스 메시 내의 서비스들과 관련된 메트릭을 살펴볼 수 있습니다. Grafana를 사용하여 사용자 프로필 서비스에 대한 더 많은 정보를 얻어보겠습니다.

## Grafana 살펴보기 (Explore Grafana)

먼저 Grafana 사용자 인터페이스를 살펴보겠습니다.

<blockquote>
<i class="fa fa-terminal"></i>
Grafana 콘솔을 엽니다. Grafana의 엔드포인트를 확인합니다:
</blockquote>

```execute
echo $(oc get route grafana -n %username%-istio --template='https://{{.spec.host}}')
```
<p><i class="fa fa-info-circle"></i> 액세스 승인 요청 창이 뜨면 'Allow selected permissions'를 클릭합니다.</p>

<blockquote>
<i class="fa fa-desktop"></i>
새 브라우저 탭에서 이 URL로 이동합니다. OpenShift에 접속할 때 제공받은 자격 증명(ID/PW)과 동일한 자격 증명으로 로그인합니다.
</blockquote>

로그인하면 다음과 같은 Grafana 콘솔 화면이 나타납니다:

<img src="images/grafana-welcome.png" width="600"><br/>
*Grafana 환영 화면*

<br>

<blockquote>
<i class="fa fa-desktop"></i>
왼쪽 바에서 위에서 두 번째 아이콘(Dashboards)에 마우스를 올리고 'Manage'를 선택합니다. 'istio' 폴더를 확장합니다.
</blockquote>

다음과 같이 표시되어야 합니다:

<img src="images/grafana-istio.png" width="1024"><br/>
*Grafana Istio 대시보드 목록*

<br>

메트릭을 보기 전에 애플리케이션에 트래픽 부하를 먼저 전송해야 합니다.

<blockquote>
<i class="fa fa-terminal"></i>
애플리케이션 사용자 인터페이스(UI)에 부하를 전송합니다:
</blockquote>

```execute
while true; do curl -s -o /dev/null $GATEWAY_URL; done
```

<br>

부하 전송이 실행되는 동안, 대시보드를 살펴보겠습니다.

<blockquote>
<i class="fa fa-desktop"></i>
Grafana에서 'Istio Mesh Dashboard'를 선택합니다.
</blockquote>

다음과 같이 표시되어야 합니다:

<img src="images/grafana-istio-mesh.png" width="1024"><br/>
*Grafana Istio Mesh 대시보드*

<br>

이를 통해 서비스들과 관련된 전반적인 메트릭을 한눈에 파악할 수 있습니다. 예를 들어, 전체적으로 에러 응답이 없다는 것을 알 수 있으며 각 서비스의 처리량(throughput)과 지연 시간(latency)을 빠르게 요약하여 보여줍니다. 하지만 아직 사용자 프로필 서비스에 대한 데이터가 누락되어 있으므로 해당 서비스에도 부하를 전송해 보겠습니다.

<blockquote>
<i class="fa fa-terminal"></i>
터미널에서 다른 탭을 엽니다. 사용자 프로필 서비스에 부하를 전송합니다:
</blockquote>

```execute-2
GATEWAY_URL=$(oc get route istio-ingressgateway -n %username%-istio --template='http://{{.spec.host}}')
while true; do curl -s -o /dev/null $GATEWAY_URL/profile; done
```

Mesh 대시보드가 동적으로 업데이트됩니다. 이제 다음과 같이 보여야 합니다:

<img src="images/grafana-istio-mesh-updated.png" width="1024"><br/>
*업데이트된 Grafana Istio Mesh 대시보드*

<br>

userprofile 서비스에 두 가지의 서로 다른 워크로드(userprofile(버전 1) 및 userprofile-2)가 있는 것을 확인할 수 있습니다. userprofile-2로의 호출이 엄청나게 느립니다. 서비스 대시보드를 선택하여 서비스와 연관된 메트릭을 더 자세히 조사할 수 있습니다.

<blockquote>
<i class="fa fa-desktop"></i>
'Service' 열에서 userprofile FQDN에 마우스를 올린 다음 선택합니다.
</blockquote>

다음과 같이 서비스 뷰 화면으로 이동합니다:

<img src="images/grafana-istio-service.png" width="1024"><br/>
*Grafana Istio Service 대시보드*

<br>

이것은 사용자 프로필 서비스 전용 메트릭입니다. 'Service Workloads' 아래로 스크롤하면 해당 서비스의 서로 다른 워크로드 버전별 세부 분석 내용을 볼 수 있습니다.

<blockquote>
<i class="fa fa-desktop"></i>
'Service Workloads' 아래의 'Incoming Request Duration by Source'에 마우스를 올려봅니다.
</blockquote>

다음과 같이 표시되어야 합니다:

<img src="images/grafana-istio-service-duration.png" width="1024"><br/>
*Grafana Istio Service 대시보드 - 요청 소요 시간*

<br>

이는 Mesh 대시보드에서 보았던 지연 시간을 시각적으로 명확하게 나타내 주며, 버전 2의 지연 시간이 훨씬 더 높다는 사실을 확실하게 보여줍니다.

<br>


[1]: https://grafana.com
