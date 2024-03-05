---
layout: post
title: Google Search Console 인덱싱 자동화2
date: 2024-03-05 09:10 +0900 
description: Google Search Console 인덱싱 자동화, 서버리스로 업그레이드
category: [블로그, 인덱싱] 
tags: [색인, 페이지, 인덱싱, 자동화, 서버리스] 
pin: false
math: true
mermaid: true
---
Google Search Console 인덱싱 자동화2, 서버리스로 업그레이드
<!--more-->


## Google Search Console 인덱싱자동화


>  여기에서는 노션 자동화와 관련된 내용이 들어간다. 노션 자동화 프로젝트는 작년에 진행했지만, 아직 포스팅을 하지 못했다. 관련 내용은 업데이트되는 대로 링크를 추가할 예정이다.


### 배경


[이전 포스팅](https://www.handongbee.com/posts/Search-Console-%EC%9D%B8%EB%8D%B1%EC%8B%B1%EC%9E%90%EB%8F%99%ED%99%94/) 다음버전에 대한 이야기이다. 기존 포스팅에서는 사이트맵의 모든 URL에 대하여 인덱싱을 진행했다. 하지만, 이제는 새롭게 포스팅되는 페이지에 대해서만 인덱싱을 진행하면 된다. 과거처럼 모든 사이트맵의 URL을 순회하며 인덱싱을 진행하는 것은 비효율적이다. 


또한, API 요청에 제한이 있다. 인덱싱을 진행하는 `publish API`는 **200/days**, 인덱싱되었는지 확인하는 `getMetadata`는 **200/min**이다. 페이지 개수가 200개가 넘어가면 문제가될 가능성이 크다. (물론 람다의 최대 시간은 15분이며, `getMetadata`를 우선적으로 하면 동작은 하겠지만 좋은 방법은 아니다.)


### 문제해결


**문제를 해결하기위해 기존의 람다를 업데이트하여 포스팅과 동시에 포스팅된 페이지를 인덱싱한다.**


필자와 같은 경우에는 노션에서 원고를 작성하고, 람다를 통해 노션에서 GitHub Blog로 업로드를 진행한다. 그렇기에 람다에서 새롭게 포스팅되는 페이지일 경우 바로 인덱싱을 진행하면 된다.


기존의 람다의 언어를 Node.js 18.x 버전을 사용했기에 이전 포스팅과는 다르게 자바스크립트로 인덱싱 코드를 작성했다. 

1. 우선, Google API를 사용하기 위한 인증 정보는 AWS Secret Manager에 저장해둔다.
2. google-auth-library를 이용하여 Google API에 접근할 클라이언트를 생성하고, axios를 통해 API 요청을 보낸다.

인증 정보는 [이전 포스팅](https://www.handongbee.com/posts/Search-Console-%EC%9D%B8%EB%8D%B1%EC%8B%B1%EC%9E%90%EB%8F%99%ED%99%94/)과 같은 JSON 파일이다. 아래의 `const data` 부분에서 URL만 본인의 블로그 접두사로 바꾸는 수정만 해준다면 아래의 코드를 그대로 이용할 수 있다.


```javascript
const data = {
                url: 'https://www.handongbee.com/posts/' + post_title,
                type: 'URL_UPDATED',
            };
```


### 결과


**CloudWatch > Log groups**에서 결과를 확인하면 아래와 같이 성공적으로 포스팅과 인덱싱이 작동되는 모습을 확인할 수 있다. 아래는 가장 최근 블로그 포스팅 로그이다.


```bash
40c64a85-7f9c-45b0-930a-89b28d754ce8	INFO	Response Data: {
  urlNotificationMetadata: {
    url: 'https://www.handongbee.com/posts/S3%20%EB%B3%B4%EC%95%88%EA%B4%80%EB%A0%A8',
    latestUpdate: {
      url: 'https://www.handongbee.com/posts/S3%20%EB%B3%B4%EC%95%88%EA%B4%80%EB%A0%A8',
      type: 'URL_UPDATED',
      notifyTime: '2024-03-05T07:45:03.710121382Z'
    }
  }
```


### 소스코드


[GitHub](https://github.com/han-0315/notion2github?tab=readme-ov-file)에서 자세한 코드를 확인할 수 있습니다.


```javascript
import { GoogleAuth } from 'google-auth-library';
import axios from 'axios'
import AWS from 'aws-sdk';

const SCOPES = ['https://www.googleapis.com/auth/indexing'];
const secretManager = new AWS.SecretsManager();


export default class GoogleSearch {
    constructor() {
    }
    async getSecret() {
        return new Promise((resolve, reject) => {
            secretManager.getSecretValue({
                SecretId: 'googleAPI'
            }, (err, data) => {
                if (err) {
                    console.error('SecretsManager Error:', err);
                    reject(err);
                } else {
                    if ('SecretString' in data) {
                        resolve(JSON.parse(data.SecretString));
                    } else {
                        const buff = Buffer.from(data.SecretBinary, 'base64');
                        resolve(JSON.parse(buff.toString('ascii')));
                    }
                }
            });
        });
    }

    async authenticate(post_title) {
        try {
            const secret = await this.getSecret();
            const auth = new GoogleAuth({
                credentials: secret,
                scopes: ['https://www.googleapis.com/auth/indexing'],
            });
            const client = await auth.getClient();
            const url = `https://indexing.googleapis.com/v3/urlNotifications:publish`;

            const data = {
                url: 'https://www.handongbee.com/posts/' + post_title,
                type: 'URL_UPDATED',
            };

            try {
                const response = await axios({
                    method: 'post',
                    url: url,
                    data: data,
                    headers: {
                        'Authorization': `Bearer ${(await client.getAccessToken()).token}`,
                        'Content-Type': 'application/json',
                    },
                });

                console.log('Response Data:', response.data);
            } catch (error) {
                if (error.response) {
                    // 요청이 이루어졌으나 서버가 2xx 범위가 아닌 상태 코드로 응답한 경우
                    console.error('Error Response:', error.response.data);
                    console.error('Status:', error.response.status);
                    console.error('Headers:', error.response.headers);
                } else if (error.request) {
                    // 요청이 이루어졌으나 응답을 받지 못한 경우
                    console.error('Error Request:', error.request);
                } else {
                    // 요청을 설정하는 동안에 발생한 에러
                    console.error('Error Message:', error.message);
                }
                console.error('Config:', error.config);
            }
        } catch (error) {
            console.error('Authentication Error:', error.message);
        }
    }
}
```

