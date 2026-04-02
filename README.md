# 🚀 Kubernetes DevSecOps 프로젝트

## 📋 목차
1. [프로젝트 개요](#프로젝트-개요)
2. [아키텍처](#아키텍처)
3. [환경 구성](#환경-구성)
4. [Spring Boot 앱 설정](#spring-boot-앱-설정)
5. [K8s 리소스 구성](#k8s-리소스-구성)
6. [MySQL Master-Slave 이중화](#mysql-master-slave-이중화)
7. [DevSecOps 적용](#devsecops-적용)
8. [배포 순서](#배포-순서)
9. [검증 방법](#검증-방법)
10. [트러블슈팅](#트러블슈팅)
11. [추후 발전 방향](#추후-발전-방향)

---

## 프로젝트 개요

Spring Boot 애플리케이션과 MySQL을 Kubernetes 환경에서 **실무 구조**로 운영하는 프로젝트입니다.

### 핵심 요구사항
- Spring Boot App과 MySQL을 **파드 분리**하여 실무 구조 구현
- ConfigMap + Secret으로 **민감정보 분리** 적용
- NetworkPolicy, ResourceLimit 등 **DevSecOps** 구현
- MySQL **Master-Slave 이중화** 및 **읽기/쓰기 분리**

---

## 아키텍처

<img src="./imgs/architecture.png">

### 노드 구성
| 노드 | 역할 | IP |
|------|------|-----|
| server01 | Control Plane | 10.0.2.15 |
| server02 | Worker Node | 10.0.2.20 |
| server03 | Worker Node | 10.0.2.25 |

---

## 환경 구성

### Kubernetes 클러스터
- K8s 버전: v1.30.14
- CNI: Calico v3.27.3
- Container Runtime: containerd 1.7.28

### 기술 스택
- Spring Boot 3.5.10
- Java 17
- MySQL 8.0
- Gradle

---

## Spring Boot 앱 설정

### application.properties

```properties
spring.application.name=step04_empApp

server.servlet.context-path=/emp
server.port=8081

# JSP 파일 경로 설정
spring.mvc.view.prefix=/WEB-INF/views/
spring.mvc.view.suffix=.jsp

# Master DataSource (쓰기)
spring.datasource.master.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.master.jdbc-url=jdbc:mysql://${MYSQL_MASTER_HOST}:${MYSQL_PORT}/${MYSQL_DATABASE}?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=Asia/Seoul
spring.datasource.master.username=${MYSQL_USER}
spring.datasource.master.password=${MYSQL_PASSWORD}

# Slave DataSource (읽기)
spring.datasource.slave.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.slave.jdbc-url=jdbc:mysql://${MYSQL_SLAVE_HOST}:${MYSQL_PORT}/${MYSQL_DATABASE}?useSSL=false&allowPublicKeyRetrieval=true&serverTimezone=Asia/Seoul
spring.datasource.slave.username=${MYSQL_USER}
spring.datasource.slave.password=${MYSQL_PASSWORD}

# JPA/Hibernate
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.hibernate.ddl-auto=update
spring.jpa.database-platform=org.hibernate.dialect.MySQLDialect
```

> ⚠️ HikariCP 직접 설정 시 `url` 대신 `jdbc-url` 을 사용해야 함

### RoutingDataSource.java

```java
package edu.fisa.ce.config;

import org.springframework.jdbc.datasource.lookup.AbstractRoutingDataSource;
import org.springframework.transaction.support.TransactionSynchronizationManager;

public class RoutingDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
            ? "slave" : "master";
    }
}
```

### DataSourceConfig.java

```java
package edu.fisa.ce.config;

import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.jdbc.DataSourceBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;

import javax.sql.DataSource;
import java.util.HashMap;
import java.util.Map;

@Configuration
public class DataSourceConfig {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.master")
    public DataSource masterDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.slave")
    public DataSource slaveDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Primary
    @Bean
    public DataSource routingDataSource(
            @Qualifier("masterDataSource") DataSource masterDataSource,
            @Qualifier("slaveDataSource") DataSource slaveDataSource) {

        RoutingDataSource routingDataSource = new RoutingDataSource();

        Map<Object, Object> dataSourceMap = new HashMap<>();
        dataSourceMap.put("master", masterDataSource);
        dataSourceMap.put("slave", slaveDataSource);

        routingDataSource.setTargetDataSources(dataSourceMap);
        routingDataSource.setDefaultTargetDataSource(masterDataSource);

        return routingDataSource;
    }
}
```

### Service 클래스 @Transactional 적용

```java
// 읽기 메소드 → Slave 라우팅
@Override
@Transactional(readOnly = true)
public List<DeptDTO> getDeptAll() { ... }

// 쓰기 메소드 → Master 라우팅
@Override
@Transactional
public Emp2DTO addEmp(Emp2DTO emp) { ... }
```

> ⚠️ `jakarta.transaction.Transactional` 이 아닌 `org.springframework.transaction.annotation.Transactional` 을 사용해야 라우팅이 동작함

### Dockerfile

```dockerfile
FROM openjdk:17
ARG JAR_FILE=*.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

---

## K8s 리소스 구성

### secret.yaml

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
type: Opaque
data:
  MYSQL_USER: dXNlcjAx          # user01 (base64)
  MYSQL_PASSWORD: dXNlcjAx      # user01 (base64)
  MYSQL_ROOT_PASSWORD: cm9vdA== # root (base64)
```

> Base64 인코딩 방법: `echo -n 'user01' | base64`

### configmap.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  MYSQL_MASTER_HOST: "mysql-master"
  MYSQL_SLAVE_HOST: "mysql-slave"
  MYSQL_PORT: "3306"
  MYSQL_DATABASE: "fisa"
```

### mysql-master-config.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-master-config
data:
  my.cnf: |
    [mysqld]
    server-id=1
    log-bin=mysql-bin
    binlog-do-db=fisa
```

### mysql-slave-config.yaml

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-slave-config
data:
  my.cnf: |
    [mysqld]
    server-id=2
    relay-log=relay-log
    read-only=1
```

### mysql-master.yaml

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-master
spec:
  serviceName: mysql-master
  replicas: 1
  selector:
    matchLabels:
      app: mysql-master
  template:
    metadata:
      labels:
        app: mysql-master
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_ROOT_PASSWORD
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_USER
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_PASSWORD
            - name: MYSQL_DATABASE
              value: "fisa"
          volumeMounts:
            - name: mysql-config
              mountPath: /etc/mysql/conf.d
            - name: mysql-data
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-config
          configMap:
            name: mysql-master-config
        - name: mysql-data
          emptyDir: {}   # ⚠️ 개발 환경용 - 파드 재시작 시 데이터 초기화됨
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-master
spec:
  type: ClusterIP
  selector:
    app: mysql-master
  ports:
    - port: 3306
      targetPort: 3306
```

### mysql-slave.yaml

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql-slave
spec:
  serviceName: mysql-slave
  replicas: 1
  selector:
    matchLabels:
      app: mysql-slave
  template:
    metadata:
      labels:
        app: mysql-slave
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_ROOT_PASSWORD
            - name: MYSQL_DATABASE
              value: "fisa"
          volumeMounts:
            - name: mysql-config
              mountPath: /etc/mysql/conf.d
            - name: mysql-data
              mountPath: /var/lib/mysql
      volumes:
        - name: mysql-config
          configMap:
            name: mysql-slave-config
        - name: mysql-data
          emptyDir: {}   # ⚠️ 개발 환경용 - 파드 재시작 시 데이터 초기화됨
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-slave
spec:
  type: ClusterIP
  selector:
    app: mysql-slave
  ports:
    - port: 3306
      targetPort: 3306
```

### app-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spring-app
  template:
    metadata:
      labels:
        app: spring-app
    spec:
      containers:
        - name: spring-app
          image: ksj3865/kubedevops:v3
          ports:
            - containerPort: 8081
          env:
            - name: MYSQL_MASTER_HOST
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: MYSQL_MASTER_HOST
            - name: MYSQL_SLAVE_HOST
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: MYSQL_SLAVE_HOST
            - name: MYSQL_PORT
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: MYSQL_PORT
            - name: MYSQL_DATABASE
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: MYSQL_DATABASE
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_USER
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_PASSWORD
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          readinessProbe:
            tcpSocket:
              port: 8081
            initialDelaySeconds: 30
            periodSeconds: 10
          livenessProbe:
            tcpSocket:
              port: 8081
            initialDelaySeconds: 60
            periodSeconds: 30
```

### app-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: spring-app-service
spec:
  type: NodePort
  selector:
    app: spring-app
  ports:
    - port: 80
      targetPort: 8081
      nodePort: 30080
```

### networkpolicy.yaml

```yaml
# mysql-master 접근을 spring-app 파드만 허용
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: mysql-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: mysql-master
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: spring-app
      ports:
        - protocol: TCP
          port: 3306
```

---

## MySQL Master-Slave 이중화

### 1. Master 복제 계정 생성

```bash
kubectl exec -it mysql-master-0 -- mysql -u root -proot -e \
  "CREATE USER 'replicator'@'%' IDENTIFIED WITH mysql_native_password BY 'replicator123';"
kubectl exec -it mysql-master-0 -- mysql -u root -proot -e \
  "GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';"
kubectl exec -it mysql-master-0 -- mysql -u root -proot -e "FLUSH PRIVILEGES;"
```

### 2. Master 상태 확인 (File, Position 값 메모 필수!)

```bash
kubectl exec -it mysql-master-0 -- mysql -u root -proot -e "SHOW MASTER STATUS;"
```

### 3. Slave 복제 설정

```bash
# MASTER_LOG_FILE, MASTER_LOG_POS는 위 SHOW MASTER STATUS 결과값으로 교체
kubectl exec -it mysql-slave-0 -- mysql -u root -proot -e \
  "CHANGE MASTER TO
    MASTER_HOST='mysql-master',
    MASTER_USER='replicator',
    MASTER_PASSWORD='replicator123',
    MASTER_LOG_FILE='{File 값}',
    MASTER_LOG_POS={Position 값},
    GET_MASTER_PUBLIC_KEY=1;"

kubectl exec -it mysql-slave-0 -- mysql -u root -proot -e "START SLAVE;"
```

### 4. 복제 상태 확인

```bash
kubectl exec -it mysql-slave-0 -- mysql -u root -proot -e "SHOW SLAVE STATUS\G" | \
  grep -E "Slave_IO_Running|Slave_SQL_Running"
# 둘 다 Yes 이면 정상!
```

---

## DevSecOps 적용

### 1. Secret으로 민감정보 분리
- DB 비밀번호를 Base64 인코딩하여 Secret에 저장
- ConfigMap에는 비민감 설정(HOST, PORT, DATABASE)만 저장

### 2. NetworkPolicy로 네트워크 격리
- mysql-master에 spring-app 파드만 접근 허용
- 다른 파드에서 MySQL 접근 차단

### 3. Resource Limit
- CPU/메모리 requests/limits 설정으로 DoS 방지 및 안정적 운영

### 4. ReadinessProbe / LivenessProbe
- tcpSocket으로 헬스체크, 비정상 파드 자동 감지 및 재시작

---

## 배포 순서

```bash
# 1. Secret
kubectl apply -f secret.yaml

# 2. ConfigMap
kubectl apply -f configmap.yaml
kubectl apply -f mysql-master-config.yaml
kubectl apply -f mysql-slave-config.yaml

# 3. MySQL Master 배포
kubectl apply -f mysql-master.yaml
kubectl wait --for=condition=ready pod -l app=mysql-master --timeout=120s

# 4. 데이터 복원
kubectl cp fisa_clean.sql mysql-master-0:/tmp/fisa_clean.sql
kubectl exec -it mysql-master-0 -- mysql -u root -proot fisa -e "source /tmp/fisa_clean.sql"

# 5. 복제 계정 생성
kubectl exec -it mysql-master-0 -- mysql -u root -proot -e \
  "CREATE USER 'replicator'@'%' IDENTIFIED WITH mysql_native_password BY 'replicator123';"
kubectl exec -it mysql-master-0 -- mysql -u root -proot -e \
  "GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';"

# 6. Master 상태 확인 (File, Position 메모!)
kubectl exec -it mysql-master-0 -- mysql -u root -proot -e "SHOW MASTER STATUS;"

# 7. MySQL Slave 배포
kubectl apply -f mysql-slave.yaml
kubectl wait --for=condition=ready pod -l app=mysql-slave --timeout=120s

# 8. Slave 복제 설정 (File, Position은 6번 결과값으로 교체)
kubectl exec -it mysql-slave-0 -- mysql -u root -proot -e \
  "CHANGE MASTER TO MASTER_HOST='mysql-master', MASTER_USER='replicator',
   MASTER_PASSWORD='replicator123', MASTER_LOG_FILE='{File 값}',
   MASTER_LOG_POS={Position 값}, GET_MASTER_PUBLIC_KEY=1;"
kubectl exec -it mysql-slave-0 -- mysql -u root -proot -e "START SLAVE;"

# 9. Spring Boot 배포
kubectl apply -f app-deployment.yaml
kubectl apply -f app-service.yaml

# 10. NetworkPolicy 배포
kubectl apply -f networkpolicy.yaml

# 11. 전체 확인
kubectl get pods -o wide
kubectl get services
kubectl get networkpolicy
```

---

## 검증 방법

### 1. 파드 상태 확인

```bash
kubectl get pods -o wide
# mysql-master-0       1/1   Running
# mysql-slave-0        1/1   Running
# spring-app-xxx (x3)  1/1   Running
```

### 2. 앱 접속 테스트

```bash
# 어느 워커 노드 IP로 접속해도 동일하게 동작
curl http://10.0.2.20:30080/emp
curl http://10.0.2.25:30080/emp
```

### 3. Master-Slave 복제 확인

```bash
# Master에 데이터 추가
kubectl exec -it mysql-master-0 -- mysql -u root -proot fisa -e \
  "INSERT INTO dept VALUES(60, 'DEV', 'SEOUL');"

# Slave에서 복제 확인 (60번 부서가 보이면 복제 성공!)
kubectl exec -it mysql-slave-0 -- mysql -u root -proot fisa -e \
  "SELECT * FROM dept;"
```

### 4. NetworkPolicy 동작 확인

```bash
# Spring Boot 파드에서 MySQL 접근 (응답 있어야 함 ✅)
kubectl exec -it {spring-app Pod명} -- wget -qO- mysql-master:3306

# 관계없는 파드에서 MySQL 접근 (타임아웃 되어야 함 ✅)
kubectl run testpod --image=busybox:1.28 --rm -it --restart=Never -- sh
wget -qO- mysql-master:3306   # 응답 없음 = 차단 성공
```

---

## 트러블슈팅

### 1. MySQL Pod Pending 상태

**증상**
```
mysql-0   0/1   Pending   0   77s
Events: pod has unbound immediate PersistentVolumeClaims
```

**원인** PVC가 연결될 스토리지 없음

**해결**
```yaml
volumes:
  - name: mysql-data
    emptyDir: {}   # volumeClaimTemplates 제거 후 emptyDir 사용
```

---

### 2. Spring Boot → MySQL DNS 연결 실패

**증상**
```
java.net.UnknownHostException: mysql-service
```

**원인** Calico CNI 노드 간 IP 충돌로 노드 간 DNS 통신 불가

**해결**
```bash
DATASTORE_TYPE=kubernetes KUBECONFIG=~/.kube/config \
  calicoctl patch node server02 -p '{"spec":{"bgp":{"ipv4Address":"10.0.2.20/24"}}}'

kubectl delete pod -n kube-system -l k8s-app=calico-node
```

---

### 3. MySQL Slave 복제 인증 오류

**증상**
```
Authentication plugin 'caching_sha2_password' requires secure connection
```

**원인** MySQL 8.0 기본 인증방식이 보안 연결 필요

**해결**
```bash
kubectl exec -it mysql-master-0 -- mysql -u root -proot -e \
  "ALTER USER 'replicator'@'%' IDENTIFIED WITH mysql_native_password BY 'replicator123';"
```

---

### 4. Slave SQL_Running: No 오류

**증상**
```
Slave_IO_Running: Yes
Slave_SQL_Running: No  (Last_Errno: 1396)
```

**원인** Master의 CREATE USER 명령이 Slave에 복제되면서 계정 없어 충돌

**해결**
```bash
kubectl exec -it mysql-slave-0 -- mysql -u root -proot -e \
  "CREATE USER 'replicator'@'%' IDENTIFIED WITH mysql_native_password BY 'replicator123';"

kubectl exec -it mysql-slave-0 -- mysql -u root -proot -e "STOP SLAVE;"
kubectl exec -it mysql-slave-0 -- mysql -u root -proot -e "SET GLOBAL SQL_SLAVE_SKIP_COUNTER=1;"
kubectl exec -it mysql-slave-0 -- mysql -u root -proot -e "START SLAVE;"
```

---

### 5. HikariCP jdbcUrl 오류

**증상**
```
HikariPool-1 - jdbcUrl is required with driverClassName
```

**원인** `@ConfigurationProperties`로 직접 DataSource 설정 시 HikariCP는 `url` 대신 `jdbc-url` 요구

**해결**
```properties
# 변경 전
spring.datasource.master.url=jdbc:mysql://...

# 변경 후
spring.datasource.master.jdbc-url=jdbc:mysql://...
```

---

### 6. @Transactional 라우팅 안 되는 문제

**증상** `readOnly=true` 설정해도 Master로만 요청이 감

**원인** `jakarta.transaction.Transactional` 사용 시 `TransactionSynchronizationManager` 동작 안 함

**해결**
```java
// 변경 전
import jakarta.transaction.Transactional;

// 변경 후
import org.springframework.transaction.annotation.Transactional;
```

---

### 7. securityContext로 파드가 죽는 문제

**증상** `CrashLoopBackOff` 또는 `CreateContainerConfigError`

**원인** Dockerfile에 USER 설정이 없어 root로 실행되는데 `runAsNonRoot: true` 설정 시 충돌

**해결 A** (빠른 해결)
```yaml
# app-deployment.yaml에서 securityContext 제거
```

**해결 B** (완전한 해결)
```dockerfile
RUN useradd -m appuser
USER appuser
```

---

## 최종 구현 요약

```
✅ Spring Boot (replicas: 3) + MySQL Master-Slave 파드 분리
✅ Secret (민감정보) + ConfigMap (설정) 분리
✅ MySQL Master-Slave 복제 (DB 이중화)
✅ 읽기/쓰기 분리 (RoutingDataSource)
✅ NetworkPolicy (mysql-master 접근 제한)
✅ Resource Limit (DoS 방지)
✅ ReadinessProbe / LivenessProbe (헬스체크)
✅ Rolling Update (무중단 배포)

⚠️ emptyDir 사용 중 (개발 환경) → 운영 환경에서는 PVC 적용 필요
```

---

## 추후 발전 방향

### 1. PVC 적용 (데이터 영속성 확보)

현재 `emptyDir` 을 사용하고 있어 파드가 재시작되면 MySQL 데이터가 초기화됩니다. 운영 환경에서는 반드시 PVC로 교체해야 합니다.

```yaml
# mysql-master.yaml 수정
volumes:
  - name: mysql-data
    persistentVolumeClaim:
      claimName: mysql-master-pvc
```

```yaml
# mysql-master-pvc.yaml 추가
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-master-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/mysql-master
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - server02
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-master-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

---

### 2. Jenkins CI/CD 파이프라인 구축

코드 변경 시 자동으로 빌드 → 보안 스캔 → 배포까지 이어지는 파이프라인을 구축합니다.

#### 전체 흐름

```
코드 Push (GitHub)
      ↓
Jenkins 감지 (Webhook)
      ↓
Gradle 빌드
      ↓
Docker 이미지 빌드
      ↓
Trivy 보안 스캔  ← DevSecOps 핵심
      ↓
취약점 없으면 Docker Hub 푸시
      ↓
K8s 자동 배포 (kubectl rollout)
```

#### Jenkinsfile 예시

```groovy
pipeline {
    agent any

    environment {
        IMAGE_NAME = "ksj3865/kubedevops"
        IMAGE_TAG  = "v${BUILD_NUMBER}"
    }

    stages {
        stage('Build') {
            steps {
                sh './gradlew clean build -x test'
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Trivy Scan') {
            steps {
                sh "trivy image --exit-code 1 --severity HIGH,CRITICAL ${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh "docker login -u $USER -p $PASS"
                    sh "docker push ${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage('Deploy to K8s') {
            steps {
                sh "kubectl set image deployment/spring-app spring-app=${IMAGE_NAME}:${IMAGE_TAG}"
                sh "kubectl rollout status deployment/spring-app"
            }
        }
    }

    post {
        failure {
            echo '빌드 또는 보안 스캔 실패 - 배포 중단'
        }
    }
}
```

> Trivy 단계에서 HIGH/CRITICAL 취약점 발견 시 파이프라인이 즉시 중단되어 안전하지 않은 이미지는 배포되지 않습니다.
