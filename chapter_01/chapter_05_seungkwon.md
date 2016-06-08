## 이번 챕터에서 가장 중요하게 봐야하는것은 무엇인가?

유저 관점에서 입력하는 <input> node에 데이터를 입력해서 그 데이터가 올바르게 저장되었는지
확인하는데 그 중점을 두는것이 목표임

두가지 방법으로 접근하는데
1. selenium을 사용한다. - 기능테스트
2. 일반적인 Request를 사용한다 - 단위테스트

```
def test_signup_user(self):

    self.browser.get('http://localhost:8000/users/signup')

    user_id = self.browser.find_element_by_id('user_id').send_keys('test_id')
    user_pwd_first = self.browser.find_element_by_id('firstpw').send_keys('test12345')
    user_pwd_checked = self.browser.find_element_by_id('secondpw').send_keys('test12345')
    user_email = self.browser.find_element_by_id('user_email').send_keys('spark0017@naver.com')
    signup_btn = self.browser.find_element_by_id('signup').click()
    self.browser.implicitly_wait(1000)

    user = User.objects.count()

    self.assertEqual(
        user,
        1
    )
```

책에 나와있는것은 단위테스트 위주로 나와서 기능 테스트인 selenium을 사용해서 함.

csrf_token을 어떻게 정규식으로 걸러낼것인가?

```
    def modal_signup_user_model(self):

        rendered_html = render_to_string('users/signup.html')

        # csrf_token ignore
        csrf_regex = r'<input[^>]+csrfmiddlewaretoken[^>]+>'
        toast_reget = r'^카카오'
        observed_html = re.sub(csrf_regex, '', self.response.content.decode())

        self.assertEqual(
            observed_html,
            rendered_html,
        )

```

## 발생된 문제
javascript을 사용해서 client rendering 으로 발생된 template은 어떻게 봐야할까?
`render_to_string` 을 사용하면 javascript는 rendering되지 않는다.
이것에 대해서 어떻게 해야하는지 확인해볼 필요가 있다.
ex.) toast popup의 경우 test가 책에 나와있는 내용만으론 잡히지 않는다.