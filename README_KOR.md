# Kafka 클러스터 Ansible 배포

이 저장소는 Zookeeper를 포함한 Kafka 클러스터를 배포하고 관리하기 위한 Ansible 기반 솔루션을 제공합니다. 이는 Prometheus 모니터링이 포함된 고가용성, 내결함성 Kafka 클러스터를 설정하기 위한 자동화되고 재현 가능한 방법을 제공합니다.

## 목차

1. [기능](#기능)
2. [사전 요구 사항](#사전-요구-사항)
3. [시작하기](#시작하기)
   - [Vagrant 환경](#vagrant-환경)
   - [프로덕션 배포](#프로덕션-배포)
4. [구성](#구성)
   - [Zookeeper](#zookeeper)
   - [Kafka](#kafka)
   - [모니터링](#모니터링)
5. [보안 고려 사항](#보안-고려-사항)
6. [교훈](#교훈)
7. [기여하기](#기여하기)
8. [라이선스](#라이선스)

## 기능

- 3개의 Zookeeper 노드와 3개의 Kafka 브로커가 있는 Kafka 클러스터 자동 배포
- 배포 중 Kafka 힙 크기 동적 구성
- Kafka 및 Zookeeper를 위한 통합 Prometheus 모니터링
- 다양한 배포 환경 지원 (Vagrant, AWS, 온프레미스)
- 재사용 가능하고 사용자 정의 가능한 Ansible 역할 및 플레이북

## 사전 요구 사항

- Ansible 2.9 이상
- Docker 및 Docker Compose
- Vagrant (로컬 테스트용)
- 대상 배포 환경에 대한 액세스 (Vagrant, AWS, 온프레미스)

## 시작하기

### Vagrant 환경

1. 저장소 복제:

   ```bash
   git clone https://github.com/thenameisvikash/kafkacluseterplaybook.git
   cd kafkacluseterplaybook
   ```

2. Vagrant 환경 시작:

   ```bash
   vagrant up
   ```

3. Kafka 클러스터 배포:

   ```bash
   ansible-playbook -i vagrant.yml deploy-kafka-cluster.yml
   ```

### 프로덕션 배포

1. 저장소 복제:

   ```bash
   git clone https://github.com/thenameisvikash/kafkacluseterplaybook.git
   cd kafkacluseterplaybook
   ```

2. 배포 환경에 맞게 인벤토리 파일 업데이트 (예: `inventory/aws.yml`, `inventory/onprem_keybased.yml`).

3. Kafka 클러스터 배포:

   ```bash
   ansible-playbook -i onprem_sudopassword.yml deploy-kafka-cluster.yml
   ```

`<인벤토리_파일>`을 환경에 맞는 적절한 인벤토리 파일로 교체하세요.

## 구성

### Zookeeper

Zookeeper 구성은 `group_vars/all.yml` 파일에 정의되어 있습니다. 주요 설정은 다음과 같습니다:

- `zookeeper_data_dir`: Zookeeper 데이터 경로
- `zookeeper_datalog_dir`: Zookeeper 데이터 로그 경로
- `zookeeper_servers`: Zookeeper 앙상블 정의

### Kafka

Kafka 구성도 `group_vars/all.yml` 파일에 정의되어 있습니다. 주요 설정은 다음과 같습니다:

- `kafka_port`: Kafka 브로커가 사용하는 포트
- `kafka_log_segment_bytes`: 단일 로그 파일의 최대 크기
- `kafka_log_retention_bytes`: 로그 파일의 최대 총 크기
- `kafka_log_retention_ms`: 로그 파일의 최대 보존 시간

### 모니터링

Ansible 플레이북은 Kafka 및 Zookeeper를 위한 Prometheus 모니터링을 설정합니다. Prometheus 구성은 `roles/common/templates/prometheus-config.yml` 파일에 정의되어 있습니다.

## 보안 고려 사항

제공된 설정에는 다음과 같은 기본 보안 조치가 포함됩니다:

- Kafka에 대한 SSL/TLS 암호화 활성화
- SASL 인증 구현
- 주제에 대한 접근을 제어하기 위한 ACL 구성

그러나 특정 요구 사항과 모범 사례에 따라 보안 조치를 검토하고 강화하는 것이 중요합니다.

## 교훈

- 적절한 로그 보존 설정으로 시작하고 필요에 따라 확장
- Kafka 클러스터를 면밀히 모니터링하고 중요 이벤트에 대한 경고 설정
- 보안은 일회성이 아닌 지속적인 프로세스
- Zookeeper 데이터를 정기적으로 백업하고 복원 프로세스 테스트
- 더 쉬운 유지 관리와 문제 해결을 위해 모든 것을 문서화

## 기여하기

기여는 환영합니다! 문제를 발견하거나 개선 제안이 있으시면 새 이슈를 생성하거나 풀 리퀘스트를 제출해 주세요.

## 라이선스

이 프로젝트는 [MIT 라이선스](LICENSE) 하에 라이선스가 부여됩니다.