## ✏ 프로젝트 소개

### 나처럼 말하는 봇, '구협봇' 만들기!

-----

 단톡방에서 내가 나눈 대화를 학습하여, 마치 내가 말하는 것과 같은 챗봇을 구현하는 것을 목표로 하고 있습니다.
 
 기본적인 NLP모델이나, LLM을 이용하여 학습한 후 배포하는것 까지를 목표로 하고 있습니다.

 대화에 관한 내용은 개인정보이므로 업로드하지 않았습니다.

## 프로젝트 기간
**START  : 2024.06.06**
<br>
**END : 2024.06.??**

## ⌨ 개발 환경
### Language
------
<img src="https://img.shields.io/badge/language-%23121011?style=for-the-badge"><img src="https://img.shields.io/badge/python-3776AB?style=for-the-badge&logo=python&logoColor=white"><img src="https://img.shields.io/badge/3.11.9-515151?style=for-the-badge"><br>


### IDE 
------
<img src="https://img.shields.io/badge/ide-%23121011?style=for-the-badge"><img src="https://img.shields.io/badge/visual studio code-007ACC?style=for-the-badge&logo=visual studio code&logoColor=white"><br>
<img src="https://img.shields.io/badge/ide-%23121011?style=for-the-badge"><img src="https://img.shields.io/badge/jupyter-F37626?style=for-the-badge&logo=Jupyter&logoColor=white"> 


## 데이터 수집

(chat_to_df.ipynb)

<img src="https://img.shields.io/badge/Line-00C300?style=for-the-badge">

친구들과의 단톡방이 있는 LINE에서 데이터를 .txt파일로 추출함

2020년 12월 4일부터 2024년 6월 5일까지의 데이터 (1279일)

<img src="https://github.com/GuhyupLee/GuhyupBot/assets/160453988/fa1756d9-113f-41e6-9fa3-10db6521b708">

시간, 보낸 사람, 메세지의 형식으로 저장되어 있음

채팅의 정보 수집은 단톡방 인원 5명의 동의를 전부 받음


<img width="411" alt="image" src="https://github.com/GuhyupLee/GuhyupBot/assets/160453988/bcaeb684-10ef-42d9-849e-6f1987d917c6">

.txt파일을 pandas를 이용하여 DataFrame으로 변경하였음.

<img width="411" alt="image" src="https://github.com/GuhyupLee/GuhyupBot/assets/160453988/19832cce-b6cc-42f6-ba0f-72ec4c4c4fa3">

최종적으로, 날짜와 시간을 기록한 DataTime, 수신인을 기록한 Sender, 메세지를 기록한 Message. 총 3개의 Column으로 DataFrame을 완성

Row의 수는 474,706개

Sender는 이니셜 및 적절한 영어로 대체함

## 데이터 전처리

(preprocessing.ipynb)

그저 시간순으로 나열된 대화 로그에서 중요한건, 어디서 어디까지가 일련의 '대화'인지 알아내는것이었다.

일단 나(LSN)의 말투를 쓰는 봇을 개발하기 위함이므로 다음 요소를 고려함

- 대화를 내가 시작한 경우를 제외함
- 챗의 전송 시간 간격이 일정 시간 이하(여기선 1시간)일 경우 대화가 성립되었다고 판단
- 대화는 한번의 채팅으로 끝나지 않으므로, 내가 채팅을 치고, 상대방이 채팅을 칠때 까지의 모든 대화 내용을 문장(리스트)으로 기억함
- 마지막 채팅 전송 이후, 아무도 채팅을 치지 않고 1시간이 지나면 대화가 종료됐다고 판단.

이렇게 하면 '대화'라는 묶음을 전처리 할 수 있다고 기대하였음.

```
def refine_conversation(df):
    # 대화를 저장할 리스트 초기화
    conversations = []
    current_conversation = []  # 현재 대화를 저장할 리스트 초기화
    last_message_time = None  # 마지막 메시지 시간을 추적하기 위한 변수 초기화

    # 데이터프레임의 각 행을 반복 처리, 진행 상황을 표시하기 위해 tqdm 사용
    for index, row in tqdm(df.iterrows(), total=df.shape[0], desc="Processing Conversations"):
        if row['Sender'] != 'LSN':
            # 마지막 메시지 시간과 현재 메시지 시간의 차이가 1시간을 초과하면 대화를 종료
            if current_conversation and (row['DateTime'] - last_message_time > timedelta(hours=1)):
                # 현재 대화를 'Prev_Message'와 'My_Response'로 분리하여 conversations 리스트에 추가
                prev_message = ' '.join([msg['Message'] for msg in current_conversation if msg['Sender'] != 'LSN'])
                my_response = ' '.join([msg['Message'] for msg in current_conversation if msg['Sender'] == 'LSN'])
                if my_response.strip():  # My_Response가 공백이 아닌 경우에만 추가
                    conversations.append({
                        'Prev_Message': prev_message,
                        'My_Response': my_response
                    })
                # 현재 대화 초기화
                current_conversation = []

            # 현재 메시지를 현재 대화 리스트에 추가
            current_conversation.append(row)
            # 마지막 메시지 시간 업데이트
            last_message_time = row['DateTime']
            
        elif row['Sender'] == 'LSN':
            if current_conversation:
                # LSN의 응답을 현재 대화에 추가하고 대화를 종료
                current_conversation.append(row)
                prev_message = ' '.join([msg['Message'] for msg in current_conversation if msg['Sender'] != 'LSN'])
                my_response = ' '.join([msg['Message'] for msg in current_conversation if msg['Sender'] == 'LSN'])
                if my_response.strip():  # My_Response가 공백이 아닌 경우에만 추가
                    conversations.append({
                        'Prev_Message': prev_message,
                        'My_Response': my_response
                    })
                # 현재 대화 초기화
                current_conversation = []
            # 마지막 메시지 시간 업데이트
            last_message_time = row['DateTime']

    # 마지막 대화가 남아 있을 경우 추가
    if current_conversation:
        prev_message = ' '.join([msg['Message'] for msg in current_conversation if msg['Sender'] != 'LSN'])
        my_response = ' '.join([msg['Message'] for msg in current_conversation if msg['Sender'] == 'LSN'])
        if my_response.strip():  # My_Response가 공백이 아닌 경우에만 추가
            conversations.append({
                'Prev_Message': prev_message,
                'My_Response': my_response
            })

    # 대화 목록을 데이터프레임으로 반환
    return pd.DataFrame(conversations)
```




<img width="600" alt="image" src="https://github.com/GuhyupLee/GuhyupBot/assets/160453988/eec9084e-c9e5-4e86-bb98-88d077fef835">


총 72,935개의 대화 묶음(상대방의 대화 전송 -> 나의 답변)을 생성하였음.

547행과 같이 맥락에서 벗어난 대화 묶음도 존재하지만, 대체적으로는 잘 전처리 되었다고 봄

문제점은, 이것을 수치화할 방법을 찾지 못하였음.

## 데이터 학습

### LLaMa3를 이용하여 LoRA를 생성

(Lora.ipynb)

[학습에 이용한 모델] (https://huggingface.co/beomi/Llama-3-Open-Ko-8B-Instruct-preview/tree/main)

[학습에 도움을 준 링크] (https://hypro2.github.io/llama2-lemon/)

모델을 처음부터 파인튜닝 하기에는 데이터셋의 양이 많지않고, 목적에도 맞지 않다고 생각하였음

따라서 LoRA(low-rank adaptation of large language models)를 이용하여 학습을 시도함

대략적인 방법은 비슷했으나, 데이터셋을 구성하는 방식에 큰 차이가 있었음.
JSON파일을 재정의 하여, 이를 극복함

<img width="260" alt="image" src="https://github.com/GuhyupLee/GuhyupBot/assets/160453988/a6c1e95b-24b9-4183-80fb-a7644c220fc3">

내가 제일 좋아하는 음식이 피자인데, LoRA로 학습한 후의 답변도 일맥상통 하였다.

<img width="260" alt="image" src="https://github.com/GuhyupLee/GuhyupBot/assets/160453988/57c53a68-5ee2-47be-b9fd-7e44ff0419a9">

이것도 정답. 물론 이런 결과는 내가 단톡방에서 가장 많이 쓰는 단어를 학습한 결과기도 하다.

<img width="600" alt="image" src="https://github.com/GuhyupLee/GuhyupBot/assets/160453988/8cf42e71-4966-4389-93da-710409fcabb6">

단톡방에서 내가 가장 많이 썼던 말을 도식화한 워드 클라우드, 피자와 일본이 상위권에 있는것을 확인할 수 있다.


## 문제점 및 느낀점.

처음 데이터 전처리에 6시간이라는 많은 시간이 소요된다고 나옴 -> 코드의 문제점을 해결하고, 짧은 시간으로 줄일 수 있었다.

LoRA로 학습한 결과가 생각보다 멍청함 -> 모든 Layer를 사용한게 문제가 아닐까 생각중이다.