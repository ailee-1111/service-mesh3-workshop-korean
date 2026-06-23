# 서비스 버전 간 트래픽 분할 (Splitting Traffic Amongst Service Versions)

이제 애플리케이션의 성능 문제를 수정할 차례입니다. 앞선 실습에서는 새로운 버전을 배포하고 모든 트래픽(100%)을 해당 신규 버전으로 곧바로 전송했습니다. 이번에는 Istio 트래픽 라우팅을 사용해 카나리(canary) 배포를 수행하고 트래픽을 세밀하게 분할해 보겠습니다.

## 기능 수정 (Feature Fix)

사용자 프로필 서비스의 성능 문제를 수정한 코드가 이미 'workshop-feature-fix' 브랜치에 작성되어 있습니다.  

<blockquote>
<i class="fa fa-terminal"></i>
이 기능 브랜치에서 새로운 빌드를 생성합니다:
</blockquote>

```execute
oc new-app -f ./config/app/userprofile-build.yaml \
  -p APPLICATION_NAME=userprofile \
  -p APPLICATION_CODE_URI=https://github.com/RedHatGov/service-mesh-workshop-code.git \
  -p APPLICATION_CODE_BRANCH=workshop-feature-fix \
  -p APP_VERSION_TAG=3.0
```

<p><i class="fa fa-info-circle"></i> 이미지스트림(imagestream)이 이미 존재하므로 빌드 생성 시의 실패 오류 메시지는 무시하셔도 됩니다.</p>

<blockquote>
<i class="fa fa-terminal"></i>
빌드를 시작합니다:
</blockquote>

```execute
oc start-build userprofile-3.0 -F
```

빌더가 소스 코드를 컴파일하고 베이스 이미지를 사용하여 배포 가능한 이미지 아티팩트를 생성합니다. 최종적으로 빌드가 성공한 것을 확인할 수 있습니다.

출력 결과 (일부):
```
...
[INFO] [io.quarkus.deployment.pkg.steps.JarResultBuildStep] Building thin jar: /tmp/src/target/userprofile-1.0-SNAPSHOT-runner.jar
[INFO] [io.quarkus.deployment.QuarkusAugmentor] Quarkus augmentation completed in 7988ms
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  01:41 min
[INFO] Finished at: 2020-02-24T19:13:59Z
[INFO] ------------------------------------------------------------------------...
```

빌드가 완료되면 이미지는 OpenShift 로컬 레지스트리에 저장됩니다.

<blockquote>
<i class="fa fa-terminal"></i>
이미지가 생성되었는지 확인합니다:
</blockquote>

```execute
oc describe is userprofile
```

출력 결과 (일부):
```
...

3.0
  no spec tag

  * image-registry.openshift-image-registry.svc:5000/microservices-demo/userprofile@sha256:da74d277cc91c18226fb5cf8ca25d6bdbbf3f77a7480d0583f23023fb0d0d7df
      12 seconds ago

2.0
  no spec tag

  * image-registry.openshift-image-registry.svc:5000/microservices-demo/userprofile@sha256:147d836e9f7331a27b26723cbb99f2b667e176b4d5dd356fea947c7ca4fc24a6
      16 minutes ago
...
```

최신 이미지에는 '3.0' 태그가 있어야 합니다.

<blockquote>
<i class="fa fa-terminal"></i>
로컬 이미지 참조 주소를 가져옵니다:
</blockquote>

```execute
USER_PROFILE_IMAGE_URI=$(oc get is userprofile --template='{{.status.dockerImageRepository}}')
echo $USER_PROFILE_IMAGE_URI
```

출력 결과 (예시):
```
image-registry.openshift-image-registry.svc:5000/microservices-demo/userprofile
```

애플리케이션 배포를 위해 'userprofile-deploy-v3.yaml' 배포 파일이 준비되어 있습니다.

<blockquote>
<i class="fa fa-terminal"></i>
이미지 URI를 사용하여 서비스를 배포합니다:
</blockquote>

```execute
sed "s|%USER_PROFILE_IMAGE_URI%|$USER_PROFILE_IMAGE_URI|" ./config/app/userprofile-deploy-v3.yaml | oc create -f -
```

<blockquote>
<i class="fa fa-terminal"></i>
사용자 프로필 배포 상태를 모니터링합니다:
</blockquote>

```execute
oc get pods -l deploymentconfig=userprofile --watch
```

출력 결과:
```
userprofile-3-xxxxxxxxxx-xxxxx              2/2     Running        0          53s
userprofile-2-xxxxxxxxxx-xxxxx              2/2     Running        0          13m
userprofile-xxxxxxxxxx-xxxxx                2/2     Running        0          22h
```

<br>

## 트래픽 라우팅 (Traffic Routing)

사용자 프로필 서비스 신규 버전에 대한 [카나리 배포(Canary Release)][1]를 시작해 보겠습니다. 사용자 트래픽의 90%는 버전 1로 보내고, 10%는 최신 버전(v3)으로 라우팅되도록 설정합니다.

<blockquote>
<i class="fa fa-terminal"></i>
가장 선호하는 에디터나 bash를 통해 Virtual Service 설정을 확인합니다:
</blockquote>

```execute
cat ./config/istio/virtual-service-userprofile-90-10.yaml
```

출력 결과 (일부):
```
...
---
  http:
  - route:
    - destination:
        host: userprofile
        subset: v1
      weight: 90
    - destination:
        host: userprofile
        subset: v3
      weight: 10 
---
...
```

'weight(가중치)' 파라미터가 각 서비스 서브셋으로 전송할 트래픽의 비율을 결정합니다.

<blockquote>
<i class="fa fa-terminal"></i>
라우팅 규칙을 배포합니다:
</blockquote>

```execute
oc apply -f ./config/istio/virtual-service-userprofile-90-10.yaml
```

<blockquote>
<i class="fa fa-terminal"></i>
(Grafana 실습 등에서 이미 실행 중이지 않은 경우) 사용자 프로필 서비스에 지속적으로 부하를 전송합니다:
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

<p><i class="fa fa-info-circle"></i> URL을 분실한 경우 다음 명령으로 확인할 수 있습니다:</p>

`echo $KIALI_CONSOLE`

<blockquote>
<i class="fa fa-desktop"></i>
'Versioned app graph' 뷰로 전환하고 'Last 1m'으로 변경합니다.  
</blockquote>
<blockquote>
<i class="fa fa-desktop"></i>
'No edge labels' 드롭다운을 'Request Distribution'으로 변경합니다.  
</blockquote>

트래픽이 사용자 프로필 서비스의 버전 1과 버전 3 사이에서 대략 90%와 10% 비율로 나뉘는 것을 볼 수 있습니다.

<img src="images/kiali-userprofile-90-10.png" width="1024"><br/>
*90:10 트래픽 분할이 적용된 Kiali 그래프*

이를 통해 모든 사용자에게 한꺼번에 영향을 주지 않고, 극히 일부의 사용자에 한해 사용자 프로필의 새로운 기능을 안전하게 격리하여 검증할 수 있습니다.  

신규 기능 검증에 성공하여 확신이 선다면, 최신 버전으로 보내는 트래픽 비율을 늘릴 수 있습니다.


<blockquote>
<i class="fa fa-terminal"></i>
가장 선호하는 에디터나 bash를 통해 Virtual Service 설정을 확인합니다:
</blockquote>

```execute
cat ./config/istio/virtual-service-userprofile-50-50.yaml
```

출력 결과 (일부):
```
...
---
  http:
  - route:
    - destination:
        host: userprofile
        subset: v1
      weight: 50
    - destination:
        host: userprofile
        subset: v3
      weight: 50
---
...
```

이 예시에서는 두 버전 간에 트래픽을 정확히 절반씩 균등하게(50:50) 라우팅합니다. 이는 A/B 테스트 같은 고급 배포 방식에 적용할 수 있는 유용한 기술입니다.

<blockquote>
<i class="fa fa-terminal"></i>
라우팅 규칙을 배포합니다:
</blockquote>

```execute
oc apply -f ./config/istio/virtual-service-userprofile-50-50.yaml
```

<blockquote>
<i class="fa fa-terminal"></i>
아직 부하를 전송하고 있지 않은 경우, 사용자 프로필 서비스에 부하를 전송합니다:
</blockquote>

```execute
while true; do curl -s -o /dev/null $GATEWAY_URL/profile; done
```

<br>

Kiali에서 변경 사항을 다시 검사합니다.

<blockquote>
<i class="fa fa-desktop"></i>
왼쪽 내비게이션 바에서 'Graph'로 이동합니다.
</blockquote>
<blockquote>
<i class="fa fa-desktop"></i>
'Versioned app graph' 뷰로 전환합니다. 'No edge labels' 드롭다운을 'Request Distribution'으로 변경합니다.  
</blockquote>

사용자 프로필 서비스의 버전 1과 버전 3 사이에 대략 50:50 비율로 트래픽이 분할되어 있는 것을 볼 수 있습니다.

<img src="images/kiali-userprofile-50-50.png" width="1024"><br/>
*50:50 트래픽 분할이 적용된 Kiali 그래프*

마지막 단계로, 모든 사용자가 이 신규 버전을 사용하도록 전환할 준비가 되었습니다.

<br>

<blockquote>
<i class="fa fa-terminal"></i>
가장 선호하는 에디터나 bash를 통해 Virtual Service 설정을 확인합니다:
</blockquote>

```execute
cat ./config/istio/virtual-service-userprofile-v3.yaml
```

출력 결과 (일부):
```
...
---
  http:
  - route:
    - destination:
        host: userprofile
        subset: v3
---
...
```

<blockquote>
<i class="fa fa-terminal"></i>
라우팅 규칙을 배포합니다:
</blockquote>

```execute
oc apply -f ./config/istio/virtual-service-userprofile-v3.yaml
```

<blockquote>
<i class="fa fa-terminal"></i>
아직 부하를 전송하고 있지 않은 경우, 사용자 프로필 서비스에 부하를 전송합니다:
</blockquote>

```execute
while true; do curl -s -o /dev/null $GATEWAY_URL/profile; done
```

<br>

Kiali에서 변경 사항을 다시 검사합니다.
<blockquote>
<i class="fa fa-desktop"></i>
왼쪽 내비게이션 바에서 'Graph'로 이동합니다. 
</blockquote>
<blockquote>
<i class="fa fa-desktop"></i>
'Versioned app graph' 뷰로 전환합니다. 'No edge labels' 드롭다운을 'Request Distribution'으로 변경합니다.  
</blockquote>


이제 모든 트래픽이 사용자 프로필 서비스의 v3 버전으로 온전히 흐르는 것을 볼 수 있습니다.

<img src="images/kiali-userprofile-v3.png" width="1024"><br/>
*v3 라우팅이 적용된 Kiali 그래프*

<br>

브라우저에서 새 버전의 프로필 서비스를 최종 테스트해 보겠습니다.

<blockquote>
<i class="fa fa-desktop"></i>
브라우저에서 헤더의 'Profile' 섹션으로 이동합니다.  
</blockquote>

<p><i class="fa fa-info-circle"></i> URL을 분실한 경우 다음 명령으로 확인할 수 있습니다:</p>

```execute
echo $GATEWAY_URL
```

<br>

다음 화면이 표시되어야 합니다:

<img src="images/app-profilepage-v3.png" width="1024"><br/>
 *프로필 페이지*

<br>

## 요약 (Summary)

축하합니다, Istio를 사용한 트래픽 분할 구성을 성공적으로 마쳤습니다!

주요 내용은 다음과 같습니다:

* Virtual Service에서 'weight' 파라미터를 수정하여 다른 서비스 버전으로 전송되는 트래픽의 백분율 비율을 손쉽게 조정할 수 있습니다.
* Kiali 서비스 그래프는 트래픽이 서비스 메시 내에서 흐르는 대로 트래픽 분할 상태를 동적이고 직관적으로 포착해 줍니다.

[1]: https://martinfowler.com/bliki/CanaryRelease.html
[2]: https://martinfowler.com/bliki/BlueGreenDeployment.html
