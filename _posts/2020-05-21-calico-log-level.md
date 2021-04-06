---
layout: post
title: Calico Log Level
categories:
  - Troubleshooting
tags:
  - Infra
  - DevOps
  - Kubernetes
  - Calico
  - Elasticsearch
---

개발용 Kubernetes에 설치한 Grafana에서 경보메세지가 왔다.
Grafana에 설정한 알럿은 디스크 관련 경보 뿐이었으니 디스크에 문제가 생긴 것이 분명했다.
로깅용 Elasticsearch의 디스크 용량을 꽤 낮게 잡았고 별도로 로그 삭제 정책을 설정하지 않았기 때문에
디스크 사용량이 금새 임계치를 넘어가는 것은 예견된 문제였다.

Elasticsearch를 확인해보니 Calico에서 발생하는 info 로그로 인해 생각보다 빨리 디스크가 소모되고 있었다.

개발용 로그는 전부 보존할 필요가 없으므로 일단 오래된 로그들은 삭제한다.

```bash
curl -s -XGET localhost:9200/_cat/indices | awk '$3 ~ /logstash.*/ { i=$3; gsub(/\./,"",i); if (int(substr(i,10,18)) < int('$(date +%Y%m%d --date '30 days ago')')) print "localhost:9200/"$3; }' | xargs curl -XDELETE
```

사실 개발용 서버야 로그를 주기적으로 삭제하는 것만으로도 크게 문제는 안될 것이다.
하지만 서비스가 계속 추가되는 것을 고려하여 불필요한 로그 자체를 줄이는 것이 필요해보였다.

`calico-node` 데몬셋 설정에서 `FELIX_LOGSEVERITYSCREEN` 값을 `warning`으로 변경했다.

```bash
kubectl edit -n kube-system ds calico-node
```

```yaml
...

spec:
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      k8s-app: calico-node
  template:
    metadata:
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ""
      creationTimestamp: null
      labels:
        k8s-app: calico-node
    spec:
      containers:
      - env:
        - name: FELIX_LOGSEVERITYSCREEN
          value: warning

...
```

[Calico Log Level Issue](https://github.com/projectcalico/calico/pull/2116)  
[Calico Documents](https://docs.projectcalico.org/reference/felix/configuration)  
