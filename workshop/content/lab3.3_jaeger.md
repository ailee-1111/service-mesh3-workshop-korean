# Jaeger를 이용한 분산 추적 (Distributed Tracing with Jaeger)

[Jaeger][1]는 서비스 메시를 통과하는 요청들의 흐름을 추적할 수 있도록 돕는 분산 추적 도구입니다. 이는 마이크로서비스 아키텍처에서 발생하는 성능 문제를 디버깅할 때 엄청난 강점을 발휘합니다. 

## Jaeger 살펴보기 (Explore Jaeger)

먼저 Jaeger 사용자 인터페이스를 살펴보겠습니다.

<blockquote>
<i class="fa fa-desktop"></i>
Jaeger 콘솔을 엽니다. Jaeger의 엔드포인트를 확인합니다: 
</blockquote>

```execute
echo $(oc get route jaeger -n %username%-istio --template='https://{{.spec.host}}')
```

액세스 승인 요청 창이 뜨면 'Allow selected permissions'를 클릭합니다.

<blockquote>
브라우저에서 이 URL로 이동합니다. OpenShift에 접속할 때 제공받은 자격 증명(ID/PW)과 동일한 자격 증명으로 로그인합니다. 
</blockquote>

로그인하면 다음과 같은 Jaeger 콘솔 화면이 나타납니다:

<img src="images/jaeger-welcome.png" width="1024"><br/>
*Jaeger 환영 화면*

<br>

메시 내부에서 요청이 어떻게 흐르는지 알아보기 위해 추적 데이터를 생성해야 합니다.

<blockquote>
<i class="fa fa-terminal"></i>
애플리케이션 사용자 인터페이스(UI)에 부하를 전송합니다:
</blockquote>

```execute
for ((i=1;i<=100;i++)); do curl -s -o /dev/null $GATEWAY_URL; done
```

<br>
여러분의 서비스에 생성된 추적 데이터를 검사해 보겠습니다.  

왼쪽 바의 **Search** 아래에서, 'Service'로 `app-ui.%username%`를 선택하고, **Operation**으로 `boards-%username%.svc.cluster.local`를 선택합니다.  

다음과 같이 표시되어야 합니다:

<img src="images/jaeger-search-boards.png" width="400"><br/>
*Boards 서비스 추적 검색*

<br>

<blockquote>
<i class="fa fa-desktop"></i>
'Find Traces' 버튼을 클릭하면 Jaeger가 Boards 서비스로 연결되는 추적 목록을 새로 로드합니다.
</blockquote>

<img src="images/jaeger-boards-traces.png" width="1024"><br/>
*Boards 서비스로의 추적 내역*

<br>

<blockquote>
<i class="fa fa-desktop"></i>
이 추적 목록 중 하나를 선택합니다.  
</blockquote>

표시된 정보 중에 'Duration' 및 'Total Spans'가 포함되어 있는 것을 볼 수 있습니다. 'Duration'은 이 추적을 완료하여 응답을 보내고 받기까지 걸린 총 시간을 의미합니다. 'Total Spans'는 생성된 스팬(span)의 총 개수를 가리키며, 각각의 스팬은 이 추적 과정에서 수행된 개별 작업 단위를 나타냅니다. 아래의 예시에서, 'app-ui'는 총 9.52ms가 소요되었고 그중 5.14ms는 boards 서비스를 호출하는 데 소요되었습니다. boards 서비스 자체는 응답을 반환하기까지 3.56ms가 소요되었습니다.

<img src="images/jaeger-boards-example.png" width="1024"><br/>
*Boards 서비스 예시*

<br>

스팬을 직접 클릭하여 개별 스팬에 대한 더 자세한 정보를 볼 수 있습니다.  

<blockquote>
<i class="fa fa-desktop"></i>
다음과 같이 트리 최하단의 스팬 2개를 확장합니다:
</blockquote>

<img src="images/jaeger-boards-expanded.png" width="1024"><br/>
*확장된 Boards 서비스 뷰*

각 스팬은 전체 소요 시간 대비 해당 스팬의 실행 시간 및 시작 시간을 보여줍니다. 'Tags' 아래에서는 HTTP URL, 메서드, 응답 결과와 같은 추가 정보를 볼 수 있습니다. 마지막으로, 이 스팬을 실행한 프로세스의 실제 IP를 확인할 수 있습니다. 이 IP들이 실제 트래픽을 처리한 파드(Pod)의 IP와 일치하는지 검증할 수 있습니다.

<br>

<blockquote>
<i class="fa fa-terminal"></i>
app-ui 파드의 IP를 확인합니다:
</blockquote>

```execute
oc get pods -l deploymentconfig=app-ui -o jsonpath='{.items[*].status.podIP}{"\n"}'
```

<blockquote>
<i class="fa fa-terminal"></i>
boards 파드의 IP를 확인합니다:
</blockquote>

```execute
oc get pods -l deploymentconfig=boards -o jsonpath='{.items[*].status.podIP}{"\n"}'
```

파드 IP는 여러분이 확장한 스팬 내의 프로세스 IP와 일치해야 합니다.

<br>

## 사용자 프로필 디버깅하기 (Debug User Profile)

우리가 배운 내용을 토대로 사용자 프로필 서비스의 성능 문제를 디버깅해 보겠습니다.

<blockquote>
<i class="fa fa-terminal"></i>
사용자 프로필 서비스에 부하를 전송합니다:
</blockquote>

```execute
for ((i=1;i<=5;i++)); do curl -s -o /dev/null $GATEWAY_URL/profile; done
```

<p><i class="fa fa-info-circle"></i> 프로필 서비스가 느리게 작동하므로 완료될 때까지 기다리십시오.</p>

<br>
추적 결과를 확인합니다.  
<br>

왼쪽 바의 **Search** 아래에서, **Service**로 `app-ui.%username%`를 선택하고, 'Operation'으로 `userprofile-%username%.svc.cluster.local`를 선택합니다.  
</blockquote>

<blockquote>
<i class="fa fa-desktop"></i>
'Find Traces'를 선택하면 Jaeger가 사용자 프로필 서비스로의 추적 목록을 새로 불러옵니다.
</blockquote>

<img src="images/jaeger-userprofile-traces.png" width="1024"><br/>
*사용자 프로필 서비스로의 추적 내역*

일부 추적은 매우 빠르고(몇 ms 수준), 일부 추적은 매우 느린(약 10초 수준) 것을 확인할 수 있습니다.  

<br>

<blockquote>
<i class="fa fa-desktop"></i>
먼저 빠른 추적 중 하나를 선택하고, 가장 밑에 있는 스팬을 확장해 봅니다.  
</blockquote>

다음과 같은 화면이 나타나야 합니다:

<img src="images/jaeger-userprofile-fast.png" width="1024"><br/>
*빠르게 동작한 사용자 프로필 서비스*

위의 예시에서, 추적이 완료되는 데 총 13.48ms가 소요되었습니다. 사용자 프로필 서비스 자체는 실행 후 응답을 반환하기까지 3.5ms가 걸렸습니다. 이 요청을 처리한 파드가 버전 1(v1) 사용자 프로필 서비스임을 확인할 수 있습니다.

<br>

<blockquote>
<i class="fa fa-terminal"></i>
userprofile-v1 파드의 IP를 확인합니다:
</blockquote>

```execute
oc get pods -l deploymentconfig=userprofile,version=1.0 -o jsonpath='{.items[*].status.podIP}{"\n"}'
```

파드 IP는 여러분이 확장한 스팬 내의 프로세스 IP와 일치해야 합니다.

<br>

<blockquote>
<i class="fa fa-desktop"></i>
이번에는 느린 추적 중 하나를 선택하고 가장 밑에 있는 스팬을 확장합니다.
</blockquote>

다음과 같은 화면이 나타나야 합니다:

<img src="images/jaeger-userprofile-slow.png" width="1024"><br/>
*느리게 동작한 사용자 프로필 서비스*

이 화면을 통해 요청에 소요된 총 시간이 userprofile 서비스 자체에서 소모되었다는 것을 쉽게 알 수 있습니다. 위의 예시에서는 5.23ms에 실행을 시작하여 완료하는 데 10초가 걸렸습니다. 이 요청을 처리한 파드가 버전 2(v2) 사용자 프로필 서비스임을 추가로 확인할 수 있습니다.

<br>

<blockquote>
<i class="fa fa-terminal"></i>
userprofile-v2 파드의 IP를 확인합니다:
</blockquote>

```execute
oc get pods -l deploymentconfig=userprofile,version=2.0 -o jsonpath='{.items[*].status.podIP}{"\n"}'
```

파드 IP는 여러분이 확장한 스팬 내의 프로세스 IP와 일치해야 합니다.

<br>

이 시점에서 버전 2 소스에 직접적인 성능 문제가 있다는 것이 아주 명백해졌습니다. 예시는 매우 단순했지만, 서비스 메시 내에서 여러 서비스 호출이 얽혀있는 복잡한 네트워크 환경일수록 이러한 분산 추적(Distributed Tracing) 기능은 디버깅에 엄청난 도움을 줍니다.

<br>


[1]: https://www.jaegertracing.io
