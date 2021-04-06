---
layout: post
title: 'Prometheus Federation'
categories:
  - Note
tags:
  - Infra
  - DevOps
  - Prometheus
  - Federation
---

개발용 쿠버네티스 환경을 점검하면서 프로메테우스와 같은 모니터링 시스템을 클러스터 내부에 두는 것이 올바른 방법일지 의문이 들었다.
보통 쿠버네티스 내부에 프로메테우스를 설치하는 것으로 보이나 만약에 쿠버네티스 클러스터가 완전히 기능을 상실했을 때 다운 당시 어떤 문제가 발생했었는지를 알 수 없는 상태에 빠질 수 있기 때문이다.
클러스터의 가용성이 충분히 보장된다면 모르겠지만 직접 관리하는 클러스터에 이번이 처음으로 쿠버네티스 클러스터를 다루는 것인 만큼 신중을 기하고 싶었다.

우선 앤서블을 통해 쿠버네티스 외부에 프로메테우스를 설치했다.

```yaml
- name: Check if Prometheus installed
  stat:
    path: /usr/local/prometheus
  register: prometheus_server_installed

- name: Download Prometheus
  unarchive:
    src: https://github.com/prometheus/prometheus/releases/download/v2.18.1/prometheus-2.18.1.linux-amd64.tar.gz
    dest: /usr/local/
    remote_src: yes
  when: prometheus_server_installed.stat.exists == false

- name: Create a symbolic link
  file:
    src: /usr/local/prometheus-2.18.1.linux-amd64
    dest: /usr/local/prometheus
    state: link

- name: Create prometheus user
  user:
    name: prometheus
    shell: /bin/false
    create_home: false

- name: Make a Prometheus config directory
  file:
    path: /usr/local/etc/prometheus
    state: directory
    owner: prometheus
    group: prometheus

- name: Make a Prometheus data directory
  file:
    path: /usr/local/var/lib/prometheus
    state: directory
    owner: prometheus
    group: prometheus
    recurse: yes

- name: Copy a Prometheus config file
  copy:
    src: prometheus.yml
    dest: /usr/local/etc/prometheus.yml

- name: Copy a Prometheus systemd service file
  copy:
    src: prometheus.service
    dest: /etc/systemd/system/prometheus.service

- name: Enable and start Prometheus service
  systemd:
    name: prometheus
    state: started
    enabled: yes

- name: Install HTTPD via YUM
  package:
    name:
      - httpd
      - mod_ssl
    state: present

- name: Copy a Prometheus HTTPD config file
  template:
    src: prometheus.conf
    dest: /etc/httpd/conf.d/prometheus.conf
    mode: '0644'

- name: Copy certificate files
  copy:
    src: '{ item.src }'
    dest: '{ item.dest }'
  with_items:
    - src: '{ prometheus_server_ssl_cert_src }'
      dest: '{ prometheus_server_ssl_cert_dest }'
    - src: '{ prometheus_server_ssl_ca_src }'
      dest: '{ prometheus_server_ssl_ca_dest }'
    - src: '{ prometheus_server_ssl_key_src }'
      dest: '{ prometheus_server_ssl_key_dest }'

- name: Enable and start HTTPD
  systemd:
    name: httpd
    state: started
    enabled: yes
```

서비스 파일은 아래와 같이 준비한다.

```yaml
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/prometheus/prometheus \
    --config.file /usr/local/etc/prometheus.yml \
    --storage.tsdb.path /usr/local/var/lib/prometheus/ \

[Install]
WantedBy=multi-user.target
```

프로메테우스가 API를 통해 쿠버네티스의 메트릭을 수집하도록 하려면 API 도메인, CA 인증서, 로그인 토큰이 필요하다.
[프로메테우스에서 제공하는 예시 파일](https://github.com/prometheus/prometheus/blob/master/documentation/examples/prometheus-kubernetes.yml)과 [kubernetes_sd_config 설정 문서](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config)를 확인하여 설정파일을 작성했으며 CA 파일을 리눅스 시스템에서 신뢰하도록 등록해주었다.

프로메테우스가 정상적으로 쿠버네티스 API와 통신하며 일부 메트릭을 수집하는 걸 확인했으나 pod과 같이 클러스터 외부에 오픈되어있지 않은 리소스의 메트릭은 수집하지 못하는 문제가 발생했다.
문제가 발생하고 나서야 왜 프로메테우스를 쿠버네티스 내부에서 구동하는 시나리오가 대부분이었는지 이해가 됐다.

기왕 프로메테우스 서버를 셋업했기 때문에 쿠버네티스와 관련된 메트릭만 기존 쿠버네티스 프로메테우스에서 수집하고 나머지 서버와 서비스들에 대한 메트릭은 새로 셋업한 프로메테우스에서 수집하기로 했다.
우선 프로메테우스를 매번 번갈아가면서 볼 수는 없으므로 페더레이션을 진행한다.

본래라면 쿠버네티스 프로메테우스에 엔드포인트를 오픈하고 해당 엔드포인트를 통해 페더레이션을 구성하는 게 좋겠지만 우선 페더레이션이 정상적으로 동작하는지 확인하기 위해 프로메테우스 서버 내에 kubectl을 설치하여 임시로 포트포워딩했다.

```yaml
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']

  - job_name: 'federate'
    scrape_interval: 15s
    honor_labels: true
    metrics_path: '/federate'
    params:
      'match[]':
        - '{job="kubernetes-apiservers"}'
        - '{job="kubernetes-nodes"}'
        - '{job="kubernetes-pods"}'
        - '{job="kubernetes-nodes-cadvisor"}'
        - '{job="kubernetes-service-endpoints"}'
    static_configs:
      - targets:
        - 'localhost:9091'
```

간단한 설정으로 프로메테우스에서 쿠버네티스 프로메테우스의 메트릭을 정상적으로 읽을 수 있었다.
이제 쿠버네티스 프로메테우스 서버를 오픈하여 다시 페더레이션을 진행하면 될 것 같다.
하는 김에 `kubernetes-nodes`나 `kubernetes-apiservers` 같이 외부에서도 충분히 메트릭을 수집할 수 있는 잡들은 외부 프로메테우스에서 진행하도록 처리하면 좋을 것 같다.
