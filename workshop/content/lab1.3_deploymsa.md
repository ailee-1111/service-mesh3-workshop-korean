# 서비스 메시로 애플리케이션 배포하기 (Deploying an App into the Service Mesh)

이제 마이크로서비스 애플리케이션을 배포할 시간입니다. 이번에 작업할 애플리케이션은 사용자가 공유 보드에 댓글을 게시할 수 있는 페이스트 보드(paste board) 애플리케이션입니다. 다음은 아키텍처 다이어그램입니다:


<img src="images/architecture-highlevel.png" width="800"><br/>

*애플리케이션 아키텍처*

마이크로서비스는 싱글 사인온(SSO), 사용자 인터페이스(UI), 보드 애플리케이션, 컨텍스트 스크래퍼(context scraper)를 포함합니다. 이 시나리오에서는 이러한 서비스들을 배포한 다음 새로운 사용자 프로필(user profile) 서비스를 추가할 예정입니다.

<br>

## 마이크로서비스 배포하기 (Deploy Microservices)

소스 코드로부터 애플리케이션 이미지를 빌드한 다음 클러스터에 리소스를 배포합니다.

소스 파일명은 '{microservice}-fromsource.yaml'과 같습니다. 각 파일에는 서비스 메시에 사이드카 프록시를 주입하도록 지시하는 'sidecar.istio.io/inject' 어노테이션이 추가되어 있습니다.

<blockquote>
<i class="fa fa-terminal"></i>
'app-ui' 파일의 어노테이션을 확인합니다:
</blockquote>

```execute
cat config/app/app-ui-fromsource.yaml | grep -B 1 sidecar.istio.io/inject
```

출력 결과:
```
	annotations:
	  sidecar.istio.io/inject: "true"
```

<br>

이제 마이크로서비스를 배포해 보겠습니다.

<blockquote>
<i class="fa fa-terminal"></i>
boards 서비스를 배포합니다:
</blockquote>

```execute
oc new-app -f ./config/app/boards-fromsource.yaml \
  -p APPLICATION_NAME=boards \
  -p NODEJS_VERSION_TAG=16-ubi8 \
  -p GIT_URI=https://github.com/RedHatGov/service-mesh-workshop-code.git \
  -p GIT_BRANCH=workshop-stable \
  -p DATABASE_SERVICE_NAME=boards-mongodb \
  -p MONGODB_DATABASE=boardsDevelopment
```

<blockquote>
<i class="fa fa-terminal"></i>
context scraper 서비스를 배포합니다:
</blockquote>

```execute
oc new-app -f ./config/app/context-scraper-fromsource.yaml \
  -p APPLICATION_NAME=context-scraper \
  -p NODEJS_VERSION_TAG=16-ubi8 \
  -p GIT_BRANCH=workshop-stable \
  -p GIT_URI=https://github.com/RedHatGov/service-mesh-workshop-code.git
```

<blockquote>
<i class="fa fa-terminal"></i>
사용자 인터페이스(UI)를 배포합니다:
</blockquote>

```execute
oc new-app -f ./config/app/app-ui-fromsource.yaml \
  -p APPLICATION_NAME=app-ui \
  -p NODEJS_VERSION_TAG=16-ubi8 \
  -p GIT_BRANCH=workshop-stable \
  -p GIT_URI=https://github.com/RedHatGov/service-mesh-workshop-code.git \
  -e FAKE_USER=true
```

<blockquote>
<i class="fa fa-terminal"></i>
마이크로서비스 데모 설치 상태를 모니터링합니다:
</blockquote>

```execute
oc get pods --watch
```

<br>

몇 분간 기다립니다. 'app-ui', 'boards', 'context-scraper' 파드가 실행 중인 것을 확인할 수 있어야 합니다. 예시:

```
NAME                                    READY   STATUS      RESTARTS   AGE
app-ui-1-build                          0/1     Completed   0          64m
app-ui-1-xxxxx                          2/2     Running     0          62m
app-ui-1-deploy                         0/1     Completed   0          62m
boards-1-xxxxx                          2/2     Running     0          62m
boards-1-build                          0/1     Completed   0          64m
boards-1-deploy                         0/1     Completed   0          62m
boards-mongodb-1-xxxxx                  2/2     Running     0          64m
boards-mongodb-1-deploy                 0/1     Completed   0          64m
context-scraper-1-build                 0/1     Completed   0          64m
context-scraper-1-xxxxx                 2/2     Running     0          62m
context-scraper-1-deploy                0/1     Completed   0          62m
rhsso-operator-xxxxxxxxx-xxxxx          1/1     Running     0          15h
```

<br>

각 마이크로서비스 파드는 애플리케이션 자체와 Istio 프록시라는 두 개의 컨테이너를 실행합니다.

<blockquote>
<i class="fa fa-terminal"></i>
'app-ui' 파드 내부의 컨테이너를 출력합니다:
</blockquote>

```execute
oc get pods -l app=app-ui -o jsonpath='{.items[*].spec.containers[*].name}{"\n"}'
```

출력 결과:
```
app-ui istio-proxy
```

<br>

## 애플리케이션 액세스하기 (Access Application)

애플리케이션이 배포되었습니다! 하지만 사용자 인터페이스를 통해 애플리케이션에 접속할 방법이 필요합니다.

Istio는 서비스 메시 외곽(edge)에 로드 밸런서를 구성할 수 있는 [Gateway][1] 리소스를 제공합니다. 다음 단계는 Gateway 리소스를 배포하고 애플리케이션 사용자 인터페이스로 라우팅되도록 로드 밸런서를 구성하는 것입니다.

<blockquote>
<i class="fa fa-terminal"></i>
게이트웨이 구성 및 라우팅 규칙을 생성합니다:
</blockquote>

```execute
oc create -f ./config/istio/gateway.yaml
```

애플리케이션에 접속하려면 로드 밸런서의 엔드포인트가 필요합니다.

<blockquote>
<i class="fa fa-terminal"></i>
로드 밸런서의 URL을 가져옵니다:
</blockquote>

```execute
GATEWAY_URL=$(oc get route istio-ingressgateway -n %username%-istio --template='http://{{.spec.host}}')
echo $GATEWAY_URL
```

<blockquote>
<i class="fa fa-desktop"></i>
새 브라우저 탭에서 다음 URL로 이동합니다. 예시:
</blockquote>

```
http://istio-ingressgateway-userx-istio.apps.cluster-naa-xxxx.naa-xxxx.example.opentlc.com:6443
```

<br>

애플리케이션 사용자 인터페이스가 나타납니다. 새로운 보드를 만들고 공유 보드에 게시물을 작성해 보세요.

예시:

<img src="images/app-pasteboard.png" width="1024"><br/>
 *새로운 보드 생성*

## 요약 (Summary)

축하합니다, 마이크로서비스 애플리케이션 설치를 완료했습니다!  

주요 내용은 다음과 같습니다:

* 데모 마이크로서비스 애플리케이션은 페이스트 보드 애플리케이션입니다.
* 'sidecar.istio.io/inject' 어노테이션은 마이크로서비스 파드에 사이드카 프록시를 주입하도록 Istio에 지시합니다.
* Gateway 리소스는 서비스 메시 내부로의 인바운드 연결을 허용하도록 외곽 로드 밸런서를 구성합니다.


[1]: https://istio.io/docs/reference/config/networking/gateway/
