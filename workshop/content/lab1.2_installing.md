# 준비 작업 (Setup)

여러분은 OpenShift 클러스터에서 이 실습들을 진행하게 됩니다. 먼저 콘솔과 CLI를 통해 클러스터에 액세스할 수 있는지 테스트합니다.

## OpenShift

<blockquote>
<i class="fa fa-desktop"></i> 대시보드의 Console 탭을 확인하면 다음과 같은 화면이 표시되어야 합니다:
</blockquote>

<img src="images/openshift-welcome.png" width="1024"><br/>
 *OpenShift 시작 화면*

<br>

이 실습의 대부분은 OpenShift `oc` CLI를 사용하여 명령을 실행합니다. 


<blockquote>
<i class="fa fa-terminal"></i> 이미 웹 터미널에서 클러스터에 로그인되어 있는 상태여야 합니다.
</blockquote>

**Terminal** 탭으로 전환하고 다음 명령을 실행해 보세요:

```execute
oc whoami
```
*코드 블록의 우측 상단에 있는 재생(play) 버튼을 클릭하면 명령이 자동으로 실행됩니다.*

사용자 이름이 표시되어야 합니다: %username%.

강사가 프로젝트를 미리 구성해 놓았을 것입니다.

<blockquote>
<i class="fa fa-terminal"></i> 프로젝트 목록을 확인합니다:
</blockquote>

```execute
oc projects
```

두 개의 프로젝트가 표시되어야 합니다: 사용자 프로젝트 (예: '%username%') 및 '%username%-istio'.  

<br>

<blockquote>
<i class="fa fa-terminal"></i> 사용자 프로젝트로 전환합니다. 예시:
</blockquote>

```execute
oc project %username%
```

<br>

프로젝트를 살펴보겠습니다.

<blockquote>
<i class="fa fa-terminal"></i> 프로젝트의 파드(Pod) 목록을 확인합니다:
</blockquote>

```execute
oc get pods
```

출력 결과 (예시):

```
NAME                                    READY   STATUS    RESTARTS   AGE
rhsso-operator-xxxxxxxxx-xxxxx          1/1     Running   0          15h
```

RH-SSO 오퍼레이터(Operator)는 나중에 진행할 보안 실습에서 사용됩니다.

<br>

## 애플리케이션 코드 (Application Code)
다음으로 애플리케이션 코드의 로컬 복사본이 필요합니다.

<blockquote>
<i class="fa fa-terminal"></i> 리포지토리를 클론(Clone)합니다:
</blockquote>

```execute
git clone https://github.com/RedHatGov/service-mesh-workshop-code.git
```

<blockquote>
<i class="fa fa-terminal"></i> workshop-stable 브랜치로 체크아웃(Checkout)합니다:
</blockquote>

```execute
cd service-mesh-workshop-code && git checkout workshop-stable
```

## Istio
강사가 클러스터에 Istio를 이미 설치해 놓았을 것입니다. 클러스터에서 올바르게 실행 중인지 확인해 보겠습니다.

%username%-istio 프로젝트는 여러분을 전용으로 하는 서비스 메시입니다.

<blockquote>
<i class="fa fa-terminal"></i> 서비스 메시 프로젝트의 파드 목록을 확인합니다:
</blockquote>

```execute
oc get pods -n %username%-istio
```

출력 결과:

```
NAME                                      READY   STATUS    RESTARTS   AGE
grafana-xxxxxxxxx-xxxxx                   2/2     Running   0          5h30m
istio-egressgateway-xxxxxxxx-xxxxx        1/1     Running   0          5h30m
istio-ingressgateway-xxxxxxxxx-xxxxx      1/1     Running   0          5h30m
istio-telemetry-xxxxxxxxx-xxxxx           2/2     Running   0          5h25m
istiod-workshop-install-xxxxxxxxx-xxxxx   1/1     Running   0          5m28s
jaeger-xxxxxxxxxx-xxxxx                   2/2     Running   0          5h25m
kiali-xxxxxxxxxx-xxxxx                    1/1     Running   0          5h25m
prometheus-xxxxxxxxx-xxxxx                2/2     Running   0          5h30m
```

기본 컨트롤 플레인 컴포넌트는 Istio 데몬인 `istiod`입니다. `istiod`는 [트래픽 관리(Traffic Management)][1], [텔레메트리(Telemetry)][2], 그리고 [보안(Security)][3]을 처리합니다. `istio-ingressgateway`는 서비스 메시를 위한 로드 밸런서입니다. 다음 실습에서 마이크로서비스 애플리케이션과 함께 이를 구성해 볼 것입니다.

[1]: https://istio.io/docs/concepts/traffic-management/
[2]: https://istio.io/docs/concepts/observability/
[3]: https://istio.io/docs/concepts/security/
