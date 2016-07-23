# 입력 유효성 검사 및 테스트 구조화

> 이후에는 몇 장에 걸쳐서 사용자 입력 테스트와 유효성 검증에 대해 다룬다

### FT 유효성 검사: 빈 작업 아이템 방지

새로운 사용자 시나리오 작성과, 테스트 코드 분리를 시작한다.

#### 테스트 건너뛰기

일단 테스트가  모두 통과된 상태에서 리팩터링을 권장한다. `skip` 데코레이터로 새로운 테스트 함수를
잠시 꺼둔다다

```{.python}
#functional_test/tests.py

from unittest import skip

...

@skip
def test_cannot_add_empty_list_items(self):
```

**"레드, 그린, 리팩터"의 리팩터를 잊지 말자**
 - TDD가 문제 되는 경우는, 오로지 테스트가 통과되는 것에만 초점을 맟추기 때문이다
 - "TDD는 만병통치 약이 아니다" 설계적 측면을 항상 생각해야 한다.
 - `레드,그린,리팩터`는 테스트 통과가 아니라 설계를 개선하기 위한 리팩터링에 시간을 투자해야 함을 의미한다
 - 최적의 리팩터링 방식은 매번 떠오르지 않는다. 이미 작업이 진행된 와중 좋은 아이디어가 떠오를 수도 있다.
 - 동작 상태가 보장되지 않는 코드 변경의 경우도 생길 수있다. 이땐 작업 메모장에 이후 해야할 작업으로 기록해두고, 동작 상태가 될 때까지 기다렸다가 리팩터링하는 것이 좋다

####기능 테스트를 여러 파일로 분할하기

각 테스트를 개별 클래스로 나눈다. 아직은 하나의 파일 내에 존재한다.

```{.python}
# functional_test/tests.py

class FunctionalTest(StaticLiveServerCase):

	@classmethod
	def setUpClass(cls):

	.
	.
	.

class NewVisitorTest(FunctionalTest):

	deftest_can_start_a_list_and_retrieve_it_later(self):
	.
	.
	.
```

나눈 이후 테스트를 통과하는지 확인한다. 통과한다면 다음으로 base.py를 만들고 클래스별로 파일을 나눈후 base.py의 기본클래스를 상속한다. 물론 나눈 이후 다시 기능테스트 결과를 확인한다. 모두 통과했다면 skip을 제거한다.

####개별 테스트 파일 실행

커밋

####FT에 살 붙이기

이제서야 테스트코드를 작성한다. 이후 테스트를 실행하면 실패한다

---

###모델-레이어 유효성 검증

Django에선 모델과 상위 폼 계층에서 유효성 검증을 할 수 있다. 저자는 모델 계층을 선호한다.
 - 데이터베이스 무결성 규칙 부분을 직접 테스트할 수 있다.
 - 더 안전하다.

####단위 테스트를 여러 개 파일로 리팩터링하기

뷰와 모델 테스트로 파일을 나눈 후, 유닛테스트를 확인한다.

####단위 테스트 모델 유효성 검증과 self.assertRaises 컨텍스트 관리자

```{.python}
# lists/tests/test_models.py

from django.core.exceptions import ValidationError

class ListAndItemModelsTest(TestCase):
	
	...

	def test_cannot_save_empty_list_items(self):
		list_ = List.objects.create()
		item = Item(list=list_, test='')
		with self.assertRaises(ValidationError):
			item.save()
```

위 테스트에서 `save()`메서드가 ValidationError를 일으켜야 한다. 하지만 
```
empty_list_items
    item.save()
AssertionError: ValidationError not raised
```
이처럼 ValidationError가 일어나지 않았다고 한다.

####수상한 Django: 모델 저장은 유효성 검사가 되지 않는다

위 테스트가 실패한 이유는 무엇일까. TextField가 `blank=False`임에도 말이다.

책에서 설명하듯 직관적이지 못한 이유이긴 하지만, Django 모델이 저장 처리에 대해선 유효성 검사를 못하기 때문이다.

하지만 이는 sqlite의 경우로, 텍스트 컬럼에 null 제약을 강제적으로 뷰여할 수 없기 때문에, save메서드가 빈값을 그냥 통과 시키는 것이다. 

이를 위해 Django에서 유효성 검증을 수동으로 해주는 메서드가 있다

```{.python}
# lists/tests/test_models.py

with self.assertRaises(ValidationError):
	item.save()
	item.full_clean()
```

이로써 테스트가 성공한다. 

208페이지 마지막줄..?????

---

###뷰를 통한 모델유효성 검증

뷰 계층에서 유효성 검증을 해보자. HTML을 통해 에러를 표시 한다.

```{.html}
<!-- lists/templates/base.html -->

<form>
	<input type="text" id="id_new_item"
		class="form-control input-lg" 
		name="item_text" placeholder="작업 아이템 입력">
	{% csrf_token %}
	{% if error %}
		<div class="form-group has-error">
			<span class="help-block">{{ error }}</span>
		</div>
	{% endif %}
</form>
```

에러를 템플릿으로 전달하는 것은 뷰 함수가 담당한다. 당연히 단위테스트부터 시작이다.
두가지 패턴을 사용 할 것이다.

```{.python}
# lists/tests/test_views.py

def test_validation_errors_are_sent_back_to_home_page_template(self):
	response = self.client.post('/list/new', data={'item_text': ''})
	self.assertEqual(response.status_code, 200)
	self.assertTemplateUsed(response, 'home.html')
	expected_error = "You can't have an empty list item"
	self.assertContains(response, expected_error)
```

 - URL을 하드코딩하는 것을 경고한다. 작업 메모장에 이것을 리팩터링하도록 기록한다.
 - `new_list`뷰에서 full_clean을 호출한다.(해당 뷰에서도 하드코딩된 url을 볼 수있다. 기록해두자)

앞서 말한 두가지 패턴중 하나, try/except를 이용해 에러를 잡는 시도를 해본다.

```{.python}
# lists/views.py

from django.core.exceptions import ValidationError

def new_list(request):
	list_ = List.objects.create()
	item = Item.objects.create(text=request.POST['item_text'], list=list_)
	try:
		item.full_clean()
	except ValidationError:
		pass
	return redirect('/lists/{list_id}/'.format(list_id=list_.id,))
# 이코드는 다시 302 != 200 결과를 보여준다
```

렌더링된 템플릿을 반환해서 템플릿 체크

```{.python}
# lists/views.py

except ValidationError:
	return render(request, 'home.html')

# 테스트를 다시 실행하면 response에서 오류가 날것이다.
# 새로운 템플릿 변수를 이용한다

except ValidationError:
	error = "You can't have an empty list item"
	return render(request, 'home.html', {"error": error})
```

하지만 제대로 동작하지 않는다. 무엇이 문제일지 `print`를 이용해 디버깅해본다

```{.python}
# lists/tests/test_views.py

expected_error = "You can't have an empty list item"
print(response.content.decode())
self.assertContains(response, expected_error)
```

이스케이프 문제 `You can't ...`문제로 테스트를 통과하지 못한다.

```python
from django.utils.html import escape

...

expected_error = escape("You can't have and empty list item")
```

####데이터베이스에 잘못된 데이터가 저장됐는지 확인

앞선 과정 중 무언가 이상한 부분이 있었다. view에서 유효성 검사에 실패해도 객체를 생성하는 부분이다.

이에 새로운 단위 테스트를 추가해서 빈 아이템이 저장되지 않도록 한다.

```python
# lists/tests/test_views.py

class NewListTest(TestCase):
	...

	def test_invalide_list_items_arent_saved(self):
		self.clientpost('/lists/new/', data={'item_text': ''})
		self.assertEqual(List.objects.count(), 0)
		self.assertEqual(Item.objects.count(), 0)
```

우려한대로 `AssertionError: 1 != 0` 실패한다.(url 마지막 '/'슬래시를 주의하자!) 뷰를 수정한다

```python
# lists/views.py

def new_list(request):
	list_ = List.objects.create()
	item = Item(text=request.POST['item_text'], list=list_)
	try:
		item.full_clean()
		item.save()
	except ValidationError:
		list_.delete()
		error = "You can't have an empty list item"
		return render(request, 'home.html', {"error": error})
	return redirect('lists/{list_id}/'.format(list_id=list_.id))
```

FT를 실행한다 `$python manage.py test functional_test.test_list_item_validation`

비록 실패하지만 진전이 있다고 말한다. 로그를 보면 첫 번째 테스트는 통과했지만 두 번째 테스트에서 실패한 것이고, 이는 두 번째 빈 아이템 등록에 대한 테스트이기 때문이다. 여기서 저자는 커밋을 주문한다.

-

###Django 패턴: 폼 렌더링 뷰와 같은 뷰에서 POST 요청 처리

아직은 무슨 이유인지 모르지만 약간 다른 접근법을 이용한다. 허나 이 방법이 Django의 일반적인 방법이라 말한다. 폼 렌더링에 사용하는 뷰를 사용해 POST 요청을 처리하는 것이다. RESTful 하지 않지만, 동일 URL을 이용해서 폼뿐만 아니라 사용자 입력 처리 시 발생하는 에러도 출력한다는 큰 이점이 있다.

```python
# lists/templates/list.html

{% block form_action %}/lists/{{ list.id }}/{% endblock %}
```

이로써 하드코딩된 url은 2개! `/lists/new`와 `/lists/{{ list.id }}`

> **메모장**
> - views.py 에서 하드코딩된 url 제거
> - list.html과 home.html의 폼에서 하드코딩된 url 제거

이는 원래 기능 테스트를 망가뜨린다. view_list 페이지가 POST 요청을 처리하지 못하기 때문이다

기능테스트로 확인해 볼 수있다.

####리팩터링: new_item 기능을 view_list로 옮기기

POST 요청을 저장하는 NewItemTest의 모든 테스트를 ListViewTest로 옮긴다. 이를 통해 .../add_item이 아닌 기본 모곡 url을 가리키도록 만든다

그런이후 view_list 뷰에서 POST요청을 처리하는 코드를 작성한다

```python
# lists/views.py

def view_list(request, list_id):
	list_ = List.objects.get(id=list_id)
	if request.method == 'POST':
		Item.objects.create(text=request.POST['item_text'], list=list_)
		return redirect('/lists/{list_id}'.format(list_id=list_.id))
	return render(request, 'list.html', {'list': list_})
```

이제 `add_item`관련 url, view 코드는 전부 삭제한다. 이후 FT를 실행하고, 204페이지의 상황으로 돌아간것을 볼 수 있다.

**_짚고 넘어가기_**

"실패 테스트 상태에서는 절대 리팩터링하지 않는다"는 규칙이 기억나는가? 우리는 그 규칙을 깼다. 하지만 이번 경우 신규 기능이 동작하려면 리팩터링이 필요했기 때문에 용납이 된다. 하지만 책에서 제한을 두었다. 단위테스트가 실패하는 상태에선 절대 리팩터링을 하면 안되지만, 기능테스트에 한해 허용한다. 클린테스트를 선호한다면 skip 데코레이션을 써도 된다.

####view_list에서 모델 유효성 검증 구현

모델 검증 규칙에 맞추어 기존 리스트에 아이템을 추가하는 처리가 필요하다. 이를 위한 새로운 단위 테스트를 작성한다.

```python
def test_validation_errors_end_up_on_lists_page(self):
	list_ = List.obejcts.create()
	response = self.client.post(
		'/lists/{list_id}/'.format(list_id=list_.id),
		data={'item_text': ''}
	)
	self.assertEqual(response.status_code, 200)
	self.assertTemplateUsed(response, 'list.html')
	expected_error = escape("You can't have na empty list item")
	self.assertContains(response, expected_error)
```

> 슬래시 오류를 잊지 말자!

아직은 뷰가 모든 POST 요청을 유효성 검증 없이 리다이렉트 하기 때문에 에러가 발생한다.

```python
def view_list(request, list_id):
	list_ = List.objects.get(id=list_id)
	error = None

	if request.method == 'POST'
		try:
			item = Item(text=request.POST['item_text'], list=list_)
			item.full_clean()
			item.save()
			return redirect('/lists/{list_id}/'.format(list_id=list_.id))
		except ValidationError:
			error = "You can't have na empty list item"

	return render(request, 'list.html', { 'list': list_, 'error': error })
```

하지만 두 뷰가 try/except 를 사용하여 투박한 느낌이든다. 개인적으로도 별로 좋아하지 않는 구문

리팩터링을 미뤄두고 메모장에 추가한다

> **메모장**
> - views.py에서 하드코딩된 url 제거
> - list.html과 home.html의 폼에서 하드코딩된 url 제거
> - 뷰에 있는 중복된 검증 로직 제거

-

###리팩터링: 하드코딩된 URL 제거

`urls.py`에 있는 name을 사용한다. 각 폼의 action 등록부분을 {% url 'url_name' %}을 이용하여 리팩터링한다.

물론 이후 테스트 통과-

####get_absolute_url을 이용한 리다이렉션

```python
# lists/views.py

def new_list(request):
	[...]
	return redirect('view_list', list_.id)
```

장고를 처음 접할 때 힘든점은 암묵적으로 모델과 연결된 url 설정이다. `get_absolute_url`을 `reverse`로 오버라이딩하면 뷰에 인자를 넘길 수 있게된다.

```python
# lists/models.py

from django.core.urlresolvers import reverse

class List(models.Model):
	
	def get_absolute_url(self):
		return reverse('view_list', args=[self.id])
```

> 메모장
> - ~~views.py에서 하드코딩된 url 제거~~
> - ~~list.html과 home.html의 폼에서 하드코딩된 url 제거~~
> - 뷰에 있는 중복된 검증 로직 제거

####테스트 구조화와 리팩터링 팁

**테스트 폴더 사용**
 - tests라는 폴더를 사용한다
 - 기능 테스트를 위해, 기능이나 사용자 스토리 단위로 테스트를 그룹화한다.
 - 단위 테스트를 위해, 테스트 대상 코드 파일별로 별도 테스트 파일을 만든다. Django에선 일반적으로 `test_models.py, test_views.py, test_forms.py`
 - 적어도 하나의 기본 테스트를 만들어서 다른 함수와 클래스가 이를 사용하도록 한다.

**"레드, 그린, 리팩터의 리팩터"를 잊지 말자**

테스트를 하는 이유는 코드를 리팩터링 하기 위해서이다. 테스트를 이용해서 가능한 간결한 코드를 만들도록 한다.

**실패 테스트 상태에서 리팩터링하지 않는다.**
 - 대부분의 경우 옳다
 - 하지만 현재 작업하고있는 FT는 해당되지 않는다
 - 아직 작성하지 않은 코드를 테스트하기 위해 "skip"을 사용할 수 있다.
 - 일단 리팩터링이 필요한 것들을 기록해 두고 진행 중인 작업을 계속한다. 동작 상태가 되면 그때 리팩터링한다.
 - 코드 커밋 시에는 반드시 skip을 제거하도록 한다. 이런 상태를 파악하기 위해서는 diff를 이용해서 변경 내역을 줄 단위로 검토해야 한다.