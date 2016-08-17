# 멋있게 만들기: 레이아웃, 스타일링, 테스트

###레이아웃과 스타일을 기능적으로 테스트하기

지독하게 까다로운 정적파일 설정을 위해 스모크 테스트를 할 예정이다.
```{.python}
# functional_tests/test.py 

class NewVisitorTest(LiveServerTestCase):

	def test_layout_and_styling(self):
		# 에디스는 메인 페이지를 방문한다
		self.browser.get(self.live_server_url)
		self.browser.set_window_size(1024,768)

		# 그녀는 입력 상자가 가운데 배치된 것을 본다
		inputbox = self.browser.find_element_by_id('id_new_item')
		self.assertAlmostEqual(
			inputbox.location['x'] + inputbox.size['width']/2.
			512,
			delta=10
		)
```
어떤 기준인지 모르겠지만 우선 굉장히 지저분한 **'편법'**으로 테스트를 통과한다. 테스트를 확장하여 신규 작업 목록 페이지에서도 입력 상자가 가운데 배치되는지 확인한다.

```{.python}
#functional_tests

	# 그녀는 새로운 리스트를 시작하고 입력 상자가 가운데 배치된 것을 확인한다
	inputbox.send_keys('testing\n')
	inputbox = self.browser.find_element_by_id('id_new_item')
	self.assertAlmostEqual(
		inputbox.location['x'] + inputbox.size['width']/2.
		512,
		delta=10
	)
```

`git reset --hard`로 괴이한 코드는 리셋한다

---
###멋있게 만들기: CSS프레임워크 이용

block이라는 연속 영역 기능을 이용하여 커스터마이징에 더 집중하도록 한다

---
###부트스트랩 통합하기

---
###Django의 정적 파일

`settings.py`에서 STATIC_URL의 기본으로 '/static/'이 할당돼있다. 이는 '/static/'으로 시작되는 요청을(개발 서버에서만)받으면, Django가 모든 서브 디렉터리에서 'static'폴더를 검색하여 처리한다.

####StaticLiveServerTestCase로 교체하기

기능 테스트는 실패하는 모습을 볼 수 있는데 이는 테스트 서버가 정적 파일을 찾을 수 없기 때문이다.
테스트의 상속 클래스를 교체한다.

###부트스트랩 컴포넌트를 이용한 사이트 외형 개선

###사용자 지정 CSS 사용하기

###얼버무리고 넘어간 것: collectstatic과 다른 정적 디렉터리

골치아픈 작업이 남았다. collectstatic 명령어로 정적 파일을 관리하게 될 텐데 이를 위해 settings.py에 경로 설정을 해준다.

```{.python}
# settings.py

# Static files (CSS, JavaScript, Images)
# https://docs.djangoproject.com/en/1.9/howto/static-files/
STATIC_URL = '/static/'
STATIC_ROOT = os.path.abspath(os.path.join(BASE_DIR, '../static'))
```