# 스테이징 사이트를 이용한 배포 테스트

> 제품으로 출시되기 전까지는 모든 것이 재미있고 게임과 같다

###항상 그렇듯이 테스트부터 시작

테스트 임시 서버가 실행되는 주소를 변경한다
```{.python}
#functional_tests/test.py

import sys

class NewVisitorTest(StaticLiveServerCase):

	# 전체 테스트 메소드 이전에 한 번만 실행
	@classmethod
	def setUpClass(cls):
		#liveserver라는 커맨드 라인 인수를 탐색. 이후 테스트에 입력할 예정
		for arg in sys.argv:
			# 인수가 있다면 setUpClass를 건너뛰고 스테이징 서버 URL을 저장
			if 'liveserver' in arg:
				cls.server_url = 'http://' + arg.split('=')[1]
				return
		super().setUpClass()
		cls.server_url = cls.live_server_url

	@classmethod
	def tearDownClass(cls):
		#원래 조건절 if cls.server_url == cls.live_server_url: 이 붙으나 오류 발생. 
		super().tearDownClass()
```

FT결과 당연히 실패^ㅛ^

-

###도메인 취득

###수동으로 서버를 호스트 사이트로 프로비저닝하기

####사용자 계정, SSH, 권한

-

###코드를 수동으로 배포

####데이터베이스 위치 조정

```{.python}
#settings.py

# abspath를 먼저 실행하는게 임포트 문제를 예방하는 좋은 팁이라고한다.
BASE_DIR = os.path.abspath(os.path.dirname(os.path.dirname(__file__)))

DATABASES = {
	'default': {
		'ENGINE': 'django.db.backends.sqlite3',
		'NAME': os.path.join(BASE_DIR, '../database/sqlite3'),
	}
}
```

이후 마이그레이트 하고 테스트!

서버로 넘어가 클론

####Virtualenv

우리에겐 pyenv가 있다

####간단한 NGINX 설정(간단한..?)

```{.lua}
-- /etc/nginx/sites-available/superlist

server {
	listen 80;
	server_name superlist;

	location / {
		-- 로컬 포트 8000으로 오는 모든 요청을 장고로 보내서 응답하도록 프록시 한다
		proxy_pass http://localhost:8000;
	}
}
```

이후 서버에서 `$sudo ln -s /etc/nginx/sites-available/superlist /etc/nginx/sites-enabled/superlist`

설정파일은 sites-available에 두고 symlink는 sites_enabled에 둔다. 우분투에서 자주 사용되는 Nginx 설정 저장 방법이라 한다. 이 구조가 사이트를 쉽게 시작하고 종료할 수 있도록 한다.
이후 sites-enabled에 default파일은 삭제하고 Nginx 재시작후 runserver 실행. 작동을 확인한다.

로컬로 돌아와 기능테스트를 한다. `python manage.py test functional_tests --liveserver=staging.noreeter.com`

DB설정을 아무것도 하지 않았기에 에러가 뜬다.(이미 runserver 단계에서 경고를 보았을 것이다) 서버에서 마이그레이트 후 runserver. 로컬로 돌아와 다시 기능 테스트를 하면 성공!

-

###Gunicorn으로 교체

서버에 Gunicorn 설치후, `$gunicorn superlist.wsgi:application` 실행...을 하고 사이트를 들어가면 CSS가 깨질것이다. 로컬에서 기능테스트를 해봐도 에러...지금까지 장고 개발서버가 알아서 static파일을 알아서 전달해 줬다면, Gunicorn은 그렇지 못하다. 이제는 Nginx에게 이일을 일임해야 한다.

####Nginx를 통한 정적 파일제공

```{.lua}
-- /etc/nginx/sites-available/superlist

server {
	listen 80;
	server_name superlist;

	location /static {
		alias /home/ubuntu/TDD_superlist/static;
	}

	location / {
		proxy_pass http://localhost:8000;
	}
}
```

`$sudo service nginx reload` 이후 gunicorn을 다시 실행하면 잘 작동하는 모습을 볼 수 있다.(테스트도 통과)

####유닉스 소켓으로 교체하기

```{.lua}
-- /etc/nginx/sites-available/superlist

server {
	
	-- code

	location / {
		-- 장고와 Gunicorn이 어떤 도메인 상에서 실행되고 있는지 알게 한다. 이후 ALLOWED_HOSTS 보안 설정에 사용됨
		proxy_set_header Host $host;
		proxy_pass http://unix:/tmp/staging.noreeter.com.socket;
	}
}
```

Nginx 리로드 후 `$gunicorn --bind unix:/tmp/staging.noreeter.com.socket superlist.wsgi:application`

브라우저에서 잘 보인다면 로컬에서 기능 테스트를 다시 확인

####DEBUG를 False로 설정하고 ALLOWED_HOSTS 설정하기

```{.python}
# settings.py

DEBUG = False

TEMPLATE_DEBUG = DEBUG
```

####Upstart를 이용한 부팅 시 Gunicorn가동

```{.lua}


description "Gunicorn server for staging.noreeter.com"

start on net-device-up
stop on shutdown

respawn

setuid ubuntu
chdir /home/ubuntu/TDD_superlist/superlist

exec /home/ubuntu/.pyenv/path/to/gunicorn/binary \
--bind unix:/tmp/staging.noreeter.com.socket \
superlist.wsgi:application

```

 > exec 에서 gunicorn 바이너리 파일이 깔린 경로에서 gunicornd을 실행한다. pyenv라고 해도 cd가 먹히지 않기 때문에 경로 입력은 필수다.


####변경사항 저장: Gunicorn을 requirements.txt에 추가

-

###자동화

####프로비저닝
 1. 사용자 계정과 홈폴더가 있다고 가정한다
 2. apt-gt nginx git
 3. pyenv, virtualenv, autoenv 설정
 4. 가상 호스트를 위한 Nginx설정을 추가한다
 5. Gunicorn을 위한 Upstart 처리를 추가한다

####배포
 1. repository clone
 2. virtualenv 활성화
 3. `$pip install -r requirements.txt
 4. 데이터베이스를 위한 migrate
 5. 정적 파일을 위한 collectstatic
 6. `DEBUG = False` 및 `ALLOWED_HOSTS` 설정
 7. Gunicorn재시작
 8. FT를 실행하여 정상동작 하는지 확인

Nginx와 Upstart 설정 파일을 어딘가에 저장해서 이후에 재사용할 수 있도록 해두는 것이 필요.

`$mkdir deploy_tools`

```{.lua}
-- deploy_tools/nginx.template.conf

server {
	listen 80;
	server_name SITENAME;

	location /static {
		alias /home/ubunut/projectname/static;
	}

	location / {
		proxy_set_header Host $host;
		proxy_pass http://unix:/tmp/SITENAME.socket;
	}
}
```
```
-- deploy_tools/gunicorn-upstart.template.conf

start on net-device-up
stop on shutdown

respawn

setuid ubuntu
chdir /home/ubuntu/TDD_superlist/superlist

exec /home/ubuntu/.pyenv/path/to/gunicorn/binary \
--bind unix:/tmp/SITENAME.socket \
superlist.wsgi:application
```

