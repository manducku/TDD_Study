###5장 사용자 입력 저장하기

**csrf-token setting**
```
<form method="POST">
            <input name="item_text" placeholder='작업 아이템 입력' id='id_new_item'></input>
        </form>
```
```
<form method="POST">
            <input name="item_text" placeholder='작업 아이템 입력' id='id_new_item'></input>
            {% csrf_token %}
        </form>
```

####서버에서 POST 요청 처리
폼에 action 속성을 지정하지 않았기 때문에, 동일 URL을 전달해서 기본 설정된 페이지를 다시 표시한다

```
def test_home_page_can_save_POST_request(self):
        request = HttpRequest()
        request.method = 'POST'
        request.POST['item_text']='신규 작업 아이템'

        response = home_page(request)

        self.assertIn('신규 작업 아이템', response.content.decode())
```

home_page view 함수에서 post 요청에 따른 logic을 추가하지 않았으므로 assert error가 발생 

> 참고) 
> 테스트 코드 작성 컨벤션      
> 설정(Setup), 처리(Exercise), 어설션(Assert) 파트를 나누어 작성

```
    def test_home_page_can_save_POST_request(self):
        request = HttpRequest()
        request.method = 'POST'
        request.POST['item_text']='신규 작업 아이템'

        response = home_page(request)
        print(response)

        self.assertIn('신규 작업 아이템', response.content.decode())
        expected_html = render_to_string(
                'lists/home.html',
                {'new_item_text': '신규 작업 아이템'}
                request=request
                )
        self.assertEqual(response.content.decode(), expected_html)
```
reder_to_string 함수의 두번째 인자로 parameter 값 전달 
response에 view 처리가 없으므로 실패함

> **get csrf token in render_to_string**
 
> To get the csrf token to work when using render_to_string, you need to supply the request object so that the context processors run.

> In Django 1.8+, you can simply pass the request as an argument

> return render_to_string('index.html', request=request)
On earlier versions, you can use a RequestContext.

> from django.template import RequestContext
render_to_string('index.html', context_instance=RequestContext(request))

<br>
>**Dictionary Data Get**
>It allows you to provide a default value if the key is missing:

> dictionary.get("bogus", None)
> returns None, whereas

> dictionary["bogus"]
> would raise a KeyError.


**2번째 task 추가**
```
        inputbox = self.browser.find_element_by_id('id_new_item')
        inputbox.send_keys('React 공부하기')
        inputbox.send_keys(Keys.ENTER)

        table = self.browser.find_element_by_id('id_list_table')
        rows = table.find_elements_by_tag_name('tr')
        self.asserIn('1: TDD 공부하기', [row.text for row in rows])
        self.asserIn('2: React 공부하기', [row.text for row in rows])
```

같은 코드가 연속적으로 나오므로, Refactoring(Code smell)

```
def check_for_row_in_list_table(self, row_test):
        table = self.browser.find_element_by_id('id_list_table')
        rows = table.find_elements_by_tag_name('tr')
        self.assertIn(row_text, [row.text for row in rows]
```

참고) test_로 시작하는 메소드만 테스트로 실행되므로, `check_for_row_in_list_table`은 test 진행 시, 실행되지 않음 

####Django ORM과 첫 모델
객체 관계형 맵핑 - DB의 테이블 레도크, 칼럼 형태로 저장돼 있는 데이터를 추상화한 것. 객체 지향 코딩 방식을 이용해서 데이터베이스를 처리할 수 있음

```
from lists.models import Item 

class ItemModelTest(TestCase):
    
    def test_saving_and_retrieving_items(self):
        first_item = Item()
        first_item.text = '첫번째 아이템'
        first_item.save()
        
        second_item = Item()
        second_item.text = '두번째 아이템'
        second_item.save()
        
        saved_items = Item.objects.all()
        self.assertEqual(saved_items.count(), 2)

        first_saved_item = saved_items[0]
        second_saved_item = saved_items[1]

        self.assertEqual(first_saved_item, '첫번째 아이템')
        self.assertEqual(second_saved_item, '두번째 아이템')

```

models.py에서 Item 클래스를 가져와 초기화 해주고, 입력할 값들을 입력해 줌 
UnitTest는 Django 모듈 내에서 Django의 기능을 활용하여 Test
FunctionTest는 Django 모듈 밖에서 셀레늄을 통해 테스트 


####POST를 데이터베이스에 저장하기
**lists/test.py**
```
def test_home_page_can_save_a_POST_request(self):
        request = HTTPRequest()
        request.method = 'POST'
        request.POST['item_text']= 'A new list item'
        response = homepage(request)

        self.assertEqual(Item.objects.count(), 1)
        new_item = Item.objects.first()
        self.assertEqual(new_item.text, 'A new list item')

        self.assertIn('A new list item', response.content.decode())
        expected_html = render_to_string(
                'home.html',
                {'new_item_text': 'A new list item'}
                )
        self.assertEqual(response.content.decode(), expected_html)
```

**To-Do List**
Django import 규칙 확인 - from은 Root 폴더로부터 시작 됨
