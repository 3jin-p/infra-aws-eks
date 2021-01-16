*작업문서*
[eks+fargate 구성](https://github.com/3jin-p/study/blob/master/infra/aws/eks)
[efkStack 구성](https://github.com/3jin-p/study/blob/master/infra/aws/efkstack)

``` bash
cd eks  
kubectl apply -f deployment.yaml
kubectl apply -f fluentd-configmap.yaml
```
[로그생성 서버 및 Fluentd 컨테이너 DockerFile](https://github.com/3jin-p/sj)
  
---  
소스는 개인적인 구현내용 저장용입니다. 
구성을 하고싶다면 위 작업문서를 참고해주세요

