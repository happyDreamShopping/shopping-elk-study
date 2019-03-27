## 실습 회의 
- 각자 데이터를 색인할 것인가? 
    - 그룹 단위로 한다면, 한 클러스터에 한 인덱스로 공유?
    - 개인 단위로 한다면, 한 클러스터에 각자 색인? 


### 실습 주제 및 데이터
- 실습 주제 및 데이터를 정합니다.
- 1: stack overflow
    - BigQuery dataset
    - 건영님의 졸업작품을 참고한다. 
    - https://www.kaggle.com/stackoverflow/stackoverflow

- 2(공통주제): stack overflow developer survey
    - agg 등 분류/집계 쿼리 연습에 좋다. 
    - https://www.kaggle.com/stackoverflow/stack-overflow-2018-developer-survey

- 3: reddit comments
    - http://files.pushshift.io/reddit/comments/
        - http://files.pushshift.io/reddit/comments/sample_data.json


### 실습 데이터로 뭘 할것인가? 
- 팀전
    - 조모임 
    - 조모임 끝나고 나서 발표
    - 평가 및 회고 

- stack 팀, reddit 팀
    - stack : 김건영, 이정혁, 정휘준
    - reddit : 김슬아, 남상욱, 이성준

### 실습 일정 
- 실습 기간: 3.25 ~ 4.5 (2주)
- 실습 클러스터 롤링 upgrade. 
    - 6.6
    - 이번 주(3.22) 안에. 
    - ansible-playbook 으로 구성. 
    - 슬아님 클러스터도 있음. 

- 다음주(3.25) 월 이후: 팀별 일정 진행
- 4.5 팀별 발표 및 회고 


### 실습 방법 
- 팀별로 스키마 구성, data pipeline 구성
- 검색 목표 설정(쿼리 기반, 페이지네이션 기반 등...)
- stackoverflow survey는 공통 주제로 진행해본다. 
- REST API server / React.js 등 옷을 입혀본다(자유롭게!)

### 기타 질문
- 있으면 남겨주세요. 
