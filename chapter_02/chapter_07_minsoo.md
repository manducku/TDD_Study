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

