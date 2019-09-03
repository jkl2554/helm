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
    sentinel client-reconfig-script mymaster /data/conf/failover.sh         # 추가된 행(Failover 시 failover.sh스크립트 실행)
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

    setup_defaults() {
        echo "Setting up defaults"
        if [ "$INDEX" = "0" ]; then
            echo "Setting this pod as the default master"
            redis_update "$ANNOUNCE_IP"
            sentinel_update "$ANNOUNCE_IP"
            sed -i "s/^.*slaveof.*//" "$REDIS_CONF"
            /data/conf/kubectl label pod $SERVICE-server-$INDEX role=master --overwrite -n {{.Release.Namespace}}  # 첫 실행시 마스터 설정을 위해 추가된 행
        else
            DEFAULT_MASTER="$(getent hosts "$SERVICE-server-0.$SERVICE.{{.Release.Namespace}}.svc.cluster.local" | awk '{ print $1 }')" # 서비스의 갯수를 줄이기 위해 변경된 행
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



