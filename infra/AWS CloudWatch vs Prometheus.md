# AWS CloudWatch vs Prometheus

대표적인 모니터링 서비스 CloudWatch와 Prometheus를 비교해 보자!

## AWS CloudWatch란?

![image](https://github.com/COW-edu/COW-Spring-2/assets/59856002/9b6c8db9-e773-4a4a-ad00-aa0a96aa3258)

* AWS 리소스 사용의 실시간 모니터링 기능 지원
* 다양한 이벤트들을 수집하여 로그파일로 저장
  * S3 파일 업로드, 삭제, 접근 거부, RDS 접속 시도 등등 모든 이벤트
* 알림 설정을 통해 SNS나 AWS Lambda로 전송 가능
  * 로그파일을 읽지 않아도 무슨일이 언제 왜 일어났는지 알려준다.
* EC2, RDS, S3, EB에 모두 적용 가능

### 모니터링이란?
서비스 운영 중에 발생할 수 있는 이슈 및 오류에 대비하기 위해 관련 데이터를 수집하고 기록하는 것이다.
애플리케이션 또는 네트워크와 같은 IT 환경에서 측정 가능한 수치나 지표를 말한다.

모니터링 방식에는 2가지가 있다 : pull-based / push-based
이중 Prometheus와 CloudWatch는 모두 pull-based이며, 대상 (target)으로부터 메트릭 값을 받아온다.
Target에 직접적으로 접근하여 데이터를 Scraping하며, 그 대상이 되는 Target은
데이터를 노출시킬 방법이 필요한데, 이러한 생태계가 잘 구축되어 있다.

### 어떤 것을 모니터링 할 것인가?
주 대상은 이벤트이며, 이벤트에는 context가 있다.
예를 들어 HTTP 요청에는 들어오고 나가는 IP 주소, 요청 URL, 쿠키, 사용자 정보가 포함된다.
모든 이벤트에 대한 컨텍스트를 파악하면 디버깅에 큰 도움이 되고 기술적인 관점과 비즈니스 관점 모두에서
시스템의 수행 방법을 이해할 수 있지만, 처리 밎 저장해야 하는 데이터의 양이 늘어난다.

### CloudWatch 모니터링 종류
* Basic Monitoring
  * 5분 간격으로 최소의 Metrics 제공
* Detailed Monitoring
  * 1분 간격으로 자세한 Metrics 제공

### Alarm이란?
* 임의로 정해둔 값에 도달할 시 Alarm이 울린다.
  * 알람에 대한 특정 이벤트를 작동시킬 수 있음.
* 정해놓은 지출 임계값을 초과할 경우 SNS를 통하여 경고를 함

### CloudWatch 용어
* Namespace : 특정 서비스 또는 리소스 그룹에 대한 메트릭들을 묶어서 분류
  * EC2, S3, RDS는 각각 다른 Namespace에 속한다.
  * 해당 네임스페이스에 속한 메트릭을 사용해서 리소스의 상태를 모니터링
* Metric(지표)
  * 리소스의 상태를 Metric(지표)로 기록한다.
  * EC2 CPU 사용률 등 그래프로 사용 가능
* Dimension
  * Metric을 더 세부적으로 분류
  * 메트릭 데이터를 그룹화, 필터링해서 자세한 검색이 된다.

## Prometheus
메트릭 기반의 오픈소스 모니터링 시스템이다.
프로메테우스는 애플리케이션과 인프라의 성능을 분석할 수 있다.
간단한 텍스트 형식을 통해 쉽게 메트릭을 게시할 수 있다.

* Alert는 PromQL이라는 메트릭을 검색하기 위한 고유 쿼리 언어를 통해 정의할 수 있다.
* 결과는 Grafana같은 대시보드 시스템에서 볼 수 있다.(Prometheus + Grafana)의 조합
* k8s나 Docker Engine은 Metirc을 내보내는 Endpoint를 자체적으로 지원하므로, 프로메테우스와 연동 시 별도의 소스코드 없이 모니터링 시스템 구축이 가능하다.

### Architecture
![image](https://github.com/wonjunYou/WIL/assets/59856002/1eb206ac-1753-4f7a-a330-1a997883365b)

* Prometheus-server
  * 다음과 같은 3가지 역할이 있다.
  * Node Exporter, 메트릭을 수집해 오는 수집기
  * 수집한 시계열 metric 데이터를 저장하는 시계열 DB
  * 저장된 데이터의 query, 수집 대상의 상태를 확인할 수 있는 웹 UI.

* Node Exporter
  * 노드의 시스템 메트릭 정보를 HTTP로 공개하는 역할이다.
  * 설치된 노드에서 특정 파일을 읽어 프로메테우스 서버가 수집할 수 있는 Metric으로 변환, Node Exporter에서 HTTP 서버로 공개한다.
  * 대상 : HW, OS, CPU, 메모리, 디스크, 파일시스템 등

* kubelet : 개별 컨테이너 메트릭
* Client Library : 누군가 메트릭을 만들기 위한 계측 코드가 필요함. 이 부분에 관여하는 역할.
  * 코드로 메트릭을 정의하고, 원하는 코드에 인라인으로 계측 기능을 추가할 수 있다.

### 차이점?
프로메테우스와 비교했을 때, CloudWatch는 AWS SQS를 사용하여 알람을 전달 할 수 있지만 
프로메테우스는 다른 시스템과 결합해서 사용하기 보다 손쉽다.(이미 제공하고 있는 라이브러리 + webhook 적용) 
또한 Dashboard를 적용하기 위해서는 CloudWatch는 비용이 추가로 든다.(대시보드당 3$)
반면 프로메테우스는 Grafana를 사용하여 CloudWatch 보다 더 나은 그리고 PromQL을 활용하여 더 다양하게 그려낼 수 있다.

![image](https://github.com/wonjunYou/WIL/assets/59856002/d0911518-efeb-448c-b08e-2e8632f5b2d1)

## References
* https://docs.aws.amazon.com/ko_kr/AmazonCloudWatch/latest/monitoring/cloudwatch_architecture.html
* https://velog.io/@hyunshoon/Monitoring-Prometheus-%EC%B4%9D-%EC%A0%95%EB%A6%AC
* https://hudi.blog/spring-boot-actuator-prometheus-grafana-set-up/
* https://woo-chang.tistory.com/78
* https://timewizhan.tistory.com/entry/Prometheus