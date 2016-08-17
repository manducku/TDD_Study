#2부 웹 개발 핵심편


###11장 간단한 폼

> 앞장 마지막 부분에서는 뷰의 입력 검증 코드가 만이 중복된 상태에서 마무리를 지었다.    
> Django는 사용자 입력 검증과 검증 에러 메시지 출력을 위해 form 클래스 사용을 권장하고 있다.
> 이번 장에서는 단위 테스트 정리에도 시간을 투자 할 것이다.

----------------------------------------------------------
##유효성 검증 로직을 폼으로 옮기기

>Django의 복잡한 뷰에서 냄새가 나기에 뷰에 있는 로직을 폼으로 옮기고 싶다.    
>혹은 모델 클래스에 있는 사용자 지정 메소드로 만들고 싶다.    
>또 다른 경우에는 아예 Django를 사용하지 않고 다른 모듈로 만들어서 사용하는 경우도 있다.


Django의 폼은 정말 다양한 기능을 가지고 있다.

- 사용자 입력을 처리하고 검증해서 에러로 출력할 수 있다.
- HTML 입력 요소를 표시하기 위한 템플릿으로 사용할 수 있으며, 에러 메시지도 제공한다.
- 뒤에서도 보겠지만, 일부 폼은 데이터를 데이터베이스에 저장할 수도 있다.(Django ModelForm으로 변경 부분)



###단위 테스트를 통한 form API알아보기

단위 테스트를 이용해서 폼에 관련된 작은 실험을 해보자.

**In `lists/tests/test_forms.py`**

```python
from django.test import TestCase

from lists.forms import ItemForm


class ItemFormTest(TestCase):

    def test_case_renders_item_text_input(self):
        form = ItemForm()
        self.assertIn(''placeholder="작업 아이템 입력"', form.as_p())
        self.assertIn(''class="form-control input-lg"', form.as_p())
```

`form.as_p()`는 폼을 HTML로 렌더링하는 역할을 한다. 이제 이 테스트를 통과할 수 있도록 간단한 form을 하나 만들어 보도록 하자

**In `lists/forms.py`**

```python
from django import forms


class ItemForm(forms.Form):
	item_text = forms.CharField(
		widget=forms.fields.TextInput(attrs={
			'placeholder': '작업 아이템 입력',
			'class' : 'form-control input-lg',
			}
	),
	
```
`widget`명령어는 form필드 입력 부분을 커스터마이징 할 때 이용된다. 위의 예시는 `placeholer`에 `widget`을적용한 것 이다.


```
                                  질문
                                  
개발 주도 테스트 : 예비 코딩을 위한 단위 단위 테스트

새로운 API를 사용하기 시작할 때는 정식 TDD단계에 들어가기 전까지 여러가지를 시도해볼 수 있다.
이때는 인터랙티브 콘솔을 이용하거나 예비 코드를 작성하는 것도 허용돈다.

여기서도 실험적으로 폼 API를 단위 테스트에 사용했다
```

###Django ModelForm으로 변경

모델에 정의한 검증 코드를 재사용해서 폼에 적용하도록 한다. Django에서는 모델용 폼을 자동으로 생성할 수 있는 특수한 클래스를 제공한다. `ModelForm` 이것은 `Meta`라는 특수한 속성을 이용해서 설정한다.

**In `lists/forms.py`**

```python
from django froms

from lists.models Item


class ItemForm(froms.models.ModelForm):

	class Meta:
		model = Item
		fields = ('text',)
```

`class Meta`에서는 폼이 어떤 모델을 이용할지와 어떤 필드를 사용할지를 정의한다.    
`ModelForm`은 다른 타입의 필드에 HTML 폼의 입력 타입을 할당한다거나, 기본 검증로직을 적용하는 등 다양한 처리가 가능하다.[참고링크](https://docs.djangoproject.com/en/1.7/topics/forms/modelforms/)


`placeholder`와 CSS 클래스가 빠졌지만 `name="item_text"` 대신 `name="text"`를 사용하고 있다는 것을 볼 수 있다. 그리고 `input`이 아닌 `textarea`를 사용하고 있기 때문에 우리가 원하는 UI를 만들 수 는 없다.    
그래서 우리는 `widget`을 오버라이드해서 ModelfForm필드를 적용해야한다.

**In `lists/forms.py`**

```python
class ItemForm(forms.models.ModelForm):

	class Meta:
		model = Item
		fields = ('text',)
		widgets = {
			'text' : forms.fields.TextInut(
				attrs={
					'placeholder' : '작업 아이템 입력',
					'class' : 'form-control input-lg',
					}),
				}
			 )
			 
```


###폼 유효성 검증 커스터마이징 및 테스트

이제 `ModelForm`이 모델에 정의한 것과 같은 검증 규칙을 사용하는지 확인해보자.    
그리고 데이터를 폼에 어떻게 전달하는지 (마치 사용자가 입력한 데이터처럼) 배우도록 한다.


**In `lists/tests/test_forms.py`**

```python
def test_form_validation_frm_blank_items(self):
	form = ItemForm(data={'text':''})
	form.save()
```

```
ValueError: The Item could not be created because the data didn't validate.
```
빈 아이템 텍스트를 저장하려고 하자 에러를 출력한다.    
그러면 여기에 특정 에러 메시지를 지정할 수 있는지 알고싶다. 폼 유효성을 검증하는 API인 `is_valid`함수를 이용한다.

**In `lists/tests/test_forms.py`**

```python
from lists.forms import ItemForm


class ItemFormTest(TestCase):

    def test_case_renders_item_text_input(self):
        form = ItemForm(data={'text':''})
        self.assertFalse(form.is_valid())
        self.assertEqual(
                form.errors['text'],
                ["You can't have an empty list item"]
            )
```

```
AssertionError: ['This field is required.'] != ["You can't have an empty list item"]
```

에러 로그를 살펴보면 Django는 기본적으로 에러 메시지를 갖고 있다는 것을 볼 수 있다.    
우리는 이 default메시지를 변경하여 특별한 메시지를 직접 만들것이다.    
아까 한것과 같이 `Meta`클래스를 이용하여 기존 `error_message`를 커스터마이징한다.

-------------------------------------

## 뷰에서 폼 이용하기

>모든 기능들을 테스트해서 배포하기에는 너무 늦는다. 우리는 `최대한 빨리 배포한다`정신을 잊어선 안된다.


###뷰에서 GET 요청과 함께 폼 이용하기
home 뷰를 단위 테스트부터 시작해본다.

**In `lists/tests/test_view.py`**

```python

class HomePageTest(TestCase):

	def test_home_page_renders_home_template(self):
        response = self.client.get('/')
        self.assertTemplateUsed(response, 'home.html')

    def test_home_page_uses_item_template(self):
        response = self.client.get('/')
        self.assertIsInstance(response.context['form'], ItemForm)
```

그러면 p233의 form을 home 페이지 뷰에 적용하고 `base.html`에 마찬가지로 적용한다.    
그러고 나서 단위테스트를 돌리면 너무 긴 코드 비교에 에러 로그를 보기 힘든 상황에 마주하는데 이를 해결하기 위해 `assertMultiLineEqual`메소드를 사용한다.

**In `lists/tests/test_views.py`**

```python
class HomePageTest(TestCase):
    maxDiff = None
    
    [...]
    
    def test_home_page_returns_correct_html(self):
        request = HttpRequest()
        response = home_page(request)
        expected_html = render_to_string('home.html', request=request)
        self.assertMultiLineEqual(response.content.decode(), expected_html)

```

`assertMultiLineEqual`는 긴 문자열을 diff형태로 보여주지만 너무 길경우 뒷부분은 자르도록 초기설정 되어있다.
초기설정을 지우기 위해서 `maxDiff = None` 명령어를 사용한다.    
그 후에 테스트를 해보면 `render_to_string`이 폼을 모르기 때문에 에러가 발생한다.

**In `lists/test/test_views.py`**

```python
def test_home_page_returns.correct_html
	request = HttpRequest()
	response = home_page(request)
	expected_html = render_to_string(
		'home.html',
		{'form':ItemForm()}
	)
	self.assertMultiLineEqual(responsecontent.decode(), expected_html)

```

이와 같이 `render_to_string`함수에 폼을 랜더링하는 코드를 넣어줌으로써 테스트를 통과시킨다.

###찾아서 고치기

폼이 변경되어 기존에 사용하던 id와 name 속성이 더이상 작동하지 않는다.    
`functional_test`에 적혀있는 `id_new_item`가 form이 바뀌면서 사라졌기 때문에 입력상자를 찾지 못하는 것이 그 이유다.

```
grep id_new_item functional_tests/test*
```
명령어를 통해서 `grep id_new_item`가 몇 개나 잡혀있는지 한번 확인해 보자

```
functional_tests/test_layout_and_styling.py:        inputbox = self.browser.find_element_by_id('id_new_item')
functional_tests/test_layout_and_styling.py:        inputbox = self.browser.find_element_by_id('id_new_item')
functional_tests/test_list_item_validation.py:        self.browser.find_element_by_id('id_new_item').send_keys('\n')
functional_tests/test_list_item_validation.py:        self.browser.find_element_by_id('id_new_item').send_keys('우유 사기\n')
functional_tests/test_list_item_validation.py:        self.browser.find_element_by_id('id_new_item').send_keys('\n')
functional_tests/test_list_item_validation.py:        self.browser.find_element_by_id('id_new_item').send_keys('tea 만들기\n')
functional_tests/test_simple_list_creation.py:        inputbox = self.browser.find_element_by_id('id_new_item')
functional_tests/test_simple_list_creation.py:        inputbox = self.browser.find_element_by_id('id_new_item')
functional_tests/test_simple_list_creation.py:        inputbox = self.browser.find_element_by_id('id_new_item')
```
이것들을 모두 수정해주어야 한다.    

추천하는 방법은 `base.py`에 새로운 헬퍼 메소드를 작성하는 것이다.

**In `functional_tests/base.py`**

```python
class FunctionalTest(StaticLiveServerCase):
	[...]
	def get_item_input_box(self):
		return self.browser.find_element_by_id('id_text')
```

```
grep id_new_item functional_tests/tests*
```
명령어를 통해 `form`모델로 바꿔준 것들을 사용한다.


-----------------------------------------------------------------------------------

##POST 요청을 받는 뷰에서 폼 이용

>유효성 검증 부분을 중심으로 new_list 뷰를 위한 단위 테스트를 수정해야한다.


**In `lists/tests/test_views.py`**

```python
class NewListTest(TestCase):
	[...]
	
	def test_validation_error_are_sent_back_to_home_page_template(self):
        response = self.client.post('/lists/new', data={'text': ''})
        self.assertEqual(response.status_code, 200)
        self.assertTemplateUsed(response, 'home.html')
        expected_error = escape("You Can't Have An Empty List Item")
        self.assertContains(response, expected_error)

```

##new_list 뷰를 위한 단위 테스트 수정

이 테스트를 두 어썰션으로 분리하겠다.

- 검증 오류가 있으면 200 코드와 함께 home 템플릿을 출력한다.
- 검증 오류가 있으면 응답이 에러 메시지를 포함해야한다.
- (추가)검증 오류가 있으면 폼 객체를 템플릿에 전달한다.

####**[해당 코드 생략]**



###폼을 이용해서 템플릿에 에러 메시지 출력하기
템플릿에서 에러 메시지를 출력하기 위한 폼이 아직 적용되지 않았기 때문에 통과하지 못하고 있다. 이 문제를 해결하기 위해서는 폼 테스트를 해야한다.

**In `lists/templates/base.html`**

```python
<form method="POST" action="{% block form_action %}{% endblock %}">
	{{ form.text }}
    {% csrf_token %}
    {% if error %}
    	<div class="from-group has-error">
        	<span class="help-block">{{ error }}</span>
        </div>
    {% endif %}
</form>

```

변경 후 테스트 해보면 모든 템플릿에서 에러를 출력하는 방식을 바꿔놓은 바람에 에러 메시지를 표시하지 못한다는 에러가 발생한다.

```
Creating test database for alias 'default'...
................F.....
======================================================================
FAIL: test_validation_errors_end_up_on_lists_page (lists.tests.test_views.ListViewTest)
----------------------------------------------------------------------
Traceback (most recent call last):
  File "/Users/hanminsoo/Documents/TDD_test/TDD_Test/superlists/lists/tests/test_views.py", line 158, in test_validation_errors_end_up_on_lists_page
    self.assertContains(response, expected_error)
  File "/Users/hanminsoo/.pyenv/versions/TDD_test/lib/python3.5/site-packages/django/test/testcases.py", line 403, in assertContains
    msg_prefix + "Couldn't find %s in response" % text_repr)
AssertionError: False is not true : Couldn't find 'You Can&#39;t Have An Empty List Item' in response

----------------------------------------------------------------------
Ran 22 tests in 0.095s

FAILED (failures=1)
```

--------------------------------------------------
##폼 자체의 save메소드를 사용해보기

>폼이 데이터를 데이터베이스에 저장할 수 있다는 건 앞서 말했다.    
>하지만 우리가 만든 서비스에서는 제대로 작동 되지 않을 것이다.(아이템이 어떤 목록에 저장돼야하는지 알야하 하기 때문)

테스트부터 시작

**In `lists/tests/test_form.py`**

```python
def test_form_save_handles_saving_to_a_list(self):
    form = ItemForm(data={'text':'do me'})
    new_item = form.save()
```

하지만 아이템이 어떤 목록(lists)에 속해있지 않기 때문에 (외래키가 없음)에러가 발생한다.

```
django.db.utils.IntegrityError: NOT NULL constraint failed: lists_item.list_id
```

그러면 우리는 form으로 만들어진 item이 어떤 list에 속해야 하는지 정해줘야할 필요가 있다.

**In `lists/tests/test_form.py`**

```python
def test_form_save_handles_saving_to_a_list(self):
	list_ = List.objects.create()
    form = ItemForm(data={'text':'do me'})
    new_item = form.save(for_list=list_)
    self.assertEqual(new_item, Item.objects.first())
    self.assertEqual(new_item.text, 'do me')
    self.assertEqual(new_item.list, list_)
```

`for_list`라는 변수를 따로 정의해주자

**In `lists/form.py`**

```python
def save(self, for_list):
	self.instance.list = for_list
	return super().save()
```

`.instance`는 데이터베이스 객체를 나타낸다.    
수동으로 객체를 생성하는 방법, `commit=False`를 이용해서 저장하는 방법 등    






