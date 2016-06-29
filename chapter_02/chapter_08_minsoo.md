#2부 웹 개발 핵심편

>우리가 만든 사이트를 배포하는 방법을 배우고, 실제 운영 가능한 진짜 웹서버에 배포하도록 한다.

###8장 스테이징 사이트를 이용한 배포 테스트


----------------------------------------------------------
##TDD와 배포시 주의할 사항

배포 시 주의해야 할 사항들

- 정작 파일(CSS, 자바스크립트, 이미지 등등)
	- 이 파일들을 제공하기 위해 웹 서버에 특수한 설정을 해주어야한다.
- 데이터베이스
	- 권한이나 경로 문제가 있을 수 있기 때문에 테스트 데이터 관리에 유의해야 한다.
- 의존 관계
	- 개발한 소프트 웨어와 연계되어 있는 패키지를 서버PC에도 설치해 주어야한다.
	
해결책    

- 실제 운영 사이트와 동일한 황경의 스테이징(staging)사이트를 이용한다.
- 스테이징 사이트에 대해 기능테스트를 사용할 수 있다.
- virtualenv를 사용한다.
- `자동화` 된 스크립트를 사용해서 신규 버전을 배포하고, 이를 스테이징 서버와 운영 서버에 동시 배포함으로써 `운영 서버 = 스테이징 서버` 상태로 만들 수 있다.

```
제시된 해결책

로컬 PC -(자동화)-> 스테이징 사이트 -(자동화)-> 운영사이트
```

**이번장의 개요**    

```
1. 스테이징 서버에서 실행할 수 있도록 FT를 수정한다.
2. 서버를 구축하고 거기에 필요한 모든 소프트웨어를 설치한다. 또한 스테이징과 운영 도메인이 이 서버를 가르키도록 설정한다.
3. Git을 이용해서 코드를 서버에 업로드한다.
4. Django개발 서버를 이용해서 스테이징 사이트에서 약식 버전의 사이트를 테스트한다.
5. Virtualenv 사용법을 배워 서버에 있는 파이썬 의존 관계를 정리한다.
6. 과정을 진행하면서 지속적으로 FT를 실행한다. 이를 통해 단계별로 무엇이 동작하고, 무엇이 동작하지 않는지를 체크한다.
7. Gunicon, Upstart, 도메인 소켓 등을 사용해서 사이트를 운영 서버에 배포하기 위한 설정을 한다.
8. 설정이 정상적으로 동작하면 스크립트를 작성해서 수동으로 했던 작업을 자동화하도록 한다.(사이트 베포 자동화)
9. 동일 스크립트를 이용해서 운영 버전의 사이트를 실제 도메인에 배포한다.

```


----------------------------------------------------------

##항상 그렇듯이 테스트부터 시작

>기능테스트를 스테이징 사이트에서도 실행되도록 수정

**In `functional_tests/tests.py`**

```python
import sys

class NewVisitorTest(StaticLiveServerTestCase):

    @classmethod
    def setUpClass(cls):
        for arg in sys.argv:
            if 'liveserver' in arg:
                cls.server_url = 'http://' + arg.split('=')[1]
                return
            super().setUpClass()
            cls.server_url = cls.live_server_url

    @classmethod
    def tearDownClass(cls):
        if cls.server_url == cls.live_server_url:
            super().tearDownClass()
            
 ```
 
**[sys.argv에 대한 설명](http://ngee.tistory.com/159)**    
**[cls에 대한 설명](http://stackoverflow.com/questions/4613000/what-is-the-cls-variable-used-in-python-classes)** 질문, instance method - class method 차이
 
`LiveServerTestCase`는 몇가지 특정한 제약사항을 갖는다.    
그 중 하나는 자체 테스트 서버에서 사용된다고 가정하는 것이다.    
**우리는 이를 실제 운영서버에서 실행하게 만들고 싶다.**



**In `functional_tests/tests.py`**

```python
def setUpClass(cls):
    for arg in sys.argv:
        if 'liveserver' in arg
            cls.server_url = 'http://' + arg.split('=')[1]
            return
        super().setUpClass()
        cls.server_url = cls.live_server_url

@classmethod
def tearDownClass(cls):
    if cls.server_url == cls.live_server_url:
        super().tearDownClass()
            
 ```

----------------------------------------------------------

##도메인명 취득

도메인을 사면된다. 

> 메인도메인의 서브도메인도 가능하다

----------------------------------------------------------
## 수동으로 서버를 호스트 사이트로 프로비저닝하기

배포의 2가지 단계

- 신규 서버를 프로비저닝(Provisioning)해서 코드를 호스팅 할 수 있도록 한다.
- 신규 버전의 코드를 기존 서버에 배포한다.

혹자는 서비스를 배포할 때 마다 새로운 서버를 사용하는 것을 좋아하지만(PythonAnywhere를 통해) 규모가 큰 사이트나기존 사이트의 대규모 업데이트 시에만 필요한 것이다.    
간단한 사이트에서는 두 단계로 나누는 것이 좋다.    
최종적으로는 두가지를 자동화하겠지만, 지금은 수동 프로비저닝으로도 충분하다.    

>소프트웨어(WAS, DBMS, 어플리케이션 포함)를 시스템에 설치/배포 하고 필요한 구성 셋팅 작업을 해서 실행 할 수 있도록 해 놓는것을 ‘소프트웨어 프로비저닝' 이라 부르기도 한다.    
가상화 시스템, 유틸리티 컴퓨팅, 클라우드 컴퓨팅 환경에서는, 이렇게 IT 인프라 자원을 할당, 배치, 배포, 구성하는 작업들을 ‘자동화’ 하는 기능을 요구하고 있는데, 그래서 ‘자동 프로비저닝’ 이란 개념이 많이 이야기 되고 있다.

####사이트 호스트할 곳 정하기

사이트 호스팅을 위해서 대체로 두가지가 존재한다.

- 자체 서버(혹은 가상서버)운영
- Heroku, DotClout, OpenShift, PythonAnywhere 같은 Platform-As-A-Service(PaaS) 서비스 이용

####서버 구축하기

서버를 구축하기 위해서는 서버로 사용될 PC를 갖고 있어야한다.
AWS를 이용하여 INSTANCE를 만들어 사용하겠다.

1. 도메인을 구매한다.  [한국전자인증사이트 링크](https://www.cosmotown.co.kr/)
2. AWS Route53에서 구매한 도메인을 등록한다.
3. AWS EC2에서 새로운 Instance를 만든다.
4. Key를 발급받는다.
5. 받은 Key를 이용하여 서버로 사용될 PC에 ssh통신을 통해서 접속한다.


####사용자 계정, SSH, 권한

SSH통신을 이용해 Ubuntu에 접속한다.

```
ubuntu@ip-172-31-31-15:~$ sudo su
root@ip-172-31-31-15:/home/ubuntu# useradd -m -s /bin/bash elspeth
root@ip-172-31-31-15:/home/ubuntu# usermod -a -G sudo elspeth
root@ip-172-31-31-15:/home/ubuntu# passwd elspeth
root@ip-172-31-31-15:/home/ubuntu# su - elspeth
elspeth@ip-172-31-31-15:~$ 
```

####Nginx 설치

[Nginx 설치 링크](http://paphopu.tistory.com/20)


####FT를 이용해서 도메인 및 Nginx가 동작하는지 확인한다.

```
elspeth@ip-172-31-31-15:/$ python manage.py test functional_test/ --liveserver=tddgoat1.amull.net
```


```
======================================================================
FAIL: test_can_start_a_list_and_retrieve_it_later (functional_tests.tests.NewVisitorTest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/Users/hanminsoo/Documents/TDD_test/TDD_Test/superlists/functional_tests/tests.py", line 38, in test_can_start_a_list_and_retrieve_it_later
    self.assertIn('To-Do', self.browser.title)
AssertionError: 'To-Do' not found in 'Welcome to nginx!'

----------------------------------------------------------------------
Ran 2 tests in 4.756s
```

----------------------------------------------------------
##코드를 수동으로 배포

>스테이징 사이트를 복사해보고 Nginx와 Django가 제대로 상호작용이 되는지 확인한다.

####데이터베이스의 위치조정

신규 소스코드의 업데이트가 있더라도 DB는 그대로 유지하기 위해 위치를 변경한다

```python
DATABASES = {
	'default' : {
		'ENGINE' : 'django.db.backends.sqlite3',
		'NAME' : os.path.join(BASE_DIR, '../database/db.sqlite3'),
	}
}
```

```
elspeth@ip-172-31-31-15:/$ python manage.py migrate --noinput
```


####virtualenv생성

(pyenv virtualenv 사용)


----------------------------------------------------------

##Nginx 설정

```
elspeth@ip-172-31-31-15:/$ sudo vim /usr/local/nginx/conf/nginx.conf
```






----------------------------------------------------------

##운영 준비 배포 단계

> 기본 소프트웨어가 동작하는 걸 확인 했지만 개발하는데 쓰이는 테스트 서버를 운영 서버로 놓기에는 문제가 있다.
> 게다가 매번 runserver하는 것도 말이안되는 행동이다.

####Gunicorn으로 교체

```
elspeth@ip-172-31-31-15:/$ pip install gunicorn
elspeth@ip-172-31-31-15:/$ gunicorn superlists.wsgiapplcation
```

이후 기능테스트를 하면 CSS가 적용되지 않는 문제를 확인할 수 있다.

```
$ python manage.py test functional_tests --liveserver=tddgoat1.amull.net
Creating test database for alias 'default'...
.F
======================================================================
FAIL: test_layout_and_styling (functional_tests.tests.NewVisitorTest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/Users/hanminsoo/Documents/TDD_test/TDD_Test/superlists/functional_tests/tests.py", line 112, in test_layout_and_styling
    delta=10
AssertionError: 78.5 != 512 within 10 delta

----------------------------------------------------------------------
Ran 2 tests in 8.507s

FAILED (failures=1)
Destroying test database for alias 'default'...
```

####Nginx를 통한 정적파일 제공

collectstatic을 실행해서 모든 정적 파일을 Nginx가 찾을 수 있는 폴더에 복사

----------------------------------------------------------

```
elspeth@ip-172-31-31-15:/$ python manage.py collectstatic
```

이후 Nginx가 정적파일을 제공할 수 있도록 설정해줌

```
server{
	listen 80;
	server_name tddgoat1.amull.net;
	
	location /static {
		alias /home/elspeth/sites/tddgoat1.amull.net/static;
	}
	
	[...]
```

Gunicorn과 Nginx를 재시작한다.


####유닉스 소켓으로 교체하기

>스테이징과 운영서버를 동시에 가동하는 경우에는 둘다 포트 8000번을 사용할 수 없다. 그리고 다른 포트를 사용하게 하는 방법은 별로 좋지 않다.    
>가장 좋은 해결책은 유닉스의 도메인 소켓을 이용하는 방법이다.(Nginx - Gunicorn이 소켓 파일을 이용해서 서로 커뮤니케이션 하게 만든다.)

```
server{
	listen 80;
	server_name tddgoat1.amull.net;
	
	location /static {
		alias /home/elspeth/sites/tddgoat1.amull.net/static;
	}
	
location / {
	    proxy_set_header   Host $host;
        proxy_pass   http://unix:/tmp/tddgoat1.amull.net.socket;
        }
``` 

proxy_set_header는 Django와 Gunicorn이 어떤 도메인에서 실행되는지 알게 하게하는 역할을 맡는다. 이 설정은 뒤에 ALLOWED_HOSTS 보안 설정에도 사용된다.

그리고 Nginx, Gunicorn을 재시작한다.(Gunicorn은 socket을 사용한다.)

```
elspeth@ip-172-31-31-15:$ sudo service nginx restart
elspeth@ip-172-31-31-15:$ gunicorn --bind \ unix:/tmp/tddgoat1.amull.net.socket superlists.wsgi:application
```

####DEBUG를 False로 설정하고 ALLOWED_HOSTS 설정하기

>Django의 DEBUG기능은 개발자에겐 매우 편리하지만 더불어 크래커(cracker)에게도 사이트를 해킹하기 매우 좋은 정보를 보여주게 된다.     
>이를 막기 위해서 DEBUG내용을 보여주지 않도록 배포 버전에서는 False로 바꿔준다.

**In `superlists/settings.py`**

```
DEBUG = False

TEMPLATE_DEBUG = DEBUG

ALLOWED_HOSTS = ['tddgoat1.amull.net']
```

[What is **TEMPLATE_DEBUG**?](http://stackoverflow.com/questions/25275252/what-is-djangos-template-debug-setting-for)

이후 Nginx와 Gunicorn을 재시작한다.

####Upstart를 이용한 부팅 시 Gunicorn 가동

>서버를 부팅할 때 자동적으로 Gunicorn이 실행되도록 하고싶다. 이것은 Ubuntu의 Upstart를 이용해서 구현이 가능하다.
>Ubuntu에는 `sudo start`명령어를 통해서 스크립트 문을 실행할 수 있도록 만들어져 있다.    
>이는 `/etc/init/`폴더에서 만들 수 있으며 파일의 확장자는 `.conf`로 설정해 주어야한다.


**In `/etc/init/tddgoat1.amull.net.conf`**

```
description "Gunicorn Server for tddgoat1.amull.net"


start on net-device-up
stop on shutdown

respawn

setuid elspeth
chdir /home/elspeth/sites/tddgoat1.amull.net/TDD_Test/source

exec /home/elspeth/.pyenv/versions/3.5.1/envs/sites/bin/gunicorn --bind \ unix:/tmp/tddgoat1.amull.net.socket superlists.wsgi:application
```

**Tip**    
`.conf`파일 에러 로그확인 방법    
`sudo cd /var/log/upstart/`에서 이름에 맞는 `.conf`파일 찾아서 sudo 권한으로 접근
