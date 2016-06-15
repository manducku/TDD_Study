# TDD_Study

1부 6장 자료

---------------------------------------------------------------------------
###기능 테스트 내에서 테스트 격리

기능테스트를 할때마다 이전 테스트의 아이템들이 데이터베이스에 남아있는 문제가 발생함.
단위테스트는 Django내부에서 별도의 DB를 사용해서 테스트가 끝나면 DB가 초기화되는 장점이 있음
그러므로 기능테스트를 Django 내부로 이동시킬 필요가 있다

git mv 명령어를 통해 git이 인식하도록 유도함
```
git mv functional_tests.py functional_tests/tests.py
```

`LiveServerTestCase`클래스를 상속받아 8000포트를 사용하지 않고 `live_server_url`함수를 사용
이로써 `python manage.py`를 통해서 기능 테스트를 실행시킬 수 있다

```
git diff -M
```
명령어는 `이동한 파일을 감지`하는 명령어로 functional_tests.py와 functional_tests/tests.py가 같은 파일이라는 것을
인지한다.


---------------------------------------------------------------------------
###필요한 경우에는 최소한의 설계를

>사용자가 각각의 개별 URL을 갖고 있어서 다른 사용자의 URL은 볼 수 없도록 만들어야함

**YAGNI**
설계에 대해 생각하다 보면 생각을 멈추는 것이 쉽지 않은데 그러한 과한 열정을 자제하기 위해서는
스스로 절제하는 능력을 키워야한다.(You ain't gonna need it)

**REST**
Representational State Transfer 웹 설계 방법 중 하나 (데이터의 구조를 URL에 일치시키는 것)
예를 들어 새로운 목록을 만들기 위해서 /lists/new라는 URL을 사용하여 POST요청을 한다

----------------------------------------------------------------------------

###TDD를 이용한 새로운 설계 반영

아이템을 전송하자마자 새로운 목록을 만들어서 신규 작업 아이템을 추가함

In `functional_tests/tests.py`
```python
inputbox.send_keys(Keys.ENTER)
edith_list_url = self.browser.current_url
self.assertRegex(edith_list_url, '/lists/.+')
self.check_for_row_in_list_table('1:공작깃털 사기')
```

`assertRegex`는 지정한 정규표현과 문자열이 일치하는지 확인해주는 함수

-------------------------------------------------------------------------------

###새로운 설계를 위한 반복

>Regexp의 문제를 갖고있는 FT가 두번째 아이템에 개별 URL과 식별자를 적용하는 문제를 해결

현재 하나의 URL에만 대응시키기 위해 직접 URL을 명시해줌

```python
def home_page(request):
    if request.method == 'POST':
        Item.objects.create(text=request.POST['item_text'])
        return redirect('/lists/the-only-list-in-the-world/')
```

--------------------------------------------------------------------------------
###Django 테스트 클라이언트를 이용해 뷰, 템플릿, URL을 한번에 테스트하기


`TestCase`의 속성인 `self.client`(테스트 클라이언트)를 이용해 테스트할 URL을 get!

```python
class ListViewTest(TestCase):
    def test_display_all_items(self):
    [...]

    response = self.client.get('/lists/.../world/')
    
    [...]
```

이후 urls.py, views.py 에서 해당 테스트에 대한 url경로를 설정해준다.

메인 페이지와 목록 뷰의 기능이 다르기 때문에 home.html, list.html을 구분하는 테스트를 작성

In`lists/tests.py`
```python
class ListViewTest(TestCase):
    def test_uses_list_template(self):
        response = self.clinet.get('/lists/the-only-list-in-the-world/')
        self.assertTemplateUsed(reponse, 'list.html')

    def test_display_all_items(self):
        [...]
```

list.html을 사용하도록 views.py에서 view_list수정

In`lists/views.py`
```python
def view_list(request):
    items = Item.objects.all()
    return render(request,'list.html',{'items':items})
```
--------------------------------------------------------------------------------
###목록 아이템 추가를 위한 URL, View

**주의**
꼬리 슬래시 주의 `/new`, `/new/`의 차이점 `/new`는 데이터베이스에 변경을 가하는 "액션" URL인 경우이다.

In`superlists/urls.py`
```python
urlpatterns = [
    url(r'^admin/', admin.site.urls),

    url(r'^$', home_page, name='home'),

    url(r'^lists/the-only-in-the-world/$', view_list, name='view_list'),
    url(r'^lists/new$', new_list, name='new_list'),
]
```

In`lists/tests.py`
```python
def test_redirects_after_POST(self):
    response = self.client.post(
        '/lists/new',
        date={'item_text': '신규 작업 아이템'}
    )
    self.assertRedirects(reponse, '/lists/the-only-list-in-the-world/')
```
>헬퍼함수 assertRedirects를 사용하지 않고서도 오류가 뜨지 않았음


추가된 아이템을 보는 작업을 list.html이 대신 해주기 때문에 home.html에 있는 불필요한
View, Test에 있는 기능들을 삭제할 수 있다.

In`lists/views.py`

```python
def home_page(request):
    return render(request, 'home.html')
```
`test_home_page_only_saves_items_when_necessary`테스트 삭제


새롭게 만들어낸 `/lists/new` URL을 home.html, views.html form의 action에 적용

---------------------------------------------------------------------------------
###모델 조정

[model test link](https://github.com/deadlylaid/TDD_Test/blob/master/superlists/lists/tests.py)


외래 키관계
In `lists/models.py`
```python
class List(models.Model):
    pass

class Item(models.Model):
    text = models.TextField(default='')
    list = models.TextField(default='')
```

`AssertionError = 'List object' != <List: List object>`
Django가 List객체를 문자열로 해석하여 저장하고 있는 것을 볼 수 있음

이를 해결하기 위해서 `Foriengkey`를 사용해야한다.

In `lists/models.py`
```python
class List(models.Model):
    pass

class Item(models.Model):
    text = models.TextField(default='')
    list = models.TextField(List, default=None)
```

**궁금증**
(p117)이전까지 Item을 list에 종속시키는 작업을 완료했으나 아이템을 만들때마다
목록이 새로 생긴다는 말이 무슨뜻인지 모르겠다

---------------------------------------------------------------------------------
###각 목록(list)이 하나의 고유 URL을 갖아야 한다.
각 목록이 고유의 URL을 갖기 위해선 URL에 기본키를 넣는 것이 가장 효과적이다
그러므로 URL에 `list_.id`를 넣으려 한다.

```python
class ListViewTest(TestCase):
    def test_uses_list_template(self):
        response = self.client.get('/lists/%id/' %(list_.id))
```

404 Error를 해결하기 위해 urls.py에 추가
```python
urlpatterns = [
    url(r'^admin/', admin.site.urls),

    url(r'^$', home_page, name='home'),

    url(r'^lists/(.+)/$', view_list, name='view_list'),
    url(r'^lists/new$', new_list, name='new_list'),
]
```

**알기**
캡쳐그룹(.+)은 `/`뒤에 나오는 문자열에 일치된다.


list_.id를 통해서 각각의 목록마다 고유의 URL을 주었으므로 이제 `the-only-list-in-the-world`는
필요없다.

In`list/views.py`
```python
def new_list(request):
    list_ = List.objects.create()
    Item.objects.create(text=request.POST['item_text'], list=list_)
    return redirect('/lists/%d/' % (list_.id,))
```

-------------------------------------------------------------------------------
###기존 목록에 아이템을 추가하기 위한 또 다른 뷰
`/list/<list_id>/add_item`을 통해서 새로운 아이템을 기존의 list에 추가하도록 한다.

[list/test.py link](https://github.com/deadlylaid/TDD_Test/blob/master/superlists/lists/tests.py)


**주의**
urls.py에 존재하는 `url(r'^lists/(.+)/$', view_list, name='view_list')`가 `lists/<list_id>/add_item`까지
한번에 잡아버리는 경우가 발생한다. 이렇게 되면 301 ! = 302에러가 발생할 수 있다.
`url(r'^lists/(/d+)/$', view_list, name='view_list')`로 URL을 수정해서 숫자만 출력하도록 유도한다.


form에 해당 action으로 넣어야할 URL을 지정해준다
In`lists/Template/list.html`
```html
<form method="POST" action="/list/{{list.id}}/add_item">
```

View가 list를 template에 전달하도록 유도

```python
def test_passes_correct_list_to_template(self):
other_list = List.objects.create()
correct_list = List.objects.create()

response = self.client.get('/lists/%d/' % (correct_list.id,))

self.assertEqual(response.context['list'], correct_list)
```

그 후에 list.html에서 item을 lists로 변경
```python
{% for item in list.item_set.all %}
```


