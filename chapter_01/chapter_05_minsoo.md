# TDD_Study

1부 5장 자료

---------------------------------------------------------------------------
####기능 테스트에서 예측하지 못한 에러가 발생할 경우 디버깅 해야할 사항
 - print문을 사용해서 현재 페이지 텍스트를 확인해보기
 - 에러 메시지를 더 자세한 정보를 출력하도록 개선
 - 수동으로 사이트 열어보기
 - time.sleep을 이용해서 테스트를 잠시 중지시킴

---------------------------------------------------------------------------
###CSRF TOKEN

사용자 입력을 저장하기 위해서는 djando에서 구현되어있는 'csrf_token'을 사용하도록 합니다.
csrf_token을 통하여 POST요청을 위조해서 request를 보내는 공격을 막을 수 있습니다.

**Issue**

>***django>1.8이후 csrf_token사용시 unittest가 통과되지 않는 문제***

csrf_token 을 사용하여 POST요청을 보내도록 설계한 후 unittest를 할 경우
AssertionError가 뜨는 것이 확인됨
```script
Creating test database for alias 'default'...
FF.
======================================================================
FAIL: test_home_page_can_save_a_POST_request (lists.tests.HomePageTest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/path/to/my/Documents/projects/ttd/superlists/lists/tests.py", line 33, in test_home_page_can_save_a_POST_request
      self.assertEqual(response.content.decode(), expected_html)
      AssertionError: '<!DO[280 chars]     <input type=\'hidden\' name=\'csrfmiddlew[198 chars]l>\n' != '<!DO[280 chars]     \n    </form>\n    <table id="id_list_tab[101 chars]l>\n'

======================================================================
FAIL: test_home_page_returns_correct_html (lists.tests.HomePageTest)
----------------------------------------------------------------------
Traceback (most recent call last):
File "/path/to/my/Documents/projects/ttd/superlists/lists/tests.py", line 19, in test_home_page_returns_correct_html    self.assertEqual(response.content.decode(), expected_html)
AssertionError: '<!DO[280 chars]     <input type=\'hidden\' name=\'csrfmiddlew[183 chars]l>\n' != '<!DO[280 chars]     \n    </form>\n    <table id="id_list_tab[86 chars]l>\n'

----------------------------------------------------------------------
Ran 3 tests in 0.012s

FAILED (failures=2)
Destroying test database for alias 'default'...
(ttd)
```

이유인 즉슨 우리는 CSRF보호를 요청할 때 단순히 '{% csrf_token %}'이라는 명령어로 쉽게 사용하였지만 실제로 response받은 home_page는
```html
<input type='hidden' name='csrfmiddlewaretoken' value='ScastWK6UtLY4KNb5jBrYKFH4O8EZ8I6' />
```
라는 csrf 전용 hidden 타입의 input이 추가되기 때문에
render_to_string되어 랜더링된 home.html코드와 달라질 수 밖에 없는 것입니다.

이 문제를 해결하기 위해서 기존에 보냈던 httprequest와 같은 request를 사용할 필요가 있습니다.

**In `test_home_page_returns_correct_html():`**

```python
request = HttpRequest()

# ... other test code ...

expected_html = render_to_string('home.html', request=request)
```

**In ``test_home_page_can_save_a_POST_request():``**

```python
request = HttpRequest()
request.method = 'POST'
request.POST['item_text'] = 'A new list item'

# ... other test code ...

expected_html = render_to_string(
    'home.html',
        {'new_item_text': 'A new list item'},
    request=request,
)
```

####TDD code 작성 방법론
- 설정(Setup)
- 처리(Exercise)
- 어설션(Assert)
으로 나누어 한 줄씩 띄어쓰기

---------------------------------------------------------------------------
###Helper Method를 활용하여 반복되는 코드 묶어놓기

>Helper Method란?
>다른 메소드의 동작을 도와주기위해서 만들어진 메소드이다.
>이것은 일반적으로 하나의 복잡한 Task를 여러개의 작은 Task로
>나누는 데에 사용된다.(이때 작은 Task가 helper method의 역할을 수행한다)
[관련링크](http://forums.devshed.com/java-help-9/helper-method-350163.html)

test_로 시작하는 메소드만 테스트로 실행되기 때문에 helper method는 test_로 시작하는
선언을 해서는 안됩니다.

---------------------------------------------------------------------------
###Django ORM
객체 관계형 맵핑(이하 ORM)은 클래스에서 선언된 하나의 인스턴스를 하나의 데이터베이스
튜플로 정의한 것을 말합니다.

---------------------------------------------------------------------------
###테스트의 모듈화
*하나의 테스트는 하나의 기능만 테스트 해야한다*

- 버그추적의 용이함을 위해
- 앞의 Assertion이 실패할 경우 뒤의 Assertion의 상태를 파악할 수 없어기 때문에


---------------------------------------------------------------------------
###Python Template Engine - JinJa2

[관련링크](http://jinja.pocoo.org/)
