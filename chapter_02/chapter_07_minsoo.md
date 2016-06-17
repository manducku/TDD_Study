#2부 웹 개발 핵심편

>TDD는 django 뿐만 아니라 정적 파일, 폼 데이터 검증, 자바스크립트 테스트, 배포 작업들도 포함된다.


###7장 레이이웃,스타일링 테스트

---------------------------------------------------------------------------
##레이아웃 스타일을 기능적으로 테스트

스모크(smoke)테스트를 통해서 CSS가 올바르게 로딩됬는지 확인할 수 있다.    
FT의 `NewVisitorTest`클래스에 새로운 테스트를 추가한다.

**In`functional_tests/tests.py`**

```python
def test_layout_and_styling(self):
        #민수는 메인 페이지를 방문한다.
        self.browser.get(self.live_server_url)
        self.browser.set_window_size(1024, 768)

        #그는 입력 상자가 가운데 배치된 것을 본다
        inputbox = self.browser.find_element_by_id('id_new_item')
        self.assertAlmostEqual(
                inputbox.location['x'] + inputbox.size['width'] / 2,
                512,
                delta=10
                )
```
해당 테스트는 잘못된 방향으로 TDD를 진행되게 할 수 있기때문에 편법을 사용함    
확인 후 삭제하게될 코드

코드해석
    
- 창의 크기를 고정시킴
- 입력 요소를 찾고 크기와 위치를 취득
- assertAlmostEqual 함수를 이용하여 반올림처리

 
그 후 신규 작업 목록 페이지에도 입력 상자가 가운데 배치되는지 확인    

**In`functional_tests/tests.py`**

```python
inputbox.send_keys('testing\n')
        inputbox = self.browser.find_element_by_id('id_new_item')
        self.assertAlmostEqual(
                inputbox.location['x'] + inputbox.size['width'] / 2,
                512,
                delta=10
                )
```

**질문**    
inputbox.send_keys('testing\n')가 무엇인가?

--------------------------------------------------------------------------------
##멋있게 만들기 : CSS 프레임워크 이용

Bootstrap을 이용해 CSS 코딩

```
$ wget -O bootstrap.zip https://github.com/twbs/bootstrap/releases/download/\
v3.1.0/bootstrap-3.1.0-dist.zip
$ unzip bootstrap.zip
$ mkdir lists/static
$ mv dist lists/static/bootstrap
$ rm bootstrap.zip
```

**주의**    
Mac에서는 wget이 기본적으로 설치되어있지 않으므로 직접 설치해주어야한다.    
그런데 `MAC OS 10.11.1`이상 버전에서는 `./configure --with-ssl=openssl`명령어를 이용해 설치 시    

```
configure: error: --with-ssl=openssl was given, but SSL is not available.
```
이런 에러가 발생하게 되는데 이를 해결하기 위해서는 다른 방법을 이용해서 설치해야 한다.
그 방법 중 하나가 `brew`명령어를 통해서 설치하는 방법인데 먼저

```
ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```
명령어로 `brew install`작업을 한 후에

```
brew install wget --with-libressl
```
명령어로 wget 최신 버전을 설치한다.(이 명령어는 기본적으로 다운, 압축풀기, ./configure까지 한번에 해준다.)
[참고](http://stackoverflow.com/questions/33886917/how-to-install-wget-in-capitan-mac-10-11)

Static파일을 경로에 맞춘 후에 Bootstrap을 적용할 수 있는 코드를 확인해본다.
[참고](http://getbootstrap.com/getting-started/)


--------------------------------------------------------------------------------
##Django 템플릿 상속

Bootstrap을 모든 페이지에 적용해야 하기때문에 각 페이지마다 적용 코드를 입혀야하는 번거로움이 생긴다.    
이를 해결하기 위해 Django에서 지원하는 템플릿 상속 기능을 사용하면 이를 해결할 수 있다.

home.html과 list.html의 공통점을 따로 뽑아서 공통인자로 묶는다면 마치 `헬퍼 함수`처럼 템플릿을 사용할 수 있다.

```
$ cp superlists/lists/templates/home.html superlists/lists/templates/base.html
```
명령어를 통해서 home.html을 복사해서 base.html을 만들고    
자식 템플릿이 커스퍼마이징 할 수 있도록 'block'을 설정

**In `/lists/templates/base.html`**

```html
<html>
  <head>
    <title>To-Do lists</title>
  </head>

  <body>
    <h1>{% block header_text %}{% endblock %}</h1>
    <form method="POST" action="{% block form_action %}{% endblock %}">
      <input name="item_text" id="id_new_item" placeholder="작업 아이템 입력" />
      {% csrf_token %}

    </form>
    {% blcok table %}
    {% endblock %}
  </body>
</html>
```

각각 home.html 과 list.html을 base.html을 상속 받을 수 있도록 수정한다.

**In `/lists/templates/home.html`**

```html
{% extends 'base.html' %}

{% block header_text %}To-Do 작업 목록 시작{% endblock %}

{% block form_action %}/lists/new{% endblock %}
```


**In `/lists/templates/list.html`**

```html
{% extends 'base.html' %}

{% block header_text %}Your To-Do list{% endblock %}

{% block form_action %}/lists/{{ list.id }}/add_item{% endblock %}

{% block table %}
  <table id="id_list_table">
    {% for item in list.item_set.all %}
    <tr>
      <td>
        {{ forloop.counter }}: {{ item.text }}
      </td>
    </tr>
    {% endfor %}
  </table>
{% endblock %}
```

--------------------------------------------------------------------------------
##Bootstrap 통합하기

base.html에 bootstrap을 적용함으로써 base.html을 상속받는 모든 html이 bootstrap의 영향을    
받게 되었다.

**In `list/tempates/base.html`**

```html
[...]
<head>
    <title>To-Do lists</title>
    <meta name="viewport" content="width-device-width, initial-scale=1.0">
    <link herf="css/bootstrap.min.css" rel="stylesheet" media="screen">
  </head>
[...]  
```

#####그리드를 이용한 행과 열 조정


**In`lists/templates/base.html`**

```html
<html lang="euc-kr">

  <head>
    <title>To-Do lists</title>
    <meta name="viewport" content="width-device-width, initial-scale=1.0">
    <link herf="css/bootstrap.min.css" rel="stylesheet" media="screen">
  </head>

  <body>
    <div class="container">
      <div class="row">
        <div class="col-md-6 col-md-offset-3">
          <div class="text-center">
            <h1>{% block header_text %}{% endblock %}</h1>
            <form method="POST" action="{% block form_action %}{% endblock %}">
              <input name="item_text" id="id_new_item" placeholder="작업 아이템 입력" />
              {% csrf_token %}
            </form>
          </div>
        </div>
      </div>
    </div>
    <div class="row">
      <div class="col-md-6 col-md-offset-3">
        {% block table %}
        {% endblock %}
      </div>
    </div>
  </body>
</html>
```

**주의**    
Django(뿐만 아니라 일반 웹서버에서)에서 정적 파일을(Static File)을 다루기 위해서는 기본적으로 두가지를 고려해야한다.

1. URL이 정적 파일을 위한 것인지 뷰 함수를 경유해서 제공되는 HTML을 위한 것인지 구분할 수 있는가?
2. 사용자가 원할 때 어디서 정적 파일을 찾을 수 있는가?

p143정적파일은 특정URL을 파일과 매칭시키는 역할을 한다    
작업 아이템1을 위해 django가 요구하는 것은 url의 접두사를 정의 하는 것    
특정 접두사로 시작하는 URL은 정적 파일을 위한 요청이라고 인식하게 된다.    
이 접두사는 `superlists/settings.py`에 `STATIC_URL='/static/'`이라고 정의되어 있다.    
때문에 Django에서 정적 파일을 인식하기 위해서는 static이라는 디렉토리에 정적파일을 넣고 경로 앞에 `/static/`을 붙여야한다.

**In`lists/templates/base.html`**

```html
<link herf="/static/bootstrap/css/bootstrap.min.css" rel="stylesheet" media="screen">
```

실제로 `manage.py runserver`를 해보면 `STATIC FILE`이 적용된 것을 확인 할 수 있다.

#####StaticLiveServerTestCase로 교체
하지만 Function Test를 실행해도 여전히 같은 오류가 나는 것을 확인 할 수 있는데    
이는 `runserver`명령어는 `static file`의 위치를 찾을 수 있지만 `LiveServerTest`는 그러한 능력을 갖추고 있지 못하기    
때문이다.    
그러므로 `StaticLiveServerTestCase`로 교체하여 사용하도록 한다.

--------------------------------------------------------------------------------

##BootStrap Component를 이용한 사이트 외형 개선

####점보트론
부트스트램은 Jumbotron이라는 클래스를 제공하는데 이는 페이지에 있는 특정 콘텐츠를 강조해주는 스타일을 말한다.
여기선 메인페이지의 헤더와 폼을 강조하기 위해 사용한다.

**In `lists/templates/base.html`**

```html
<div class="col-md-6 col-md-offset-3 jumbotron">
```

이 후 input form과 table의 클래스를 추가한다.

--------------------------------------------------------------------------------

##사용자 지정 CSS 사용하기

타이틀과 입력상자 사이에 간격을 만들고자 한다.

**In`lists/templates/base.html`**

```html
  <head>
    <title>To-Do lists</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link href="/static/bootstrap/css/bootstrap.min.css" rel="stylesheet" media="screen">
    <link href="/static/base.css" rel="stylesheet" media="screen">
  </head>
```

**In `lists/static/base.css`**

```css
#id_new_item{
	margin-top: 2ex;
	}
```

--------------------------------------------------------------------------------

##collectstatic과 다른 정적 디렉터리

Django 개발 서버는 폴더 내부에 있는 모든 static파일들을 찾아 제공하지만 배포 서버에서 이렇게 하는 것은 매우 비효율 적이다.
그래서 `collectstatic`이라는 명령어를 사용해서 static파일을 모조리 한곳에 모아 배포용으로 만들어둘 필요가 있다.

수집된 정적파일은 `settings.py`의 `STATIC_ROOT`에서 설정해준 경로에 저장된다.

**주의**    
정적파일은 레포지토리 밖에 있어야한다. 이유는 `lists/static`폴더 내부에 있는 파일과 동일하기 때문에 코드관리를 해줄 필요가 없다.

**In `superlists/settings.py`**

```python
STATIC_ROOT = os.path.abspath(os.path.join(BASE_DIR, '../static'))
```


```
$ python manage.py collectstatic
```
명령어를 통해 지정된 디렉토리 안으로 복사한다.

모아진 css는 관리사이트에 있는 css까지 모조리 복사한 것이므로 이것은 현재 공부단계에 필요없으니
`settings.py`에서 이 기능 꺼줍니다.


```python
INSTALLED_APPS = [
    #'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    'django_extensions',

    'lists',
]
```
