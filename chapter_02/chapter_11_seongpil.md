#간단한 폼

> Django는 사용자 입력 검증과 검증 에러 메시지 출력을 위해 `Form` 클래스 사용을 권장한다

###유효성 검증 로직을 폼으로 옮기기

Django 폼의 기능
 - 사용자 입력을 처리하고 검증하여 에러로 출력할 수 있다
 - HTML입력 요소를 표시하기 위한 템플릿으로 사용할 수 있으며, 에러 메시지도 제공한다
 - 일부 폼은 데이터를 데이터베이스에 저장할 수도 있다

####단위 테스트를 통한 폼 API 알아보기

책을 따라하면 무언가 이상한 점을 느꼈을 것이다. 우리는 분명 여러가지 테스트를 먼저 실패하고 코드를 작성해왔는데,
이부분은 금방금방 기능코드를 작성해 버린다. 하지만 때에 따라서는 이 방식도 문제가 없다고 얘기한다.

 > 새로운 API를 사용하기 시작할 때는 정식 TDD 단계에 들어가기 전까지 여러가지 시도를 해볼 수있다.
이때는 인터액티브 콘솔을 이용하거나 예비 코드를 작성하는 것도 허용된다.(하지만 이런 임시 코드는 모두 폐기하고 이후에 정식으로 재작성해야 한다.)

####Django ModelForm으로 변경

```python
# lists/forms.py

from django import forms.py

from lists.models import Item


# ModelForm은 다른 타입 필드에 HTML 폼의 입력 타입을 할당한다거나
# 기본 검증 로직을 적용하는 등 다양한 처리가 가능하다
class ItemForm(forms.models.ModelForm):

	# Meta에서 폼이 어떤 모델, 필드를 이용할지 정의
	class Meta:
		model = Itemfields = ('text',)
```

이후 테스트를 하면 placeholder와 CSS 클래스가 빠진 것을 알 수 있는데 아래와 같이 처리한다

```python
class ItemForm(forms.models.ModelForm):
	
	class Meta:
		model = Item
		fields = ('text',)
		widgets = {
			'text': forms.fields.TextInput(attrs={
				'placeholder': '작업 아이템 입력',
				'class': 'form-control input-lg',
				}),
		}
```

####폼 유효성 검증 커스터마이징 및 테스트

```python
# lists/tests/test_forms.py

def test_form_validation_for_blank_items(self):
	form = ItemForm(data={'text': ''})
	form.save()
```

과연 ModelForm이 모델에 정의한 것과 같은 검증 규칙을 사용하는지 확인한다. 테스트 실패로 이를 확인한다.

이제 특정 에러 메시지를 지정할 수 있는지 보도록 하자. 데이터를 저장하기 전에 폼 유효성을 검증하는 is_vaild라는 함수다.

```python
# lists/tests/test_forms.py
def test_form_validation_for_blank_items(self):
	form = ItemForm(data={'text': ''})
	self.assertFalse(form.is_valid())
	self.assertEqual(
		form.errors['text'],
		["You can't have an empty list item"]
	)
```

form.is_valid()를 호출하면 `True`나 `False`를 반환한다. 하지만 이 기능 역시 부작용이 있다. 필드명과 에러를 매칭시키는 딕셔너리 부분이 문제다.( 하나의 필드가 복수의 에러 메시지를 가질 수 있다) 장고는 에러 메시지를 가지고 있다. `error_messages`라는 Meta 변수를 커스터마이징 하면 된다.

-

###뷰에서 폼 이용하기

> 빈 아이템 검증, 고유성 검증..."가능한 빨리 배포하라, 빨리 코드를 머지하라"와 배치된다.

####뷰에서 GET 요청과 함께 폼 이용하기

홈 뷰 단위 테스트부터 시작.

 -`test_home_page_returns_correct_html`
 -`test_root_url_resolves_to_home_page_view`

두 뷰는 `HttpRequest`를 통해 실행되는 테스트이다. 이를 Django의 테스트 클라이언트로 변경한다. 신규 테스트 결과가 이전과 같은지 비교하기 위해 이전 테스트를 남겨둔다.

```python
# lists/tests/test_views.py

def test_home_page_returns_correct_html(self):
	response = self.client.get('/')
	# assertTemplateUsed 라는 헬퍼 메서드 사용
	self.assertTemplateUsed(response, 'home.html')

def test_home_page_uses_itemform(self):
	response = self.client.get('/')
	# 뷰가 적합한 폼을 사용하는지 확인
	self.assertIsInstance(response.context['form'], ItemForm)
```

테스트 결과 `KeyError: 'form'`이 뜬다. 이것보고 home_page뷰에 템플릿 변수 생각을 할순 없었지만
넣어준다.

```python
# lists/views.py

def home_page(request):
	return render(
		request,
		'home.html',
		{'form': ItemForm()}
	)
```

템플릿에서는 `form`변수를 이용한다

```html
<form method="POST" action="{% block form_action %}{% endblock %}">
	{{ form.text }}
	{% csrf_token %}
	{% if error %}
	....
```

이후 `HomePageTest`의 두 테스트에서 에러가 나야한다. 하지만 나지 않는다. 버전 문제라 추측한다. 일단 책 내용을 따라간다.
신규테스트만을 남기고 이전 테스트는 주석 처리했다.

####찾아서 고치기

FT를 진행하면 오류가 뜬다. 폼이 변경되면서 id와 name을 찾을 수 없기 때문이다. 이를 위해 '찾아서 고치기' 작업이 많이 필요하다.

`$ grep id_new_item functional_tests/test*`

이로써 `id_new_item`를 사용하고 있는 부분을 찾았다. 이는 리팩터링을 위한 좋은 정보가 된다. `base.py`에 새로운 헬퍼 메서드를 작성하자

```python
# functional_tests/base.py

class FunctionalTest(StaticLiveServer):
	[...]
	def get_item_input_box(self):
		return self.browser.find_element_by_id('id_text')
```

`id`와 `name`이 바뀌었고, 헬퍼 메서드도 작성했다. 적용된 모든 곳을 수정해야 한다. 앞선 명령어와

 - `$ grep -r id_new_item lists/`
 - `$ grep -Ir item_text lists`

두 명령어로 찾아가며 수정한다. 이후 단위 테스트를 확인하고, 기능 테스트도 실행한다. 기능테스트는 실패한다.

`new_list` 뷰에서 유효성 검증 후 폼을 전달해 주지 못해 페이지에 입력할 수 있는 폼이 없기 때문이다. 커밋 후 진행한다.

-

###POST 요청을 받는 뷰에서 폼 사용하기

`new_list`를 위한 단위 테스트를 수정해야 한다.

####new_list뷰를 위한 단위 테스트 수정

`test validation_errors_are_sent_back_to_home_page_template` 테스트는 한번에 너무 많은 테스트를 하고 있다. 이를 단순화 하고, 두 가지 다른 어설션으로 분리하도록 한다.

 - 검증 오류가 있으면 200 코드와 함께 home 템플릿을 축력한다.
 - 검증 오류가 있으면 응답이 에러 메시지를 포함해야 한다.

또한 신규 어설션도 추가한다.

 - 검증 오류가 있으면 폼 객체를 템플릿에 전달한다.

그리고 하드코딩된 문자열로 에러를 표시하는 대신에 상수를 이용하도록 한다.

```python
from lists.forms import ItemForm, EMPTY_LIST_ERROR
[...]

class NewListTest(TestCase):
	[...]

	def test_for_invalid_input_renders_home_template(self):
		response = self.client.post('/lists/new', data={'text': ''})
		self.assertEqual(response.status_code, 200)
		self.assertTemplateUsed(response, 'home.html')

	def test_validation_errors_are_shown_on_home_page(self):
		response = self.client.post('/lists/new', data={'text': ''})
		self.assertContains(response, escape(EMPTY_LIST_ERROR))

	def test_for_invalid_input_passes_form_to_template(self):
		response = self.client.post('/lists/new', data={'text': ''})
		self.assertIsInstance(response.context['form'], ItemForm)
```

####뷰에서 폼 사용하기

```python
# list/views.py

def new_list(request):
	form = ItemForm(data=request.POST)
	if form.is_valid():
		list_ = List.objects.create()
		Item.objects.create(text=request.POST['text'], list=list_)
		return redirect(list_)
	else:
		return render(request, 'home.html', {'form': form})
```

####폼을 이용해서 템플릿에 에러 메시지 출력하기

템플릿에서 에러 메시지를 출력하기 위한 폼이 아직 적용돼지 않아 테스트가 실패한다.

```html
# lists/tmplates/base.html

<form method="POST" action="{% block form_action %}{% endblock %}">
	{{ form.text }}
	{% csrf_token %}
	{% if form.errors %}
		<div class="form-group has-error">
			<div class="help-block">{{ form.text.errors }}</div>
		</div>
	{% endif %}
</form>
```

예상치 못한 실패가 발생한다. 이는 `view_list` 때문에 일어났다. 모든 템플릿에서 에러 메시지 출력 방식을 form 형식으로 변경했기 때문에, 템플릿 변수로 직접 전달한 에러 메시지는 더이상 표시하지 못하기 때문이다. `view_list`뷰도 수정해야한다는 뜻이다.

-

###다른 뷰에 폼 적용하기

현재 `view list`뷰는 GET, POST 요청을 처리하고 있다. 우선 GET 요청에 사용된 폼부터 확인해보자. 새로운 테스트를 만든다.

```python
# lists/tests/test_views.py

class ListViewTest(TestCase):
	[...]

	def test_displays_item_form(self):
		list_ = List.objects.create()
		response = self.client.get('/lists/{list_id}/'format(list_id=list_.id))
		self.assertIsInstance(response.context['form'], ItemForm)
		self.assertContains(response, 'name="text"')
```

결과는 `KeyError: 'form'`

이를위해 최소한의 기능 구현을 한다.

```python
# lists/views.py

def view_list(request, list_id):
	[...]

	form = ItemForm()
	return render(request, 'list.html', {
		'list': list_,
		'form': form,
		''error': error,
	})
```

####짧은 테스트들을 위한 헬퍼 메서드

이제 두 번째 뷰에서 폼 에러를 이용할 수 있게 해야 한다. 잘못된 입력 데이터용 단일 테스트를 몇 개의 테스트로 나누도록 한다.

```python
# lists/tests/test_views.py

class ListViewTest(TestCase):
	[...]

	# test로 시작하는 test 함수가 아닌 헬퍼 함수
	def post_invalid_input(self):
	list_ = List.objects.create()
	return self.client.post(
		'/lists/{list_id}/'.format(list_id=list_.id),
		data={'text': ''},
	)

	# 이후 각 테스트별로 헬퍼함수를 적용한다
	def test_for_invalid_input_nothing_saved_to_db(self):
		self.post_invalid_input()
		self.assertEqual(Item.objects.count(), 0)

	def test_for_invalid_input_renders_list_template(self):
		response = self.post_invalid_input()
		self.assertEqual(response.status_code, 200)
		self.assertTemplateUsed(response, 'list.html')

	def test_for_invalid_input_passes_form_to_template(self):
		response = self.post_invalid_input()
		self.assertIsInstance(response.context['form'], ItemForm)

	def test_for_invalid_input_shows_error_on_page(self):
		response = self.post_invalid_input()
		self.assertContains(response, escape(EMPTY_LIST_ERROR))
```

헬퍼함수를 통해 하나의 view 테스트를 여러개로 쪼갤 수 있었다. 이로써 발생하는 문제를 정확하게 찾아낼수 있게된다.
아직은 테스트를 해도 에러가 뜬다. 이제 폼을 사용하도록 뷰를 다시 작성한다.

```python
def view_list(request, list_id):
	list_ = List.objects.get(id=list_id)
	form = ItemForm()
	if request.method == 'POST':
		form = ItemForm(data=request.POST)
		if form.is_valid():
			Item.objects.create(text=request.POST.get('text'), list=list_)
			return redirect(list_)
	return render(request, 'list.html', { 'lists': list_, 'form': form})
```

이제 유닛, 기능 테스트를 모두 통과한다. ~~_마음이 편안해진다_~~

> 지금까지 우리는 앱에 큰변화를 주었고 성공했다. name과 ID를 가진 입력 필드는 모든 처리의 핵심이 되기 때문이다.    
> 지금까지 7~8개의 파일을 수정, 리팩터링 했다. 이와 같이 테스트가 없었다면 결과를 보기까지 많은 걱정을 했을 것이다.    
> 적은 기능이지만 많은 테스트로 탄탄한 코드 품질을 낼 수 있었고, 믿음이 생긴다. 좋은 보상이다.

-

###폼 자체 save 메소드 사용

좋은 소식이 있다. 뷰를 더 단순하게 할 수 있다고 저자는 말한다.

```python
# lists/tests/test_forms.py

def test_form_save_handles_saving_to_a_list(self):
	form = ItemForm(data={'text': 'do me'})
	new_item = form.save()
```

Django가 호락호락 하지 않다. 어떤 리스트를 참조하는지 명시하지 않았기 때문이다.

```python
# lists/tests/test_forms.py
from lists.models import List, Item

[...]

def test_form_save_handles_saving_to_a_list(self):
	list_ = List.objects.create()
	form = ItemForm(data={'text': 'do me'})
	new_item = form.save(for_list=list_)
	self.assertEqual(new_item, Item.objects.first())
	self.assertEqual(new_item.text, 'do me')
	self.assertEqual(new_item.list, list_)
```

예상 했듯 폼의 save 메서드에서 에러를 뿜는다. 그리고 책에서는 save 메서드를 오버라이드 한다(좋은 구현인지 의구심이 든다)

```python
# lists/forms.py

def save(self, for_list):
	self.instance.list = for_list
	return super().save()
```

폼의 `instance` 속성은 데이터베이스 객체를 나타내며, 수정 또는 생성을 할 수 있다. 

이후 `new_list`뷰를 리팩터링 한다.

```python
def new_list(request):
	form = ItemForm(data=request.POST)
	if form.is_valid():
		list_ = List.objects.create()
		form.save(for_list=list_)
		return redirect(list_)
	else:
		return render(
			request,
			'home.html',
			{
				"form": form,
			}
		)
```

다음은 `view_list` 차례다.

```python
def view_list(request, list_id):
	list_ = List.objects.create()
	form = ItemForm()
	if request.method == 'POST'
		form = ItemForm(request.POST)
		if form.is_valid():
			form.save(for_list=list_)
			return rediret(list_)
	return render(
		request,
		'list.html',
		{
			'list': list_,
			'form': form
		}
	)
```

---

####가벼운 뷰

뷰가 복잡해지거나 이를 위한 테스트를 작성해야 한다면, 로직 분산을 고려해야한다. 이번 장 처럼 폼이 좋은 대상이 될 수있다.    
혹은 모델 클래스에 있는 사용자 작성 메서드도 좋은 대안이 될 수 있다. 앱의 복잡성에 따라서는 Django 외부 파일이나    
자체 클래스 및 함수에 핵심 로직을 구현하는 방법도있다.

> 스쿨에서 다뤘던 `utils`가 좋은 예가 되겠다.

####하나의 테스트는 한 가지만 테스트해야 한다

경험상 하나의 테스트에 하나의 이상의 어설션이 있다면 불안정 하다. 가끔 두 어설션이 밀접하게 연계돼 있어서 같이 있어야할    
경우도 있다. 하지만 처음 작성한 테스트의 경우 여러 개의 테스트를 내포한 채 마무리 되는 경우가 있다. 이때는 테스트를    
나누어서 재작성하는 것이 좋다. __헬퍼 함수__는 테스트가 비대해지는 것을 방지해 준다.