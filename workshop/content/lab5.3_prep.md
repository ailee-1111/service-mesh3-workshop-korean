# 싱글 사인온 (Single Sign-On)
로그인, 회원가입 및 역할 기반 권한 부여(RBAC)는 Red Hat SSO(Keycloak)를 통해 처리됩니다.
SSO는 OpenShift 또는 보유하고 계신 Red Hat의 미들웨어 제품 서브스크립션의 일부로 포함되어 있습니다!

## 설치하기
서비스 메시가 사용자를 인증하고 JWT를 생성하기 위해 SSO를 활용할 것이므로 이를 설치해야 합니다. 여기서는 오퍼레이터(Operator)를 통해 여러분의 네임스페이스에 설치하게 됩니다.

### 리소스 사용자 정의
여기 있는 `.yaml` 파일들을 확인하고 실습 클러스터의 도메인 및 사용자 프로젝트에 맞게 구성하십시오. 실습 편의를 위해 SSO 보안 설정을 다소 완화해 놓았으므로, 프로덕션 환경에서는 보안 구성을 더욱 강화해야 합니다 (예: CORS 설정 및 와일드카드 사용 제한 등).

### SSO 인스턴스 생성 및 구성
이 작업은 CLI 또는 웹 콘솔에서 수행할 수 있습니다. 웹 콘솔을 사용하는 경우 상단의 '+' 버튼을 클릭하고 아래 단계의 파일을 드래그 앤 드롭하면 됩니다.

<blockquote>
<i class="fa fa-terminal"></i> 쿠버네티스 리소스를 사용하여 Keycloak 인스턴스를 생성합니다:
</blockquote>

```execute
oc apply -f ./config/sso/sso-keycloak.yaml
```

<blockquote>
<i class="fa fa-terminal"></i> Keycloak 포드(Pod)가 준비 상태가 될 때까지 몇 분간 기다립니다:
</blockquote>

```execute
oc wait --for=condition=Ready pod/keycloak-0 --timeout=300s
```

Output:
```
pod/keycloak-0 condition met
```

라우트 연결을 허용하기 위해 Keycloak 포드에 레이블을 추가합니다:

```execute
oc label pod keycloak-0 maistra.io/expose-route=true
```

<blockquote>
<i class="fa fa-terminal"></i> 다음과 같이 쿠버네티스 리소스를 통해 렐름(realm), 역할(roles), 클라이언트(clients) 및 사용자(users)를 생성하고 구성합니다:
</blockquote>

```execute
sed "s|%APP_URL%|$GATEWAY_URL|" ./config/sso/sso-realm.yaml | oc create -f -
```

```execute
oc apply -f ./config/sso/sso-user1.yaml
```

```execute
oc apply -f ./config/sso/sso-user2.yaml
```

### SSO 관리자 콘솔 로그인
<blockquote>
<i class="fa fa-terminal"></i>
SSO 콘솔을 엽니다. 접속을 위한 엔드포인트를 확인합니다:
</blockquote>

```execute
echo $(oc get route keycloak --template='https://{{.spec.host}}')
```

<p>
<i class="fa fa-info-circle"></i>
Keycloak이 실행된 후에도 한동안 초기화 프로세스가 진행됩니다. 포드 로그를 지켜보고 있다면 "Admin console listening on"이라는 문구가 나타났을 때 로그인할 준비가 완료된 것입니다.
</p>
<br>

관리자(admin) 사용자 이름은 `admin`이며 비밀번호는 자동으로 생성됩니다. 이 정보는 `credential-workshop-keycloak`이라는 시크릿(Secret)에 base64로 인코딩되어 저장되어 있습니다.

<blockquote>
<i class="fa fa-terminal"></i>
다음 명령을 실행하여 암호를 출력합니다:
</blockquote>

```execute
echo $(oc get secret/credential-workshop-keycloak -o jsonpath="{.data.ADMIN_PASSWORD}") | base64 --decode && echo
```

<blockquote>
<i class="fa fa-desktop"></i> SSO 웹 콘솔로 이동하여 "Administration Console"을 선택하고, 방금 확인한 암호를 사용하여 "admin" 계정으로 로그인합니다.
</blockquote>

<br>

### SSO 웹 콘솔에서 설정 수정하기
<blockquote>
<i class="fa fa-desktop"></i> 로그인이 완료되었으면, 왼쪽 메뉴에서 "Users"를 클릭한 다음 "View all users"를 선택합니다.
</blockquote>

<blockquote>
<i class="fa fa-desktop"></i> "demo" 사용자를 선택하고 "Credentials" 탭으로 이동합니다.
</blockquote>

<blockquote>
<i class="fa fa-desktop"></i> 비밀번호를 "demo"로 새로 설정하거나 재설정합니다 (Temporary 체크박스를 "off"로 비활성화했는지 확인해 주세요).
</blockquote>

<blockquote>
<i class="fa fa-desktop"></i> "theterminator" 사용자에 대해서도 동일하게 처리하고, 비밀번호를 "illbeback"으로 설정합니다.
</blockquote>

<br>

### 이제 SSO 웹 콘솔에서 사용자를 위한 몇 가지 역할을 생성합니다.


<blockquote>
<i class="fa fa-desktop"></i> "theterminator" 사용자가 선택된 상태에서 "Role Mappings" 탭을 클릭합니다.
</blockquote>

<blockquote>
<i class="fa fa-desktop"></i> "cool-kids" 역할을 클릭하여 강조 표시한 후, "Add selected" button을 눌러 해당 역할을 부여합니다.
</blockquote>


*목록에 cool-kids가 표시되지 않는 경우*

<blockquote>
<i class="fa fa-desktop"></i> 왼쪽 사이드바에서 "Roles"를 클릭하고 "View all roles"를 선택합니다.
</blockquote>

<blockquote>
<i class="fa fa-desktop"></i> 화면 오른쪽 상단 부근에 있는 "Add Role"을 클릭합니다.
</blockquote>

<blockquote>
<i class="fa fa-desktop"></i> 역할 이름란에 "cool-kids"와 간단한 설명을 입력한 뒤 "Save"를 클릭합니다.
</blockquote>

<blockquote>
<i class="fa fa-desktop"></i> 다시 "Users" 메뉴로 돌아가 "theterminator"를 선택하면, 렐름 역할 매핑(realm role mapping)을 정상적으로 진행할 수 있습니다.
</blockquote>


<br>

## APP UI가 이 SSO 서비스를 사용하도록 설정하기
이전에는 SSO 연동 없이 애플리케이션을 간편하게 테스트하기 위해 임시 로그인(fake login) 설정을 사용하고 있었습니다. 이제 이 구성을 제거하고 app-ui 서비스를 재배포해 보겠습니다.

<blockquote>
<i class="fa fa-terminal"></i>
CLI 환경에서 다음 명령을 실행합니다:
</blockquote>

```execute
SSO_SVC=$(oc get route keycloak --template='{{.spec.host}}')
oc set env dc/app-ui NODE_TLS_REJECT_UNAUTHORIZED=0 FAKE_USER=false SSO_SVC_HOST=$SSO_SVC
```

<br/>

## 접속 정보 및 API 문서
- 관리자 로그인 주소: https://keycloak-%username%.apps.%cluster_subdomain%/
- 일반 사용자 로그인 주소: https://keycloak-%username%.apps.%cluster_subdomain%/auth/realms/microservices/account
- Keycloak에 대해 보다 세부적으로 이해하려면 아래 관련 문서를 참고해 보세요:
  - [공식 문서][1]
  - [업스트림 프로젝트 문서][2]
  - [블로그 예제][3]

<br/>

[1]: https://access.redhat.com/documentation/en-us/red_hat_single_sign-on/7.3/html-single/red_hat_single_sign-on_for_openshift/
[2]: https://www.keycloak.org/documentation.html
[3]: https://developers.redhat.com/blog/2020/01/29/api-login-and-jwt-token-generation-using-keycloak/
