---
layout: post
title: "Amazon EKS 클러스터에 IAM 사용자 API 사용 권한 부여"
categories:
- Hands-on
tags:
- Amazon EKS
- Amazone IAM
- Kubernetes
- RBAC
---
EKS 클러스터 설정 뒤 다른 AWS 사용자들이 해당 클러스터의 상태를 확인하거나 조작하길 바랄 수 있다.

동일한 계정을 여러 사용자들이 돌려쓰는 것은 보안상 위험하며 필요한 사용자에게 각자 최소한으로 필요한 권한만을 부여할 필요가 있다.

이 실습문서에서는 IAM 계정을 사용하여 EKS 클러스터 API 접근 권한을 부여해볼 것이다.
실습을 위해서는 기초적인 AWS, EKS 그리고 쿠버네티스 지식이 필요하며 실습 전 AWS CLI 환경이 준비되어 있고 EKS 클러스터가 생성되어 있어야한다.

# 실습 컨셉

표면적인 컨셉은 *특정 IAM 그룹에 속한 IAM 유저에게 EKS 클러스터 API의 사용 권한을 부여하는 것*이다.

구체적으로 들어가면 특정 IAM 그룹에 속한 IAM 유저에게 특정 AWS 역할을 위임받을 수 있도록 설정하고
해당 AWS 역할을 위임받은 유저가 EKS 클러스터의 특정 그룹에 속하도록 연결시키는 것이다.
이후 EKS 클러스터의 그룹에 `RoleBinding`을 통해 API를 사용하기 위한 권한을 부여한다.

1. AWS 역할 AROLE을 만든다.
2. AWS 역할 AROLE을 위임받은 IAM 유저가 EKS 클러스터의 특정 그룹 KGROUP에 속하도록 설정한다.
3. KGROUP에 속한 Kubernetes 계정이 view 역할(권한)을 가지도록 설정한다.
4. IAM 그룹 AGROUP을 만들고 AWS 역할 AROLE을 위임받을 수 있는 권한을 부여한다.
5. IAM 유저를 AGROUP에 소속시킨다.
6. AUSER가 AGROUP에 속함으로서 AROLE을 위임받게 되고 AROLE를 위임받아 최종적으로 Kuberentes view 권한을 받는다.

IAM 유저가 EKS 클러스터 API를 사용할 수 있도록 권한을 부여하기 위해 가장 빠르고 쉬운 방법은 유저 그 자체를 쿠버네티스 그룹에 맵핑하는 것이다.
하지만 유저가 늘어날수록 관리가 힘들어질 것이므로 IAM 그룹을 통해 신규 유저 생성시 그룹에 추가만 해주면 권한을 부여받을 수 있도록 구성한다.

이 실습예제에는 꼭 필요하지 않은 부분들도 약간씩 포함되어 있다.
실제 구성시에는 필요하지 않은 부분이나 다른 식으로 맞출 수 있는 부분은 자신이 사용하는 시스템에 맞춰 조절하면 된다.

# 실습

## AWS 유저/그룹 생성

`dev1` 유저를 생성한다.

```bash
aws iam create-user --user-name dev1
```

*출력:*

```json
{
    "User": {
        "Path": "/",
        "UserName": "dev1",
        "UserId": "***",
        "Arn": "arn:aws:iam::<AccountID>:user/dev1",
        "CreateDate": "2021-04-07T06:20:50+00:00"
    }
}
```

AWS CLI를 사용하기 위해 액세스키를 생성한다.

```bash
aws iam create-access-key --user-name dev1
```

*출력:*

```json
{
    "AccessKey": {
        "UserName": "dev1",
        "AccessKeyId": "<AccessKeyID>",
        "Status": "Active",
        "SecretAccessKey": "<SecretAccessKey>",
        "CreateDate": "2021-04-07T07:05:36+00:00"
    }
}
```

`Developers` 그룹을 생성한다.

```bash
aws iam create-group --group-name Developers
```

*출력:*

```json
{
    "Group": {
        "Path": "/",
        "GroupName": "Developers",
        "GroupId": "***",
        "Arn": "arn:aws:iam::<AccountID>:group/Developers",
        "CreateDate": "2021-04-07T06:24:46+00:00"
    }
}
```

`dev1` 유저를 `Developers` 그룹에 추가한다.

```bash
aws iam add-user-to-group --user-name dev1 --group-name Developers
```

`dev1` 계정에 대한 설정을 로컬에 저장한다.

```bash
aws configure --profile dev1
```

```
AWS Access Key ID [None]: <AccessKeyID>
AWS Secret Access Key [None]: <SecretAccessKey>
Default region name [None]: <Region>
Default output format [None]: json
```

## AWS 정책/역할 설정

EKS 클러스터에 대한 정보를 읽을 수 있도록 DescribeCluster 정책을 생성한다.
이 정책은 이후 역할에 추가해줄 것이다.

지정한 EKS 클러스터에 대한 정보만 취득할 수 있도록 `Resource`에 클러스터 ARN을 명시해준다.

EKS-IAM 연동 자체에 필요한 작업은 아니지만 AWS CLI를 통해 `$HOME/.kube/config` 파일을 업데이트하기 위해서는 `eks:DescribeCluster` 권한이 필요하다.
이 예제에서는 역할에 정책을 연결하지만 유저에 직접 연결하거나 그룹에 연결시켜도 상관없고 kubeconfig 파일을 직접 작성한다면 해당 작업은 넘어가도 된다.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "eks:DescribeCluster",
            "Resource": "arn:aws:eks:<Region>:<AccountID>:cluster/<ClusterName>"
        }
    ]
}
```

위 내용으로 `eks-<ClusterName>-describe.json` 파일을 생성 후 아래 커맨드로 정책을 생성한다.

```bash
aws iam create-policy --policy-name EKSMyExampleDescribe --policy-document file://eks-<ClusterName>-describe.json
```

*출력:*

```json
{
    "Policy": {
        "PolicyName": "EKSMyExampleDescribe",
        "PolicyId": "***",
        "Arn": "arn:aws:iam::<AccountID>:policy/EKSMyExampleDescribe",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2021-04-07T07:40:06+00:00",
        "UpdateDate": "2021-04-07T07:40:06+00:00"
    }
}
```

이제 EKS와 연동할 역할을 생성한다.
이후 EKS 클러스터에 이 역할을 가진 사용자에 대해 `RoleBinding`을 생성해줄 것이다.

EKS-IAM 연동에는 역할 자체만 있으면 된다.
실제로 어떤 정책을 연결할 필요는 없다.
여기서 정책에 `eks:DescribeCluster` 권한을 부여하는 것은 이전에 말한 것처럼 어디까지나 kubeconfig를 편하게 업데이트하기 위함이다.

```json
{
    "Version": "2012-10-17",
    "Statement": {
        "Effect": "Allow",
        "Principal": { "AWS": "arn:aws:iam::<AccountID>:root" },
        "Action": "sts:AssumeRole"
    }
}
```

```bash
aws iam create-role --role-name EKSMyExampleViewer --assume-role-policy-document file://eks-<ClusterName>-viewer.json
aws iam attach-role-policy --role-name EKSMyExampleViewer --policy-arn "arn:aws:iam::<AccountID>:policy/EKSMyExampleDescribe"
```

```bash
{
    "Role": {
        "Path": "/",
        "RoleName": "EKSMyExampleViewer",
        "RoleId": "***",
        "Arn": "arn:aws:iam::<AccountID>:role/EKSMyExampleViewer",
        "CreateDate": "2021-04-07T07:41:51+00:00",
        "AssumeRolePolicyDocument": {
            "Version": "2012-10-17",
            "Statement": {
                "Effect": "Allow",
                "Principal": {
                    "AWS": "arn:aws:iam::<AccountID>:root"
                },
                "Action": "sts:AssumeRole"
            }
        }
    }
}
```

AWS 유저가 `EKSMyExampleViewer` 역할을 위임받을 수 있도록 `sts:AssumeRole` 권한을 포함한 정책을 생성한다.
이전에 만들었던 역할에 대해서만 위임받을 수 있도록 `Resource`를 명시해준다.

생성된 정책은 유저에게 직접 연결하거나 그룹을 통해 연결하면 된다.
아래 예제에서는 그룹에 정책을 연결했다.

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "iam:ListRoles",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::<AccountID>:role/EKSMyExampleViewer"
        }
    ]
}
```

```bash
aws iam create-policy --policy-name EKSMyExampleViewerRoleAssume --policy-document file://eks-<ClusterName>-viewer-role-assume.json
aws iam attach-group-policy --group-name Developers --policy-arn "arn:aws:iam::<AccountID>:policy/EKSMyExampleViewerRoleAssume"
```

```json
{
    "Policy": {
        "PolicyName": "EKSMyExampleViewerRoleAssume",
        "PolicyId": "***",
        "Arn": "arn:aws:iam::<AccountID>:policy/EKSMyExampleViewerRoleAssume",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2021-04-07T07:43:32+00:00",
        "UpdateDate": "2021-04-07T07:43:32+00:00"
    }
}
```

AWS CLI를 통해 `dev1` 계정으로 `aws eks describe-cluster`를 실행해보면 실패할 것이다.
`eks:DescribeCluster` 권한이 포함된 역할을 위임받을 수 있는 권한은 있지만 아직 실제로 해당 역할을 위임받은 것은 아니기 때문이다.

```bash
aws eks --profile dev1 describe-cluster --name <ClusterName>
```

```
An error occurred (AccessDeniedException) when calling the DescribeCluster operation:
User: arn:aws:iam::<AccountID>:user/dev1 is not authorized to perform: eks:DescribeCluster on resource:
arn:aws:eks:ap-northeast-2:<AccountID>:cluster/<ClusterName>
```

`aws sts assume-role` 명령어로 지정한 역할을 사용하는 세션을 시작할 수도 있으나
이후 계속 해당 역할을 사용할 예정이므로 아래 예제에서는 `$HOME/.aws/config` 파일을 조작하여 역할을 위임받은 프로필을 추가할 것이다.

기존에 `dev1` 계정을 `aws configure` 명령어로 추가했다면 `$HOME/.aws/config` 파일에 `dev1` 계정의 프로필이 있을 것이다.
그 하위에 `dev1-eks` 프로필을 직접 추가해준다.

프로필에는 `role_arn`과 `source_profile`을 작성해준다.
각각 위임받을 역할의 ARN과 원본 프로필을 써주면 된다.

이후 AWS CLI 사용시 `--profile dev1-eks`로 `EKSMYExampleViewer` 역할을 위임받아 사용할 수 있다.

```
[profile dev1]
region = ap-northeast-2
output = json
[profile dev1-eks]
region = ap-northeast-2
output = json
role_arn = arn:aws:iam::<AccountID>:role/EKSMyExampleViewer
source_profile = dev1
```

파일 작성 후 방금 추가한 `dev1-eks` 프로필을 사용하여 다시 `aws eks describe-cluster` 명령어를 실행하면 정상적으로 `<ClusterName>` EKS 클러스터에 대한 정보를 확인할 수 있다.

```bash
aws eks --profile dev1-eks describe-cluster --name <ClusterName>
```

`eks:DescribeCluster` 권한이 생겼으므로 `aws eks update-kubeconfig` 명령어를 사용하여 클러스터에 대한 설정파일을 로컬에 내려받을 수 있다.

```bash
aws eks --profile dev1-eks --region ap-northeast-2 update-kubeconfig --name <ClusterName> --alias <ClusterName>
```

하지만 여전히 `kubectl` 커맨드는 사용할 수 없다.
역할을 위임받았지만 해당 역할이 EKS 클러스터와 연결되어있지 않기 때문이다.

```bash
kubectl get pods -A
```

*출력:*

```
error: You must be logged in to the server (Unauthorized)
```

## Kubernetes 역할 설정

쿠버네티스 설정을 변경하기 위해 기존에 EKS 클러스터를 생성한 프로필로 컨택스트를 전환한다.
쿠버네티스 설정 파일을 관리할 줄 안다면 직접 파일을 수정하거나 `kubectl config use-context`명령어를 사용해도 되고
아니라면 그냥 아래 명령어를 실행해서 그냥 설정을 덮어쓰자.

```bash
aws eks --profile default --region ap-northeast-2 update-kubeconfig --name <ClusterName> --alias <ClusterName>
```

우선 관리편의성을 위해 `developers` 그룹을 생성한 뒤 빌트인으로 제공하는 `ClusterRole/view`를 해당 그룹에 바인딩한다.
`developers_rolebinding.yml` 파일을 생성한 뒤 아래 내용으로 채워넣고 적용한다.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: developers-view
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f developers_rolebinding.yml
```

이제 `developers` 그룹에 속한 계정은 `view` 권한을 사용할 수 있다.
`view` 권한은 몇가지 리소스에 대한 읽기 권한을 포함한다.

## AWS 역할 - Kubernetes 연동

그럼 이번엔 `role/EKSMyExampleViewer` 역할을 사용하는 IAM 계정이 `developers` 그룹에 속하도록 컨피그맵을 수정한다.

```bash
kubectl edit -n kube-system configmap/aws-auth
```

`mapRoles` 항목에 `role/EKSMyExampleViewer` 역할이 `developers` 그룹에 연결되도록 항목을 하나 추가해준다.

```yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::<AccountID>:role/MyExampleEKSNode
      username: system:node:{{EC2PrivateDNSName}}
    - groups:
      - developers
      rolearn: arn:aws:iam::<AccountID>:role/EKSMyExampleViewer
      username: developer
kind: ConfigMap
metadata:
  creationTimestamp: "2021-04-07T03:23:28Z"
  managedFields:
...
```

설정 적용 후 다시 `dev1-eks`로 kubeconfig를 변경하여 `kubectl` 명령어를 실행해보면 정상적으로 쿠버네티스 역할을 부여받았음을 알 수 있다.

```bash
aws eks --profile dev1-eks --region ap-northeast-2 update-kubeconfig --name <ClusterName> --alias <ClusterName>
kubectl get pods -A
```

*출력:*

```
NAMESPACE      NAME                                    READY   STATUS    RESTARTS   AGE
istio-system   istio-ingressgateway-5656d66f9f-5d64d   1/1     Running   0          6h20m
istio-system   istiod-7ddfdf84f8-t88tz                 1/1     Running   0          6h20m
kube-system    aws-node-c645j                          1/1     Running   0          6h30m
kube-system    aws-node-dx7fn                          1/1     Running   0          6h30m
kube-system    aws-node-fbs7d                          1/1     Running   0          6h30m
kube-system    aws-node-tk4rq                          1/1     Running   0          6h30m
kube-system    coredns-6fb4cf484b-7vcjb                1/1     Running   0          6h36m
kube-system    coredns-6fb4cf484b-n47rv                1/1     Running   0          6h36m
kube-system    kube-proxy-dc5ps                        1/1     Running   0          6h31m
kube-system    kube-proxy-npbhx                        1/1     Running   0          6h31m
kube-system    kube-proxy-q7zp8                        1/1     Running   0          6h31m
kube-system    kube-proxy-qx5w8                        1/1     Running   0          6h31m
lens-metrics   kube-state-metrics-64d759d9db-jplpl     1/1     Running   0          6h11m
lens-metrics   node-exporter-6h4hz                     1/1     Running   0          6h11m
lens-metrics   node-exporter-fmvh4                     1/1     Running   0          6h11m
lens-metrics   node-exporter-wbsm5                     1/1     Running   0          6h11m
lens-metrics   node-exporter-zxgsv                     1/1     Running   0          6h11m
lens-metrics   prometheus-0                            1/1     Running   0          6h11m
```

`view` 역할에는 노드에 대한 정보를 읽을 권한은 없기 때문에 `kubectl get node` 명령어를 실행할 경우 아래와 같은 에러메세지를 볼 수 있다.

```bash
kubectl get nodes
```

*출력:*

```
Error from server (Forbidden): nodes is forbidden: User "developer" cannot list resource "nodes" in API group "" at the cluster scope
```

# 정리

일부 이름이 겹치는 부분이 있어 헷갈릴 수 있으나 AWS 그룹와 EKS 클러스터 내 그룹 등은 서로 아무런 연관관계가 없다.
**EKS 클러스터에서는 AWS 계정의 역할만 본다.**
AWS에서 그룹을 지정하거나 `eks:DescribeCluster` 권한을 주는 것들은 그저 관리편의상 진행했을 뿐이다.
결과적으로 어떻게든 해당 AWS 유저가 EKS에서 `aws-auth` 컨피그맵에서 지정한 역할을 위임받게만 만들면 된다.

# 참고

[Managing users or IAM roles for your cluster](https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html)

[How do I assume an IAM role using the AWS CLI?](https://aws.amazon.com/premiumsupport/knowledge-center/iam-assume-role-cli/?nc1=h_ls)
