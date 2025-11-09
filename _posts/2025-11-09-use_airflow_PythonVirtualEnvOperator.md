---
created: 2025-11-09
title: Airflow PythonVirtualEnvOperator 사용하기
layout: post
tags: [Airflow]
category: 공부
math: true
---

A/B 테스트를 위한 유저 그룹을 생성하는 과정에서 EKS Airflow 환경에 설치되지 않은 라이브러리가 필요했는데요. 이 문제를 해결하기 위해  `PythonVirtualEnvOperator`를 사용하게 되었습니다. 여기저기서 에러를 많이 마주쳐서, 해결 방법을 글로 남겨봅니다.

먼저 기본적인 Airflow DAG 구성은 다음과 같다고 가정하겠습니다.

```python
# 디렉터리 구조
ab_test/
├── etl_dag.py
└── modules/
    ├── extract.py
    ├── transform.py
    └── load.py
```

```python
from airflow.operators.python import PythonOperator, PythonVirtualenvOperator
from modules import extract, transform, load

# ...(DAG 설정 생략)...

with dag:
	extract_task = PythonOperator(
	    task_id='extract',
	    python_callable=extract.extract_function
	)
	
	# 특정 task에서 PythonVirtualenvOperator 사용
	transform_task = PythonVirtualenvOperator(  
	    task_id='transform',
	    python_callable=transform.transform_function,
	    requirements=['special-library==1.0.0']
	)
	
	load_task = PythonOperator(
	    task_id='load',
	    python_callable=load.load_function
	)
	
	extract_task >> transform_task >> load_task
```


## PythonVirtualEnvOperator 동작 방식

우선 PythonVirtualEnvOperator가 어떻게 동작하는지 알아보겠습니다.

PythonVirtualEnvOperator는 이름에서도 알 수 있듯이, PythonOperator와 달리 별도의 서브 프로세스에서 격리된 가상환경을 생성합니다. 가상환경에 필요한 라이브러리를 설치하고 task를 실행한 후, DAG이 완료되면 해당 가상환경과 프로세스를 종료하는 방식입니다.

이 과정에서 PythonVirtualEnvOperator는 함수를 pickle로 직렬화하여 새로운 가상환경 프로세스에 전달하는데요, 이 때 DAG 내의 함수와 외부 모듈의 함수를 저장하는 방식이 다릅니다.

- **DAG 파일(__main__) 내에 정의된 함수:** 함수의 실제 코드를 포함하여 직렬화
- **외부 모듈의 함수:** 모듈 경로와 함수 이름을 참조로 저장(실제 코드는 포함하지 않음)

따라서 외부 모듈에 정의된 함수를 사용하면, 가상환경에서는 `modules/` 디렉터리가 존재하지 않아 역직렬화 시 다음과 같은 에러를 마주하게 됩니다.

```Python
ModuleNotFoundError: No module named 'modules'
TypeError: cannot pickle 'module' object
```


## 에러 해결 방법

해결 방법은 간단합니다. PythonVirtualenvOperator에서 사용할 함수를 **외부 모듈이 아닌 DAG 파일 내부에 정의**하면 됩니다. 이 때 함수 내부에서 필요한 라이브러리는 함수 안에서 import 해야한다는 것을 잊지 마세요:)

```python
from airflow.operators.python import PythonOperator, PythonVirtualenvOperator
from modules import extract, load

# ...(DAG 설정 생략)...

def transform_function():
	import special_library    # 함수 내부에서 import 필요
	pass


with dag:
	extract_task = PythonOperator(
	    task_id='extract',
	    python_callable=extract.extract_function
	)
	
	# 특정 task에서 PythonVirtualenvOperator 사용
	transform_task = PythonVirtualenvOperator(  
	    task_id='transform',
	    python_callable=transform_function,
	    requirements=['special-library==1.0.0']
	)
	
	load_task = PythonOperator(
	    task_id='load',
	    python_callable=load.load_function
	)
	
	extract_task >> transform_task >> load_task
```


## PythonVirtualenvOperator에서 context를 사용하는 방법

PythonVirtualenvOperator는 **다른 operator와 context를 공유하지 않는다**는 점도 주의해야 합니다. 필요한 데이터는 `op_kwargs`를 통해 명시적으로 전달해서 사용할 수 있는데요, Jinja 템플릿을 통해 XCom 데이터를 참조할 수도 있습니다.

```python
transform_task = PythonVirtualenvOperator(  
    task_id='transform',
    python_callable=transform_function,
    requirements=['special-library==1.0.0'],
    op_kwargs={    
        'data': "{{ ti.xcom_pull(task_ids='extract', key='result') }}",
        'execution_date': "{{ ds }}"
    }    # 명시적으로 필요 데이터 전달
)
```

XCom을 통해 데이터를 전달할 때는 이중 직렬화 문제가 발생할 수 있기 때문에 `to_json` 대신  `to_dict`을 사용하는 것이 좋습니다.

```python
# JSON 문자열을 저장하면 Airflow가 다시 json.dumps() 호출 → 이중 인코딩
ti.xcom_push(key='data', value=data.to_json('records'))

# Python dict를 저장하면 Airflow가 json.dumps()를 한 번만 호출 → 정상 직렬화
ti.xcom_push(key='data', value=data.to_dict('records'))
```


## 정리

이렇게 EKS Airflow 서버에 설치되지 않은 라이브러리를 사용하기 위해 `PythonVirtualenvOperator`를 활용하는 방법에 대해 알아보았는데요. 핵심 포인트를 다시 정리하면 다음과 같습니다.

1. **PythonVirtualenvOperator의 함수는 DAG 파일 내부에 정의하기**
2. **필요한 라이브러리는 함수 내부에서 import**
3. **Context는 공유되지 않으므로** `op_kwargs`로 명시적으로 데이터 전달
4.  **XCom 데이터는 dict 형태로 저장**하여 이중 직렬화 문제를 방지

사실 제일 편한 방법은 클라우드 엔지니어에게 요청하는 것(?) 같습니다. 자주 사용할 라이브러리라면 더욱이요 :)