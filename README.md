# BUILD & PUSH CONTAINER IMAGE

## BUILD IMAGE

### How to Create Dockerfile
Dockerfile은 Docker 컨테이너를 빌드하는 데 사용되는 텍스트 파일입니다. 이 파일은 컨테이너 이미지를 생성하기 위한 명령어와 설정을 정의하며, Docker 엔진에게 어떻게 컨테이너를 구성할지를 알려줍니다. Dockerfile을 작성하여 컨테이너 이미지를 만들면, 해당 이미지를 사용하여 여러 컨테이너를 실행할 수 있습니다.
```
##  Dockerfile
# 베이스 이미지 정의
FROM openjdk:17

# 작업 디렉토리 설정
WORKDIR /app

# 파일 복사
COPY ./gradle /app

# 명령어 실행
RUN gradle clean && gradle build -x test

# 환경 변수 설정
ENV JAVA_OPTS=-Xms1G -Xmx1G

# 포트 노출
EXPOSE 8080

# 컨테이너 실행 시 실행될 명령어
CMD ["java", "-jar", "/app/build/app.jar"]
```
1. FROM: 베이스 이미지를 지정합니다. 모든 Dockerfile은 기본적으로 다른 이미지를 기반으로 합니다.
2. WORKDIR: 작업 디렉토리를 설정합니다. 해당 디렉토리에서 나머지 명령어들이 실행됩니다.
3. COPY: 로컬 파일이나 디렉토리를 컨테이너 내부로 복사합니다.
4. RUN: 쉘 명령어를 실행합니다. 이미지 빌드 시에 실행되며, 패키지 설치, 의존성 설정 등을 수행합니다.
5. ENV: 환경 변수를 설정합니다. 애플리케이션 실행에 필요한 환경 변수를 지정할 수 있습니다.
6. EXPOSE: 컨테이너가 사용하는 포트를 외부에 노출합니다. 이는 실행 중인 컨테이너에서 해당 포트에 접근할 수 있도록 해줍니다.
7. CMD: 컨테이너가 시작될 때 실행될 명령어를 설정합니다. Dockerfile 당 하나만 존재하며, ENTRYPOINT와 함께 사용될 수 있습니다.

Dockerfile은 Docker 컨테이너를 구성하는 방법을 기술하는 스크립트이며, 이를 통해 반복 가능하고 일관된 환경에서 애플리케이션을 실행할 수 있습니다.

## BUILD AND PUSH CONTAINER IMAGES TO THE CONTAINER REGISTRY
컨테이너 이미지를 빌드하고 이미지를 dockerHub 에 푸시합니다.
macOS에서 이미지를 빌드하면 arm64 아키텍처로 이미지가 생성됩니다. 법용 리눅스 환경에서 사용하기 위해서는 amd64 아케텍처가 필요합니다. Docker 엔진에서 제공하는 buildx 를 사용하면 쉽게 이미지를 빌드하고 푸시 할수 있습니다.
### Login to dockerHub
```
docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: *******
Password: *******
```

### Build and Push Blue Application
데모에 사용할 블루 어플리케이션을 빌드하고 푸시합니다.
```
# build & push v1
docker build --build-arg="APP_NAME=blue" --build-arg="APP_VERSION=v1" -t nationminu/blue:v1 . -f docker/Dockerfile --no-cache
# docker push nationminu/blue:v1


# build multiple platform & push image
docker buildx build --platform linux/amd64,linux/arm64 --push --build-arg="APP_NAME=blue" --build-arg="APP_VERSION=v1" -t nationminu/blue:v1 . -f docker/Dockerfile --no-cache
docker buildx build --platform linux/amd64,linux/arm64 --push --build-arg="APP_NAME=blue" --build-arg="APP_VERSION=v2" -t nationminu/blue:v2 . -f docker/Dockerfile --no-cache
```

### Build and Push Green Application
데모에 사용할 그린 어플리케이션을 빌드하고 푸시합니다.
```
# build & push
docker build --build-arg="APP_NAME=green" --build-arg="APP_VERSION=v1" -t nationminu/green:v1 . -f docker/Dockerfile --no-cache
# docker push nationminu/green:v1


# build multiple platform & push image
docker buildx build --platform linux/amd64,linux/arm64 --push --build-arg="APP_NAME=green" --build-arg="APP_VERSION=v1" -t nationminu/green:v1 . -f docker/Dockerfile --no-cache
docker buildx build --platform linux/amd64,linux/arm64 --push --build-arg="APP_NAME=green" --build-arg="APP_VERSION=v2" -t nationminu/green:v2 . -f docker/Dockerfile --no-cache
```

### Local Test with Container image
로컬에서 빌드된 이미지를 검증합니다.
```
docker run --rm -d -p "8080:80" nationminu/blue:v1
curl localhost:8080/version.html

docker run --rm -d -p "8180:80" nationminu/green:v2
curl localhost:8180/version.html
```

## DEPLOY TO KUBERNETES
준비된 컨테이너 이미지를 사용하여 kubernetes에서 배포 테스트합니다.
### create namespace
```
kubectl create namespace pod-demo
```

### How to Create Pods
```
# create pod
kubectl run blue-pod --image=nationminu/blue:v1 -n pod-demo

# create service
kubectl expose pod blue-pod --name=blue-svc -n pod-demo --port=80 --target-port=80

# create ingress
kubectl create ingress blue-ing --class=nginx --rule="blue.handle.local/*=blue-svc:80" -n pod-demo --dry-run=client -o yaml

# patch image
kubectl patch pod blue-pod -p '{"spec":{"containers":[{"name":"blue-pod","image":"nationminu/blue:v2"}]}}' -n pod-demo
```

### clean up
```
kubectl delete namespace pod-demo
```

## KUBERNETES DEPLOYMENT STRATEGY DEMO
쿠버네티스에서 어플리케이션 배포 전략을 반영한 데모입니다.

### recreate deployment strategy
kubernetes deployments 에서 제공하는 컨테이너 recreate 배포 전략을 적용한 데모입니다.
```
# 네임스페이스 생성
$ kubectl apply -f recreate-deployment/namespace.yaml
namespace/recreate-demo created

# 어플리케이션 배포
$ kubectl apply -f recreate-deployment/deployment -n recreate-demo
deployment.apps/blue-deployment created
ingress.networking.k8s.io/blue-ing created
service/blue-svc created

# 어플리케이션 스케일 아웃 1 -> 3
$ kubectl scale deployment blue-deployment --replicas 3 -n recreate-demo
deployment.apps/blue-deployment scaled

# 어플리케이션 이미지 변경 blue -> green
$ kubectl set image deployment blue-deployment blue=nationminu/green:v1 -n recreate-demo
deployment.apps/blue-deployment image updated

# 파드 모니터링
$ kubectl get pod -n recreate-demo -w

# 초기화
$ kubectl delete -f recreate-deployment/namespace.yaml
namespace "recreate-demo" deleted
```

### rolling deployment strategy
kubernetes deployments 에서 제공하는 컨테이너 rollingUpdate 배포 전략을 적용한 데모입니다.
```
# 네임스페이스 생성
$ kubectl apply -f rolling-deployment/namespace.yaml
namespace/rolling-demo created

# 어플리케이션 배포
$ kubectl apply -f rolling-deployment/deployment -n rolling-demo
deployment.apps/blue-deployment created
ingress.networking.k8s.io/blue-ing created
service/blue-svc created

# 어플리케이션 스케일 아웃 1 -> 3
$ kubectl scale deployment blue-deployment --replicas 3 -n rolling-demo
deployment.apps/blue-deployment scaled

# 어플리케이션 이미지 변경 blue -> green
$ kubectl set image deployment blue-deployment blue=nationinu/green:v1 -n rolling-demo
deployment.apps/blue-deployment image updated

# 파드 모니터링
$ kubectl get pod -n rolling-demo

# 초기화
$ kubectl delete -f rolling-deployment/namespace.yaml
namespace "rolling-demo" deleted
```

### blue-green deployment strategy
kubernetes service를 이용하여 blue-green 배포 전략을 적용한 데모입니다.
```
# 네임스페이스 생성
$ kubectl apply -f blue-green-deployment/namespace.yaml
namespace/blue-green-demo created

# 블루 어플리케이션 배포
$ kubectl apply -f blue-green-deployment/blue -n blue-green-demo
deployment.apps/blue-deployment created
ingress.networking.k8s.io/blue-green-ing created
service/blue-green-svc created

# 그린 어플리케이션 배포
$ kubectl apply -f blue-green-deployment/green -n blue-green-demo
deployment.apps/green-deployment created

# 서비스 변경 blue -> green
$ vi blue-green-deployment/service.yaml
...
  selector:
    app: demo
    version: blue -> green

# 서비스 변경 적용
$ kubectl apply -f blue-green-deployment/blue/service.yaml -n blue-green-demo
service/blue-green-svc configured

# 서비스 변경 확인
$ curl -H "Host: blue-green.handle.local" 192.168.0.100/version.html;

# 초기화
kubectl delete -f blue-green-deployment/namespace.yaml
```

### canary deployment strategy
kubernetes ingress(nginx)를 이용하여 canary 배포 전략을 적용한 데모입니다.
```
# 네임스페이스 생성
$ kubectl apply -f canary-deployment/namespace.yaml
namespace/canary-demo created

# 블루 어플리케이션 생성
$ kubectl apply -f canary-deployment/blue -n canary-demo
ingress.networking.k8s.io/blue-canary-ing created
deployment.apps/blue-deployment created
service/blue-svc created

# 그린 어플리케이션 생성(카나리 정책 포함 ingress 생성)
$ kubectl apply -f canary-deployment/green -n canary-demo
ingress.networking.k8s.io/green-canary-ing created
deployment.apps/green-deployment created
service/green-svc created

# 카나리 테스트 5% 요청 그린 어플리케이션으로 접속
$ for i in {1..20}; do curl -H "Host: canary.handle.local" 192.168.0.100/version.html; done
blue:v1
blue:v1
blue:v1
blue:v1
green:v1
blue:v1
green:v1
blue:v1
blue:v1
blue:v1
blue:v1
blue:v1
blue:v1
blue:v1
blue:v1
blue:v1
blue:v1
blue:v1
blue:v1

# 카나리 ingress 정책 변경 요청 5% -> 50%
$ vi canary-deployment/green/canary-ingress.yaml
  annotations:
    kubernetes.io/ingress.class: nginx
    # Enable the canary release feature.
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "5" -> "50"

# 카나리 정잭 변경 적용
$ kubectl apply -f canary-deployment/green/canary-ingress.yaml -n canary-demo
ingress.networking.k8s.io/green-canary-ing configured

# 카나리 테스트 50% 요청 그린 어플리케이션
$ for i in {1..20}; do curl -H "Host: canary.handle.local" 192.168.0.100/version.html; done
green:v1
green:v1
green:v1
green:v1
blue:v1
blue:v1
blue:v1
green:v1
blue:v1
green:v1
green:v1
blue:v1
green:v1
blue:v1
green:v1
green:v1
blue:v1
green:v1
green:v1
blue:v1

# 초기화
kubectl delete -f canary-deployment/namespace.yaml
```


# ETC Command
### patch
```
# kubectl patch deployment blue-deployment -n deployment-demo --patch-file blue/patch/image_patch.yaml
# kubectl patch deployment blue-deployment -n deployment-demo -p '{"spec": {"template": {"spec": {"containers": [{"name": "blue", "image": "nationminu/blue:v2"}]}}}}'
kubectl set image deployment blue-deployment blue=nationminu/blue:v2 -n deployment-demo
kubectl annotate deployments.apps blue-deployment -n deployment-demo kubernetes.io/change-cause="patch blue v2"
```

### annotations
```
kubectl annotate deployments.apps blue-deployment -n deployment-demo kubernetes.io/change-cause="blue scale to 3"
```