---
layout: post
title: Google Search Console 인덱싱 자동화3
date: 2024-03-10 09:00 +0900 
description: Google Search Console 인덱싱 자동화, SQS 도입
category: [블로그, 인덱싱] 
tags: [색인, 페이지, 인덱싱, 자동화, 서버리스] 
pin: false
math: true
mermaid: true
---
Google Search Console 인덱싱 자동화3, SQS 도입
<!--more-->


## 문제 상황


[이전 포스팅](https://www.handongbee.com/posts/Search-Console-%EC%9D%B8%EB%8D%B1%EC%8B%B1%EC%9E%90%EB%8F%99%ED%99%942/)에서 포스팅을 진행하는 Lambda에서 곧바로 Google API를 호출하여 인덱싱을 진행했다. 포스팅은 람다에서 GitHub으로 포스팅하고, GitHub Pages 작업은 Actions 파이프라인에 의해 완료된다.


“API로 호출해도 곧바로 처리하지 않고 작업 큐에 있다가 처리하니 Actions 작업 이후에 처리하겠지”라는 생각으로 람다에서 Google API를 호출했고, 응답도 아래와 같이 잘왔다. 


![Untitled.png](/assets/img/post/Search%20Console%20인덱싱자동화3/1.png)


하지만, 어느 정도 시간이 지나고 “페이지를 찾을 수 없어(404) 색인을 생성하지 못했다는” 메일이 왔다. **상황마다 다르겠지만 인덱싱 API를 실시간에 가깝게 처리**하는 것 같다. 실시간에 가깝게 처리되면, Actions 작업은 약30초 정도 소요되기에 API 요청을 보낼 때는 아직 포스팅이 완료되지 않을 가능성이 크다. 


![Untitled.png](/assets/img/post/Search%20Console%20인덱싱자동화3/2.png)


## 가장 쉬운 해결 방법


Lambda에서 포스팅이 완료된 후 GitHub Actions이 작동한다. GitHub Actions은 1분 이내로 작업이 마무리된다. 가장 빠르고 쉬운 해결책은 Lambda에서 모든 작업이 완료하고 일부로 1분 이상 대기하고, 페이지 인덱싱 API를 호출하면 된다. 평소 람다의 실행시간을 확인해보면 10초 내외이기에, 여기서 1분을 더해도 람다 최대 실행시간(15분)보다는 적다.


하지만, 나는 종종 여러 개의 포스팅을 동시에 진행했다. 작업을 노션에서 하기에, 노션에서 작성하고 한 번에 각 페이지에 대한 람다를 호출한다. 이럴 경우 지속해서 커밋이 발생하기에 GitHub Actions 작업은 취소되고, 최종 커밋에서 작업이 이뤄진다. 이런 상황에 대비하면 넉넉잡아 5분 정도 후에 API를 호출하면 될 것 같다. 람다의 최대 실행시간은 15분이기에 가능은 하나, 대기를 위해 5분 이상을 의미없이 실행만 시킨다는 것 옳지 않아 다른 방법을 찾게되었다.


만약 위의 방법으로 진행한다면, 언어에 맞는 wait 혹은 sleep 함수를 통해 대기시키면 된다.


## SQS를 이용한 해결 방법


SQS에서는 메시지를 보낼 때, 대기시간을 설정할 수 있다. 예를 들어 메시지에 대기시간을 5분으로 설정하면 5분 후에 메시지가 큐에 노출되어 소비자에게 전달된다. 이를 이용하여 포스팅을 진행하는 Lambda에서 페이지 URL을 SQS에 전달한다. SQS는 대기시간(5분)이 지나면 Google Search Console Indexing을 진행하는 람다를 호출한다. 람다는 API를 처리한다. 이러면 무의미하게 대기하는 시간이 없어진다. 또한, API 처리를 하는데 문제가 생기면 SQS의 메시지가 그대로 남으니, 다시 처리하는 복원력이 생긴다. 


### SQS 생성


SQS 서비스를 검색 > Create Queue 버튼을 클릭한다. 여러 설정 옵션이 나온다. 그중에서 중요한 설정값에 대해서만 설명한다. 나머지는 권장 옵션을 따라해도 무방하다.


**Visibility timeout**은 하나의 메시지를 소비자가 작업할 때, 해당 메시지를 다른 소비자가 소비하지 않도록 감추는 시간이다. 이 시간은 우리가 추후 생성할 람다의 최대 실행시간(1분)보다 크게 설정한다. 필자는 100초로 설정했다.


**Access policy**에서 기존 람다가 해당 SQS에 메시지를 보낼 수 있도록 권한을 열어준다.


나머지 설정값을 알맞게 넣어주고 SQS를 생성한다.


### 기존 람다 수정


기존의 람다 코드에서 포스팅을 완료한 후 종료하기 전 페이지에 대한 URL을 SQS에 전송한다. 여기서 `delaySeconds`은 지연시간을 담당하며 상황에 맞게 설정하면 된다. 필자는 3분으로 설정했다. `MessageBody`에는 URL을 입력한다.


```javascript
async sendMessage(post_title) {

        const params = {
            QueueUrl: this.queueUrl,
            MessageBody: base_url + post_title,
            DelaySeconds: delaySeconds
        };

        try {
            const result = await sqs.sendMessage(params).promise();
            console.log('Message sent successfully', result.MessageId);
            
        } catch (error) {
            console.error('Error sending message to SQS', error);
     
        }
    }
```


### 이벤트 스키마


```json
{
  "Records": [
    {
      "messageId": "d115af49-d481-4518-93af...",
      "receiptHandle": "AQEBoLMuGh70fIrt...",
      "body": {
        "data": "https://www.handongbee.com/posts/S3%20%EC%9D%B4%EB%B2%A4%ED%8A%B8%20%EC%8B%A4%EC%8A%B5"
      },
      "attributes": {
        "ApproximateReceiveCount": "1",
        "AWSTraceHeader": "Root=1-65ec89a8-320818240d154ce77437c065;Parent=645355b6467ec5b3;Sampled=0;Lineage=518e8703:0",
        "SentTimestamp": "1710000565147",
        "SenderId": "AROAQFFUSRHQABEXSJOM5:notion2github-blog-iBSnWFPGdQFV",
        "ApproximateFirstReceiveTimestamp": "1710000745147"
      },
      "messageAttributes": {},
      "md5OfBody": "3d4d66ad09c28d056057477d8ae0e31a",
      "eventSource": "aws:sqs",
      "eventSourceARN": "arn:aws:sqs:ap-northeast-2:...:IndexingQueue",
      "awsRegion": "ap-northeast-2"
    }
  ]
}
```


### 기존 람다 권한 수정


람다에는 SQS에 메시지를 보낼 수 있는 권한을 부여한다. 

1. Lambda IAM Role에 들어가서, 사용할 SQS 리소스에 대한 “**SQS:SendMessage**” 정책을 부여한다.
2. SQS의 접근 정책에서 Lambda에 대한 권한을 열어주면 된다.

1, 2번 방법 중에 하나만 설정해도 동작한다.


### SQS에서 메시지를 받아 인덱싱을 처리하는 람다


이제 인덱싱을 처리하는 람다를 작성한다. 코드는 아래와 같고, 보안을 위해 사전에 Google API 시크릿 정보를 AWS Secrets Manager에 올려두었다. 


```python
import json
import boto3
from google.oauth2 import service_account
import googleapiclient.discovery
import requests

def get_secret():
    secret_name = "googleAPI"
    region_name = "ap-northeast-2"

    # AWS Secret Manager 클라이언트 생성
    session = boto3.session.Session()
    client = session.client(
        service_name='secretsmanager',
        region_name=region_name
    )

    try:
        get_secret_value_response = client.get_secret_value(SecretId=secret_name)
    except Exception as e:
        raise e
    else:
        if 'SecretString' in get_secret_value_response:
            secret = get_secret_value_response['SecretString']
            return json.loads(secret)
        else:
            decoded_binary_secret = base64.b64decode(get_secret_value_response['SecretBinary'])
            return json.loads(decoded_binary_secret)

# Google API 인증 후 페이지 크롤링을 요청하는 함수
def authenticate_and_notify(url):
    secret_info = get_secret()
    credentials = service_account.Credentials.from_service_account_info(secret_info, scopes=['https://www.googleapis.com/auth/indexing'])
    
    service = googleapiclient.discovery.build('indexing', 'v3', credentials=credentials)
    body = {
        'url': url,
        'type': 'URL_UPDATED'
    }
    response = service.urlNotifications().publish(body=body).execute()
    print('Response from Google Indexing API:', response)

# Lambda Event 핸들러
def lambda_handler(event, lambda_context):
    print("Received event: ", event)
    try:
        # 이벤트에서 URL 가져오기
        url = event['Records'][0]['body']['data']
        print("URL from event:", url)
        authenticate_and_notify(url)
        return {
            'statusCode': 200,
            'body': json.dumps('URL Updated: ' + url)
        }
    except Exception as e:
        print(e)
        return {
            'statusCode': 500,
            'body': json.dumps('Error updating URL')
        }

```


### 인덱싱 Lambda 권한


인덱싱 람다는 SQS 큐에서 메시지를 받고, 메시지를 삭제할 권한과 Secrets Manager에서 시크릿 정보를 읽어올 권한이 필요하다.

1. SQS에 대해서는 LambdaSQSQueueExecutionRole에 해당하는 ”ReceiveMessage”, "GetQueueAttributes”,  “DeleteMessage” 정책을 연결한다.
2. Secret Manager에 대해서는 Google API 시크릿 정보를 담아둔 리소스에 대해 GetSecretValue 정책을 부여한다.(만약, 시크릿 매니저를 사용하지 않았다면 생략해도 된다.)

### SQS과 Lambda 통합


이제 SQS와 Lambda를 통합하여 SQS 메시지가 Lambda에 전달되도록 설정한다.

1. 람다 메인 화면에서 다이어그램에 존재하는 **Add trigger** 버튼 혹은 **Configureation > Triggers > Add trigger** 페이지로 이동한다.
2. Trigger Source로 SQS를 선택한다.
3. 기존에 생성한 SQS를 선택한다.

위의 과정이 끝나면, 이제 SQS에 들어오는 메시지는 람다에게 전달된다.


![Untitled.png](/assets/img/post/Search%20Console%20인덱싱자동화3/3.png)


### 결과


이제 전체 프로세스를 테스트 해보자. 노션에서 테스트 페이지에 대한 Lambda를 실행시킨다. 노션에서 깃허브 블로그로 포스팅하는 Lambda가 실행되며, **포스팅 후에 SQS에 메시지**를 보낸 것을 확인할 수 있다. 


**[포스팅 Lambda가 실행되는 모습]**


![Untitled.png](/assets/img/post/Search%20Console%20인덱싱자동화3/4.png)


메시지를 SQS에서 대기시간을 거쳐 두 번째 람다로 전달된다. 이제 람다에서는 페이지 URL을 받아 Google API로 인덱싱을 요청한다.


**[SQS에서 위의 메시지를 받아 처리하는 모습]**


![Untitled.png](/assets/img/post/Search%20Console%20인덱싱자동화3/5.png)


이제 Google Search Console에서 요청을 받아, 인덱싱을 처리한다. 구글에서 작업하는 데 보통 1일 정도 소요된다. 이제 인덱싱이 완료되면 아래와 같이 Google 검색에 포스팅이 노출된다.


![Untitled.png](/assets/img/post/Search%20Console%20인덱싱자동화3/6.png)


**[위의 프로세스를 통해 포스팅된 가장 최근 포스트이다.]**

