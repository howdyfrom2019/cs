# Browser - DNS - Web의 관계
<br />

###Q. 주소창에 `google.com`을 입력했을 때 무슨 일이 일어나는가?
<br />

1. 사용자가 브라우저에 `www.google.com`을 입력.
2. 브라우저는 해당 DNS 주소에 해당하는 IP 주소를 확인.
   * 이 과정에서 기존에 캐싱해둔 DNS 정보를 찾아본다.
3. HTTP 통신(TCP)를 사용하여 `HTML`문서를 요청.
4. 웹 서버는 WAS(웹 어플리케이션 서버)에 db에서 문서를 조회 요청.
5. 성공 시 해당 결과를 보내줌.
6. 브라우저는 해당 화면을 바탕으로 내용을 렌더링

## DNS Query
<br />

### Q. DNS 서버는 어떻게 IP 주소를 찾아내는 걸까?

ISP(Internet Service Provider)는 DNS서버가 호스팅하고 있는 서버의 IP 주소를 찾기 위해 `DNS query`를 날린다.  
DNS 서버들을 오가면서 에러가 날 때까지 반복적으로 검색한다.  

#### www.google.com 의 IP를 찾아내는 간략한 단계

> 🌐 **Root Domain** | .  
> > 🌐 **Top-Level Domains** | edu, org, io, **com**, kr  
> > > 🌐 **Second-level Domains** | naver, google, daum, apple
> > > > 🌐 **Third-level Domains** | www, downloads, dev

1. DNS recursor가 `root` name server에 요청.
2. `.com` name server로 리다이렉트.
3. `google.com` name server로 리다이렉트.
4. `www.google.com` name server로 리다이렉트.
5. 해당 주소에 매칭되는 IP 주소 찾기.
6. DNS recursor로 해당 주소를 보내기.  

## Q. 웹 어플리케이션 서버(WAS)의 역할은 무엇인가?
<br />

* 웹서버는 정적인 콘텐츠(html, 이미지 등)를 처리한다.(Apache, NginX, IIS)  
* 하나의 웹 서버가 모든 요청을 감당할 수 없기 때문에 `웹서버 - db` 간의 미들웨어로 WAS를 사용한다.  
* WAS는 DB를 조회해서 동적 웹에 필요한 컨텐츠 (jsp, asp, php)을 처리한다.  

## Q. 그렇다면 node.js는 웹서버인가?
<br />

* Chrome V8 JS 엔진으로 빌드된 **JavaScript 런타임.**
* 웹서버는 아니지만 직접 HTTP 서버를 작성하여 웹 서버를 띄울 수 있는 환경을 만들 수 있음.
* node.js는 정적파일 제공과 WAS를 모두 담당한다.
* Express.js를 사용하면 DB연결

## HTTP 통신 오류 코드 정의
> **1XX |** 정보가 담긴 메시지.  
> **2XX |** 성공. (response 보냄)  
> **3XX |** 클라이언트를 다른 URL로 리다이렉트.  
> **4XX |** 클라이언트 측에서 에러 발생.  
> **5XX |** 서버 측에서 에러 발생.  