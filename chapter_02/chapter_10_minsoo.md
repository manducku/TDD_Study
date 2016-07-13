#2부 웹 개발 핵심편


###10장 입력 유효성 검사 및 테스트 구조화
>이 후 몇장에 걸처서 입력 테스트, 유효성 검사에 대해 다룬다.


----------------------------------------------------------
##FT 유효성 검사 : 빈 작업 아이템 방지

사용자의 이용 수준에 따라 작업 목록이 자칫 지저분해질 수 있는 문제가 생긴다.(예를 들어 빈 작업 아이템을 등록한다던가 중복으로 등록한다던가) 우리는 이러한 실수가 일어날 수 있는 가능성을 배제하고 싶다.

**In `functional_tests/tests.py`**

```python
def test_cannot_add_empty_list_items(self):
#에디스는 메인 페이지에 접속해서 빈 아이템을 실수로 등록하려고 한다
# 입력상자가 비어있는 상태에서 엔터키를 누른다

#페이지가 새로고침 되고, 빈아이템을 등록할 수 없다는
#메시지가 표시된다.

#다른 아이템을 입력하면 정상처리된다

#고의로 다시한번 빈 아이템을 등록한다

#리스트 페이지에 에러메시지가 표시된다.

#아이템을 입력하면 정상작동한다.

self.fail('wirte me!')
```

>기능 테스트는 사용자 스토리와 밀접한 관련이 있다는 것을 알 수 있다. 일이 더 진행되기 전에 테이트 파일을 나누어 하나의 테스트 파일에 한 개의 테스트만 진행되도록 수정한다.



###테스트 건너뛰기

***리팩터링할때 테스트가 모두 통과된 상태에서 진행하는 것이 좋다.*** 앞에서 `self.fail`함수를 이용해서 테스트를 고의적으로 실패시켰으니 `skip`데코레이터를 활용해서 테스트를 잠시 꺼둔다

**In `funtional_tests/test.py`**

```python
from unittest import skip

[...]

@skip
def test_cannot_add_empty_list_items(self):

[...]

```


> 테스트를 하는 동안 `skip`메소드를 사용하는 것은 다소 위험성이 내제되어있다 그 이유인 즉슨 `skip`메소드를 사용했다는 사실을 잊고 테스트가 통과했다는 메시지 하나로 `commit`을 할 수 있기 때문이다.    
> 그래서 사용해야할 것이 바로 TDD메모장 이다.



```
TDD메모장

1. test_cannot_add_empty_list_items함수에 skip메소드를 사용했다.

```



###기능 테스트를 여러 파일로 분할하기

각 테스트를 개별 클래스로 분할하고 싶다.(이 단계에선 아직 하나의 파일 내에 모든 테스트들이 존재한다.)


**In `functional_test/test.py`**

```python
class FunctionalTest(StaticliveServerTestCase):

	@classmethod
	def setUpClass(cls):
	[...]
	
	@classmethod
	def tearDownClass(cls):
	[...]
	
	def setUp(self):
	[...]
	
	def tearDown(self):
	[...]
	
	def check_for_row_in_list_table(self, row_text):
	[...]
	
class NewVisitorTest(FunctionalTest):

	def test_can_start_a_list-and_retrieve_it_later(self):
	[...]
	
class LayoutAndStylingTest(FunctionalTest):
	
	def test_layout_and_styling(self):
	[...]
	
class ItemValidationTest(FunctionalTest):
	
	@skip
	def test_cannot_add_empty_list_items(self):
	[...]
	

```


테스트가 통과되는 것을 확인한 후에 `functional_test`폴더 내부에 존재하는 `test.py`를 `base.py`로 변경하고 헬퍼클래스를 넣은 후에 각각 `functional_test/item_simple_list_creation.py`, `functional_test/test_layout_and_styling.py`, `functional_test/test_list_item_validation.py`파일을 추가하여 수정한다.    
그리고 테스트가 통과한 것을 확인했다면 앞서 사용했던 `skip` 메소드를 지우고 커밋한다

```
TDD 메모장
1. test_cannot_add_empty_list_items함수에 skip메소드를 사용했다.
2. skip method를 삭제했다
```


###FT에 살 붙이기

**In `functional_test/test_list_item_validation.py`**


```python
from .base import FunctionalTest


class ItemValidationTest(FunctionalTest):

    def test_cannot_add_empty_list_item(self):
         #에디스는 메인 페이지에 접속해서 빈 아이템을 실수로 등록하려고 한다
        # 입력상자가 비어있는 상태에서 엔터키를 누른다
        self.browser.get(self.server_url)
        self.browser.find_element_by_id('id_new_item').send_keys('\n')

        #페이지가 새로고침 되고, 빈아이템을 등록할 수 없다는
        #메시지가 표시된다.
        error = self.browser.find_element_by_css_selector('.has-error')
        self.assertEqual(error.text, "빈 아이템은 등록할 수 없다.")

        #다른 아이템을 입력하면 정상처리된다
        self.browser.find_element_by_id('id_new_item').send_keys('1: 우유 사기\n')
        self.check_for_row_in_list_table('1: 우유 사기')

        #고의로 다시한번 빈 아이템을 등록한다
        self.browser.find_element_by_id('id_new_item').send_keys('\n')

        #리스트 페이지에 에러메시지가 표시된다.
        self.check_for_row_in_list_table('1: 우유 사기')
        error = self.browser.find_element_by_css_selector('.has-error')
        self.assertEqual(error.text, "빈 아이템을 등록할 수 없습니다")

        #아이템을 입력하면 정상작동한다.
        self.browser.find_element_by_id('id_new_item').send_keys('tea 만들기\n')
        self.check_for_row_in_list_table('1: 우유 사기')
        self.check_for_row_in_list_table('2: tea 만들기')
```

- .has-error 함수는 부트스트랩 CSS에서 지원하는 파일로써 에러 텍스트를 표시한다.


테스트를 시도하면 예상된 에러가 나타난다.

```
selenium.common.exceptions.NoSuchElementException: Message: Unable to locate element: {"method":"css selector","selector":".has-error"}
```


------------------------------------------------------

##모델-레이어 유효성 검증

Django에서는 두 단계로 유효성 검증을 할 수 있다.  
  
- 모델 계층에서 유효성 검증
- 상위 계층에 있는 폼(Form)단에서 유효성 검증

데이터베이스나 데이터베이스 무결성 규칙 부분을 직접 테스트 가능하며, 더 안전하다는 이유로 `모델 계층에서의 유효성 검증`을 사용하는 사람들이 있다.


###단위 테스트를 여러 개 파일로 리팩터링하기
>앞서 우리가 기능의 유효성을 검사하기 위해서 유닛테스트를 진행했다.    
>우리는 이 유닛테스트를 리펙터링할 생각이다.

`lists/tests.py` 폴더를 View Test, Model Test로 나누어 `lists/tests/test_views.py`와 `lists/tests/test_models.py`로 리펙터링하자

**In `lists/tests/test_views.py`**

```python
[...]


class HomePageTest(TestCase):
	[...]


class NewListTest(TestCase):
	[...]
		
```



**In `lists/tests/test_models.py`**

```python
[...]

class ListAndItemModelTest(TestCase):
	[...]

```

###단위 테스트 모델 유효성 검증과 self.assertRaises 켄텍스트 관리자

빈 작업 아이템이 들어왔을 경우 에러코드를 발생시키기 위해서 일단 새로운 메소드를 추가한다.

**In `lists/tests/test_models.py`**

```python
from django.core.exceptions import ValidationError

[...]

class LIstAndItemModelsTest(TestCase):
	[...]
	
	def test_cannot_save_empty_list_items(self):
		list_ = List.objects.create()
		item = Item(list=list_, text='')
		with self.assertRaises(ValidationError):
			item.save()
```

테스트를 실행하면 예상했던대로 에러가 뜬다

```
AssertionError: ValidationError not raised
```

### 수상한 Djgno:모델 저장은 유효성 검사가 되지 않는다.(질문)

> 나의 의문점, TestField가 애초에 blank =False라면 text에 빈값이 들어가는 순간 error가 떠야하는 것이 아닌가?

Django가 갖고 있는 이상한 점을 발견할 수 있는데 사실상 앞서 나왔던 에러는 사실 pass되었어야할 테스트다.    
그 이유인 즉슨 Django 모델 필드를 살펴보면 `TestField`가 `blank=False` 로 초기 설정되어있는 것을 볼 수 있다.    
애초에 `TextField`는 빈 값을 허용해서는 안된다.    
그럼에도 테스트가 실패하는 이유는 Django모델이 저장 처리에 대해서는 유효성 검사를 하지 못해서이다.    
데이터베이스 저장과 관련된 처리에선 에러가 발생하지만 SQLite의 경우, 텍스트에 `null`값을 강제로 부여하지 못하기 때문에 save메소드가
빈 값을 그냥 통과시켜 버린다.(원래는 데이터베이스가 빈 값에 null이라는 값을 부여해서 테스트가 통과되어야하지만, SQLite는 빈 값에 null을 부여하는 기능이 없어서
save메소드가 그냥 그대로 통과시키다보니 빈값을 받고 에러가 뜨는것인가?)

Django의 유효성 검증을 수동으로 하기 위해서 `full_clean`이라는 메소드를 사용한다.

**In `lists/tests/test_models.py`**

```python
with self.assertRaises(validationError):
	item.save()
	item.full_clean()
```

이것을 통해서 텍스트 필드를 `blank=true`로 해야한다는 것을 알 수 있었다(난 아직도 모르겠다.)


---------------------------------------------------------------

##뷰를 통한 모델 유효성 검증

뷰 계층에서 모델 유효성을 검증하면서 이것을 템플릿으로 가져오는 일을 해보겠다.    
이를 통해 사용자가 유효성 검증을 확인할 수 있다.(이게 뭔소리야?)
HTML을 통해서 에러를 표시하는 방법을 알아보자

템플릿이 에러 변수를 제대로 전달했는지 확인함과 동시에 폼 옆에 이것을 출력한다.

**In `lists/templates/base.html`**

```html
<!DOCTYPE html>
<html lang="euc-kr">

  <head>
    <title>To-Do lists</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <link href="/static/bootstrap/css/bootstrap.min.css" rel="stylesheet" media="screen">
    <link href="/static/base.css" rel="stylesheet" media="screen">
  </head>

  <body>
    <div class="container">
      <div class="row">
        <div class="col-md-6 col-md-offset-3 jumbotron">
          <div class="text-center">
            <h1>{% block header_text %}{% endblock %}</h1>
            <form method="POST" action="{% block form_action %}{% endblock %}">
              <input name="item_text" id="id_new_item" class="form-control input-lg"placeholder="작업 아이템 입력" />
              {% csrf_token %}
              {% if error %}
              	<div class="form-group has-error">
              		<span class="help-block">{{ error }}</span>
              	</div>
              {% endif %}
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

에러를 템플릿으로 전달하는 역할은 뷰 함수가 담당한다. 약간 다른 두가지 에러 처리 패턴을 사용하려한다

**1. 신규 목록용 URL과 view가 home.html페이지와 동일한 템플릿을 출력하고, 에러 메시지를 함께 출력한다.**


**In `lists/tests/test_views.py`**

```python
class NewListTest(TestCase):
	
	[...]
	def test_validation_errors_are_sent_back_to_home_page_template(self):
		response = self.client.post('/lists/new', data={'item_text':''})
		self.assertEqual(respnse.status_code, 200)
		self.assertTemplateUsed(response, 'home.html')
		expeted_error = "You Can Not ... item"
		self.assertContains(response, expected_error)
```


여기서 우리는 `/lists/new`라는 URL을 하드코딩했지만 이것은 DRY원칙에 위배되며 코드의 유동성에 대처하지 못하므로 굉장히 좋지 못한 코딩방식이다.
(실제로 개발하다가 URL을 바꿀수도 있으니 말이다.)    
때문에 나중에 우리는 이 URL을 유동적바꿔야한다.

다시 테스트로 돌아와서... 해보면 뷰가 status code 302를 돌려주며 에러가 뜨는 것을 확인하게 될 것이다.    
뷰에서 full_clean()을 호출해본다

**In `lists/views.py`**

```python
def new_list(request):
    list_ = List.objects.create()

    item = Item.objects.create(text=request.POST['item_text'], list=list_)
    item.full_clean()
    return redirect(
            '/lists/%d/' % (list_.id,)
           )
```
뷰를 보면 제거해야할 하드코딩 URL후보를 찾을 수 있다. 이것을 메모장에 기록하도록한다.

```
TDD 메모장
1. test_cannot_add_empty_list_items함수에 skip메소드를 사용했다.
2. skip method를 삭제했다
3. views.py에서 하드코딩된 URL을 제거
```

테스트를 해보면 유효성 검증이 예외 상황을 발생시키고 이것은 뷰를 통해서 표시된다.

```
django.core.exceptions.ValidationError: {'text': ['This field cannot be blank.']}
```

우리는 이 에러를 잡아주기 위해서 먼저 try/exept를 사용해서 접근해보겠다.

**In `lists/views.py`**

```python
def new_list(request):
    list_ = List.objects.create()

    item = Item.objects.create(text=request.POST['item_text'], list=list_)
    item.full_clean()
    
    try:
        item.full_clean()
    except ValidationError:
        pass
        
    return redirect(
            '/lists/%d/' % (list_.id,)
            )
```

테스트를 돌려보면 `302 ! = 200` 에러를 돌려주는 것을 볼 수 있게된다.    

그러면 렌더링된 템플릿을 반환해서 템플릿 체크를 해보자

**In `lists/views.py`**

```python
except ValidationError:
	return render(request, 'home.html')
```

그렇게 되면 `AssertionError: False is not ture` 에러를 돌려주게 된다.    

신규 템플릿 변수를 사용해서 해결한다.

**In `lists/views.py`**

```python
except ValidationError:
	error = "You can't have an empty list item"
	return render(request, 'home.html', {"error": error})
	
```  
하지만 Django에서 `어퍼스트로피` 가 HTML을 이스케이프 히기 때문에 이상한 코드가 반환되어 일치하지 않는다    

그걸 해결하기 위해 `escape`메소드를 사용한다

**In `lists/tests/test_views.py`**

```python
from django.utils.html import escape

[...]

	expected_error = espcae("You can't have an empty list item")
	self.assertContaions(response, expected_error)
```

그러면 테스트가 통과하는 것을 확인 할 수 있다.


###데이터베이스에 잘못된 데이터가 저장되어있는지 확인

앞선 테스트 코드의 잘못된 점은 바로 유효성 체크가 실패해도 객체를 생성한다는 것에 있다.

**In `lists/views.py`**

```python
item = Item.objects.create(test=request.POST['item_text'], list=list_)
try:
	item.full_clean()
except ValidatoinError:
	[...]
```

새로운 단위테스트를 추가해서 빈 아이템이 저장되는 것을 막는다

**In `lists/tests/test_views.py`**

```python
   def test_invalid_list_items_arent_saved(self):
        self.client.post('/lists/new', data={'item_text': ''})
        self.assertEqual(List.objects.count(),0)
        self.assertEqual(Item.objects.count(),0)
```

먼저 빈 아이템을 저장해서 테스트를 돌려보면 현재 object가 1개라고 표시되며 에러가 발생하는 것을 볼 수 있다.    
우리는 코드 수정을 통해 이 테스트 코드를 통과시켜야한다.



**In `list/views.py`**

```python
 try:
        item.full_clean()
        item.save()
    except ValidationError:
        list_.delete()
        error = "You Can't Have An Empty List Item"
        return render(request, 'home.html',{"error":error})

```

이렇게 하면 단위테스트가 통과된다. 기능테스트도 마찬가지 일까?

완료되었으면 `commit`하자

-----------------------------------------------------------------

##Django 패턴 : 폼 렌더링 뷰와 같은 뷰에서 POST요청 처리

>이번 접근 법은 Django에서는 일상적인 방법인 뷰를 사용해서 POST요청을처리하는 것을 해볼 것이다.
>이 방법은 RESTful URL 모델에 적합하지 않으나, 동일 URL을 사용해서 폼만 아니라 사용자 입력시 발생하는 에러도 출력하는 이점이 있다.

현재는 하나의 뷰와 URL을 이용해서 목록을 출력하고 별도 뷰와 URL을 이용해서 목록 아이템을 추가하는 일을 하고있다. 이를 합칠것이다.

**In `lists/templats/list.html`**

```html
{% block form_action %}/lists/{{ list.id }}/{% endblock %}
```

이 작업을 하면서 또 URL을 하드코딩하고있다. 이것도 제거해야한다.


```
TDD 메모장
1. test_cannot_add_empty_list_items함수에 skip메소드를 사용했다.
2. skip method를 삭제했다
3. views.py에서 하드코딩된 URL을 제거
4. list.html과 home.html의 폼에서 하드코딩된 URL을 제거한다.
```

이기능은 view_list페이지가 POST요청을 어떻게 처리하는지 모르기 때문에 기능테스트를 망가뜨릴 수 있다.

###리팩터링 : new_item 기능을 view_list로 옮기기

**In `lists/tests/test_views.py`**

```python
class ListViewtest(TestCase):
    [...]

    def  test_can_save_a_POST_reqeust_to_an_existing_list(self):
        other_list = List.objects.create()
        correct_list = List.objects.create()

        self.client.post(
            '/lists/%d/' % (correct_list.id,),
            data = {'item_text':'기존 목록에 신규 아이템'}
            )

```


변경후에 이전 처럼 1개의 Error가 뜨는 것을 확인 하고 `Commit`한다.


###view_list에서 모델 유효성 검증 구현

모델 검증 규칙에 맞춰 리스트 아이템을 추가하는 처리가 필요하다.    
새로운 단위 테스트를 작성해보자.(home 페이지에 적용한 것과 거의 비슷한 몇가지만 수행하면 된다.)


아래의 코드는 아이템 텍스트에 빈 값을 보낼때 리디렉션이 되지 않고    
`You Can't Have An Empty List Item`이라는 값이 뜨게 만든다는 테스트 코드이다.


**In `lists/tests/test_views.py`**

```python
def test_validation_errors_end_up_on_lists_page(self):
    list_ = List.objects.create()
    response = self.client.post(
    '/lists/%d/' % (list_.id,),
    data = {'item_text':''}
    )
    self.assertEqual(response.status_code, 200)
    self.assertTemplateUsed(response, 'list.html')
    expected_error = escape("You Can't Have An Empty List Item")

    self.assertContains(response, expected_error)

```

하지만 뷰에서는 어떤 유효성 검증도 하지 않은채로 모든 POST요청을 `/lists/%d' % (list_.id)`로 리디렉션
해버리기 때문에 `302!=200`에러가 발생한다.    

때문에 빈 값이 들어오면 그냥 `home.html`을 랜더링 하도록 유도할 필요가 있다.

**In `lists/views.py`**

```python
def view_list(request, list_id):

    list_ = List.objects.get(id=list_id)
    error = None

    if request.method == 'POST':
        try:
        item = Item(text=request.POST['item_text'], list=list_)
        item.full_clean()
        item.save()
        return redirect('/lists/%d/' % (list_.id,))
    except ValidationError:
        error = "You Can't Have An Empty List Item"

    return render(
        request,
        'list.html',
        {'list':list_,
        'error': error}
    )
```
현재 `views.py`내부에 `try/except` 두 번이나 중복해서 사용했다.
이건 별로 모양새가 좋지 못하다.

```
TDD 메모장
1. test_cannot_add_empty_list_items함수에 skip메소드를 사용했다.
2. skip method를 삭제했다
3. views.py에서 하드코딩된 URL을 제거
4. list.html과 home.html의 폼에서 하드코딩된 URL을 제거한다.
5. 뷰에 현재 중복 되어있는 try/except 검증 로직을 제거한다.

```

-------------------------------------------------------------------------

##리팩터링 : 하드코딩된 URL 제거해야할

>우리는 개발을 하다보면 어쩔 수 없이 URL의 주소를 바꿔어야할 상황이 생기곤한다.    
>하지만 그렇게 바꾼다면 우리가 하드코딩해버린 URL을 일일히 찾아가서 손수 수정해줘야하는 악몽같은 일이 발생하게 된다.

이 문제를 해결해주기 위해서 Django는 `name=parameter`라는 최고로 유용한 방법을 고안해냈다.


**In `lists/urls.py`**

```python
from django.conf.urls import url
from django.contrib import admin

from lists.views import home_page, view_list, new_list

urlpatterns = [
    url(r'^admin/', admin.site.urls),

    url(r'^$', home_page, name='home'),

    url(r'^lists/(\d+)/$', view_list, name='view_list',),
    url(r'^lists/new$', new_list, name='new_list'),
]
```

###{% url %} 템플릿 태그

`home.html`에서 하드코딩된 URL을 Django 템플릿 태그로 변경한다.    
템플릿 태그는 URL의 "name"을 가리킨다.


**In `lists/templates/home.html`**

```HTML
{% block from_action %}{% url 'new_list' %}{% endblock %}
```

**In `lists/templates/list.html`**

```HTML
{% block from_action %}{% url 'view_list' list.id %}{% endblock %}
```

그 후 테스트가 통과하는 것을 확인하자

