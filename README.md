# Nginx, Kubernetes 기반 애플리케이션 성능 및 확장성 테스트 🚀

## 1. 개요 📝

대규모 서비스에서는 많은 트래픽과 요청을 안정적으로 처리하기 위해 확장성과 복구 능력이 필요 합니다. Spring Boot 애플리케이션과 Nginx를 리버스 프록시로 사용하는 구성은 일반적인으로 자주 활용되지만, 대규모 트래픽이 발생할 때 이를 효과적으로 처리하기 위해서는 Kubernetes와 같은 컨테이너 오케스트레이션이 필요합니다.

Kubernetes의 자동 확장성, 로드 밸런싱, 복구 기능 등을 통해 시스템의 안정성과 성능을 극대화합니다. 이번 프로젝트에서는 Spring Boot 애플리케이션을 Nginx를 통해 리버스 프록시로 배포하고, 과부하 테스트를 통해 시스템의 성능을 측정했습니다. 또한, 트래픽이 증가함에 따라 **Horizontal Pod Autoscaler (HPA)** 를 설정하여 자동으로 Pod 수를 확장하는 방식으로 대응했습니다.

<br>

## 2. 기획 🎯

1. Spring Boot 애플리케이션과 Nginx를 컨테이너화하고 Kubernetes 클러스터에서 배포합니다.
2. HPA를 활용하여 CPU 사용량에 따라 Pod가 자동으로 확장되는지 검증합니다.
3. JMeter를 사용하여 대규모 트래픽 상황에서 애플리케이션이 성능 저하 없이 대응할 수 있는지 테스트합니다.

<br>

## 3. 설계 시스템 구조 🛠️

1. **Spring Boot 애플리케이션**: REST API를 제공하며, Docker 이미지로 패키징하여 Kubernetes에 배포
2. **Nginx**: 리버스 프록시로서 클라이언트 요청을 Spring Boot 애플리케이션으로 전달
3. **Kubernetes 클러스터**: Spring Boot 애플리케이션과 Nginx를 각각 Pod로 배포하고, HPA를 통해 CPU 사용량에 따라 Pod 수를 자동으로 조정
4. **부하 테스트 도구**: JMeter를 사용하여 대규모 동시 사용자 수와 요청을 시뮬레이션하고, 시스템 성능을 측정

<br>

## 4. 작업 프로세스 🔄

1. **Spring Boot Deploy / Service 배포**
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: springboot-deployment
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: springboot
     template:
       metadata:
         labels:
           app: springboot
       spec:
         affinity:
           nodeAffinity:
             requiredDuringSchedulingIgnoredDuringExecution:
               nodeSelectorTerms:
               - matchExpressions:
                 - key: kubernetes.io/hostname
                   operator: In
                   values:
                   - myserver02
         containers:
         - name: springboot
           image: gymlet/test_app:1.0
           ports:
           - containerPort: 8080
           resources:
             requests:
               memory: "256Mi"
               cpu: "250m"
             limits:
               memory: "516Mi"
               cpu: "500m"
   ```

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: springboot-service
   spec:
     selector:
       app: springboot
     ports:
     - protocol: TCP
       port: 8080
       targetPort: 8080
     type: ClusterIP
   ```

2. **Nginx Deploy / Service 배포**
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: nginx-deployment
   spec:
     replicas: 2
     selector:
       matchLabels:
         app: nginx
     template:
       metadata:
         labels:
           app: nginx
       spec:
         affinity:
           nodeAffinity:
             requiredDuringSchedulingIgnoredDuringExecution:
               nodeSelectorTerms:
               - matchExpressions:
                 - key: kubernetes.io/hostname
                   operator: In
                   values:
                   - myserver03
         containers:
         - name: nginx
           image: nginx:latest
           ports:
           - containerPort: 80
           resources:
             requests:
               memory: "128Mi"
               cpu: "100m"
             limits:
               memory: "256Mi"
               cpu: "200m"
           volumeMounts:
           - name: nginx-config
             mountPath: /etc/nginx/nginx.conf
             subPath: nginx.conf
         volumes:
         - name: nginx-config
           configMap:
             name: nginx-config
   ```

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: nginx-service
   spec:
     selector:
       app: nginx
     ports:
     - protocol: TCP
       port: 80
       targetPort: 80
     type: LoadBalancer
     externalIPs:
     - 10.0.2.4
   ```
   ```yaml
    apiVersion: v1
    kind: ConfigMap
    metadata:
    name: nginx-config
    data:
    nginx.conf: |
        events {}
        http {
        upstream springboot {
            server springboot-service:8080;
        }
        server {
            listen 80;
            location / {
            proxy_pass http://springboot;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            }
        }
        }
    ```

3. **Spring HorizontalPodAutoscaler(HPA) 배포**
    ```yaml
    apiVersion: autoscaling/v2
    kind: HorizontalPodAutoscaler
    metadata:
    name: springboot-hpa
    spec:
    scaleTargetRef:
        apiVersion: apps/v1
        kind: Deployment
        name: springboot-deployment
    minReplicas: 2
    maxReplicas: 6
    metrics:
    - type: Resource
        resource:
        name: cpu
        target:
            type: Utilization
            averageUtilization: 70
    ```
- CPU사용률이 70%가 넘으면 pod 확장


<br>

4. 테스트 환경체크

    **Kubernetes 클러스터** <br>
    ![2024-10-09 19 26 33](https://github.com/user-attachments/assets/2e63ebc7-b27e-4bcf-9e6a-f2bcee834d22)

    **Service / Deploy / Replica / Pod / HPA**
    ![2024-10-09 19 28 09](https://github.com/user-attachments/assets/6f82d3a1-bea9-459e-99b7-8c7450250509)

    > Oracle Virtual Box를 사용하여 VM1(Master Node), VM2/3(Worker node) Kubernetes 클러스터를 구축했습니다. 외부IP 10.0.2.4:80 포트와 IP를 호스트와 포트포워딩 후 Jmeter를 통해 부하를 테스트합니다.

    <br>

    > 각 VM의 자원은 CPU 2Core, Mem 4G를 할당하였으며 호스트의 자원은 CPU 4core, Mem 16G로 오버커밋된 상태입니다.

## 5. 아키텍처 🎨
![2024-10-09 20 42 03](https://github.com/user-attachments/assets/440e36e1-5cc9-439a-9aba-0b3d1bc8914e)

## 6. 트러블 슈팅 🐞
**1.Cluster Autoscaler 오작동 문제**
![2024-10-09 21 17 47](https://github.com/user-attachments/assets/7aceb4d6-14aa-4ca6-b0b6-a1d2041264d7)
<br>
>**문제🎐**: CPU가 70%가 넘으면 Spring Boot Pod가 확장되어야 하는데 Nginx도 함께 확장되는 현상 
<br>

>**원인🧨**: 사전지식이 부족한 탓에 Master Node에 Spring, Nginx 모두 배포하였고 부하테스트 시 마스터노드의 자원이 부족해지면서 클러스터 오토 스케일러가 자동으로 워커노드에 Spring, Nginx ReplicaSet을 배포하면서 위 같은 상황이 발생했습니다.

>**해결🎵**: Spring Boot는 워커노드(myserver02), Nginx는 워커노드(myserver03)에 배포될 수 있도록 Affinity Role 설정

**nginx-deployment.yaml**
```yaml
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - myserver03
```

**2.POD 매트릭 수집불가 현상**
>**문제🔒**: kubectl top pods 명령어를 통해 각 pod가 사용중인 cpu, memory 사용량을 알 수 있습니다. 하지만 각 노드의 매트릭을 수집할 수 없다는 에러 발생했습니다.

>**원인🧨**: kubelet이 TLS 인증서를 검증하고 있어서 매트릭을 수집할 수 없는 상황

![image](https://github.com/user-attachments/assets/3875b62e-c622-48cc-9b26-10b206659f66)

>**해결🎵**: TLS 인증서는 운영 환경에서는 MetricServer와 Kubelet 사이의 통신의 보안문제로 필수적으로 사용하길 권장하지만 로컬 테스트 환경이라서 비활성화 처리

```yaml
spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=10250
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls # Kubelet TLS 인증서 검증 비활성화
```

**클러스터 매트릭 수집 후 각 pod의 cpu 사용량 확인 가능**
![image](https://github.com/user-attachments/assets/01ae5bd1-d68d-474c-8259-d45106b1101e)


## 6. 부하 테스트 결과 📊
Jmeter를 사용해서 피버타임에 몇명의 유저까지 견딜 수 있는지 테스트합니다. 
### 1000명 👊
![image](https://github.com/user-attachments/assets/0eef1d52-ca11-4880-a859-b568bee08d94)
![image](https://github.com/user-attachments/assets/14c958c4-06f8-431c-ba8b-404395d37914)

### 2000명 👊
![image](https://github.com/user-attachments/assets/d3b0421d-0359-46f8-8782-69d090327523)
![image](https://github.com/user-attachments/assets/5e6e1ff9-5120-4c9d-aa19-f093d9f190e0)


### 3000명 👊
![image](https://github.com/user-attachments/assets/f7c20210-194f-403c-a6df-9f0b71494974)
![image](https://github.com/user-attachments/assets/7fe018d7-3667-4a22-8b99-43fe4cd88628)

### 5000명 👊
![image](https://github.com/user-attachments/assets/bee55f48-a4d5-49c2-8548-46720a27a686)
![image](https://github.com/user-attachments/assets/1e23dd42-4ef4-4139-9d48-b0f020a86490)

- 5000명이 넘어가자 15초정도에 성능이 급격히 저하된 후 HorizontalPodAutoscaler에 의해 Pod가 증가한 것을 확인할 수 있습니다. 스프링 애플리케이션이 실행중인 워커노드(myserver2)의 cpu 사용량이 70%가 넘었다는 것을 의미합니다.

### 7000명 👊
![image](https://github.com/user-attachments/assets/e654cc71-e960-4da7-8a0e-6f02c3cbd68b)

- 7000명 접속했을 때는 서버에 다운타임이 발생하였습니다.


## 7. 결론 및 느낀점 ✨

이번 프로젝트에서는 Spring Boot, Nginx를 Kubernetes 클러스터에 배포하고 Jmeter를 활용하여 위의 아키텍처가 어느정도의 트래픽을 감당할 수 있는지 시뮬레이션 했습니다. 로드벨런싱, HPA를 사용하여 애플케이션의 확장성과 안정성을 최적화할 수 있었습니다. 이번 부하테스트를 진행하며 어느 지점에서 에러가 발생하고 병목현상이 발생하는지 알 수 있었으며 서버의 리소스 부족이 큰 한계점이라는 것을 배웠습니다. 따라서 서버는 크고 많을수록 좋은 거 같습니다.
