---
layout: post
title:  "KEYCLOAK, SAML을 이용한 AWS SSO 연결(How to set up SSO for Amazon AWS using Keycloak)"
date:   2021-03-24
categories:  AWS IAM SSO Keycloak SAML
---

SSO를 위해서 Keycloak(SAML)를 사용했으며, Keycloak 사용자가  AWS Role에 따라서 권한이 제어되는 부분을 소개합니다.
여기에서는 Keycloak SAML를 통한 연결을 설명드립니다. 아래 연결 경로상 1번 연결에 대한 설정 사항입니다.


![SSO_for_AWS_using_SAML_Keycloak](/assets/SSO_for_AWS_using_SAML_Keycloak.png)

참고사이트:
https://docs.aws.amazon.com/ko_kr/IAM/latest/UserGuide/id_roles_providers_enable-console-saml.html

![IAM](https://docs.aws.amazon.com/ko_kr/IAM/latest/UserGuide/images/saml-based-sso-to-console.diagram.png)

AWS IAM Provider 생성 및 Keycloak Client 구성을 통해서 서로 신뢰관계를 설정합니다. 그리고 AWS Role를 생성하여 Keyclock User 또는 Group에게 매핑을 합니다.

## Step 01. keycloak Realms 생성

![01.Create_Realm](/assets/01.Create_Realm.png)

Login'탭을 열고 "Require SSL"을 "none"으로 설정합니다.

![01-1_Realm_ssl_none](/assets/01-1_Realm_ssl_none.png)

## Step 02. Keycloak Identity Provider setup

#### 1) AWS에서 제공하는 SAML Meta 파일 다운로드 및 추가 수정

SAML Meta 파일을 다운로드하고, 추가적인 Attribute를 있을 경우 추가하여  Keycloak IDP를 구성합니다.
이 Meta파일에 정의된 Attribute 값을 통해서 연결설정이 진행됩니다.

~~~
curl -o saml-metadat.xml https://signin.aws.amazon.com/static/saml-metadata.xml
~~~

더운로드 받은 파일에 추가적인 Attribute인 SessionDuration을 추가 합니다.
~~~
<RequestedAttribute isRequired="true" Name="https://aws.amazon.com/SAML/Attributes/SessionDuration" FriendlyName="SessionDuration"/>
~~~

#### 2) Keycloak client를 생성

Amazon AWS saml-metadata.xml 파일을 Import 하여 Keycloak client를 생성합니다.

![02.Create_Client](/assets/02.Create_Client.png)

saml-metadata.xml 파일을 통해서 일부 정보는 자동적으로 입력됩니다. 그리고 추가적인 정보(Base URL, SSO Url name)를 기입합니다.

![03.Realm_setting](/assets/03.Realm_setting.png)

~~~
- “Base URL” 설정: /auth/realms/[your_realm_name]/protocol/saml/clients/amazon-aws
- “IDP Initiated SSO URL Name” 설정 : amazon-aws
~~~

#### 3) 저장 및 다운로드

저장하신 후에 아래 명령어로 Keycloak 설정 파일을 다운로드 합니다.

~~~
wget https://[Keycloak URL]/auth/realms/DevOps/protocol/saml/descriptor --no-check-certificate -O keycloak-idp.xml
~~~

## Step 03. Amazon AWS Service Provider setup

#### 1) AWS Identity Provider 생성

다운로드 한 keycloak-idp.xml 파일을 AWS IAM 의 Identity Provider 생성시 사용합니다.
“IAM”, “Identity providers”로 가서  “Create Provider”를 통해서 생성합니다.

![05.AWS_IAM_Provider추가](/assets/05.add_AWS_IAM_Provider.png)

파일 선택에서 keycloak-idp.xml 파일을 업로드 합니다.
생성 한 후 Summary에서 <saml-provider-ARN>값을 확인합니다.

예시) arn:aws:iam::12345678912345:saml-provider/[SAML Privider Name]

#### 2) Role 생성

Role을 생성하여 권한을 부여 합니다.

![5-1.Role생성](/assets/5_1_Role생성.png)



## Step 04. Keycloak Identity Provider 추가 설정

#### 1) Keycloak Role 추가

AWS IAM role 정보를 가지고 Keycloak에서 Role을 추가합니다.
Keycloak Role 정보는 아래 패턴으로 추가합니다.
~~~
<IAM-Role-ARN>,<saml-provider-ARN>
~~~

![06.Add_role](/assets/06.Add_role.png)

~~~
예시) arn:aws:iam::12345678912345:role/keycloak-ec2,arn:aws:iam::12345678912345:saml-provider/keycloak_sparkproda
~~~

#### 2) “Mappers”에서 항목을 추가

3가지 Mapper를 추가 합니다.

2-1) Session Role
- Name：　Session Role
- Mapper Type：　Role list
- Role attribute name：　https://aws.amazon.com/SAML/Attributes/Role
- Friendly Name：　Session Role
- SAML Attribute Name Format：　Basic
- Single Role Attribute：　ON (필수)

![07.Role](/assets/07.Role.png)

2-2) Session Name
- Name：　Session Name
- Mapper Type：　User Property
- Property：　username
- Friendly Name：　Session Name
- SAML Attribute Name：　https://aws.amazon.com/SAML/Attributes/RoleSessionName
- SAML Attribute Name Format：　Basic

![08.Session](/assets/08.Session.png)


2-3) ession Duration

- Name：　Session Duration
- Mapper Type：　Hardcoded attribute
- Friendly Name：　Session Duration
- SAML Attribute Name：　https://aws.amazon.com/SAML/Attributes/SessionDuration
- SAML Attribute Name Format：　Basic
- Attribute value：　9000


#### 3) 기본 설정 부분 제거

"Scope"탭을 열고 "Full Scope Allowwd"를 "OFF"로 설정합니다.

![10.scope](/assets/10.scope.png)

"Client Scopes"에서 Assigned Default Client Scope 에 설정된 부분을 제거합니다.


#### 4) Keycloak 그룹 또는 사용자 생성 및 Role 할당

사용자 생성 사이드 메뉴의 "Users"를 선택하고 "Add user"를 클릭하여 수 있습니다. 또는 그룹 생성한 후  AWS SAML role을 할당합니다.

![11.Add_Group](/assets/11.Add_Group.png)



#### 5) 접속

URL를 통해서 AWS Console로 Redirect되면서, Role을 선택할 수 있는 화면으로 이동합니다. Role이 한개일 경우 AWS Console로 이동하게 됩니다.

접속 주소:
https://[Keycloak URL]/auth/realms/DevOps/protocol/saml/clients/amazon-aws

Keycloak “Base URL” 로 접속하시면 로그인/패스워드 인증 후 Redirect됩니다.


## Step 05. Amazon AWS CLI 연결

AWS 접속Token 정보 업데이트를 위해서 Saml2aws를 사용했습니다.
Saml2aws통해서 AWS CLI Profile이 생성되며, 이 Profile를 사용하여 연결합니다.

~~~
saml2aws configure -a sparkproda-saml  \
      --idp-provider KeyCloak \                               
      --username [username] \
      -r ap-northeast-2 \
      --mfa Auto \
      --url https://[Keycloak URL 주소]/auth/realms/DevOps/protocol/saml/clients/amazon-aws  \
      --skip-prompt \
      --skip-verify

account {
       URL: https://[Keycloak URL 주소]/auth/realms/DevOps/protocol/saml/clients/amazon-aws
       Username: [username]
       Provider: KeyCloak
       MFA: Auto
       SkipVerify: true
       AmazonWebservicesURN: urn:amazon:webservices
       SessionDuration: 3600
       Profile: saml
       RoleARN:
       Region: ap-northeast-2
     }

saml2aws login -a sparkproda-saml

aws --profile saml ec2 describe-instances

aws sts get-caller-identity --profile saml

{
    "UserId": "ABCABCABCABC:sparkproda",
    "Account": "123456789",
    "Arn": "arn:aws:sts::123456789:assumed-role/[Role_ID]/[Keycloak_ID]"
}
(END)
~~~

60분이 지나면 토큰 유효기간이 지나서 다시 갱신해야 합니다.
~~~
aws sts get-caller-identity --profile saml

An error occurred (ExpiredToken) when calling the GetCallerIdentity operation: The security token included in the request is expired
~~~

saml2aws: https://github.com/Versent/saml2aws

https://aws.amazon.com/ko/premiumsupport/knowledge-center/aws-cli-call-store-saml-credentials/
