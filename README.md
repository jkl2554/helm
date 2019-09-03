# Failover
ConfigMap 수정을 통해 Failover작업 실행

```
  sentinel.conf: |
{{- if .Values.sentinel.customConfig }}
{{ .Values.sentinel.customConfig | indent 4 }}
{{- else }}
    dir "/data"
    {{- $root := . -}}
    {{- range $key, $value := .Values.sentinel.config }}
    sentinel {{ $key }} {{ $root.Values.redis.masterGroupName }} {{ $value }}
    {{- end }}
    # 추가된 행(Failover 시 failover.sh스크립트 실행)
    sentinel client-reconfig-script mymaster /data/conf/failover.sh
{{- if .Values.auth }}
    sentinel auth-pass {{ .Values.redis.masterGroupName }} replace-default-auth
{{- end }}
{{- end }}

  failover.sh: |
    #!/usr/bin/env sh
    FAILOVERIP=$6
    if [ -z "$FAILOVERIP" ]; then
        FAILOVERIP=$1
    fi
    echo $FAILOVERIP
    MASTERPOD=`getent hosts "$FAILOVERIP" |awk '{ print $3}'| awk -F '.' '{print $1}'`
    if [ -z "$MASTERPOD" ]; then
        echo "MASTER is Not Found"
    else
        /data/conf/kubectl label pod --all role=slave --overwrite -n {{.Release.Namespace}}
        /data/conf/kubectl label pod $MASTERPOD role=master --overwrite -n {{.Release.Namespace}}
    fi
    ...
  
  init.sh: |
    ...

        copy_config() {
        cp /readonly-config/redis.conf "$REDIS_CONF"
        cp /readonly-config/sentinel.conf "$SENTINEL_CONF"
        cp /readonly-config/failover.sh /data/conf/

        # kubectl을 사용하기위해 추가된 행 ##
        wget -O /data/conf/stable.txt https://storage.googleapis.com/kubernetes-release/release/stable.txt
        wget -O /data/conf/kubectl https://storage.googleapis.com/kubernetes-release/release/`cat /data/conf/stable.txt`/bin/linux/amd64/kubectl
        chmod +x /data/conf/kubectl 
        ####

        # failover.sh의 실행권한 추가
        chmod +x /data/conf/failover.sh
    }

    setup_defaults() {
        echo "Setting up defaults"
        if [ "$INDEX" = "0" ]; then
            echo "Setting this pod as the default master"
            redis_update "$ANNOUNCE_IP"
            sentinel_update "$ANNOUNCE_IP"
            sed -i "s/^.*slaveof.*//" "$REDIS_CONF"
            # 첫 실행시 마스터 설정을 위해 추가된 행
            /data/conf/kubectl label pod $SERVICE-server-$INDEX role=master --overwrite -n {{.Release.Namespace}}  
        else
            # 서비스의 갯수를 줄이기 위해 변경된 행
            DEFAULT_MASTER="$(getent hosts "$SERVICE-server-0.$SERVICE.{{.Release.Namespace}}.svc.cluster.local" | awk '{ print $1 }')" 
            if [ -z "$DEFAULT_MASTER" ]; then
                echo "Unable to resolve host"
                exit 1
            fi
            echo "Setting default slave config.."
            redis_update "$DEFAULT_MASTER"
            sentinel_update "$DEFAULT_MASTER"
        fi
    }
    ...

```

* announce 서비스를 삭제해 불필요한 서비스 삭제
* master 서비스및 slave서비스를 추가해 master 및 slave에 구분되게 접속할 수 있게 구성함.
* 포드 IP를 통해 다른 포드의 이름을 알기 위해 전체 연결 서비스가 필요
* 여기에 추가적으로 announce서비스 삭제 및 master & slave서비스 추가작업 수행


# 결과
* 처음 생성 시 Statefulset이기 때문에 순서대로 생성되며, 첫번째 pod(redis-failover-ha-server-0)가 마스터로 선정된다.
```
# 헬름을 통한 설치
helm install --name redis-failover --namespace redis-test ./
# 결과 확인
kubectl get po -w --show-labels -n redis-test
NAME                               READY   STATUS     RESTARTS   AGE   LABELS
redis-failover-redis-ha-server-0   0/2     Init:0/1   0          29s   app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-0
redis-failover-redis-ha-server-0   0/2     Init:0/1   0          82s   app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-0
redis-failover-redis-ha-server-0   0/2     Init:0/1   0          83s   app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=master,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-0
redis-failover-redis-ha-server-0   0/2     PodInitializing   0          84s   app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=master,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-0
redis-failover-redis-ha-server-0   0/2     Running           0          85s   app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=master,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-0
redis-failover-redis-ha-server-0   1/2     Running           0          99s   app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=master,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-0
redis-failover-redis-ha-server-0   2/2     Running           0          100s   app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=master,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-0
redis-failover-redis-ha-server-1   0/2     Pending           0          0s     app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-1
redis-failover-redis-ha-server-1   0/2     Pending           0          1s     app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-1
redis-failover-redis-ha-server-1   0/2     Pending           0          5s     app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-1
redis-failover-redis-ha-server-1   0/2     Pending           0          15s    app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-1
redis-failover-redis-ha-server-1   0/2     Pending           0          15s    app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-1
redis-failover-redis-ha-server-1   0/2     Init:0/1          0          15s    app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-1
redis-failover-redis-ha-server-1   0/2     Init:0/1          0          86s    app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-1
redis-failover-redis-ha-server-1   0/2     PodInitializing   0          88s    app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-1
redis-failover-redis-ha-server-1   0/2     Running           0          91s    app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-1
redis-failover-redis-ha-server-1   1/2     Running           0          106s   app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-1
redis-failover-redis-ha-server-1   2/2     Running           0          110s   app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-1
redis-failover-redis-ha-server-2   0/2     Pending           0          0s     app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-2
redis-failover-redis-ha-server-2   0/2     Pending           0          0s     app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-2
redis-failover-redis-ha-server-2   0/2     Pending           0          10s    app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-2
redis-failover-redis-ha-server-2   0/2     Pending           0          10s    app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-2
redis-failover-redis-ha-server-2   0/2     Init:0/1          0          10s    app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-2
redis-failover-redis-ha-server-2   0/2     Init:0/1          0          49s    app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-2
redis-failover-redis-ha-server-2   0/2     PodInitializing   0          51s    app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-2
redis-failover-redis-ha-server-2   0/2     Running           0          53s    app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-2
```
* 최종 생성 결과
```
kubectl get po -n redis-test --show-labels
NAME                               READY   STATUS    RESTARTS   AGE     LABELS
redis-failover-redis-ha-server-0   2/2     Running   0          10m     app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=master,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-0
redis-failover-redis-ha-server-1   2/2     Running   0          8m20s   app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-1
redis-failover-redis-ha-server-2   2/2     Running   0          6m30s   app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-2
```
* 서비스에는  master서비스와 slave서비스가 로드 밸런서 형식으로 생성된다. 
```
kubectl get svc -n redis-test
NAME                             TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)                          AGE
redis-failover-redis-ha          ClusterIP      None           <none>          6379/TCP,26379/TCP               5m29s
redis-failover-redis-ha-master   LoadBalancer   10.0.38.168    52.231.78.248   6379:32682/TCP,26379:30944/TCP   5m29s
redis-failover-redis-ha-slave    LoadBalancer   10.0.204.158   52.231.70.179   6379:30517/TCP,26379:31255/TCP   5m29s
```
* master서비스는 read & write이고, slave서비스는 readonly이다.
```
# master test
redis-cli -h 52.231.78.248
52.231.78.248:6379> set key1 value1
OK
52.231.78.248:6379> get key1
"value1"

# slave test
redis-cli -h 52.231.70.179
52.231.70.179:6379> set key2 value2
(error) READONLY You can't write against a read only replica.
52.231.70.179:6379> get key1
"value1"
```
* redis master pod 삭제작업 수행
```
kubectl delete po redis-failover-redis-ha-server-0 -n redis-test
pod "redis-failover-redis-ha-server-0" deleted
```
* failover가 완료되어 마스터 포드가 변경됨
```
kubectl get po -n redis-test --show-labels
NAME                               READY   STATUS    RESTARTS   AGE     LABELS
redis-failover-redis-ha-server-0   0/2     Running   0          35s     app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-0
redis-failover-redis-ha-server-1   2/2     Running   0          9m46s   app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=slave,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-1
redis-failover-redis-ha-server-2   2/2     Running   0          7m56s   app=redis-ha,controller-revision-hash=redis-failover-redis-ha-server-6dfcd4b66f,release=redis-failover,role=master,statefulset.kubernetes.io/pod-name=redis-failover-redis-ha-server-2
```
* master read & write test
```
redis-cli -h 52.231.78.248
52.231.78.248:6379> get key1
"value1"
52.231.78.248:6379> set key3 value3
OK
52.231.78.248:6379> get key3
"value3"
```