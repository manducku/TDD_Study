#사용자 입력 저장하기

> 목표: 사용자 입력 아이템을 서버로 보내 저장하고 다시 사용자에게 보여주기

###POST 요청을 전송하기 위한 폼(Form)연동

```{.html}
# lists/templates/home.html 

<h1>Your To-Do list</h1>
<form method="POST">
	<input name="item_text" id="id_new_item" placeholder="작업 아이템 입력">
</form>

<table id="id_list_table"></table>
```
이후 FT를 실행하면 예측하지 못한 에러가 뜬다. 몇가지 디버깅 방법을 제시한다.
 - print문을 사용해서 현재 페이지 텍스트 등을 확인한다.
 - 에러메시지를 개선해서 더 자세한 정보를 출력하도록 한다.
 - 수동으로 사이트를 열어본다.
 - time.sleep을 이용해서 실행중에 있는 테스트를 잠시 정지시킨다. 개인적으로 time.sleep보다는 Ipython의 embed로 멈추는게 더 좋아보인다.

```{.python}
# funtional_test.py 

# 엔터키를 치면 페이지가 갱신되고 작업 목록에 "1: 공작깃털 사기" 아이템이 추가된다.
input.send_keys(Keys.Enter)

from Ipython import embed; embed()
table = self.browser.find_element_by_id('id_list_table')
```
FT를 다시 실행하면 CSRF 에러를 포여주는 장고 디버깅 페이지가 나타난다.

form 엘리먼트 안에 `{% csrf_token %}`을 넣어주면 해결된다.

---
###서버에서 POST 요청 처리

이전에 테스트에서 다시 FT를 실행하면 오류가 날 것이다. 이는 아직 서버와 POST요청이 연계되지 않았기 때문이다. 
이를 위해서 home_page 뷰에 적용할 새로운 유닛테스트를 만들어야한다.(TDD 이므로) 이 테스트는 POST 요청처리와
반환된 html이 신규 아이템 텍스트를 포함하고 있는지 확인하는 테스트이다.
```{.python}
# lists/tests.py 

...

def test_home_page_can_save_a_POST_request(self):
	request = HttpRequest()
	request.method = 'POST'
	request.POST['item_text'] = '신규 작업 아이템'

	response = home_page(request)

	self.assertIn('신규 작업 아이템', response.content.decode())
```

다시 UT를 실행하면 아직 html에 '신규 작업 아이템'이 없다고 한다. 이를위해 뷰에서 분기처리를 한다.
```{.python}
def home_page(request):
	if request.method == 'POST':
		return HttpResponse(request.POST['item_text'])
	return render(request. 'home.html')
```

---
###파이썬 변수를 전달해서 템플릿에 출력하기

테스트를 통과했지만 템플릿에 있는 테이블에 데이터를 추가하는 것이 목표이므로 추가 작업이 필요하다.
{{ ... }} 이 템플릿 구문으로 파이썬 변수를 html로 전달해 줄 수 있다.
```{.html}
// lists/templates/home.html

<table id="id_list_table">{{ new_item_text }}</table>
```
```{.python}
# lists/tests.py 
self.assertIn('신규 작업 아이템', response.content.decode())
expected_html = render_to_string(
	'home.html',
	{'new_item_text': '신규 작업 아이템'},
	request=request
)
self.assertEqual(response.content.decode(), expected_html)
```
render_to_string 메서드이 두번째 인자로 변수명과 값을 매칭시키고 있다. request에 request를 대입하여 csrf_token필드로 인한 오류를 방지한다.
이제 뷰에서 파이썬 변수를 처리해주어야 한다.

```{.python}
def home_page(request):
	return render(
		request,
		'home.html',
		{
			'new_item_text': request.POST.get('item_text', '')
		}
	)
```	
이제 단위테스트를 통과했다. 기능테스트를 실행하면 오류가 뜬다.
`AssertionError: False is not true: 신규 작업 아이템이 테이블에 표시되지 않는다`

```{.python}
# funtional_test.py 

self.assertTrue(
	'1: 공작깃털 사기', [row.text for row in rows]
)
```
편하게도 위 코드로 `AssertionError: '1: 공작깃털 사기' not found in ['공작깃털 사기']`
라는 메시지를 덤으로 얻는다.

> [any와 all메서드 알아보기](http://egloos.zum.com/mcchae/v/11192451)

지금의 문제는 FT가 "1:"로 시작하는 첫 번째 아이템을 요구한다는 것이다. 이를 위해 하드코딩을 먼저 실행한다.

```{.html}
<tr><td>1: {{ new_item_text }}</td></tr>
```
냄새나는 코드지만 일단 테스트는 통과한다. 하지만 두 번째 아이템을 테이블에 추가하는 테스트를 하면 바로 실패하게 된다.

---
###스트라이크 세 개면 리팩터

딱 봐도 많은 문제가 야기될 코드다. 헬퍼 함수를 만들자.
```{.python}
def check_for_in_list_table(self, row_text):
	table = self.browser.find_element_by_id('id_list_table')
	rows = table.find_elements_by_tag_name('tr')
	self.assertIn(row_text, [row.text for row in rows])
```

이후 FT를 리팩토링하고 테스트를 실행해도 아직 오류가 난다. 이제 데이터베이스를 건드릴 차례다.

---
###Django ORM과 첫 모델

lists/tests.py에 새로운 클래스를 추가한다.

```{.python}
from lists.models import Item

# code...

class ItemModelTest(TestCase):

	def test_saving_and_retrieving_items(self):
		first_item = Item()
		first_item.text = '첫 번째 아이템'
		first_item.save()

		second_item = Item()
		second_item.text = '두 번째 아이템'
		second_item.save()	

		saved_items = Item.objects.all()
		self.assertEqual(saved_items.count(). 2)

		first_saved_item = saved_item[0]
		second_saved_item = saved_item[1]
		self.assertEqual(first_saved_item.text, '첫 번째 아이템')
		self.assertEqual(second_saved_item.text, '두 번째 아이템')
```
물론 에러가 난다. 오류를 확인 했으니 이제 Item 모델을 만든다.( _이제 적응할 때도 됐다_ )
이후 makemigrations를 하고 UT를 돌린다. 테스트가 많이 진전된 것을 볼 수 있다.

---
###POST를 데이터베이스에 저장하기

이후 모델과 연동된 유닛테스트를 추가한다. 그런데 다른 테스트보다 좀 길다. 이는 더 쪼갤 수 있다는 뜻이다.
> 코드 냄새: POST 테스트가 너무 긴가?

테스트로 돌아와서 실행을 하면 `AssertionError: 0 != 1` 에러를 뿜는다. 뷰를 조정한다.
하지만 책에 나온 코드는 굉장히 무식하다. 테스트는 통과하지만...

지금까지 쌓인 께름칙한 코드를 정리하여 TO-DO 리스트를 만든다.
 - 모든 요청에 대한 비어 있는 요청은 저장하지 않는다.
 - 코드 냄새: POST 테스트가 너무 긴가?
 - 테이블에 아이템 여러 개 표시하기
 - 하나 이상의 목록 지원하기

첫번째 항목부터 실행한다. 유닛 테스트를 손본다.
```{.python}
# lists/test.py 

class HomePageTest(TestCase):
	
	# code...

	def test_home_page_only_saves_items_when_necessary(self):
		request = HttpRequest()
		home_page(request)
		self.assertEqual(Item.objects.count(), 0)
```
현재 뷰는 요청이 오면 무조건 객체를 생성하므로 테스트가 실패한다. 뷰를 다시 손본다.
```{.python}
# lists.views.py 

def home_page(request):
	if request.method == 'POST':
		new_item_text = request.POST.get('item_text')
		Item.objects.create(text=new_item_text)
	else:
		new_item_text = ''

	return render(
		request,
		'home.html',
		{
			'new_item_text': new_item_text,
		}
	)
```

---
###POST 후에 리디렉션

'new_item_text'의 중복이 우릴 거슬리게 만든다. 이것을 리다이렉션으로 해결 할 수 있다.
우선 POST요청의 렌더링 비교하는 어설션을 지우고 응답이 리다이렉션으로 홈으로 왔는지 아닌지 확인한다.
물론 뷰에서 리다이렉션 처리를 해주지 않았기 때문에 오류가 난다.(익숙해 지자) 예상된 오류이므로 뷰를 수정한다.

현재 test_home_page_can_save_a_POST_request 테스트가 두가지 테스트를 하고있다. 나누어 준다.
이제 6개의 테스트가 통과한다. TO-DO 리스트에서 1,2번 항목을 지워도 된다.

---
###템플릿에 있는 아이템 렌더링

나머지 TO-DO리스트를 해결 해야한다. 예상가능한 테스트를 먼저 작성한다. 
```{.python}
# lists/tests.py

class HomePageTest(TestCase):

	# code...

	def test_home_page_displays_all_list_items(self):
		Item.objects.create(text='item 1')
		Item.objects.create(text='item 2')

		request = HttpRequest()
		responet = home_page(request)

		self.assertIn('item 1', responet.content.decode())
		self.assertIn('item 2', responet.content.decode())	
```
다행히 예상했던 오류가 뜰것이다. 템플릿을 수정한다.
```{.html}
<table id="id_list_table">
	{% for item in items %}
		<tr><td>1: {{ item.text }}</td></tr>
	{% endfor %}
</table>
```
items에 넣어줄 변수를 뷰에서 전달해 주어야 한다. `items = Item.objects.all()`에 추가하고 render메서드 마지막 인자로 추가한다.

유닛테스트는 모두 통과한다. 그렇다면 기능테스트는 어떨까? 불행히도 실패한다. 오류메시지만으로 잘 알수가 없는 오류다. 다행히 실제 브라우저로 접속해보면 장고가 친절히 설명해줄것이다.

---
###마이그레이션을 이용한 운영 데이터베이스 생성하기

유닛테스트를 할 때 임시의 데이터베이스가 만들어지고 사라진다. 테스트 실행시 -v2를 붙이고 로그를 읽어보면 알 수 있다. 기능테스트는 실제 브라우저로 들어가기 때문에 실제 데이터 베이스가 필요하다.

`$ python manage.py migrate`

다시 기능테스트를 실행하면 아깝게도 아이템 번호의 차이로 실패한다.

```{.html}
{% for item in items %}
	<tr><td>{{ forloop.counter }}: {{ item.text }}</td></tr>
{% endfor %}
```

이전 기록이 남아서 약간 이상하지만 `$ rm db.sqlite3`로 데이터베이스를 지우고 다시 migrate한다. 약간의 버그가 있지만 FT를 마쳤다.

####알아두면 유용한 TDD 개념

 - **퇴행 :** 동작하고 있던 로직이 추가된 코드에 의해 망가지는 것
 - **예상치 못한 실패 :** 테스트가 잘못 작성됐거나 코드 퇴행을 발견했다는 것, 혹은 애플리케이션 코드 수정이 필요하다는 의미	
 - **Red/Green/Refactor :** 테스트 작성 후 예상된 실패 관측(레드), 코드 수정으로 통과 유도(그린), 리팩터로 코드 개선
 - **삼각법 :** 기존 코드에 구체적인 테스트 케이스를 추가해서 일반화(편법이 될 수도 있는)한 처리를 정당화
 - **스트라이크 세 개면 리팩터 :** 코드 중복이 세번째 일어나면 리팩토링한다.
 - **작업 메모장 :** 코딩 중 해야할 일을 기록해두는 곳. 현재 작업하고 있는 것이나 마저 못한 작업을 나중에라도 끝낼 수 있다.