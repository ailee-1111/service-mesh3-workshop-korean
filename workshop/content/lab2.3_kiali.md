# 관찰 가능성을 위한 Kiali 소개 (Introducing Kiali for Observability)

모든 마이크로서비스가 서비스 메시에서 원활하게 작동하고 있습니다. 이제 서비스 메시 토폴로지(topology)를 시각화할 차례입니다. 즉, 서비스 메시에 어떤 것들이 실행 중이고 이들이 어떻게 연결되어 있는지 확인해야 합니다.

Istio는 서비스 메시의 콘솔 뷰를 제공하는 오픈소스 프로젝트인 [Kiali][1]를 제공합니다. Kiali를 통해 서비스 메시의 상태(health)를 모니터링할 수 있으며, 이후 실습에서 다룰 메트릭 쿼리 및 분산 추적(tracing)과의 추가적인 연동 기능도 가지고 있습니다.

## Kiali 살펴보기 (Explore Kiali)

Kiali가 배포 환경을 조회할 수 있도록 설정합니다:

```execute
oc get cm kiali -n %username%-istio -o yaml | sed '/DeploymentConfig/d' | oc apply -n %username%-istio -f -
oc rollout restart deployment kiali -n %username%-istio
```

<br>

애플리케이션에 트래픽 부하(load)를 보내봅시다.

<blockquote>
<i class="fa fa-terminal"></i>
애플리케이션 사용자 인터페이스(UI)에 부하를 전송합니다:
</blockquote>

```execute
for ((i=1;i<=100;i++)); do curl -s -o /dev/null $GATEWAY_URL; done
```

<blockquote>
<i class="fa fa-terminal"></i>
사용자 프로필 서비스에 부하를 전송합니다:
</blockquote>

```execute
for ((i=1;i<=100;i++)); do curl -s -o /dev/null $GATEWAY_URL/profile; done
```

<br>

<blockquote>
<i class="fa fa-terminal"></i>
이제 Kiali 콘솔을 열어보겠습니다. Kiali의 엔드포인트를 확인합니다:
</blockquote>


```execute
echo $(oc get route kiali -n %username%-istio --template='https://{{.spec.host}}')
```

출력 결과 (예시):
```
https://kiali-userx-istio.apps.cluster-naa-xxxx.naa-xxxx.example.opentlc.com
```

<blockquote>
<i class="fa fa-desktop"></i>
브라우저에서 이 URL로 이동합니다. OpenShift에 접속할 때 제공받은 자격 증명(ID/PW)과 동일한 자격 증명으로 로그인합니다.
</blockquote>

로그인에 성공하면 Kiali 콘솔 화면이 나타납니다:

<img src="images/kiali-welcome.png" width="1024"><br/>
*Kiali 환영 화면*

서비스 메시 토폴로지를 확인해 보겠습니다.  

<br>

<blockquote>
<i class="fa fa-desktop"></i>
왼쪽 내비게이션 바에서 'Graph'로 이동하고 본인의 네임스페이스(예: user1)를 선택합니다.
</blockquote>

<blockquote>
<i class="fa fa-desktop"></i>
우측 상단의 조회 기간 창을 'Last 1m'에서 'Last 10m'로 변경합니다.
</blockquote>

<img src="images/kiali-graph.png" width="1024"><br/>
*Kiali 그래프*

그래프에는 서비스 메시의 마이크로서비스들과 이들 사이의 연결 관계가 표시됩니다.

에지 레이블(edge labels)을 통해 서비스 간에 전송되는 트래픽 정보를 조사할 수 있습니다.

<br>

<blockquote>
<i class="fa fa-desktop"></i>
'Display'를 클릭하고 'Show Edge Labels' 아래에서 'Request Rate'를 선택합니다.
</blockquote>

이제 마이크로서비스 간의 HTTP 트래픽 정보를 볼 수 있습니다.

<img src="images/kiali-rpsgraph.png" width="1024"><br/>
*요청 속도가 표시된 Kiali 그래프*

서비스 메시에서 실행 중인 마이크로서비스들을 좀 더 자세히 살펴보겠습니다.  

<br>

<blockquote>
<i class="fa fa-terminal"></i>
왼쪽 내비게이션 바에서 'Applications'로 이동합니다.
</blockquote>

<img src="images/kiali-apps.png" width="1024"><br/>
*Kiali 애플리케이션 뷰*

<br>

<blockquote>
<i class="fa fa-terminal"></i>
이 뷰에서 각 마이크로서비스의 상세 정보를 확인할 수 있습니다. 'app-ui'를 선택해 보세요.
</blockquote>


<img src="images/kiali-appui.png" width="1024"><br/>
*Kiali App UI 뷰*

해당 마이크로서비스의 'Health(상태)'를 확인할 수 있습니다.  

<br>

<blockquote>
<i class="fa fa-terminal"></i>
'Traffic' 탭으로 이동하면 해당 마이크로서비스의 인바운드 및 아웃바운드 호출 내역을 확인할 수 있습니다.
</blockquote>

<img src="images/kiali-appuitraffic.png" width="1024"><br/>
*Kiali App UI 인바운드 및 아웃바운드 트래픽 뷰*

<br>

[1]: https://kiali.io
