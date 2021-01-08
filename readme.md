AWS-EKS with Fargate 
------

## 사전준비  

- AWS CLI 설치 
버전 1.18.163 이상 또는 버전 2.0.59  OR AWS-CLI-2  
`curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg" sudo installer -pkg AWSCLIV2.pkg -target /`  
- AWS 자격증명 구성  
```bash
$ aws configure
AWS Access Key ID [None]: <>
AWS Secret Access Key [None]: <>
Default region name [None]: <>
Default output format [None]: <>
```
- EKSCTL 설치  
1. Homebrew가 설치되어 있지 않다면 Homebrew 설치   
`/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"`   
2. Weaveworks Homebrew tap 설치 및 eksctl 설치 
`brew tap weaveworks/tap && brew install weaveworks/tap/eksctl`  

 eksctl 을 설치하면 kubectl 은 자동으로 설치되며 사전준비 끝!
  
### 서버구성
- 클러스터생성  
``` bash 
eksctl create cluster \ 
--name <cluster-name> \ 
--version <1.18> \ 
--region <cluster-region> \ 
--fargate
```  
으로 클러스터를 생성하면 Fargate Profile 과 함께 설정한 값들로 클러스터가 생성된다.

<참고>

fargate 란 AWS 에서 제공하는 컨테이너 서버리스 컴퓨팅

장점으로는

1. 자동으로 노드의 성능을 조절해주기 때문에 오토스케일러를 세팅할 필요가 없다.

2. Pod 실행 시간단위로 비용청구가 된다.

3. Fargate 구조상 Pod 간 기본 커널, CPU 리소스, 메모리 리소스 또는 탄력적 네트워크 인터페이스를 공유하지 않아 VM 수준의 격리가 가능하다

단점으로는 

1. Statefule 한 워크로드가 사용불가능하다  → 세션 정보를 서버가 아닌 외부에 저장하도록 설계가 필수적이다.

2. 특수 권한이 있는 DaemonSet 등의 Pod가 이용불가능하다 → DaemonSet 처럼 이용하고 싶은 컨테이너가 있다면 k8s SideCar 패턴으로 지원하여야한다. (EFK 스택을 sidecar 패턴으로 구현할 예정이다.)

3. 로드밸런서의 사용이 제한적이다. ALB or ELB + Ingress 를 사용하여야한다

등이 있다. 

eksctl get cluster

eksctl get fargateprofile --cluster <cluster-name>  

로 각각을 확인할 수 있다. 

처음 클러스터를 생성하면 default 네임스페이스에 대한 fargateprofile 만 존재한다. 

원하는 레이블과 네임스페이스에 대한 fargateprofile 의 생성이 필요하다

eksctl create fargateprofile  \
--cluster <cluster-name> \
--name <profile-name> \
--namespace <name-space for fargateprofile> \
--lables <“key1=value1, key2=value2”>

fargateprofile 이란  pod와 node가 1:1 로 매핑되는 fargate에서 

포드가 생성될때 해당 포드가 매핑될 서버리스 노드가 생성될 예약 작업을 명령해주는 세팅값이다. 

클러스터가 소유한 fargateprofile 의 네임스페이스와 레이블이 일치하는 포드가 생성되면, 클러스터는 fargate 노드를 생성한다.

2. 어플리케이션 배포 

kubectl create namespace <namespace 명>

관리의 용이를 위하여 namespace를 생성하여 배포하도록 하자

2-1 배포생성 

kubectl create deployment --namespace=<namespace> --image=<image path:tag> <deployment name>

를 입력하면 배포가 생성된다.  

kubectl edit deployment -n <namespace> <deployment name>

을 실행하여 fargateprofile에 맞는 레이블을 세팅해주면

 fargate 인스턴스에 파드가 올라온 것을 확인 할 수 있다.

kubectl get po -n <namespace>  

2-2 배포 노출 (서비스 생성)

kubectl expose deployment <deployment name> \
-n <namespace> \
--type=NodePort \
--port=<serviceport> \
--target-port=<pod port>

or 

kubectl create svc <service type> <service name> \
--namespace=<namespace> \
--tcp=<port>:<targetPort>

service type 은 Fargate를 이용할시 NodePort를 이용해야한다.

--tcp 옵션에서 port 는 서비스로 접근할 포트, targetPort는 해당 포트로 접근했을 시 연결되는 컨테이너의 포트이다.

kubectl -n <namespace> describe service <service-name>

으로 제대로 생성됨을 확인한다. 

이제 클러스터 내부에서는 서비스로 인해 생성된 해당 NodePort 로 파드에 접근할 수 있다.

클러스터 외부에서 접근할 수 있도록 세팅-

클러스터 외부에서 접근을 하려면 로드밸런서를 통하여 접근을 하도록 하여야한다.

로드밸런서는 ALB 를 사용하도록 하겠다.

IAM OIDC 공급자를 생성하여 클러스터와 연결

eksctl utils associate-iam-oidc-provider \ 
--region <region-code> \ 
--cluster <cluster name> \ 
--approve

로드 밸런서 컨트롤러에 IAM AWS를 호출할 수 있도록 하는 정책 다운로드 

curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

다운로드 받은 파일로 정책 생성

aws iam create-policy \
    --policy-name <policy-name> \
    --policy-document file://iam-policy.json

 사용할 로드 밸런서 컨트롤러에 대한 서비스 어카운트 생성 및 정책 연결

eksctl create iamserviceaccount \
  --cluster=<cluster-name> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/<policy-name> \
  --override-existing-serviceaccounts \
  --approve

 5.AWS LoadBalance Controller 설치  

kubectl apply \
-k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"

helm repo add eks https://aws.github.io/eks-charts

helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller \ 
--set clusterName=<cluster-name> \ 
--set serviceAccount.create=false \ 
--set serviceAccount.name=aws-load-balancer-controller \ 
-n kube-system

설치확인

kubectl get deployment -n kube-system aws-load-balancer-controller

6. 로드밸런서 컨트롤러와 연결할 인그레스 생성

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip 
    kubernetes.io/ingress.class: <위에서 설정한 alb 명>
  creationTimestamp: "2020-12-22T08:19:05Z"
  finalizers:
  - ingress.k8s.aws/resources
  labels:
    app: <>
  name: <인그레스명>
  namespace: <namespace>
  resourceVersion: "613797"
  selfLink: /apis/extensions/v1beta1/namespaces/kubetest/ingresses/testingress
  uid: 9cd097fb-33fc-4ca1-81f9-fa2c93e68446
spec:
  rules:
  - http:
      paths:
      - backend:
          serviceName: <인그레스가 연결할 서비스 명>
          servicePort: <서비스의 포트>
        path: /* (해당 백엔드로 연결될 uri)
        pathType: ImplementationSpecific

위 yaml 로 인그레스를 생성한다. kubectl apply -f <yaml 파일명>

kubernetes.io/ingress.class: 로드밸런서 컨트롤러가 인그레스에 연결되도록 판단하는 값이다

kubectl describe deployment <로드밸런서 컨트롤러 deployment 명> -o yaml 에서 

args --ingress-class=””  의 값을 넣어주면 된다. kubectl edit으로 컨트롤러의 클래스명을 변경하여도 상관없다. 

alb.ingress.kubernetes.io/target-type: ip  여러 속성이 있지만 fargate에서는 ip 만 사용이 가능하다.

kubectl get ingress/<인그레스 명> -n <네임스페이스> 로 확인 해보면  

NAME           CLASS    HOSTS   ADDRESS                                                                   PORTS   AGE
인그레스   <none>   *       k8s-game2048-ingress2-xxxxxxxxxx-yyyyyyyyyy.us-west-2.elb.amazonaws.com   80      2m32s

이런식으로 주소가 할당된 것을 확인할 수 있다. 이제 해당 주소로 접속하면 파드로 연결된다. 

제대로 되지 않았다면

kubectl logs -n kube-system   deployment.apps/aws-load-balancer-controller

으로 로드밸런서 컨트롤러의 로그를 확인할 수 있다.

