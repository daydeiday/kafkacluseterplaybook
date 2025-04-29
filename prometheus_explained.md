#### 요약
위 Ansible 태스크는 Kafka 및 ZooKeeper JVM 프로세스에 Prometheus JMX Exporter(Java 에이전트)를 배포하기 위한 준비 작업입니다. 첫 번째 `get_url` 태스크로 Maven 중앙 저장소에서 `jmx_prometheus_javaagent` JAR 파일을 다운로드하고, 두 번째 `template` 태스크로 사용자 정의 `prometheus-config.yml`을 대상 서버의 `/opt` 경로에 복사합니다. 이 구성 파일(`prometheus-config.yml`)의 `rules` 섹션에는 JMX MBean 이름과 속성을 정규표현식으로 매칭해 Prometheus 메트릭 네임과 라벨로 매핑하는 규칙이 정의되어 있습니다. 최종적으로 JVM을 다음과 같이 시작하도록 서비스 단위(예: systemd 유닛)나 환경변수(`KAFKA_OPTS`)를 설정하면, 각 브로커와 ZooKeeper 인스턴스에서 HTTP 포트(예: 7071)로 메트릭을 노출할 수 있게 됩니다.

## 1. Ansible 태스크 설명
### 1.1. get_url 모듈
- **기능**: 원격 호스트에 HTTP, HTTPS, FTP 등으로부터 파일을 다운로드합니다.
- **예제**:
  ```yaml
  - name: Download JMX Prometheus Java Agent
    get_url:
      url: https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.16.1/jmx_prometheus_javaagent-0.16.1.jar
      dest: /opt/jmx_prometheus_javaagent-0.16.1.jar
      mode: '0644'
  ```  
- **설명**: Maven 중앙 리포지터리에서 JMX Prometheus Java Agent 버전 0.16.1의 JAR 파일을 `/opt` 디렉토리로 복사하고, 파일 권한을 `0644`로 설정합니다  ([ansible.builtin.get_url module – Downloads files from HTTP, HTTPS ...](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/get_url_module.html?utm_source=chatgpt.com)).

### 1.2. template 모듈
- **기능**: Jinja2 템플릿을 렌더링하여 대상 호스트에 파일을 생성하거나 업데이트합니다.
- **예제**:
  ```yaml
  - name: Copy Prometheus config file
    template:
      src: prometheus-config.yml
      dest: /opt/prometheus-config.yml
      mode: '0644'
  ```  
- **설명**: 로컬 `templates/` 디렉토리 내의 `prometheus-config.yml` 템플릿을 렌더링하여 `/opt/prometheus-config.yml`로 복사하며, 파일 권한을 `0644`로 설정합니다  ([ansible.builtin.template module – Template a file out to a target host](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html?utm_source=chatgpt.com)).

## 2. JMX Prometheus Java Agent 개요
JMX Prometheus Java Agent는 JVM 프로세스에 `-javaagent` 옵션으로 붙여, MBean 데이터를 HTTP 엔드포인트로 노출하는 미들웨어입니다. Prometheus는 이 엔드포인트를 스크래핑해 JVM 및 애플리케이션 메트릭을 수집할 수 있습니다  ([JMX Exporter & Prometheus | Workshop Monitoring Kafka - Zenika](https://zenika.github.io/workshop-monitor-kafka/5_JMX_EXPORTER_PROMETHEUS.html?utm_source=chatgpt.com), [Monitor Java application with prometheus - DEV Community](https://dev.to/franzwong/monitor-java-application-with-prometheus-1nob?utm_source=chatgpt.com)).

```bash
java -javaagent:/opt/jmx_prometheus_javaagent-0.16.1.jar=7071:/opt/prometheus-config.yml \
     -jar kafka-server-start.jar config/server.properties
```  
위와 같이 시작하면, 포트 `7071`에서 `/metrics` 엔드포인트가 생성됩니다  ([Monitoring Kafka Clusters: Setup Guide for JMX Exporter ... - Medium](https://medium.com/%40mucagriaktas/monitoring-kafka-clusters-setup-guide-for-jmx-exporter-prometheus-and-grafana-71f8fd0ce37c?utm_source=chatgpt.com)).

## 3. prometheus-config.yml 구성 설명
```yaml
rules:
  - pattern: "kafka.server<type=(.+), name=(.+)><>Value"
    name: kafka_server__
  - pattern: "kafka.controller<type=(.+), name=(.+)><>Value"
    name: kafka_controller__
  # 이하 생략…
```
- **rules**: 매칭 순서대로 적용되는 규칙 목록이며, 첫 매칭 규칙에서 멈춥니다.
- **pattern**: JMX ObjectName 및 속성(Attribute)을 정규표현식으로 매칭합니다.
    - 예를 들어 `kafka.server<type=BrokerTopicMetrics, name=MessagesInPerSec><>Value` 형태의 MBean을 `kafka_server__` 네임스페이스로 매핑  ([flokkr/jmxpromo: Prometheus JMX exporter with consul ... - GitHub](https://github.com/flokkr/jmxpromo?utm_source=chatgpt.com)).
- **name**: Prometheus 메트릭 이름을 지정하며, 캡처 그룹(`(.+)`)을 사용해 동적으로 생성할 수 있습니다.
- **기타 옵션**(생략된 예시): `attrNameSnakeCase`, `valueFactor`, `labels`, `help`, `type` 등을 설정할 수 있습니다  ([flokkr/jmxpromo: Prometheus JMX exporter with consul ... - GitHub](https://github.com/flokkr/jmxpromo?utm_source=chatgpt.com)).

## 4. Kafka 클러스터 배포에서의 활용
1. **Agent 배포**: Ansible 플레이북의 `get_url`+`template` 태스크로 JVM 에이전트 및 설정을 각 브로커, ZooKeeper 노드에 배포합니다.
2. **서비스 설정**: Kafka 및 ZooKeeper 시작 스크립트나 systemd 유닛 파일에 `-javaagent` 인자를 추가하거나, `KAFKA_OPTS` 환경변수로 지정합니다  ([Monitoring Kafka Clusters: Setup Guide for JMX Exporter ... - Medium](https://medium.com/%40mucagriaktas/monitoring-kafka-clusters-setup-guide-for-jmx-exporter-prometheus-and-grafana-71f8fd0ce37c?utm_source=chatgpt.com)).
3. **Prometheus 스크래핑**:
   ```yaml
   scrape_configs:
     - job_name: 'kafka-jmx'
       static_configs:
         - targets: ['broker1:7071','broker2:7071','zookeeper1:7071']
   ```  
4. **Grafana 대시보드**: 수집된 메트릭으로 CPU, GC, Kafka Throughput, 소비자/생산자 지표 등을 시각화합니다  ([Monitoring Kafka Clusters: Setup Guide for JMX Exporter ... - Medium](https://medium.com/%40mucagriaktas/monitoring-kafka-clusters-setup-guide-for-jmx-exporter-prometheus-and-grafana-71f8fd0ce37c?utm_source=chatgpt.com)).

---

> 이처럼 Ansible을 활용하면 JMX Exporter 에이전트 배포와 설정 파일 템플릿화를 자동화하여, Kafka 클러스터 전체에 일관된 모니터링 환경을 손쉽게 구축할 수 있습니다.