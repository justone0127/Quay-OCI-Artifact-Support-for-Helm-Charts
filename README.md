## Quay OCI Artifact Support for Helm Charts

Kubernetes 초기부터 API 매니페스트 외에 주요 지원 아티팩트는 컨테이너 이미지였습니다. 컨테이너 이미지는 컨테이너 레지스트리에서 Kubernetes에 의해 저장되고 액세스됩니다. 이러한 레지스트리는 원래 Docker 이미지 형식을 지원하도록 설계되었습니다. 그러나 Docker 이외의 추가 런타임을 촉직하기 위해 컨테이너 런타임 및 이미지 형식을 둘러싼 표준화를 제공하기 위해 OCI (Open Container Initiative)가 만들어졌습니다. 대부분의 컨테이너 레지스트리는 Docker 이미지 매니페스트 V2, Schema 2 형식을 기반으로 하는 OCI 표준화를 지원합니다.



Red Hat Quay는 컨테이너 이미지와 컨테이너 관리를 지원하는 지원 도구의 전체 에코시스템을 저장하는 프라이빗 컨테이너 레지스트리입니다. Red Hat Quay 3.4 릴리스 OCI 기반 아티팩트, 특히 Helm Chart를 사용할 수 있는 새로운 기술 프리뷰 기능이 도입되었습니다. Red Hat Quay 3.6 릴리스와 함께 이 기능을 완전한 지원 가능성으로 전환되었습니다. Quay에서 Helm Chart에 대한 OCI 지원을 사용하는 방법을 소개합니다.



### 1. Quay Deployment and Configuration

해당 가이드 진행을 위해서는 Quay가 배포된 상태어야 합니다. 

구성된 Quay 인스턴스의 URL은 QuayRegistry 사용자 정의 리소스의 상태 필드 내의 registryEndpoint에서 확인하거나 다음 명령을 통해 찾을 수 있습니다.

```bash
$ oc get quayregistry -n ${PROJECT_NAME} ${QUAY_INSTANCE_NAME} -o jsonpath='{ .status.registryEndpoint }'
```

### 2. Application Registry Enable

Application Registry 기능을 활성화하기 위해서는 Quay Config 설정을 변경해야 합니다.

### 2.1 QuayConfig Console Access

- QuayConfig URL 확인

  ```bash
  $ oc get quayregistry -n ${PROJECT_NAME} ${QUAY_INSTANCE_NAME} -o jsonpath='{ .status.configEditorEndpoint }'
  ```

- QuayConfig 접속 정보 확인 (ID/PWD)

  OCP 콘솔 접속 > quay 프로젝트 선택 > Operator > 설치된 Operator > Red Hat Quay > Quay Registry > 인스턴스 선택 > *Config Editor Credentials Secret* : ${INSTANCE_NAME}-quay-config-editor-credentials-${ID}

  ![01_quayconfig_info](https://github.com/justone0127/Quay-OCI-Artifact-Support-for-Helm-Charts/blob/main/quay_helm/01_quayconfig_info.png)

  - 값 표시를 선택해서 ID/PWD를 확인

    ![02_quayconfig_info2](https://github.com/justone0127/Quay-OCI-Artifact-Support-for-Helm-Charts/blob/main/quay_helm/02_quayconfig_info2.png)

- Application Registry 설정 활성화

  ![03_application_registry](https://github.com/justone0127/Quay-OCI-Artifact-Support-for-Helm-Charts/blob/main/quay_helm/03_application_registry.png)

- Validate Configuration Changes 선택 후 재 배포

### 2.1 Confirm Application Registry Menu enabling

Quay Console에 접속하면 그림에서 보는 것처럼 APPLICATIONS 메뉴가 활성화 된 것을 확인 할 수 있습니다.

![04_application_repository](https://github.com/justone0127/Quay-OCI-Artifact-Support-for-Helm-Charts/blob/main/quay_helm/04_application_repository.png)

### 3. Installing Helm

Helm CLI 사용을 위해 바이너리 파일을 설치합니다.

- On Linux (root 권한으로 작업)

  - 설치 파일 다운로드

    ```bash
    $ curl -L https://mirror.openshift.com/pub/openshift-v4/clients/helm/latest/helm-linux-amd64 -o /usr/local/bin/helm
    ```

  - 실행 권한 설정

    ```bash
    $ chmod +x /usr/local/bin/helm
    ```

  - 버전 확인 (일반 유저로 확인)

    ```bash
    $ helm version
    ```

### 4. Adding Helm Repo 

다음 명령을 실행하여 repository를 추가합니다.

- repo 추가

  ```bash
  $ helm repo add redhat-cop https://redhat-cop.github.io/helm-charts
  ```

- repo 업데이트

  ```bash
  $ helm repo update
  ```

- charts를 로컬에 다운로드

  ```bash
  $ helm pull redhat-cop/etherpad --version=0.0.4
  ```

  - 파일 확인

    ```bash
    [hyou-redhat.com@bastion ~]$ ls -rlt
    total 36
    --- 생략 ---
    -rw-r--r--. 1 hyou-redhat.com users 3568 Dec 21 01:37 etherpad-0.0.4.tgz
    ```

    > Helm 3.8.0 버전부터, OCI 지원은 오랜 시간이 지난 후 "실험적" 기능으로 완전히 지원되었습니다. 이전 버전의 Helm CLI가 설치된 경우 환경 변수  HELM_EXPERIMENTAL_OCI=1: 을 설정해야 합니다.

### 5. Helm Chart Application Repository Configuration

Helm Chart를 Quay에 올리기 전에 먼저 Chart를 포함할 조직을 사용할 수 있어야 합니다. Quay Console에 접속하여 새로운 조직은 생성합니다.

- Quay Console 접속 > 오른쪽 위 **+** 버튼 선택 > New Organization 선택

  ![05_helm_new_organization](https://github.com/justone0127/Quay-OCI-Artifact-Support-for-Helm-Charts/blob/main/quay_helm/05_helm_new_organization.png)

- 생성할 조직이름을 입력하고 조직을 생성

  ![06_helm_creating_new_organization](https://github.com/justone0127/Quay-OCI-Artifact-Support-for-Helm-Charts/blob/main/quay_helm/06_helm_creating_new_organization.png)

- 생성된 조직 확인

  ![07_helm_organization](https://github.com/justone0127/Quay-OCI-Artifact-Support-for-Helm-Charts/blob/main/quay_helm/07_helm_organization.png)

### 6. Interacting with Helm and Quay

기존 컨테이너 이미지에 대해 Quay와 상호 작용하는 것과 유사하게 charts를 Push하고 Pull 하려면 먼저 자격 증명을 제공해야 합니다. Helm 클라이언트와 Quay 간의 통신은 HTTPS를 통해 용이해지며 Helm 3.8부터는 신뢰 할 수 있는 인증서가 있는 HTTPS를 통해 통신하는 레지스트리에 대해서만 지원이 제공됩니다.

또한, 운영 체제는 레지스트리에 의해 노출된 인증서를 신뢰해야 합니다. 향후 Helm 릴리스의 지원을 통해 원격 레지스트리와 안전하지 않게 통신할 수 있습니다. 이를 염두에 두고 운영 체제가 Quay에서 사용하는 인증서를 신뢰하도록 구성되었는지 확인하십시오.



### 6.1 Helm Registry Login

Helm Registry에 로그인하기 위해서는 Quay Registry 도메인의 신뢰 할 수 있는 인증서가 등록되어 있어야 하고, 명령을 실행하는 운영체제에도 신뢰 할 수 있도록 설정이 되어 있어야 합니다.

- Helm Registry 로그인

  ```bash
  $ helm registry login <QUAY_HOSTNAME>
  
  Username: <username>
  Password: <password>
  
  Login succeeded
  ```

  > 신뢰하지 않는 인증서의 경우 뒤에 --insecure 옵션을 추가한 후 로그인 합니다.

- 패키지 된 Charts를 Quay 레지스트리 위치를 지정하여 Push

  ```bash
  $ helm push etherpad-0.0.4.tgz oci://<QUAY_HOSTNAME>/<ORAGANIGATION>
  ```

- Push 완료 메시지

  ```bash
  Pushed: quay.apps.ocp4.sandbox298.opentlc.com/helm_team/etherpad:0.0.4
  Digest: sha256:74420a94f26201498cc0cd7356952a208391a30ac9770dd36092de97df594d3b
  ```

- Quay Console 확인

  다음과 같이 helm_team 조직 내에 etherpad라는 이름의 새로운 레포지토리가 생성되었고, 그 안에 0.0.4 버전의 charts가 업로드 된 것을 확인 할 수 있습니다.

  ![08_push_helmchart](https://github.com/justone0127/Quay-OCI-Artifact-Support-for-Helm-Charts/blob/main/quay_helm/08_push_helmchart.png)

- 로컬에 있는 Helm Chart 파일 삭제

  ```bash
  $ rm -rf etherpad-0.0.4.tgz
  ```

### 7. Install HelmCharts on OpenShift

Quay에 등록된 HelmCharts를 OpenShift에 배포해보겠습니다.

### 7.1 Helm Charts Download from Quay

- Helm Charts 다운로드

  ```bash
  $ helm pull oci://<QUAY_HOSTNAME>/helm_team/etherpad --version=0.0.4
  ```

- 출력 메시지 예시

  ```bash
  $ helm pull oci://quay.apps.ocp4.sandbox298.opentlc.com/helm_team/etherpad --version=0.0.4
  Pulled: quay.apps.ocp4.sandbox298.opentlc.com/helm_team/etherpad:0.0.4
  Digest: sha256:74420a94f26201498cc0cd7356952a208391a30ac9770dd36092de97df594d3b
  quay.apps.ocp4.sandbox298.opentlc.com/helm_team/etherpad:0.0.4 contains an underscore.
  
  OCI artifact references (e.g. tags) do not support the plus sign (+). To support
  storing semantic versions, Helm adopts the convention of changing plus (+) to
  an underscore (_) in chart version tags when pushing to a registry and back to
  a plus (+) when pulling from a registry.
  ```

- 다운로드 확인

  Quay에서 다운로드 된 Helm Charts 패키지 파일이 확인 됩니다.

  ```bash
  [hyou-redhat.com@bastion ~]$ ls -rlt
  total 36
  --- 생략 ---
  -rw-r--r--. 1 hyou-redhat.com users 3568 Dec 21 02:51 etherpad-0.0.4.tgz
  ```

### 7.2 Creating a new project

- 프로젝트 생성

  ```bash
  $ oc new-project helm-demo
  ```

- Helm Charts 설치

  ```bash
  $ helm install oci://<QUAY_HOSTNAME>/helmteam/etherpad --generate-name --version=0.0.4
  ```

- 출력 메시지 예시

  ```bash
  NAME: etherpad-1671591456
  LAST DEPLOYED: Wed Dec 21 02:57:39 2022
  NAMESPACE: helm-demo
  STATUS: deployed
  REVISION: 1
  TEST SUITE: None
  ```

- 개발자 콘솔 토폴로지뷰 확인

  ![09_topolozy_view](https://github.com/justone0127/Quay-OCI-Artifact-Support-for-Helm-Charts/blob/main/quay_helm/09_topolozy_view.png)

- Helm 확인

  OpenShift 콘솔 화면에서 Helm 메뉴에서 Helm Charts에 대한 정보를 확인할 수 있습니다.

  ![10_helm_info](https://github.com/justone0127/Quay-OCI-Artifact-Support-for-Helm-Charts/blob/main/quay_helm/10_helm_info.png)

- Helm Charts Resource 확인

  Helm Charts에 포함된 리소스 목록을 확인할 수 있습니다.

  ![11_helm_resource](https://github.com/justone0127/Quay-OCI-Artifact-Support-for-Helm-Charts/blob/main/quay_helm/11_helm_resource.png)

### 8. Application Access

라우트 정보를 통해 Application이 정상적으로 호출 되는지 확인합니다.

개발자 콘솔 토폴로지 뷰에서 컨테이너의 위로 향한 화살표 모양(라우트)을 선택하여 Application을 호출합니다.

![12_route](https://github.com/justone0127/Quay-OCI-Artifact-Support-for-Helm-Charts/blob/main/quay_helm/12_route.png)

또는 CLI 명령을 통해 라우트 정보를 확인 할 수 있습니다.

```bash
$ oc get route -n helm-demo
NAME                  HOST/PORT                                                        PATH   SERVICES              PORT   TERMINATION     WILDCARD
etherpad-1671591456   etherpad-1671591456-helm-demo.apps.ocp4.sandbox298.opentlc.com          etherpad-1671591456   http   edge/Redirect   None
```

> HOST/PORT에 나와있는 URL 정보를 통해 Application에 호출할 수 있습니다.

- 애플리케이션 접속 확인

  ![13_application](https://github.com/justone0127/Quay-OCI-Artifact-Support-for-Helm-Charts/blob/main/quay_helm/13_application.png)

> 이번 랩을 통해 Quay Registry를  이미지 레지스트리가 아닌 Helm Charts Repository로도 사용이 가능하다라는 것을 확인할 수 있었습니다. OpenShift에서는 Helm CLI 및 Helm을 통한 애플리케이션 배포, GUI 환경에서 리소스를 확인할 수 있으므로 보다 용이하게 컨테이너를 관리하는 환경에서 사용 할 수 있습니다.



