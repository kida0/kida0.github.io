---
created: 2025-11-15
title: Pickle checkpoint로 데이터 지키기
layout: post
tags: [데이터, LLM]
category: 공부
math: true
---

## 배경 설명

많은 이커머스 서비스에서 개인화 추천을 제공하고 있지만, 신규 가입 고객의 경우 데이터가 없어 추천이 어려운 **콜드 스타트(cold start)** 문제는 여전합니다.

이를 해결하기 위해 다른 기업들은 간단한 설문을 통해 고객의 니즈를 파악하는 방식을 사용하는데요, 저희는 조금 다른 접근을 시도했습니다. 고객이 속한 업종 기반의 상품을 추천하여 콜드 스타트 문제를 해결하고, 가입 직후부터 필요한 상품에 편리하게 접근할 수 있게 만들고자 했습니다. 또한 장기적으로 이 업종 데이터를 확장하여 분석에서 활용할 계획이었습니다.

그러나 문제는, 추천에 사용할만한 '업종별 상품 데이터'가 많지 않았습니다. 이 문제를 가장 쉽게 해결하는 방법은 식당명이나 메뉴를 통해 업종을 예상해보는 것이었는데요.

```
restaurant_name like '%치킨%' then '치킨'
munue like like '%짜장면%' then '중식'
```

하지만 단순 키워드 매칭만으로는 아래와 같은 케이스를 제대로 분류할 수 없었습니다.

- '치키치키뱅뱅'이라는 식당에서 '한방오리백숙'을 팔면?
- '고등어봉초밥', '과일화채', '모둠꼬치'를 파는 곳은?

결국 **LLM을 활용하여 식당명, 메뉴, 기타 정보를 종합적으로 활용해 '예상 업종'을 만드는 작업**을 진행하게 되었습니다.



## Jupyter Notebook에서 발생한 문제

Jupyter Notebook에서 Langchain으로 LLM 작업을 진행했는데요, 이 과정에서 문제들이 발생했습니다.

- for문 실행 중 종종 작업이 멈추는 현상 발생
- 함수를 실행하는 경우, 셀을 종료할 때 진행하던 데이터가 모두 날아감
- LLM API는 비용이 발생하기 때문에 중간 데이터가 날아가면 아까움

물론 함수를 짜지 않고 셀 단위로 실행하는 방법도 있지만, 더 나은 방법이 필요했습니다.



## pickle checkpoint 활용하기

`pickle checkpoint`를 사용하면 셀을 종료시켜도 중간 데이터가 날아가는 것을 막을 수 있습니다.


### (1) 체크 포인트 시작하기

우선 체크 포인트가 있는 경우 불러오고, 아니면 처음부터 실행하는 코드를 작성합니다.

```python 
import pickle
from pathlib import Path
import logging

def get_restaurant_category(
    ...,
    checkpoint_file: str = 'checkpoint.pkl',
    checkpoint_interval: int = 100
):
    # 기존 체크포인트가 있다면 불러오기
    checkpoint_path = Path(checkpoint_file)
    
    if checkpoint_path.exists():
        try:
            with open(checkpoint_path, 'rb') as f:
                checkpoint_data = pickle.load(f)
                result = checkpoint_data['result']
                start_idx = checkpoint_data['last_index'] + 1
                logging.info(f'* 체크포인트 복원: {start_idx}번째부터 시작합니다.')
                
        except Exception as e:
            logging.warning(f'* 체크포인트 로딩 실패: {e}. 처음부터 시작합니다.')
            start_idx = 0
            result = []
    else:
        start_idx = 0
        result = []
```



### (2) 주기적으로 체크포인트 저장하기

이제 일정 주기마다 중간 결과를 저장하는 체크포인트를 추가합니다. `KeyboardInterrupt`를 사용하면 작업을 수동으로 중단하더라도 데이터를 지킬 수 있습니다.

```python
try:
    for idx in range(start_idx, total_idx):
        # LLM 호출 및 업종 분류 작업
        category = run_llm(data[idx])
        result.append(category)
        
        # 주기적으로 체크포인트 저장
        if (idx + 1) % checkpoint_interval == 0:
            checkpoint_data = {
                'result': result,
                'last_index': idx,
            }
            
            with open(checkpoint_path, 'wb') as f:
                pickle.dump(checkpoint_data, f)
            logging.info(f'* 체크포인트 저장: {idx + 1}개 완료')
                
except KeyboardInterrupt:
    # 중단 시점의 결과 저장
    logging.warning(f'\n* 작업이 중단되었습니다.')
    
    checkpoint_data = {
        'result': result,
        'last_index': len(result) - 1
    }
    
    with open(checkpoint_path, 'wb') as f:
        pickle.dump(checkpoint_data, f)
    logging.warning(f'* 중단 시점 저장 완료: {checkpoint_file}')
```



### (3) 체크포인트 정리하기

반복되는 작업을 위해, 작업이 완료되면 체크포인트 파일을 정리해줍니다.

```python
finally:
    # 작업이 완료되었다면 체크포인트 파일 삭제
    if len(result) == total_idx:
        if checkpoint_path.exists():
            checkpoint_path.unlink()
            logging.info('* 작업 완료! 체크포인트 파일을 제거합니다.')
```




## 마무리

이번에는 LLM 작업이 멈추거나, Ctrl+C로 중단하더라도 데이터를 잃지 않는 방법에 대해 알아보았습니다. 요즘 LLM API 비용이 만만치 않다보니, 중간에 작업이 날아가서 다시 돌리는 일은 정말 피하고 싶은데요. pickle checkpoint로 데이터를 안전하게 지켜보세요 :)