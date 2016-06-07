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

####TDD 접근법
 1. 기능 테스트(FT)를 먼저 작성
 1. 기능 테스트(FT)가 실패한다면 단위 테스트(UT)를 통해 코드 동작의 여부를 확인
 1. 단위 테스트(UT)가 실패한다면 테스트를 통과하도록 코드를 수정(여기서 통화 할때까지 2,3을 반복)
 1. 기능 테스트(FT)를 실행

---------------------------------------------------------------------------
###Django에서의 단위 테스트
Django에서는 'unittest.TestCase'의 확장 버전인 특수한 'TestCase'를 사용합니다.
(이는 테스트가 원활이 진행되도록 Django에 최적화된 버전을 말합니다.)

TestCase를 사용하기 위해서는
```
from django.test import TestCase
```
입력을 통해 TestCase를 import 해준 후 상속받아서 사용해야 합니다.

----------------------------------------------------------------------------
####Traceback을 읽고 에러해결 하기
예제

```
ERROR: test_root_url_resolves_to_home_page_view(lists.tests.HomePageTest)
--------------------------------------------------------------------------
Traceback (most recent call last):
    File "/workspace/.../test.py", line 8, in
test_root_url_resolves_to_home_page_view
    found = resolve('/')
  File "/usr/local/lib/.../urlresolver.py",
line 485, in resolve
    return get_resolver(urlconf).reslove(path)
  File "/usr/local/lib/.../urlresolver.py",
line 353, in resolve
    raise Reslover404({'tried':tried, 'path':new_path})
django.core.urlresolvers.Resolver404: {'tried':[[<RegexURLResolver
<RegexURLPattern list> (admin:admin) ^admin/>]], 'path':''}
--------------------------------------------------------------------------
[...]
```

1. 가장먼저 보아야할 ERROR는 어떤 테스트가 실패했는지 알려주는 역할을 합니다.
(예제에서는 test_root_url_resolves_to_home_page_view 테스트에 에러가 있습니다.)
1. 해당 테스트의 어떤 부분에서 에러가 발생하는 지를 찾습니다.
(예제에서는 line 8의 found = resolve('/')부분에 에러가 있습니다.

-----------------------------------------------------------------------------
###단위 테스트의 코드 주기
1. 단위 테스트가 어떻게 실패하는지 확인(Traceback)
1. 실패한 테스트를 수정하기 위한 최소한의 코드를 변경
