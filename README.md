![Released Container Image](https://github.com/RedHatGov/service-mesh-workshop-dashboard/workflows/Released%20Container%20Image/badge.svg)

# OpenShift Service Mesh Workshop (오픈시프트 서비스 메시 워크숍)
이 콘텐츠는 OpenShift Homeroom 배포 환경과 연동되도록 설계되었습니다. 다음과 같은 고려 사항이 포함되어 있습니다:
* 실습 가이드 콘텐츠에서 사용자 또는 클러스터 맞춤형 변수에 대한 변수 보간(Variable interpolation) 지원
* `curl`을 통해 서비스에 액세스하거나 테스트하기 위한 클러스터 내부 URL 사용

Homeroom에 대한 핵심 요약(TL;DR)은 다음과 같습니다. 우리는 이러한 모든 실습 과정을 하나의 웹사이트로 구축하고, 이를 컨테이너에 넣어 워크숍 참가자들이 사용하는 OpenShift 클러스터에 배포합니다. 이를 통해 OpenShift 웹 콘솔 및 CLI 터미널과 실습 가이드라인을 한 화면에서 나란히 볼 수 있도록 지원합니다.

## 프로비저닝 방법 (How To Provision)

### RHPDS에서 신청하기 (Order from RHPDS)

카탈로그에서 [All Services] -> [Openshift Workshop] -> [OpenShift Service Mesh 2 Workshop] 경로로 이동합니다.

워크숍 예상 참가자 수에 맞게 `Number of Users` 값을 설정해 주세요.

[관리자이신 경우, 여기에서 워크숍 가이드를 확인하실 수 있습니다](https://docs.google.com/document/d/1A9pXa_sCto0ZkfeTV07eyf0vNeyk10W-o3-0TrIGBqk/edit#)


### 접속 정보 (Access Info)
> RHPDS를 통해 워크숍을 프로비저닝했거나 클러스터에서 [AgnosticD 역할 실행](./README-AgnosticD.md)을 완료했는지 확인해 주세요.

워크숍 참가자들에게 아래 URL을 제공합니다: 
```
echo https://username-distribution-homeroom.$CLUSTER_SUBDOMAIN
```

참가자들은 유효한 이메일 주소와 `LAB_USER_ACCESS_TOKEN` 환경 변수로 지정된 워크숍 비밀번호(기본값은 **redhatlabs**)를 입력해야 합니다. 로그인이 완료되면 사용자 계정과 워크숍으로 이동할 수 있는 링크가 제공됩니다.

`username-distribution` 앱에서 `/admin` 경로로 이동하여 관리자 작업을 수행할 수 있습니다. 사용자 이름으로 `admin`을 입력하고, 비밀번호로 `LAB_ADMIN_PASS` 환경 변수 값(기본값은 **pleasechangethis**)을 입력해야 합니다.

워크숍에 직접 접속하려면 아래 주소로 이동할 수 있습니다:
```
echo https://service-mesh-workshop-homeroom.$CLUSTER_SUBDOMAIN
```

### 정리 작업 (Clean Up)
워크숍 진행이 끝나면 RHPDS 클러스터를 삭제해 주세요.
