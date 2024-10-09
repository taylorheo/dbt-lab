
# dbt-lab

이 레포지토리는 Docker Compose를 사용하여 단일 호스트에서 dbt, Apache Spark, JupyterLab을 구성한 데이터 엔지니어링 실험 환경을 제공합니다.

Docker Desktop 환경의 Docker Compose를 통해 쉽게 구성 가능하도록 작성되었으며, Kubernetes 기반 구성은 추후 추가할 예정입니다.

> 작성 및 테스트는 x86 기반의 macOS 진행되었으므로 arm64 기반의 운영체제에서는 정상 작동을 보장하지 않습니다.


## 1. 사전 준비 사항

- 호스트 운영체제에 Docker 및 Docker Compose 3가 설치되어 있어야 합니다.


## 2. 서비스

- **dbt**: 데이터 트랜스포메이션을 위한 도구로, 데이터 모델링과 ETL 작업을 수행
- **Spark Master**: Apache Spark 클러스터의 마스터 노드
- **Spark Worker**: Apache Spark 클러스터의 워커 노드
- **JupyterLab**: 데이터 분석 및 시각화를 위한 대화형 개발 환경

## 2. 소스 구조

```
.
├── dbt/
├── notebooks/
├── jupyterhub/
│   └── jupyter_notebook_config.py
├── docker-compose.yaml
└── README.md
```

- `dbt/`: dbt 프로젝트 파일을 저장하는 디렉토리
- `notebooks/`: Jupyter Notebook(.ipynb) 파일을 저장하는 디렉토리
- `jupyterhub/jupyter_notebook_config.py`: JupyterLab 설정 파일



## 3. 사용 방법

### 1. 소스코드 클론

```bash
git clone [Repository URL]
cd [Repository Directory]
```


### 2. Docker Compose 실행

```bash
docker-compose up -d
```

- `-d` 옵션은 백그라운드에서 컨테이너를 실행합니다.


### 3. JupyterLab 접속

웹 브라우저에서 `http://localhost:8888`에 접속하여 JupyterLab을 사용합니다.

- 실험 환경이기 때문에 토큰 인증은 비활성화되어 있습니다. __보안 사고 방지를 위해 절대 운영환경에서 사용하지 마세요.__


### 4. Spark 클러스터 확인

- Spark Master Web UI: `http://localhost:8080`
- Spark Master는 `spark://spark-master:7077`에서 실행 중입니다.


### 5. JupyterLab에서 PySpark 사용

노트북에서 다음과 같이 SparkSession을 생성하여 Spark 클러스터에 연결할 수 있습니다.

테스트 코드는 `/notebooks/test.ipynb`에도 작성되어 있습니다.

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .appName("JupyterLabPySpark") \
    .master("spark://spark-master:7077") \
    .getOrCreate()
```

### 6. dbt 사용

`dbt` 컨테이너에 접속하여 dbt 명령어를 실행할 수 있습니다.

```bash
docker exec -it dbt bash
```

- 작업 디렉토리는 `/dbt`로 설정되어 있습니다.


## 환경 변수 및 설정

- 모든 서비스는 `spark_network`라는 Docker 네트워크를 통해 통신합니다.
- Apache Spark 버전은 3.5.0, Python 버전은 3.9로 설정되어 있습니다.
- 환경 변수 `PYSPARK_PYTHON`과 `PYSPARK_DRIVER_PYTHON`을 통해 Python 인터프리터를 지정합니다.
- `JUPYTER_TOKEN` 환경 변수를 비워둠으로써 JupyterLab의 토큰 인증을 비활성화했습니다.


## 컨테이너 상세 정보

### dbt

- **이미지**: `ghcr.io/dbt-labs/dbt-core:latest`
- **컨테이너 이름**: `dbt`
- **볼륨 매핑**: `./dbt` → `/dbt`
- **작업 디렉토리**: `/dbt`
- **명령어**: `sleep infinity` (필요할 때 명령어를 실행하기 위해 대기 상태로 유지)

### Spark Master

- **이미지**: `bitnami/spark:3.5.0`
- **컨테이너 이름**: `spark-master`
- **환경 변수**:
  - `SPARK_MODE=master`
  - 기타 Spark 보안 설정 비활성화
- **포트 매핑**:
  - `7077:7077` (Spark 마스터)
  - `8080:8080` (Spark 웹 UI)

### Spark Worker

- **이미지**: `bitnami/spark:3.5.0`
- **컨테이너 이름**: `spark-worker`
- **환경 변수**:
  - `SPARK_MODE=worker`
  - `SPARK_MASTER_URL=spark://spark-master:7077`
  - `SPARK_WORKER_MEMORY=1G`
  - `SPARK_WORKER_CORES=1`
- **의존성**: `spark-master` 컨테이너가 실행된 후 시작됩니다.

### JupyterLab

- **이미지**: `jupyter/pyspark-notebook:x86_64-spark-3.5.0` (Python 3.9 기반)
- **컨테이너 이름**: `jupyterlab`
- **포트 매핑**: `8888:8888` (JupyterLab)
- **볼륨 매핑**:
  - `./notebooks` → `/home/jovyan/work`
  - `./jupyterhub/jupyter_notebook_config.py` → `/home/jovyan/.jupyter/jupyter_notebook_config.py`
- **환경 변수**:
  - `SPARK_MASTER=spark://spark-master:7077`
  - `JUPYTER_ENABLE_LAB=yes`
  - `JUPYTER_TOKEN=""` (토큰 인증 비활성화)
  - `PYSPARK_PYTHON=python3`
  - `PYSPARK_DRIVER_PYTHON=python3`
- **의존성**: `spark-master` 컨테이너가 실행된 후 시작됩니다.


## 라이선스

이 프로젝트는 MIT 라이선스를 따릅니다.