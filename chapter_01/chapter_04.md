####4.왜 테스트를 하는 것인가?

**시시한 함수에 대해 시시한 테스트를 하는 이유**    
쉬운 함수를 통해 테스트를 지속적으로 연습해왔다면, 새로운 테스트를 추가하는 것이 자연스럽고, 테스트도 수월함.
즉, 테스트를 해야할 정도의 복잡하다고 판단 후, 테스트를 작성한다면 훨씬 많은 수고가 들어감 

**TDD가 훌륭한 이유**
- 다음에 무엇을 할지 잊어버릴 걱정이 없다는 것. 테스트를 실행하기만 하면 다음 작업이 무엇인지 가르쳐 준다 

**기능 테스트 코드**
```
    def test_can_start_a_list_and_retrieve_it_later(self):
        
        #해당 웹사이트를 확인 
        self.browser.get('http://localhost:8000')

        #웹 페이지 타이틀과 헤더가 "TO-DO"를 표시하고 있는가?
        self.assertIn('To-Do lists', self.browser.title)
        self.assertEqual(inputbox.get_attribute('placeholder'), '작업 아이템 입력')

        #task 추가 
        #"TDD 공부하기" 텍스트 상자에 입력 
        inputbox.send_keys('TDD 공부하기')

        #엔터키를 치면 페이지가 갱신되고 작업 목록에 "1: TDD 공부하기"가 추가
        inputbox.send_keys(Keys.ENTER)

        table = self.browser.find_element_by_id('id_list_table')
        rows = table.find_elements_by_tag_name('tr')
        self.assertTrue(
                any(row.text == '1:TDD 공부하기' for row in rows),
                )

        #추가 아이템을 입력할 수 있는 여분의 텍스트 상자가 존재 
        #TDD 복습하기를 입력

        #페이지 생긴되고, 두 개의 아이템 목록이 보임
        #사이트가 입력한 몰록을 저장하고 있는지 확인 

        #특정한 URL 생성
        #URL에 대한 설명도 함께 제공 

        #해당 URL에 접속하여 자신의 작업 목록이 있는지 확인 
        self.fail('finish the test!')

```

기능 테스트는 사용자 관점에서 product를 사용하는 flow를 기록

**기능 테스트 코드 상세 설명** 
> `send_keys` 
> 입력 요소에 값들을 입력하는 함수

> `find_elements_by_tag_name`, `find_element_by_id` 
> element, elements의 차이는 단일 객체를 반환하느냐 List 객체를 반환하느냐의 차이 

> `render_to_string()` 
> To cut down on the repetitive nature of loading and rendering templates, Django provides a shortcut function which largely automates the process: render_to_string() in django.template.loader, which loads a template, renders it and returns the resulting string

>`.decode()`
> 바이트 데이터를 파이썬 유니코드 문자열로 변환 

리팩토링 시에는 앱 코드와 테스트 코드를 한 번에 수정하는 것이 아니라 하나씩 수정해야 한다. 
작은 단계로 나누어 착실히 리팩토링 작업을 실행 

**TDD 프로세스**     
기능테스트 관점의 리팩터링
- 애플리 케이션 동작을 확인하기 위해서 기능테스트를 사용하지만, 단위 테스트를 변경, 추가, 제가할 수 있음을 의미     

기능 테스트 - 애플리케이션이 동작하는지 아닌지를 판단하기 위한 궁극의 수단     
단위 테스트 - 기능테스트 판단을 돕기 위한 툴    

![enter image description here](https://github.com/manducku/TDD_Study/blob/master/assets/IMG_6683-1.jpg)
