# 관찰 가능성(Observability) 깊이 알아보기 (Digging into Observability)

Istio는 서비스 메시와 그 성능을 분석할 수 있는 추가적인 기능을 제공합니다. 사용자 프로필 서비스의 새로운 버전을 배포하고 이것이 서비스 메시에 미치는 영향을 분석해 보겠습니다.

## 기능 업데이트 (Feature Update)

코드는 이미 저장소의 'workshop-feature-update' 브랜치에 작성되어 있습니다.

<blockquote>
<i class="fa fa-terminal"></i>
이 기능 브랜치에서 새로운 빌드를 생성합니다:
</blockquote>

```execute
oc new-app -f ./config/app/userprofile-build.yaml \
  -p APPLICATION_NAME=userprofile \
  -p APPLICATION_CODE_URI=https://github.com/RedHatGov/service-mesh-workshop-code.git \
  -p APPLICATION_CODE_BRANCH=workshop-feature-update \
  -p APP_VERSION_TAG=2.0
```

<p><i class="fa fa-info-circle"></i> 이미지스트림(imagestream)이 이미 존재하므로 실패 메시지는 무시하셔도 됩니다.</p>

<br>

<blockquote>
<i class="fa fa-terminal"></i>
빌드를 시작합니다:
</blockquote>

```execute
oc start-build userprofile-2.0 -F
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

<br>

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
2.0
  no spec tag

  * image-registry.openshift-image-registry.svc:5000/microservices-demo/userprofile@sha256:147d836e9f7331a27b26723cbb99f2b667e176b4d5dd356fea947c7ca4fc24a6
      2 minutes ago

1.0
  no spec tag

  * image-registry.openshift-image-registry.svc:5000/microservices-demo/userprofile@sha256:f01d00409f44962ab321517e18fb06483fadfc07b2f70c088f567acf20dc65eb
      23 hours ago
```

<p><i class="fa fa-info-circle"></i> 최신 이미지에는 '2.0' 태그가 지정되어 있어야 합니다.</p>

<br>

<blockquote>
<i class="fa fa-terminal"></i>
로컬 이미지에 대한 참조 주소를 가져옵니다:
</blockquote>

```execute
USER_PROFILE_IMAGE_URI=$(oc get is userprofile --template='{{.status.dockerImageRepository}}')
echo $USER_PROFILE_IMAGE_URI
```

출력 결과 (예시):
```
image-registry.openshift-image-registry.svc:5000/microservices-demo/userprofile
```

<br>

애플리케이션 배포를 위해 'userprofile-deploy-v2.yaml' 배포 파일이 미리 준비되어 있습니다.

<blockquote>
<i class="fa fa-terminal"></i>
이미지 URI를 사용하여 서비스를 배포합니다:
</blockquote>

```execute
sed "s|%USER_PROFILE_IMAGE_URI%|$USER_PROFILE_IMAGE_URI|" ./config/app/userprofile-deploy-v2.yaml | oc create -f -
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
userprofile-2-xxxxxxxxxx-xxxxx            2/2     Running        0          22s
userprofile-xxxxxxxxxx-xxxxx              2/2     Running        0          2m55s
```

<br>

## 애플리케이션 액세스하기 (Access Application)

브라우저에서 프로필 서비스의 새 버전을 테스트해 보겠습니다 (스포일러: 버그가 추가되었습니다).

<blockquote>
<i class="fa fa-desktop"></i>
브라우저에서 헤더의 'Profile' 섹션으로 이동합니다.  
</blockquote>

<p><i class="fa fa-info-circle"></i> URL을 분실한 경우 다음 명령으로 확인할 수 있습니다:</p>

```execute
echo $GATEWAY_URL
```

<br>

프로필 페이지는 버전 1과 버전 2 사이를 라운드 로빈(round robin) 방식으로 번갈아 호출합니다. 버전 2는 매우 느리게 로드되며 다음과 같이 표시됩니다:

<img src="images/app-profilepage-v2.png" width="1024"><br/>
 *프로필 페이지*

다음으로, 이 문제를 디버깅하기 위해 서비스 메시를 사용할 것입니다.
