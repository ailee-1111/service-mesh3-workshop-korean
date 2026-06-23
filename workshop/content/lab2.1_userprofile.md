# 마이크로서비스 빌드하기 (Building a Microservice)

<blockquote>
<i class="fa fa-desktop"></i>
브라우저에서 헤더의 'Profile' 섹션으로 이동합니다.
</blockquote>

<p><i class="fa fa-info-circle"></i> URL을 잊어버린 경우 다음 명령어로 다시 확인할 수 있습니다:</p>

```execute
echo $GATEWAY_URL
```

<br>

다음 화면이 표시되어야 합니다:

<img src="images/app-unknownuser.png" width="1024"><br/>
 *알 수 없는 프로필 페이지*

UI에 알 수 없는 사용자(unknown user)가 표시되는 이유는 애플리케이션에 프로필 서비스가 아직 없기 때문입니다. 이제 사용자 프로필을 위한 새로운 마이크로서비스를 빌드하고 이를 서비스 메시물에 추가할 것입니다.

## 애플리케이션 코드 (Application Code)

새로운 애플리케이션은 Java로 작성되었으며, 'app-ui' 및 'boards'와 같은 다른 백엔드 컴포넌트는 NodeJS로 작성되었습니다. Istio의 장점 중 하나는 실행 중인 마이크로서비스의 프로그래밍 언어에 구애받지 않는다(agnostic)는 점입니다.


<blockquote>
<i class="fa fa-terminal"></i>
리포지토리의 "UserProfile" 클래스를 살펴봅니다:
</blockquote>

```execute
cat ./code/userprofile/src/main/java/org/microservices/demo/json/UserProfile.java | grep "public UserProfile(String" -A 7
```

출력 결과:
```java
    public UserProfile(String id, String firstname, String lastname, String aboutme) {
        this.id = id;
        this.firstName = firstname;
        this.lastName = lastname;
        this.aboutMe = aboutme;
        this.createdAt = Calendar.getInstance().getTime();
    }
```

이 클래스는 사용자의 성(last name)과 이름(first name) 등의 정보를 캡슐화합니다.

<br>

또한 애플리케이션은 서비스와 상호 작용할 수 있도록 REST API를 노출합니다.

<blockquote>
<i class="fa fa-terminal"></i>
다음으로 "UserProfileService" 클래스를 살펴봅니다:
</blockquote>

```execute
cat ./code/userprofile/src/main/java/org/microservices/demo/service/UserProfileService.java | grep "UserProfile getProfile(" -B 5
```

출력 결과:
```java
    /**
     * return a specific profile
     * @param id
     * @return the specified profile
     */
    UserProfile getProfile(@NotBlank String id);
```

이 인터페이스는 사용자 프로필 정보를 조회하고 설정하기 위한 REST 메서드를 포함하고 있습니다.

<br>

## 애플리케이션 빌드하기 (Build Application)

이제 애플리케이션을 빌드할 준비가 되었습니다.  

애플리케이션 이미지를 빌드하기 위해 [BuildConfig][1]를 사용합니다. `BuildConfig` 템플릿은 이미 생성되어 있습니다.

<blockquote>
<i class="fa fa-terminal"></i>
애플리케이션 빌드에 사용되는 베이스 이미지를 확인합니다:
</blockquote>

```execute
cat ./config/app/userprofile-build.yaml | grep -A 4 sourceStrategy
```

출력 결과 (일부):
```yaml
      sourceStrategy:
        from:
          kind: ImageStreamTag
          name: java:11
          namespace: openshift
```

빌드 시 애플리케이션을 빌드하기 위해 Java 베이스 이미지를 사용하는 것을 알 수 있습니다.

<br>

<blockquote>
<i class="fa fa-terminal"></i>
빌드를 생성합니다:
</blockquote>

```execute
oc new-app -f ./config/app/userprofile-build.yaml \
  -p APPLICATION_NAME=userprofile \
  -p APPLICATION_CODE_URI=https://github.com/RedHatGov/service-mesh-workshop-code.git \
  -p APPLICATION_CODE_BRANCH=workshop-stable \
  -p APP_VERSION_TAG=1.0
```

<blockquote>
<i class="fa fa-terminal"></i>
빌드를 시작합니다:
</blockquote>

```execute
oc start-build userprofile-1.0 -F
```

빌더가 소스 코드를 컴파일하고 베이스 이미지를 사용하여 배포 가능한 이미지 아티팩트를 생성합니다. 최종적으로 빌드가 성공한 것을 확인할 수 있습니다.

출력 결과 (일부):
```
...
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  01:35 min
[INFO] Finished at: 2020-02-19T21:00:22Z
[INFO] ------------------------------------------------------------------------
...
```

<br>

빌드가 완료되면 이미지는 OpenShift 로컬 레지스트리에 저장됩니다.

<blockquote>
<i class="fa fa-terminal"></i>
이미지가 생성되었는지 확인합니다:
</blockquote>

```execute
oc get is userprofile
```

출력 결과:
```
NAME          IMAGE REPOSITORY                                                                  TAGS     UPDATED
userprofile   image-registry.openshift-image-registry.svc:5000/microservices-demo/userprofile   1.0   3 minutes ago
```

[1]: https://docs.openshift.com/container-platform/4.6/builds/understanding-buildconfigs.html
