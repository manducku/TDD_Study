###6장 최소 동작 사이트 
@(bucket list)[Django, TDD]

테스트와 테스트 사이의 격리가 필요.
현재까지의 방식은 이전의 테스트가 다음 테스트에 반영되는 문제가 있었음.

단위 테스트 별로 테스트 데이터베이스를 생성할 필요가 있음 

**해결 방법**
`test.py`에 기존 결과물을 정리하는 코드를 추가, 이 때 필요한 것이 `setup`과 `tearDown` 메소드 

`LiveServerTestCase` - 자동으로 테스트용 데이터베이스를 생성하고 기능 테스트를 위한 개발 서버를 가동
Django의 manage.py를 통해 실행 가능하며, 1.6 버전부터는 테스트 실행자가 'test'로 시작하는 파일을 자동으로 찾는다.

1. `mkdir functional_tests`
2. `touch functional_tests/__init__.py`
3. `functional_test.py` -> `tests.py`로 이전 

functional_tests를 하나의 App처럼 만듬. 
이제부터는 `python functional_tests.py`가 아닌 `python manage.py test functional_tests`로 실행 

Django의 테스트 실행자를 사용하므로 `if __name__ ='__main__'`는 삭제 

기능 테스트만 실행하기 
`python manage.py test functional_tests`
단위 테스트만 실행하기 
`python manage.py test lists`

> **YAGNI!**
> 'you ain`t gonna need it' 사용하지 않을 기능에 대해서 너무 많은 고민을 하지말라

**REST**
REST(Representational State Transfer)
REST는 간단하게 말해 데이터 구조를 URL 구조에 일치시키는 방식 

ex) `lists/<목록 식별자>/` 특정 list에 접근
 `lists/new/` 새로운 list를 추가 
 `lists/<목록 식별자>/add_item` 기존 리스트에 새로운 작업 아이템을 추가 

**신규 기능** 
에디스가 첫번째 아이템을 전송하자마자 새로운 목록을 만들어서 식규 작업 아이템을 추가한다. 그리고 그가 만든 목록에 접근할 수 있는 URL을 제공 

```
  edith_list_url = self.browser.current_url
  self.assertRegex(edith_list_url, '/lists/.+')
```

$`assertRegex`는 unittest에 부속된 헬퍼 함수로, 지정한 정규표현과 문자열이 일치하는지 확인한다. 

**기능테스트 작성**
```
		#페이지는 다시 갱신되고, 두 개 아이템이 목록에 보인다
        self.check_for_row_in_list_table('1: TDD 공부하기')
        self.check_for_row_in_list_table('2: React 공부하기')

        #새로운 사용자 프란시스가 사이트에 접속

        ##새로운 브라우저 세션을 이용해서 에디스의 정보가 쿠키를 통해 유입되는 것을 방지
        self.browser.quit()
        self.browser = webdriver.Firefox()

        #프란시스가 홈페이지에 접속한다.
        inputbox = self.browser.find_element_by_id('id_new_item')
        inputbox.send_keys('크로스핏 하기')
        inputbox.send_keys(Keys.ENTER)

        #프란시스 전용 URL을 취득한다
        fransys_list_url = self.browser.current_url
        self.assertRegex(fransys_list_url, '/lists/.+')
        self.assertNotEqual(fransys_list_url, edith_list_url)

        #에디스가 입력한 흔적이 없다는 것을 다시 확인한다
        page_text = self.browser.find_element_by_tag_name('body').text
        self.assertNotIn('TDD 공부하기', page_text)
        self.assertIn('크로스핏 하기', page_text)
```

에디스 이외에 프란시스라는 유저를 추가하여, 각 사용자별로 URL을 받는지 확인하는 FT 시나리오를 작성한다.

**단위 테스트 수정**
```
def test_home_page_can_redirect_after_POST_request(self):
        request = HttpRequest()
        request.method = 'POST'
        request.POST['item_text']= 'A new list item'

        response = home_page(request)

        self.assertEqual(response.status_code, 302)
        self.assertEqual(response['location'], '/lists/the-only-list-in-the-world/')
```
redirect 주소가 `/lists/the-only-list-in-the-world`와 같은지 확인하는 assert를 추가한다. 한 번에 한가지만 수정하기로 하였으므로, 천천히 단계적으로 접근한다.

**home_page View 수정**
```
def home_page(request):
    if request.method == 'POST':
        new_item_text = request.POST['item_text']
        Item.objects.create(text=new_item_text)
        return redirect('/lists/the-only-list-in-the-world/')
```
redirect 주소를 우리가 설정한 `/lists/the-only-list-in-the-world`로 수정 

> **장고 테스트 클라이언트**
> URL 해석 테스트, 뷰 함수를 호출해서 제대로 동작하는지 테스트, 뷰가 템플릿을 제대로 렌더링하는지 확인 하는 테스트를 한 번에 할 수 있는 Tool
>**장고 문서 참조**
>Use Django’s test client to establish that the correct template is being rendered and that the template is passed the correct context data.

```
class ListViewTest(TestCase):
        def test_home_page_can_display_all_list_item(self):
        Item.objects.create(text='item 1')
        Item.objects.create(text='item 2')

        response =self.client.get('/lists/the-only-list-in-the-world/')

        self.assertContain(response, 'item 1')
        self.assertContain(response, 'item 2')
```
직접 view 함수를 호출하지 않고, `self.client.get()`를 활용하여 url을 통해 접근 

**superlists/urls.py** 
```
	...
	
url(r'^lists/the-only-list-in-the-world/$', view_list, name='view_list'),
]
	...
```

**lists/views.py**
```
def view_list(request):
    items = Item.objects.all()
    return render(
            request, 
            'lists/home.html',
            {'items': items}
            )
```
위와 같이 변경한 후, test를 실행하면 이상 없이 test가 완료된다.

**목록 출력을 위한 별도 템플릿**
`home.html`은 하나의 입력 상자를 가지며, 새로운 템플릿인 `list.html`은 기존 아이템을 보여주는 테이블을 보여준다

*lists/tests.py* 
```
def test_uses_list_template(self):
            response = self.client.get('/lists/the-only-list-in-the-world')
            self.assertTemplateUsed(response, 'list.html')
```

>`assertTemplateUsed()`        
>whether a response was rendered using a template, by using a special method response called assertTemplateUsed

*views -> def iew_list*
```
return render(
            request, 
            'lists/list.html',
            {'items': items}
            )
```

*templates/lists/home.html*
```
<html>
    <head>
        <title>To-Do lists</title>
    </head>
    <body>
        <h1>작업 목록 시작 </h1>
        <form method="POST">
            <input name="item_text" placeholder='작업 아이템 입력' id='id_new_item'></input>
            {% csrf_token %}
        </form>
    </body>
</html>
```
`form` 부분만 내버려두고, 나머지 다 삭제
테스트를 지속적으로 하면서 다른 곳에 문제가 생기는 부분이 있는지 확인

*lists/views.py*
```
# Create your views here.
def home_page(request):
    if request.method == 'POST':
        new_item_text = request.POST['item_text']
        Item.objects.create(text=new_item_text)
        return redirect('/lists/the-only-list-in-the-world/')
```
`items`를 rendering 해줄 필요가 없으므로, 나머지 부분을 제거 

*lists/templates/list.html*
```
<form method="POST" action="/">
```
form의 action이 없어서 FT에서 계속해서 실패함.
그러므로 home함수로 form으로 입력받은 값들을 던져 줌.

**목록 아이템을 추가하기 위한 URL과 뷰**
신규 목록 생성을 위한 테스트 클래스 

데이터에 변경을 가하는 Action URL의 경우, **trailing slash**를 사용하지 않음

> **trailing slash or not?**     
>Traditionally, URLs that pointed to files did not include the trailing slash, while URLs that pointed to directories do include the trailing slash. This means that:
    
> http://webdesign.about.com/example/ is a directory, while
> http://webdesign.about.com/example is a file
> This helps speed up page loading because the trailing slash immediately tells the web server to go to that example/ directory and look for the index.html or other default file.

> When you go to a URL without the trailing slash, the web server looks for a file with that name.

<br>
```
def test_saving_a_POST_request(self):
        self.client.post(
                '/lists/new',
                data={'item_text':'신규 작업 아이템'}
                )
        self.assertEqual(Item.objects.count(), 1)
        new_item = Item.objects.first()
        self.assertEqual(new_item.text, '신규 작업 아이템')

    def test_redirects_after_POST(self):
        response = self.client.post(
                '/lists/new',
                data={'item_text':'신규 작업 아이템'}
                )
        self.assertEqual(response.status_code, 302)
        self.assertEqual(response['location'], '/lists/the-only-list-in-the-world/')
```
Post 저장과 redirect를 따로 분리하여 테스트 

**신규 목록 생성을 위한 URL 뷰**

*lists/views.py*
```
def new_list(request):
    Item.objects.create(text=request.POST['item_text'])
    return redirect('/lists/the-only-list-in-the-world/')
```

*superlist/url.py*
```
urlpatterns = [
    url(r'^$', home_page, name='home'), 
    #url(r'^admin/', admin.site.urls),
    url(r'^lists/the-only-list-in-the-world/$', view_list, name='view_list'),
    url(r'^lists/new$', new_list, name='new_list'),
]
```

**필요없는 코드와 테스트 삭제**
`home_page`와 `new_list`가 기능이 겹치므로, `home_page`에서의 POST 내용은 전부 삭제함.

**새로운 URL에 있는 FORM 가르키기**
*lists/templates/home.html, lists/template/list.html*
```
	<form method="POST" action="lists/new">
```

**모델 조정하기**

```
def test_saving_and_retrieving_items(self):
        list_ = List()
        list_.save()

        first_item = Item()
        first_item.text = '첫번째 아이템'
        first_item.list = list_
        first_item.save()
        
        second_item = Item()
        second_item.text = '두번째 아이템'
        second_item.list = list_
        second_item.save()
        saved_list = List.objects.first()
        self.assertEqual(saved_list,list_)
        
        saved_items = Item.objects.all()
        self.assertEqual(saved_items.count(), 2)

        first_saved_item = saved_items[0]
        second_saved_item = saved_items[1]

        self.assertEqual(first_saved_item.text, '첫번째 아이템')
        self.assertEqual(first_saved_item.list, list_)
        self.assertEqual(second_saved_item.text, '두번째 아이템')
        self.assertEqual(second_saved_item.list, list_)
```

새로운 List(목록) 객체를 생성하고, 각 아이템에 .list 속성을 부여해서 List 객체에 할당.
목록이 제대로 저장됐는지와 두 개 작업 아이템이 목록에 제대로 할당됐는지 확인.
**목록 개체를 서로 비교하는 것이 가능 (saved_list와 list 비교)**
 `self.assertEqual(saved_list,list_)` 이 부분 이해가 안됨 

**외래키 관계**
```
class List(models.Model):
    pass

class Item(models.Model):
   text = models.TextField(default='') 
   list = models.ForeignKey(List, default=None)
```
List와 Item 간의 ForeignKey로 연결해 준다.

허나, Test를 돌려보면 기존에 만든 Item들에 부모 List가 없어 intergrity Error가 발생 함.
또한 new_list view 함수에서 list 객체 없이 item을 생성하고 있기 때문에 문제가 발생
아래와 같이 코드를 변경해 준다. 

```
def new_list(request):
    list_ = List.objects.create()
    Item.objects.create(text=request.POST['item_text'], list=list_)
    return redirect('/lists/the-only-list-in-the-world/')
```

**각 목록이 하나의 고유 URL을 가져야 한다**
목록을 위한 고유 식별자 -> 데이터베이스가 자동 생성하는 id 필드 

두 개의 List를 Objet를 생성하여,  서로 간의 데이터를 공유하지 않는지 확인하는 테스트를 작성 

*lists/test.py*
```
def test_display_only_items_for_that_list(self):
            correct_list = List.objects.create()
            Item.objects.create(text='item 1', list=correct_list)
            Item.objects.create(text='item 2', list=correct_list)

            other_list= List.objects.create()
            Item.objects.create(text='item 3', list=other_list)
            Item.objects.create(text='item 4', list=other_list)

            response = self.client.get('/lists/%d/' %(correct_list.id))

            self.assertContains(response, 'item 1')
            self.assertContains(response, 'item 2')
            self.assertNotContains(response, 'item 3')
            self.assertNotContains(response, 'item 4')
```

parameter로 id를 받는 url을 `url.py`에 추가 

*superlists/url.py*
```
urlpatterns = [
    url(r'^$', home_page, name='home'), 
    #url(r'^admin/', admin.site.urls),
    url(r'^lists/new$', new_list, name='new_list'),
    url(r'^lists/(?P<id>)/$', view_list, name='view_list'),
]
```

*lists/views.py*
특정 list에 맞는 item들만 선별하여 render

```
def view_list(request, id):
    list_ = List.objects.get(id=id)
    items = Item.objects.filter(list=list_)

    return render(
            request, 
            'lists/list.html',
            {'items': items}
            )
```

**새로운 세상으로 가기 위한 new_list 수정**
*lists/test.py*
```
def test_redirects_after_POST(self):
        response = self.client.post(
                '/lists/new',
                data={'item_text':'신규 작업 아이템'}
                )
        self.assertRedirects(response, '/lists/the-only-list-in-the-world/')
```
`/lists/the-only-list-in-the-world/`로 redirect되어 있는 것을 새롭게 만들어진 list로 redirect 하도록 수정

*lists/test,py*
```
def test_redirects_after_POST(self):
        response = self.client.post(
                '/lists/new',
                data={'item_text':'신규 작업 아이템'}
                )
        new_list = list.objects.first()
        self.assertRedirects(response, '/lists/%d/' %(new_list.id))
```

*lists/views.py*
```
def new_list(request):
    list_ = List.objects.create()
    Item.objects.create(text=request.POST['item_text'], list=list_)
    return redirect('/lists/%d/' %(list_.id))
```
view 함수에서 새로운 list로 가도록 view 함수 수정 

하지만 현재는 모든 Post마다 list를 만드므로, FT 실행 시에 퇴행이 일어나게 된다.

**기존 목록에 아이템을 추가하기 위한 또 다른 뷰**
기존 목록에 신규 아이템을 추가하기 위한 URL과 View가 필요

*lists/test.py*
```
class NewItemTest(TestCase):
    def test_can_save_a_POST_request_to_an_existing_list(self):
        other_list = List.objects.create()
        correct_list = List.objects.create()

        self.client.post(
                'lists/%d/add_item' %(correct_list.id)),
                data={'item_text':'기존 목록에 신규 작업 아이템'}
                )
        self.assertEqual(Item.objects.count(), 1)
        new_item = Item.objects.first()
        self.assertEqual(new_item.text, '기존 목록에 신규 작업 아이템')
        self.assertEqual(new_item.list, correct_list) 

    def test_redirects_to_list_view(self):
        other_list = List.objects.create()
        correct_list = List.objects.create()

        response = self.client.post(
                'lists/%d/add_item' %(correct_list.id),
                data={'item_text': '기존 목록에 신규 작업 아이템'}
                )
        self.assertRedirects( responese, 'lists/%d/' %(correct_list.id))
```

POST가 제대로 되었는지, POST 후에 redirect가 성공적으로 진행되었는지 확인하는 코드를 작성 

**마지막 신규 URL**
*superlists/url.py*
```
urlpatterns = [
    url(r'^$', home_page, name='home'), 
    #url(r'^admin/', admin.site.urls),
    url(r'^lists/(\d+)/$', view_list, name='view_list'),
    url(r'^lists/(\d+)/add_item$', add_item, name='add_item'),
    url(r'^lists/new$', new_list, name='new_list'),
]
```
*lists/views.py*
```
def add_item(request, list_id):
    list_ = List.objects.get(id=list_id)
    Item.objects.create(text=request.POST['item_text'], list=list_)
    return redirect('/lists/%d/' %(list_.id))

```
특정 리스트에 신규 아이템 추가하는 View함수 생성 

**FORM에서 URL 사용하기**
*template/list.html*
```
<form method="POST" action="/lists/{{list.id}}/add_item">
```
`list.id`를 통해 action을 지정 

*lists/test,py*
```
def test_passed_correct_list_to_template(self):
            other_list = List.objects.create()
            correct_list = List.objects.create()

            response = self.client.get('/lists/%d' %(correct_list.id))
            self.assertEqual(response.context['list'], correct_list)
```
정확한 context가 전달되었는지 확인하는 테스트 코드 작성 

*lists/views.py*
```
def view_list(request, id):
    list_ = List.objects.get(id=id)
    return render(
            request, 
            'lists/list.html',
            {'list': list_}
            )
```
context로 list 객체 전달 

*template/list.html*
```
<html>
    <head>
        <title>To-Do lists</title>
    </head>
    <body>
        <h1>TO-DO List</h1>
        <form method="POST" action="/lists/{{list.id}}/add_item">
            <input name="item_text" placeholder='작업 아이템 입력' id='id_new_item'></input>
            {% csrf_token %}
        </form>
        <table id='id_list_table'>
            {% for item in list.item_set.all %} 
            <tr><td>{{ forloop.counter}}: {{ item.text  }} </td></tr>
            {% endfor %}
        </table>
    </body>
</html>
```
`item` 대신에 `list`가 넘어 왔으므로, for문 수정

**URL includes를 활용한 리펙토링**
*superlist/url.py*
```
from django.conf.urls import url
from django.contrib import admin
from lists.views import home_page
from django.conf.urls import include

urlpatterns = [
    url(r'^$', home_page, name='home'), 
    #url(r'^admin/', admin.site.urls),
    url(r'^lists/', include('lists.urls')),
]
```
lists aplication과 관련된 모든 값들을 지우고, include를 통해서 `lists.urls`에서 해당 url의 처리를 위임한다.

*lists/url.py*
```
from django.conf.urls import url
from django.contrib import admin
from lists.views import home_page
from lists.views import view_list
from lists.views import new_list
from lists.views import add_item

urlpatterns = [
    url(r'^(\d+)/$', view_list, name='view_list'),
    url(r'^(\d+)/add_item$', add_item, name='add_item'),
    url(r'^new$', new_list, name='new_list'),
]
```

기존에 `superlists/urls.py`에 있던 list 관련된 내용만 발췌하여 작성한다
