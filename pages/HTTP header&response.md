# HTTP 주요 헤더와 주요 응답

<br />

## Request HTTP Message
> **GET /api/user-info HTTP/2.0**
* GET | HTTP 메소드 종류
* /api/user-info | 요청보낸 주소
* HTTP/2.0 | HTTP 버전

> **HOST**
* 서버의 도메인이 나타나는 부분.

>**Accept**
* 서버에서 이런 타입으로 보내달라고 미리 요청함.

>**Authorization**
* 인증 토큰을 보내기 위해서 사용되는 헤더.
* `Basic`, `Bearer`를 붙여서 토큰의 종류를 알리고 보낸다.

>**User-Agent**
* 현재 사용자의 클라이언트를 요청한다.
* 접속자 통계를 내기 위해서 사용된다.

>**Origin**
* POST 요청 시, 해당 요청이 어떤 주소에서 보내진 것인지 기입.
* 요청이 받는 주소와 다르다면 CORS 에러가 발생한다.


## Response HTTP Message
> **Access-Control-Allow-Origin**
* 요청을 보내는 클라이언트와 받는 서버의 주소가 다르면 CORS 오류가 나낟.
* Options로 현재 서버가 CORS를 허용하는지 물어본다.

> **Allow**
* 이 주소로는 해당 메소드만 허용하겠다는 표시.
* 만약 GET 요청을 보냈는데 POST만 허용한다면 서버는 `405 Method Not Allowed`와 `Allow: POST`를 보내게 된다.

> **Location**
* 300번대 응답코드나 200번대 응답코드가 왔다면 어떤 페이지로 이동할 지 알려주는 헤더이다.

> **Content-Security-Policy**
* 다른 외부파일들을 불러오는 경우, 차단할 소스와 불러올 소스를 명시한다.
* XSS 공격 등 악성코드 공격을 사전 방지할 수 있는 설정이다.

> **set-cookie**
* 서버에서 쿠키를 보낼 정보를 넣어서 보낸다.
  * **Expires |** 만료일이 되면 쿠키를 삭제한다. *(ex. set-cookie: expires= Sat, 04-Jan-2023 00:00:00 GMT)*
  * **max-age |** 쿠키의 최대 시간을 의미한다. *(ex. set-cookie: max-age=3600(3600초))*
  * **Domain |** 명시한 도메인을 나타낸다. *(ex. set-cookie: domain=.google.com)*
  * **path |** 이 경로를 포함한 하위 경로 페이지만 쿠키가 접근됨. 일반적으로 루트(/)