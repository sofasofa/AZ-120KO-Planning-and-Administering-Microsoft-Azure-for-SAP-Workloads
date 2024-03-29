﻿---
lab:
    title: '9a: Linux를 실행하는 Azure VM에서 SAP 아키텍처 구현'
    module: '모듈 9: Azure로의 SAP 워크로드 마이그레이션'
---

# AZ 120 모듈 9: Azure로의 SAP 워크로드 마이그레이션
# 랩 9a: Linux를 실행하는 Azure VM에서 SAP 아키텍처 구현

예상 시간: 120분

이 랩의 모든 작업은 Azure Portal(Bash Cloud Shell 세션 포함)에서 수행됩니다.  

   > **참고**: Cloud Shell을 사용하지 않는 경우 랩 가상 머신에 Azure CLI를 설치([**https://docs.microsoft.com/ko-kr/cli/azure/install-azure-cli-windows?view=azure-cli-latest**](https://docs.microsoft.com/ko-kr/cli/azure/install-azure-cli-windows?view=azure-cli-latest))

랩 파일: 없음

## 시나리오
  
Adatum Corporation에서는 Azure에 배포할 SAP NetWeaver를 준비하는 과정에서 Linux의 SUSE 배포를 실행하는 Azure VM에 구현되는 고가용성 SAP NetWeaver를 설명할 데모를 구현하려고 합니다.

## 목표
  
이 랩을 완료하면 다음과 같은 역량을 갖추게 됩니다.

-   고가용성 SAP NetWeaver 배포를 지원하는 데 필요한 Azure 리소스 프로비전

-   Linux를 실행하는 Azure VM의 운영 체제를 고가용성 SAP NetWeaver 배포를 지원하도록 구성

-   Linux를 실행하는 Azure VM의 운영 체제에 고가용성 SAP NetWeaver 배포를 지원하는 클러스터링 구성

## 요구 사항

-   가용성 영역을 지원하는 Azure 지역에서 사용 가능한 DSv3 vCPU(2 x 4) 및 DSv2(1 x 1) vCPU의 수가 충분한 Microsoft Azure 구독

-   Windows 10, Windows Server 2016 또는 Windows Server 2019를 실행하고 Azure에 액세스할 수 있는 랩 컴퓨터


## 연습 1: 고가용성 SAP NetWeaver 배포를 지원하는 데 필요한 Azure 리소스 프로비전

재생 시간: 30분

이 연습에서는 Linux 클러스터링을 구성하는 데 필요한 Azure 인프라 컴퓨팅 구성 요소를 배포합니다. 여기에는 동일한 가용성 집합에서 Linux SUSE를 실행하는 Azure VM 쌍을 만드는 작업이 포함됩니다.

### 작업 1: 고가용성 SAP NetWeaver 배포를 호스트하는 가상 네트워크를 만듭니다.

1.  랩 컴퓨터에서 웹 브라우저를 시작하고 Azure Portal(https://portal.azure.com)로 이동합니다.

1.  메시지가 표시되면 이 랩에 사용할 Azure 구독에 대한 소유자 또는 기여자 역할과 구독에 연결된 Azure AD 테넌트의 전역 관리자 역할이 포함된 회사 또는 학교 또는 개인 Microsoft 계정을 사용하여 로그인합니다.

1.  Azure Portal에서 Cloud Shell의 Bash 세션을 시작합니다. 

    > **참고**: 현재 Azure 구독에서 Cloud Shell을 처음 시작하는 경우 Cloud Shell 파일을 유지할 Azure 파일 공유를 만들라는 메시지가 표시됩니다. 이 경우 기본값을 허용하면 자동으로 생성된 리소스 그룹에 스토리지 계정이 만들어집니다.

1.  Cloud Shell 창에서 다음 명령을 실행하여 이 랩용 리소스를 만들 Azure 지역(가용성 영역을 지원하는 지역)을 지정합니다. 여기서 `<region>`은 Azure 지역의 이름으로 바꾸세요.

```
LOCATION='<region>'
```

1.  Cloud Shell 창에서 다음 명령을 실행하여 지정한 지역에 리소스 그룹을 만듭니다.

```
RESOURCE_GROUP_NAME='az12003a-sap-RG'

az group create --resource-group $RESOURCE_GROUP_NAME --location $LOCATION
```

1.  Cloud Shell 창에서 다음 명령을 실행하여 이전에 만든 리소스 그룹에 단일 서브넷이 있는 가상 네트워크를 만듭니다.

```
VNET_NAME='az12003a-sap-vnet'

VNET_PREFIX='10.3.0.0/16'

SUBNET_NAME='sapSubnet'

SUBNET_PREFIX='10.3.0.0/24'

az network vnet create --resource-group $RESOURCE_GROUP_NAME --location $LOCATION --name $VNET_NAME --address-prefixes $VNET_PREFIX --subnet-name $SUBNET_NAME --subnet-prefixes $SUBNET_PREFIX
```

1.  Cloud Shell 창에서 다음 명령을 실행하여 새로 만든 가상 네트워크의 서브넷의 리소스 ID를 식별합니다.

```
az network vnet subnet list --resource-group $RESOURCE_GROUP_NAME --vnet-name $VNET_NAME --query "[?name == '$SUBNET_NAME'].id" --output tsv
```

1.  결과 값을 클립보드에 복사합니다. 이 값은 다음 작업에 필요합니다.

### 작업 2: 고가용성 SAP NetWeaver 배포를 호스트할 Linux SUSE를 실행하는 Azure VM을 프로비전하는 Azure Resource Manager 템플릿 배포

1.  랩 컴퓨터에서 브라우저를 시작하고 [**https://github.com/Azure/azure-quickstart-templates/tree/master/sap-3-tier-marketplace-image-md**](https://github.com/Azure/azure-quickstart-templates/tree/master/sap-3-tier-marketplace-image-md)로 이동합니다.

    > **참고**: Microsoft Edge 또는 타사 브라우저를 사용해야 합니다. Internet Explorer를 사용하지 마십시오.

1.  **SAP NetWeaver 3-tier compatible template using a Marketplace image - MD**라는 제목의 페이지에서 **Deploy to Azure**를 클릭합니다. 브라우저가 Azure Portal로 자동 리디렉션되고 **SAP NetWeaver 3-tier (managed disk)** 블레이드가 표시됩니다.

1.  **SAP NetWeaver 3계층(관리 디스크)** 블레이드에서 **템플릿 편집**을 클릭합니다.

1.  **템플릿 편집** 블레이드에서 **images** 변수를 찾고 해당 변수 정의 내의 **SLES 12** 섹션을 찾은 후 다음과 같이 `sku` 키의 값을 `12-SP4`로 변경합니다.

```
"sku": "12-SP4", 
```

1.  **저장**을 클릭합니다. 

1.   **SAP NetWeaver 3계층(관리 디스크)** 블레이드에서 다음 설정을 사용하여 배포를 시작합니다.

    -   구독: *Azure 구독의 이름*

    -   리소스 그룹: **az12003a-sap-RG**

    -   위치: *이 연습의 첫 번째 작업에서 지정한 것과 동일한 Azure 지역*

    -   SAP 시스템 ID: **I20**

    -   스택 유형: **ABAP**

    -   OS 유형: **SLES 12**

    -   Dbtype: **HANA**

    -   SAP 시스템 크기: **데모**

    -   시스템 가용성: **HA**

    -   관리자 사용자 이름: **student**

    -   인증 유형: **암호**

    -   관리자 암호 또는 키: **Pa55w.rd1234**

    -   서브넷 ID: *이전 작업에서 클립보드에 복사한 값*

    -   가용성 영역: **1,2**

    -   위치: **[resourceGroup().location]**

    -   _artifacts 위치: **https://raw.githubusercontent.com/Azure/azure-quickstart-templates/master/sap-3-tier-marketplace-image-md/**

    -   _artifacts 위치 SAS 토큰: *비워 둠*

1.  배포가 완료될 때까지 기다리지 말고 다음 작업으로 진행합니다. 

### 작업 3: 점프 호스트 배포

   > **참고**: 이전 작업에서 배포한 Azure VM은 인터넷에서 액세스할 수 없으므로 점프 호스트 역할을 할 Windows Server 2019 Datacenter를 실행하는 Azure VM을 배포합니다. 

1.  랩 컴퓨터의 Azure Portal에서 **+ 리소스 만들기**를 클릭합니다.

1.  **새로 만들기** 블레이드에서 **Windows Server 2019 Datacenter** 이미지에 따라 새 Azure VM 만들기를 시작합니다.

1.  다음 설정으로 Azure VM을 프로비전합니다.

    -   구독: *Azure 구독의 이름*

    -   리소스 그룹: *새 리소스 그룹* **az12003a-dmz-RG**의 이름

    -   가상 머신 이름: **az12003a-vm0**

    -   지역: *이 연습의 이전 작업에서 Azure VM을 배포한 것과 동일한 Azure 지역*

    -   가용성 옵션: **인프라 중복성 불필요**

    -   이미지: **Windows Server 2019 Datacenter**

    -   크기: **표준 D2s v3**

    -   사용자 이름: **Student**

    -   암호: **Pa55w.rd1234**

    -   공용 인바운드 포트: **선택한 포트 허용**

    -   인바운드 포트 선택: **RDP (3389)**

    -   이미 Windows 라이선스가 있습니까? **아니요**

    -   OS 디스크 유형: **표준 HDD**

    -   가상 네트워크: **az12003a-sap-vnet**

    -   서브넷: **bastionSubnet(10.3.255.0/24)***이라는 이름의 새 서브넷*

    -   공용 IP: **az12003a-vm0-ip***라는 이름의 새 IP 주소*

    -   NIC 네트워크 보안 그룹: **기본**

    -   공용 인바운드 포트: **선택한 포트 허용**

    -   인바운드 포트 선택: **RDP (3389)**

    -   가속화된 네트워킹: **꺼짐**

    -   기존 부하 분산 솔루션 뒤에 이 가상 머신을 배치: **아니요**

    -   무료로 기본 계획 사용: **아니요**

    -   부팅 진단: **꺼짐**

    -   OS 게스트 진단: **꺼짐**

    -   시스템 할당 관리 ID: **꺼짐**
	
    -   AAD 자격 증명으로 로그인(미리 보기): **해제**	

    -   자동 종료 사용 설정: **꺼짐**
	
    -   백업 사용: **해제**	

    -   확장: *없음*

    -   태그: *없음*

1.  프로비전이 완료될 때까지 기다립니다. 이 작업은 몇 분 정도 걸립니다.

> **결과**: 이 연습을 완료한 후 고가용성 SAP NetWeaver 배포를 지원하는 데 필요한 Azure 리소스가 프로비전되었습니다.


## 연습 2: Linux를 실행하는 Azure VM이 고가용성 SAP NetWeaver 배포를 지원하도록 구성

재생 시간: 30분

이 연습에서는 고가용성 SAP NetWeaver를 배포할 수 있도록 SUSE Linux Enterprise Server를 실행하는 Azure VM을 구성합니다.

### 태스크 1: 데이터베이스 계층 Azure VM의 네트워킹을 구성합니다.

   > **참고**: 이 태스크를 시작하기 전에 이전 연습에서 시작한 템플릿 배포가 정상적으로 완료되었는지 확인합니다.  

1.  랩 컴퓨터의 Azure Portal에서 **i20-db-0** Azure VM의 블레이드로 이동합니다.

1.  **i20-db-0** 블레이드에서 **네트워킹** 블레이드로 이동합니다.  

1.  **i20-db-0 - 네트워킹** 블레이드에서 i20-db-0의 네트워크 인터페이스로 이동합니다.  

1.  i20-db-0의 네트워크 인터페이스 블레이드에서 IP 구성 블레이드로 이동하여 **ipconfig1** 블레이드를 표시합니다.

1.  **ipconfig1** 블레이드에서 프라이빗 IP 주소를 **10.3.0.20**으로 설정하고 주소 할당을 **정적**으로 변경한 후에 변경 내용을 저장합니다.

1.   Azure Portal에서 **i20-db-1** Azure VM의 블레이드로 이동합니다.

1.  **i20-db-1** 블레이드에서 **네트워킹** 블레이드로 이동합니다.  

1.  **i20-db-1 - 네트워킹** 블레이드에서 i20-db-1의 네트워크 인터페이스로 이동합니다. . 

1.  i20-db-1의 네트워크 인터페이스 블레이드에서 IP 구성 블레이드로 이동하여 **ipconfig1** 블레이드를 표시합니다.

1.  **ipconfig1** 블레이드에서 프라이빗 IP 주소를 **10.3.0.21**로 설정하고 주소 할당을 **정적**으로 변경한 후에 변경 내용을 저장합니다.


### 작업 2: 데이터베이스 계층 Azure VM에 연결합니다.

1.  랩 컴퓨터에서 Azure Portal의 **az12003a-vm0** 블레이드로 이동합니다.

1.  **az12003a-vm0** 블레이드에서 원격 데스크톱을 통해 Azure VM az12003a-vm0에 연결합니다. 

1.  az12003a-vm0에 대한 RDP 세션 내의 서버 관리자에서 **로컬 서버** 보기로 이동하고 **IE 고급 보안 구성**을 끕니다.

1.  az12003a-vm0으로의 RDP 세션 내에서 [**https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html**](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html)로 이동하여 PUTTY를 다운로드하여 설치합니다.

1.  PuTTY를 사용하여 SSH를 통해 **i20-db-0** Azure VM에 연결합니다. 보안 경고를 승인하고 메시지가 표시되면 다음 자격 증명을 제공합니다.

    -   로그인 이름: **student**

    -   암호: **Pa55w.rd1234**

1.  PuTTY를 사용하여 SSH를 통해 동일한 자격 증명으로 **i20-db-1** Azure VM에 연결합니다.


### 작업 3: 데이터베이스 계층 Azure VM의 스토리지 구성을 검사합니다.

1.  i20-db-0 Azure VM에 대한 PuTTY SSH 세션 내에서 다음 명령을 실행하여 권한을 상승합니다. 

```
sudo su -
```

1.  암호를 입력하라는 메시지가 표시되면 **Pa55w.rd1234**를 입력하고 **Enter**키를 누릅니다. 

1.  i20-db-0에 대한 SSH 세션에서 다음을 실행하여 모든 SAP HANA 관련 볼륨(**/usr/sap**, **/hana/shared**, **/hana/backup**, **/hana/data** 및 **/hana/logs** 포함)이 올바르게 탑재되었는지 확인합니다.

```
df -h
```

1.  i20-db-1 Azure VM에서 이전 단계를 반복합니다.


### 작업 4: 노드 간 암호 없는 SSH 액세스 사용

1.  i20-db-0에 대한 SSH 세션에서 다음을 실행하여 vi 편집기에서 **/etc/ssh/sshd\_config** 파일을 엽니다(다른 모든 편집기 사용 가능).

```
vi /etc/ssh/sshd_config
```

1.  **/etc/ssh/sshd\_config** 파일에서 **PermitRootLogin** 및 **AuthorizedKeysFile** 항목을 찾고 다음과 같이 구성합니다.
```
PermitRootLogin yes
AuthorizedKeysFile      /root/.ssh/authorized_keys
```

1.  변경 내용을 저장하고 편집기를 닫습니다.

1.  i20-db-0에 대한 SSH 세션과 다음을 실행하여 sshd 디먼을 다시 시작합니다.

```
systemctl restart sshd
```

1.  i20-db-1 Azure VM에서 이전 4개 단계를 반복합니다.

1.  i20-db-0에 대한 SSH 세션에서 다음을 실행하여 암호가 없는 SSH 키를 생성합니다.

```
ssh-keygen -tdsa
```

1.  메시지가 표시되면 **Enter** 키를 세 번 누른 다음 다음을 실행하여 키를 표시합니다. 

```
cat /root/.ssh/id_dsa.pub
```

1.  키 값을 클립보드에 복사합니다.

1.  i20-db-1에 대한 SSH 세션에서 다음을 실행하여 **/root/.ssh/** 디렉터리를 만듭니다.

```
mkdir /root/.ssh
```

1.  i20-db-1에 대한 SSH 세션에서 다음을 실행하여 vi 편집기에서 **/root/.ssh/authorized\_keys**를 엽니다.

```
vi /root/.ssh/authorized_keys
```

1.  편집기 창에서 i20-db-1에 생성된 키를 붙여 넣습니다.

1.  변경 내용을 저장하고 편집기를 닫습니다.

1.  i20-db-1에 대한 SSH 세션에서 다음을 실행하여 암호가 없는 SSH 키를 생성합니다.

```
ssh-keygen -tdsa
```

1.  메시지가 표시되면 **Enter** 키를 세 번 누른 다음 다음을 실행하여 키를 표시합니다. 

```
cat /root/.ssh/id_dsa.pub
```

1.  키 값을 클립보드에 복사합니다.

1.  i20-db-0에 대한 SSH 세션에서 다음을 실행하여 vi 편집기에서 **/root/.ssh/authorized\_keys**를 엽니다.

```
vi /root/.ssh/authorized_keys
```

1.  편집기 창에서 i20-db-1에 생성된 키를 붙여 넣습니다.

1.  변경 내용을 저장하고 편집기를 닫습니다.

1.  i20-db-0에 대한 SSH 세션에서 다음을 실행하여 암호가 없는 SSH 키를 생성합니다.

```
ssh-keygen -t rsa
```

1.  메시지가 표시되면 **Enter** 키를 세 번 누른 다음 다음을 실행하여 키를 표시합니다. 

```
cat /root/.ssh/id_rsa.pub
```

1.  키 값을 클립보드에 복사합니다.

1.  i20-db-1에 대한 SSH 세션에서 다음을 실행하여 vi 편집기에서 **/root/.ssh/authorized\_keys**를 엽니다.

```
vi /root/.ssh/authorized_keys
```

1.  편집기 창에서 i20-db-0에 생성된 키를 새 줄에 붙여넣습니다.

1.  변경 내용을 저장하고 편집기를 닫습니다.

1.  i20-db-1에 대한 SSH 세션에서 다음을 실행하여 암호가 없는 SSH 키를 생성합니다.

```
ssh-keygen -t rsa
```

1.  메시지가 표시되면 **Enter** 키를 세 번 누른 다음 다음을 실행하여 키를 표시합니다. 

```
cat /root/.ssh/id_rsa.pub
```

1.  키 값을 클립보드에 복사합니다.

1.  i20-db-0에 대한 SSH 세션에서 다음을 실행하여 vi 편집기에서 **/root/.ssh/authorized\_keys**를 엽니다.

```
vi /root/.ssh/authorized_keys
```

1.  편집기 창에서 i20-db-1에 생성된 키를 새 줄에 붙여넣습니다.

1.  변경 내용을 저장하고 편집기를 닫습니다.

1.  구성이 정상적으로 적용되었는지 확인하려면 i20-db-0으로의 SSH 세션에서 다음 명령을 실행하여 **root** 사용자로 i20-db-0에서 i20-db-1로의 SSH 세션을 설정합니다.

```
ssh root@i20-db-1
```

1.  연결을 계속할지 묻는 메시지가 표시되면 `yes`를 입력하고 **Enter** 키를 누릅니다.  

1.  암호를 입력하라는 메시지가 표시되지 않는지 확인합니다.

1.  다음 명령을 실행하여 i20-db-0에서 i20-db-1로의 SSH 세션을 닫습니다.  

```
exit
``` 

1.  i20-db-1로의 SSH 세션에서 다음 명령을 실행하여 **root** 사용자로 i20-db-1에서 i20-db-0으로의 SSH 세션을 설정합니다. 

```
ssh root@i20-db-0
```

1.  연결을 계속할지 묻는 메시지가 표시되면 `yes`를 입력하고 **Enter** 키를 누릅니다.

1.  암호를 입력하라는 메시지가 표시되지 않는지 확인합니다.

1.  다음 명령을 실행하여 i20-db-1에서 i20-db-0으로의 SSH 세션을 닫습니다. 

```
exit
```

### 작업 5: YaST 패키지를 추가하고 Linux 운영 체제를 업데이트하고 HA 확장을 설치합니다.

1.  i20-db-0에 대한 SSH 세션에서 다음을 실행하여 YaST를 시작합니다.

```
yast
```

1.  **YaST Control Center**에서 **Software -\> Add-On Products**를 선택하고 **Enter** 키를 누릅니다. 이렇게 하면 **Package Manager**가 로드됩니다.

1.   **설치된 추가 기능 제품** 화면에서 **퍼블릭 클라우드 모듈**이 이미 설치되어 있는지 확인합니다. 그런 다음 **F9** 키 를 두 번 눌러 셸 프롬프트로 돌아옵니다.

1.  i20-db-0에 대한 SSH 세션에서 다음을 실행하여 운영 체제를 업데이트합니다(메시지가 표시되면 **y**를 입력하고 **Enter** 키를 누름).

```
zypper update
```

1. i20-db-0에 대한 SSH 세션에서 다음을 실행하여 HA 확장 종속성을 업데이트합니다(메시지가 표시되면 **y**를 입력하고 **Enter** 키를 누른 다음 **SUSE End User License Agreement**를 읽고 **q**와 **yes**를 차례로 입력하여 라이선스 계약 조건에 동의하고 **Enter** 키를 다시 누름).

```
zypper install sle-ha-release fence-agents
```

1. i20-db-1에서 이전 단계를 반복합니다.

> **결과**: 이 연습을 완료한 후 고가용성 SAP NetWeaver 배포를 지원하도록 Linux를 실행하는 Azure VM의 운영 체제가 구성되었습니다.

## 연습 3: Linux를 실행하는 Azure VM의 운영 체제에 고가용성 SAP NetWeaver 배포를 지원하는 클러스터링 구성

재생 시간: 60분

이 연습에서는 Linux를 실행하는 Azure VM의 운영 체제에 고가용성 SAP NetWeaver 배포를 지원하는 클러스터링을 구성합니다.

### 작업 1: 클러스터링 구성

1.  az12003a-vm0에 대한 RDP 세션 내의 i20-db-0에 대한 PuTTY 기반 SSH 세션에서 다음을 실행하여 i20-db-0에서 HA 클러스터 구성을 시작합니다.

```
ha-cluster-init
```

1.  메시지가 표시되면 다음 답변을 입력합니다.

    -   계속하시겠습니까(y/n)? **y**

    -   /root/.ssh/id_rsa가 이미 존재합니다. 덮어쓰시겠습니까(y/n)? **n**

    -   Address for ring0 [10.3.0.20]: **ENTER**

    -   Port for ring0 [5405]: **ENTER**

    -   SBD를 사용하시겠습니까(y/n)?: **n**

    -   Do you wish to configure a virtual IP address (y/n)?: **n**
	
   > **참고**: 이 클러스터링 설정에서는 암호가 **linux**로 설정된 **hacluster** 계정이 생성됩니다. 이 태스크의 뒷부분에서 해당 암호를 변경합니다.	

1.  az12003a-vm0에 대한 RDP 세션 내의 i20-db-1에 대한 PuTTY 기반 SSH 세션에서 다음을 실행하여 i20-db-0~i20-db-1의 HA 클러스터를 조인합니다.

```
ha-cluster-join
```

1.  메시지가 표시되면 다음 답변을 입력합니다.

    -   Do you want to continue anyway (y/n)? **y**

    -   IP address or hostname of existing node (e.g.: 192.168.1.1) \[\]: **i20-db-0**

    -   /root/.ssh/id\_rsa가 이미 존재합니다. 덮어쓰시겠습니까(y/n)? **n**

    -   /root/.ssh/id\_dsa가 이미 존재합니다. 덮어쓰시겠습니까(y/n)? **n**

    -   Address for ring0 [10.3.0.21]: **Enter**

1.  i20-db-0에 대한 PuTTY 기반 SSH 세션에서 다음을 실행하여 **hacluster** 계정의 암호를 **Pa55w.rd1234**로 설정합니다(메시지가 표시되면 새 암호 입력). 
```
passwd hacluster

```

1.  i20-db-1에서 이전 단계를 반복합니다.

### 태스크 2: corosync 구성 검토 

1.  i20-db-0에 대한 PuTTY 기반 SSH 세션(az12003a-vm0에 대한 RDP 세션에 위치)에서 다음을 실행하여 **/etc/corosync/corosync.conf** 콘텐츠를 검토합니다.

```
cat /etc/corosync/corosync.conf
```

1.  `transport: udpu` 항목과 `nodelist` 섹션을 확인합니다.
```
[...]
   interface { 
       [...] 
   }
   transport:      udpu
} 
nodelist {
   node {
     ring0_addr:     10.3.0.20
     nodeid:     1
   }
   node {
     ring0_addr:     10.3.0.21
     nodeid:     2
   } 
}
logging {
    [...]
```

1.  i20-db-1에서 이전 단계를 반복합니다.


### 작업 3: STONITH 클러스터링 옵션 구성

1.  az12003a-vm0에 대한 RDP 세션 내의 i20-db-0에 대한 PuTTY 기반 SSH 세션에서 다음 콘텐츠를 사용하여 **crm-defaults.txt**라는 새 파일을 만듭니다.

```
property $id="cib-bootstrap-options" \
  no-quorum-policy="ignore" \
  stonith-enabled="true" \
  stonith-action="reboot" \
  stonith-timeout="150s"
rsc_defaults $id="rsc-options" \
  resource-stickiness="1000" \
  migration-threshold="5000"
op_defaults $id="op-options" \
  timeout="600"
```

1.  변경 내용을 저장하고 편집기를 닫습니다.

1.  i20-db-0에 대한 PuTTY 기반 SSH 세션에서 다음을 실행하여 새로 만든 파일의 설정을 적용합니다.

```
crm configure load update crm-defaults.txt
```

### 작업 4: Azure 구독 ID 및 Azure AD 테넌트 ID의 값을 식별합니다.

1.  랩 컴퓨터의 브라우저 창에서 Azure Portal**(https://portal.azure.com)**로 이동하고 구독과 연결된 Azure AD 테넌트의 전역 관리자 역할이 있는 사용자 계정으로 로그인했는지 확인합니다.

1.  Azure Portal에서 Cloud Shell의 Bash 세션을 시작합니다. 

1.  Cloud Shell 창에서 다음 명령을 실행하여 Azure 구독 ID와 해당하는 Azure AD 테넌트 ID를 식별합니다.

```
az account show --query '{id:id, tenantId:tenantId}' --output json
```

1.  결과 값을 메모장에 복사합니다. 이 값은 다음 작업에 필요합니다.


### 작업 5: STONITH 장치에 대한 Azure AD 응용 프로그램 만들기

1.  Azure Portal에서 Azure Active Directory 블레이드로 이동합니다.

1.  Azure Active Directory 블레이드에서 **앱 등록** 블레이드로 이동한 다음 **+ 새 등록**을 클릭합니다.

1.  **응용 프로그램 등록** 블레이드에서 다음 설정을 지정하고 **등록**을 클릭합니다.

    -   이름: **Stonith 앱**

    -   지원되는 계정 유형: **이 조직 디렉터리의 계정만**

1.  **Stonith 앱** 블레이드에서 **응용 프로그램(클라이언트) ID**의 값을 메모장에 복사합니다. 이 연습의 나중에 이 값은 **login_id**로 나타납니다.

1.  **Stonith 앱** 블레이드에서 **인증서 및 암호**를 클릭합니다.

1.  **Stonith 앱 - 인증서 및 암호** 블레이드에서 **+ 새 클라이언트 암호**를 클릭합니다.

1.  **클라이언트 암호 추가** 창의 **설명** 텍스트 상자에 **STONITH 앱 키**를 입력하고 **만료** 섹션에서 기본값 **1년 후**를 유지한 다음 **추가**를 클릭합니다.

1.  결과 암호 값을 메모장에 복사합니다(이 항목은 **추가**를 클릭한 후 한 번만 표시됨). 이 연습의 나중에 이 값은 **password**로 나타납니다.


### 작업 6: STONITH 앱의 서비스 주체에게 Azure VM에 대한 사용 권한 부여 

1.  Azure Portal에서 **i20-db-0** Azure VM의 블레이드로 이동합니다.

1.  **i20-db-0** 블레이드에서 **i20-db-0 - IAM(액세스 제어)** 블레이드를 표시합니다.

1.  **i20-db-0 - IAM(액세스 제어)** 블레이드에서 다음 설정으로 역할 할당을 추가합니다.

    -   역할: **소유자**

    -   다음에 대한 액세스 권한을 할당합니다. **Azure AD 사용자, 그룹 또는 서비스 주체**

    -   선택: **Stonith 앱**

1.  이전 단계를 반복하여 Stonith 앱에 **i20-db-1 Azure** VM에 대한 소유자 역할을 할당합니다.


### 작업 7: STONITH 클러스터 장치 구성 

1.  az12003a-vm0에 대한 RDP 세션 내의 i20-db-0에 대한 PuTTY 기반 SSH 세션에서 다음 콘텐츠로 **crm-fencing.txt** 라는 파일을 만듭니다(여기서 `subscription_id`, `tenant_id`, `login_id` 및 `password`는 연습 3 작업 5에서 식별한 값의 자리 표시자입니다.

```
primitive rsc_st_azure_1 stonith:fence_azure_arm \
     params subscriptionId="subscription_id" resourceGroup="az12003a-sap-RG" tenantId="tenant_id" login="login_id" passwd="password"
primitive rsc_st_azure_2 stonith:fence_azure_arm \
     params subscriptionId="subscription_id" resourceGroup="az12003a-sap-RG" tenantId="tenant_id" login="login_id" passwd="password"
colocation col_st_azure -2000: rsc_st_azure_1:Started rsc_st_azure_2:Started
```

1.  s03-db-0의 SSH 세션에서 **crm configure load update crm-fencing.txt**를 실행하여 파일의 설정을 적용합니다.
```
crm configure load update crm-fencing.txt
```

### 작업 8: Hawk를 사용하여 Linux를 실행하는 Azure VM에서 클러스터링 구성 검토

1.  az12003a-vm0에 대한 RDP 세션에서 Internet Explorer를 시작하고 **https://i20-db-0:7630**으로 이동합니다. 그러면 SUSE Hawk 로그인 페이지가 표시됩니다.

  > **참고**: **이 사이트는 안전하지 않습니다** 메시지는 무시하세요.

1.  SUSE Hawk 로그인 페이지에서 다음 자격 증명을 사용하여 로그인합니다.

    -   사용자 이름: **hacluster**

    -   암호: **Pa55w.rd1234**

1.  클러스터 상태가 정상인지 확인합니다. 두 클러스터 노드 중 하나가 정상이 아니라는 메시지가 표시되면 Azure Portal에서 해당 노드를 다시 시작합니다.

> **결과**: 이 연습을 완료한 후 Linux를 실행하는 Azure VM에 고가용성 SAP NetWeaver 배포를 지원하는 클러스터링이 구성되었습니다.


## 연습 4: 랩 리소스 제거

재생 시간: 10분

이 연습에서는 이 랩에서 프로비전한 리소스를 제거합니다.

#### 태스크 1: Cloud Shell 열기

1.  Portal 상단에서 **Cloud Shell** 아이콘을 클릭하여 Cloud Shell 창을 열로 셸로 Bash를 선택합니다.

1.  Portal 하단의 **Cloud Shell** 명령 프롬프트에 다음 명령을 입력하고 **Enter** 키를 눌러 이 랩에서 만든 모든 리소스 그룹의 목록을 표시합니다.

```
az group list --query "[?starts_with(name,'az12003a-')]".name --output tsv
```

1.  이 랩에서 만든 리소스 그룹만 출력에 포함되어 있는지 확인합니다. 다음 태스크에서 이러한 그룹을 삭제합니다.

#### 태스크 2: 리소스 그룹 삭제

1.  **Cloud Shell** 명령 프롬프트에 다음 명령을 입력하고 **Enter** 키를 눌러 이 랩에서 만든 리소스 그룹을 삭제합니다.

```
az group list --query "[?starts_with(name,'az12003a-')]".name --output tsv | xargs -L1 bash -c 'az group delete --name $0 --no-wait --yes'
```

1.  Portal 하단의 **Cloud Shell** 프롬프트를 닫습니다.


> **결과**: 이 랩에서 사용한 리소스를 제거하여 이 연습을 완료해야 합니다.