# 페브릭을 이용한 배포 자동화

 > 민수씨의 제보로 fabric은 python 2.6 버전에서 설치

###배포를 위한 페브릭 스크립트 파헤치기

```{.python}
# deploy_tools/fabfile.py

from fabric.contrib.files import append, exists, sed
from fabric.api import env, local, run
import random

REPO_URL = 'https://github.com/goohooh/TDD_Study.git'

def deploy():
	run('sudo apt-get update')
	run('sudo apt-get upgrade')
	run('sudo apte-get install git')
    project_folder = '/home/%s/TDD_superlist' % (env.user, envhost)
    # _create_directory_structure_if_necessary(site_folder)
    _get_latest_source(project_folder)
    _update_settings(project_folder, env.host)
    _update_virtualenv(project_folder)
    _update_static_files(project_folder)
    _update_database(project_folder)
```

`env.user`, `env.host`는 커맨드라인에서 지정한 서버 주소를 저장한다

```{.python}
# deploy_tools/fabfile.py

def _get_latest_source(project_folder):
	if exists(project_folder + '/.git'):
		run('cd %s && git fetch' % (project_folder,))
	else:
		run('git clone %s' % (REPO_URL))

	current_commit = local("git lo -n 1 -format=%H", capture=True)
	run('git reset --hard %s' % (current_commit))
	# 서버의 코드 디렉터리에 발생한 모든 변경을 초기화 한다????
```

```{.python}
def _update_settings(project_folder, site_name):
	settings_path = project_folder + '/superlist/settings.py'
	# sed 메서드로 문자열 변경
	sed(settings_path, "DEBUG = True", "DEBUG = False")
	sed(settings_path,
		'ALLOWED_HOSTS = .+$',
		'ALLOWED_HOSTS = ["%s"]' % (site_name,)
	)
	secret_key_file = project_folder + '/superlist_key.py'
	if not exists(secret_key_file):
		chars = 'abcdefghijklmnopqrstuvwxyz0123456789!@#$%^&*('
		key = ''.join(random.SystemRandom().choice(chars) for _ in range(50))
		# append로 파일 끝에 새로운 줄 추가(\n붙이는것 권장)
		append(secret_key_file, "SECRET_KEY = '%s'" %(key,))
	append(settings_path, '\nfrom .secret_key import SECRET_KEY')
```

```{.python}
def _update_virtualenv(project_folder):
	bash_profile = '~/.bash_profile'
	if not exists(bash_profile):
		run('touch ~/.bash_profile')
	run('sudo apt-get install -y make build-essential libssl-dev zlib1g-dev libbz2-dev \
libreadline-dev libsqlite3-dev wget curl llvm libncurses5-dev libncursesw5-dev xz-utils')
	run('curl -L https://raw.githubusercontent.com/yyuu/pyenv-installer/master/bin/pyenv-installer | bash')
	append(bash_profile, 'export PATH="~/.pyenv/bin:$PATH"')
	append(bash_profile, 'eval "$(pyenv init -)"')
	append(bash_profile, 'eval "$(pyenv virtualenv-init -)"')
	run('source ~/.bash_profile')
	# run('pip install autoenv')
	# run('"source ~/.autoenv/activate.sh" >> ~/.bash_profile')
	run('pyenv install 3.5.1')
	run('pyenv virtualenv 3.5.1 TDD_superlist')
	run('pyenv activate TDD_superlist')
	run('pip install -r requirements.txt')
```
```{.python}
	def _update_static_files(project_folder):
		run('cd %s && python manage.py migrate --noinput' % (project_folder,))
```