# 메시(Mesh)에 새로운 서비스 추가하기 (Adding a New Service to the Mesh)

새로운 사용자 프로필 애플리케이션을 서비스 메시에 배포해야 합니다.

## 애플리케이션 배포하기 (Deploy Application)

애플리케이션 배포를 위해 미리 준비된 배포 파일인 'userprofile-deploy-all.yaml'을 사용합니다. 이 파일은 사용자 프로필 서비스와 이를 지원하는 PostgreSQL 데이터베이스를 생성합니다. 다른 소스 파일들과 마찬가지로, Istio가 사이드카 프록시를 주입하고 메시에 추가하도록 지시하는 'sidecar.istio.io/inject' 어노테이션이 추가되어 있습니다.

<blockquote>
<i class="fa fa-terminal"></i>
'userprofile' 파일에서 어노테이션을 확인합니다:
</blockquote>

```execute
cat ./config/app/userprofile-deploy-all.yaml | grep -B 1 sidecar.istio.io/inject
```

출력 결과:
```
    annotations:
      sidecar.istio.io/inject: "true"
  --
    annotations:
      sidecar.istio.io/inject: "true"
```

이 어노테이션은 userprofile 서비스와 PostgreSQL 서비스에 각각 한 번씩, 총 두 번 나타납니다.

<br>

서비스를 배포하기 전에, 이전 실습에서 빌드한 로컬 이미지의 참조 주소가 필요합니다.

<blockquote>
<i class="fa fa-terminal"></i>
다음 명령을 실행합니다:
</blockquote>

```execute
USER_PROFILE_IMAGE_URI=$(oc get is userprofile --template='{{.status.dockerImageRepository}}')
echo $USER_PROFILE_IMAGE_URI
```

출력 결과 (예시):
```
image-registry.openshift-image-registry.svc:5000/microservices-demo/userprofile
```

<blockquote>
<i class="fa fa-terminal"></i>
이 이미지 URI를 사용하여 서비스를 배포합니다:
</blockquote>

```execute
sed "s|%USER_PROFILE_IMAGE_URI%|$USER_PROFILE_IMAGE_URI|" ./config/app/userprofile-deploy-all.yaml | oc create -f -
```

<blockquote>
<i class="fa fa-terminal"></i>
사용자 프로필 배포 상태를 모니터링합니다:
</blockquote>

```execute
oc get pods -l deploymentconfig=userprofile --watch
```

<p>
<i class="fa fa-info-circle"></i>
PostgreSQL 파드가 아직 실행되지 않은 경우, userprofile 서비스에서 오류가 발생해 재시작될 수 있습니다.
</p>

출력 결과:
```
userprofile-xxxxxxxxxx-xxxxx              2/2     Running		    0          2m55s
```

<br>

다른 마이크로서비스와 마찬가지로, 사용자 프로필 서비스는 애플리케이션과 Istio 프록시를 함께 실행합니다.

<blockquote>
<i class="fa fa-terminal"></i>
'userprofile' 파드 내의 컨테이너를 출력합니다:
</blockquote>


```execute
oc get pods -l deploymentconfig=userprofile -o jsonpath='{.items[*].spec.containers[*].name}{"\n"}'
```

출력 결과:
```
userprofile istio-proxy
```

<br>

## 애플리케이션 액세스하기 (Access Application)

사용자 프로필 서비스 배포를 마쳤습니다! 브라우저에서 테스트해 보겠습니다.

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

<img src="images/app-profilepage.png" width="1024"><br/>
 *프로필 페이지*
