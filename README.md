##  Sign With Apple 예제

> 애플 로그인을 자바스크립트를 이용해서 구현할 때 겪었던 예외상황들을 기록합니다.



## 애플 페이지 상 설정

>  Sign In with App 웹용 설정 방법을 설명합니다.

App Id 와는 별개로 Service ID 를 반드시 별개로 추가 생성해주어야합니다. 아래 적힌 주소로 이동한 후  Identifiers 카테고리에서 새로운 Service ID 의 추가를 진행합니다.

https://developer.apple.com/account/resources/certificates/list

![Image](https://tva1.sinaimg.cn/large/007S8ZIlgy1ge654d0fdrj319o0pqq9k.jpg)

Service IDs 를 클릭후 다음으로 넘어갑니다.



![IMAGE](https://tva1.sinaimg.cn/large/007S8ZIlgy1ge65a6o7stj30zf0deaax.jpg)

Sign In with Apple 을 클릭한 후 Configure 에서 redirect-uri 등을 입력하여 등록한 후 Continue 를 눌러서 생성되는 .p8 파일를 받아놓은 후 Key Id 를 복사해놓습니다. (.p8 파일은 백엔드 서버에서 애플로 요청을 보낼때 사용됩니다.)



## 프론트엔드 설정

> 사용하려면 먼저 HTML 상에 메타 태그를 입력해야합니다. (프론트엔드 라이브러리에 따라서 태그를 주입하는 형식으로 사용할 수도 있습니다.)

만약 팝업을 사용할 수 있다면 애플 로그인을 그냥 js 를 통해서 사용할 수 있겠지만, 하이브리드 앱의 경우엔 인앱 브라우저 기능을 통해서 해당 페이지를 띄우므로 인앱 브라우저 상에서 팝업 기능을 사용하는 것이 제한되는 경우가 많았습니다. 따라서 저는 아래와 같은 세팅으로 popup 을 false 처리 한 후 라이브러리를 이용하였습니다.

```html
<meta name="appleid-signin-client-id" content="com.yourapp.name" />
<meta name="appleid-signin-redirect-uri" content="https://your-apple-auth-page.com" />
<meta name="appleid-signin-state" content="" />
<meta name="appleid-signin-use-popup" content="false" />
```



### 라이브러리 커스터마이징

> 기본 제공되는 sign with apple js 파일은 요청방식이 매우 특이합니다. 일단 최초에 애플 페이지로 이동되는 것은 정상적입니다, 그러나 해당 페이지로 이동된 다음 다시 위에서 HTML 태그를 통해서 `redirect-uri` 로 이동되지는 않습니다.
>
> 네이버 로그인이나 카카오 로그인처럼 원래 페이지로 리다이렉션 되는 것이 아닌, 해당 `redirect-uri` 로 `POST` 요청이 날아갑니다. 즉 sign-with-apple 의 `redirect-uri` 는 POST 요청이 날아갈 API 서버 상의 주소가 담겨야하는 것입니다.
>
> 이렇게 구현해도 되긴하지만, 저는 원래 uri 로 이동되면서 주소상으로 파라메터가 전달되는 구조의 API 가 필요했기 때문에 라이브러리를 커스터마이징 하였습니다.

애플로그인 라이브러리 파일을 다운받은 후 해당 파일을 열어서 아래내용을 검색합니다.

```js
responseMode:"form_post"
```

만약 `redirect-uri` 상으로 POST 요청이 전송되는 것이 아닌, 리다이렉션 및 주소 파라메터를 통한 결과값 전달을 하고 싶은 경우 "form_post" 으로 해당 내용을 바꿔주세요.

```js
responseMode:"fragment"
```

> 만약 이러한 작업을 직접하기를 원하지 않는 경우 이 레포지스토리에 업로드 되어있는 `custom_apple.js` 파일을 다운받아서 사용하실 수 있습니다.

해당 내용에 대한 애플 문서상 설명은 여기에 서술되어 있습니다. [Send the Required Query Parameters](https://developer.apple.com/documentation/sign_in_with_apple/sign_in_with_apple_js/incorporating_sign_in_with_apple_into_other_platforms)



## 프론트엔드 API 사용 예시

> Svelte 를 통해서 클라이어언트 상에서 Sign With Apple 을 구성한 코드의 일부 예시입니다.

"fragment" 옵션을 통해서 파라메터로 입력된 애플 로그인 결과물은 querystring  모듈로 해석할 수 있습니다.

```js
import * as AppleID from './custom_apple.js'
import querystring from 'querystring'

export let params: any = {}
declare var window: any


let randomId = params.randomId
if (randomId !== undefined) {
    Page.cookieStorage.setItem('randomId', randomId)
    onMount(async () => {
        try {
            await AppleID.auth.signIn()
        } catch (error) {
            noticeText =
                '소셜로그인에 실패하였습니다.<br>앱으로 다시 돌아가주세요.'
        }
    })
} else {
    onMount(async () => {
        let randomId = Page.getCacheItem('randomId')

        try {
            let callbackStr = window.location.href.split('#')[1]
            if(!callbackStr || callbackStr.length < 10){
                Page.Router.replace('/')
                return
            }
            let callbackData: {
                code: string
                id_token: string
            } = querystring.decode(callbackStr)

            let data = {
                randomId,
                token: JSON.stringify(callbackData),
                category: '애플',
            }

            try {
                await Page.GraphQLAPI.auth.socialLoginBrowserSave(data)
                noticeText =
                    '소셜로그인이 완료되었습니다.<br>앱으로 다시 돌아가주세요.'
            } catch (err) {
                noticeText =
                    '소셜로그인에 실패하였습니다.<br>앱으로 다시 돌아가주세요.'
            }
        } catch (e) {
            console.log(e)
            Page.Router.replace('/')
        }
    })
}
```



## 백엔드 설정

> 백엔드 구성도 만만치 않았습니다.

### p8 파일을 이용한 인증토큰 구성 예시

애플에서 받은 p8 파일이 필요합니다. 이 파일과 사용되는 TEAM_ID 와 KEY_ID 를 입력한 후 아래와 같은 함수를 구성합니다.

TEAM_ID 는 애플 계정마다 하나씩 존재하며, 아래 URL 에서 확인할 수 있습니다.
https://developer.apple.com/account/#/membership/

```js
import * as fs from 'fs'
import * as jwt from 'jsonwebtoken'

const signWithApplePrivateKey = fs.readFileSync('./signwithapple.p8') // .P8 FILE PATH 
export const getSignWithAppleSecret = () => {
    const token = jwt.sign({}, signWithApplePrivateKey, {
        algorithm: 'ES256',
        expiresIn: '10h',
        audience: 'https://appleid.apple.com',
        issuer: 'A1B2CD3E4F', // TEAM_ID
        subject: 'com.yourapp.name',
        keyid: '123AB45C67', // KEY_ID
    })
    return token
}
```



### 백엔드 애플 토큰 검증용 API 사용 예시

> 위에서 구성한 getSignWithAppleSecret 를 임포트합니다.

```js
import { getSignWithAppleSecret } from '../../updateAppleSecretKey'
import * as querystring from 'querystring'
import * as jwt from 'jsonwebtoken'
import axios from 'axios'
try {
    let parsedData: {
        code: string
        id_token: string
    } = JSON.parse(token)
    let response
    try {
        // console.log('idToken:', idToken)
        response = await axios.post(
            'https://appleid.apple.com/auth/token',
            querystring.stringify({
                grant_type: 'authorization_code',
                code: parsedData.code,
                client_secret: getSignWithAppleSecret(),
                client_id: 'com.yourapp.name',
                redirect_uri: 'https://your-apple-auth-page.com',
            }),
            {
                headers: {
                    'Content-Type':
                        'application/x-www-form-urlencoded',
                },
            }
        )
    } catch (e) {
        console.log(e)
    }

    if (!response.data.id_token)
        throw new Error('토큰 인증에 실패하였습니다.')

    console.log(
        'process.env.APPLE_SECRET',
        process.env.APPLE_SECRET
    )

    let userData: any = jwt.decode(parsedData.id_token)
    console.log('userData', userData)

    accountData.name = '이용자'
    try {
        accountData.name = userData.email.split('@')[0]
    } catch (e) {}
    accountData.tel = `apple_${userData.sub}`
    accountData.loginId = `apple_${userData.sub}`
} catch (e) {
    console.log(e)
}
```



위와 같은 방법을 통해서 JS 를 통한 애플로그인을 구현하실 수 있습니다.