---
layout: post
title: 사내 빌드 머신 + 배포 서버 아키텍처
tags: jenkins
---

## 빌드 머신 Jenkins
처음 회사에 왔을 때 빌드 머신이 구축되어 있지 않아 개발자 개개인이 로컬PC에서 빌드 후 apk를 전달하는 형태로 업무가 진행되고 있었다.
유니티 클라우드 빌드를 도입하려고 알아보다가 세세한 설정도 힘들것 같고 비용에 비해 좋은게 없을듯 해서 jenkins로 구축하기로 했다.
이전 회사에서 한번 해본 경험이 있었고 관련 자료도 인터넷에 많기에 큰 무리 없이 구축할 수 있었다.
단, 안드로이드 빌드시 2~3번에 한번씩 무한 대기 타는 현상이 발생했었다.
100% 원인을 밝혀낸건 아니지만 젠킨스에서 유니티 로그 파일 생성 폴더를 지정해 줬더니 그런 현상이 사라지게 되었다.


빌드 머신 구축 이후 개발, 릴리즈 빌드에 대한 번거로움이 많이 줄게 되었다.
버튼만 누르면 빌드 번호가 찍힌 apk를 자동으로 뽑아주고 다운로드 링크까지 제공하기 때문에 별도로 웹서버를 구축할 필요도 없었다.
하지만 이정도로는 뭔가 아쉬웠다.

내가 추가로 원했던 요구사항은
- 빌드 완료/실패시 슬랙으로 알림 보내기
- 테스트 편의를 위해 이전 2~3개 빌드까지 다운로드 할 수 있도록 제공하기
- 현재 빌드된 파일의 생성 날짜, svn revision, 빌드 넘버등을 확인할 수 있도록 하기

이정도만 자동화 되어도 빌드 관리 및 테스트 하는데 훨씬 편할것 같았다.


## 배포 자동화
결국 빌드 까지는 자동화 했지만 배포에 대한 문제가 남아 있게 된 것이다.
젠킨스에서 웹을 통해 다운로드 할 수 있는 url을 제공하기는 하지만 일일히 url을 카피해서 팀원들에게 전달해줘야 하는 번거로움은 여전했다.
또한 우리 게임은 빌드 할 때마다 파일 이름에 자동으로 버전을 붙여서 apk를 뽑아내도록 세팅했기 때문에 url이 항상 달라진다는것 또한 문제였다.
그래서 별도의 배포 서버를 구축하기로 마음 먹었다.


## 빌드 완료시 슬랙으로 알림 보내기
빌드 완료시 젠킨스에서 슬랙으로 알림 보내는 것은 인터넷에 널려 있는 자료를 참고해서 쉽게 구축할 수 있었다.
슬랙에서 api key를 생성하고 젠킨스 플러그인 세팅에서 연동한 뒤 보낼 메시지만 적어주니 간단하게 완료되었다.


## 다운로드 페이지 만들기
앞서 말한것 처럼 빌드 할 때 마다 apk파일 이름이 달라지게 되어 있으므로 고정된 url을 슬랙으로 보내는 것은 의미가 없었다.
그리고 단순히 다운로드 할 수 있는 url만 던져주는것 보다 빌드 번호, 날짜, revision등도 확인할 수 있는 html페이지를 만들어 제공한다면 더 좋을것 같았다.
테스트에 도움도 되고, 혹시나 잘못 빌드되진 않았는지 직접 확인할 수 있으니 말이다. 

간단하게 html페이지를 구성하고 ajax통신을 통해서 빌드 파일 리스트를 가져와 뿌리면 될것 같았다.
또 빌드가 끝나면 슬랙으로 해당 페이지의 url을 전달해서 다른 팀원들이 확인할 수 있게 하면 항상 최신화된 빌드 내용을 웹을 통해서 볼 수 있는 시스템이 구축되는 것이다.

깔끔한 css파일을 하나 구해서 테이블로 html페이지를 구성하는것 까지는 쉽게 진행 되었다.
다음으로 서버에서 파일 리스트를 가져와 테이블에 뿌리면 끝.
근데 젠킨스는 파일의 링크만 제공할 뿐이지 api요청을 받아 빌드 파일 리스트를 넘겨준다던가 하는 작업들은 불가능했다.
결국 별도의 api server의 구축이 필요한 것이다.

간단하게 아키텍처를 구성해 보면 다음과 같다.

젠킨스에서 빌드 완료 --> 현재 빌드된 파일의 이름과 링크를 api server에 전달 --> 전송받은 내용을 DB에 저장 --> 슬랙으로 노티피케이션

사용자는 다운로드 페이지에 접속 --> 해당 페이지의 코드에서 ajax로 api server와 통신 --> 수신된 내용을 페이지에 표시 --> 해당 링크를 눌러 빌드된 파일 다운로드


즉, 다운로드 자체는 젠킨스 서버를 통하고 해당 파일들의 리스트를 관리하는 서버와 디비 그리고 사용자가 볼 수 있게 html페이지를 구축하는것이 구현 목표이다.


## 빌드 리스트 관리 서버 구축
빌드된 파일 리스트를 html페이지에 보여주려면 서버 어디엔가 해당 리스트들이 저장되어 있어야 한다. 
DB는 번거롭기 때문에 간단하게 redis를 사용하기로 했다.
redis앞단에 nodejs로 api server를 구축하면 적은 노력으로 구축이 가능할것 같았다.
nodejs를 쓴 이유는 그냥 제일 간단히 구축할 수 있기 때문이다. 
난 c#이 주력 언어이긴 하지만 그렇다고 asp.net을 쓰자니 visual studio 세팅부터 iis까지 손이 너무 많이 든다.
java를 쓰는건 일을 너무 크게 벌리는것 같았고 php는 너무 올드했다.
그냥 아무 생각 없이 코딩 몇줄로 끝내고 싶었기 때문에 nodejs가 딱 맞았다. 

nodejs는 이번에 처음 써보는 환경이지만 어차피 큰 아키텍처는 비슷하고 어떤 도구로 구현하냐의 차이만 있는 것이기 때문에 생각보다 쉽게 진행할 수 있었다.
그냥 간단하게 http포트 열어서 express플러그인 붙인뒤 rest api를 제공하는 서버를 만들었다.
어차피 사내에서만 사용할 것이기 때문에 성능이나 안전성은 특별히 고려하지 않아도 되었다.

오히려 시간이 많이 소요된 부분은 젠킨스와 nodejs서버를 연동하는 부분이었다.
어차피 rest api통신만 하면 되기 때문에 별도의 플러그인을 쓸 필요는 없고 powershell에서 curl을 통해 쏘면 될것 같았다.
문제는 내가 powershell에 익숙하지 않아서 문법을 검색하고 검증하는데 생각보다 많은 시간이 들었다.

또 빌드된 파일의 이름과 링크를 가져오는것도 생각보다 시간이 많이 소요되었다.
왜냐하면 젠킨스에는 빌드된 링크를 제공하는 환경변수 같은것은 존재하지 않기 때문이다.
직접 관리 페이지를 열어서 링크를 손으로 복사할 순 있지만
자동화를 하려면 변수로 가져와야 하기 때문에 이 과정을 구현하는데 시간을 많이 사용했다.

결국 어찌 어찌 씨름해서 빌드된 파일의 이름과 링크를 powershell 변수에 담는데 성공했고 이것을 curl을 통해 nodejs서버로 보낼 수 있었다.
nodejs에서는 요청받은 내용을 redis에 저장하게 했으며 이 과정은 인터넷 검색을 통해 금방 끝낼 수 있었다.
마지막으로 html에서 ajax코딩을 통해 nodejs로부터 파일 리스트를 얻어와 페이지에 뿌려줬다.


## 결과
![](https://user-images.githubusercontent.com/438767/51073904-b9e48480-16ba-11e9-8f09-dcee237af9a2.png)


## 아키텍처
![](https://user-images.githubusercontent.com/438767/51073996-50657580-16bc-11e9-8d80-71bd32d56783.png)


## 정리
- 젠킨스로 사내 빌드 머신 구축
- 젠킨스와 슬랙을 연동하여 빌드 알림 자동 전송
- powershell, nodejs, redis를 사용하여 빌드된 파일 리스트 관리/출력
- html웹페이지를 통해 빌드 파일 다운로드 페이지 구축



