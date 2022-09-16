# Makefile for your Django project

During the development process, you often need to run some commands in your terminal: create migrations, run tests, linters, etc. Usually, you execute these commands regularly.

It's helpful to have shortcuts for such commands. And even better to share them among other developers on the project. For this goal, I use [Makefile](https://en.wikipedia.org/wiki/Make_(software)#Makefile). Here is the list of useful commands for a Django project.

### Installing dependencies

I'm not a big fan of [Poetry](https://python-poetry.org/). It has [some problems](https://github.com/dependabot/dependabot-core/issues?q=is%3Aissue+is%3Aopen+poetry+label%3A%22L%3A+python%3Apoetry%22) with Dependabot and doesn't play with [Renovate](https://docs.renovatebot.com/python/) at all. I use [pip-tools](https://github.com/jazzband/pip-tools) with separate `requirements.in` and `requirements-dev.in` for local and prod environments. 


```makefile
pip-install-dev:
	pip install --upgrade pip pip-tools
	pip-sync requirements.txt requirements-dev.txt

pip-install:
	pip install --upgrade pip pip-tools
	pip-sync requirements.txt

pip-update:
	pip install --upgrade pip pip-tools
	pip-compile requirements.in
	pip-compile requirements-dev.in
	pip-sync requirements.txt requirements-dev.txt
```

Commands:

- `pip-install-dev`: Installs all requirements, including local. It is the main command for installing the project's dependencies, so I put this command at the top.
- `pip-install` - I don't use this command often, but it could be helpful to run your code with production dependencies only. 
- `pip-update` Updates your `requirements.txt` and `requirements-dev.txt` after you add a new package to the `requirements.in` or `requirements-dev.in`.

### Running the project

```makefile
server:
	python manage.py migrate && python manage.py runserver

worker:
	python -m celery -A project_name worker --loglevel info
	
beat:
    python -m celery -A project_name beat --loglevel info
```

That is an obvious one. The commands to run local web server and [Celery](https://docs.celeryq.dev/en/stable/). Also, I like to apply new migrations automatically when I run the server.

### Running linters

```makefile
lint:
	flake8 palyanytsya
	mypy palyanytsya

black:
	python -m black palyanytsya

cleanimports:
	isort .
	autoflake -r -i --remove-all-unused-imports --ignore-init-module-imports project_name

clean-lint: cleanimports black lint

checkmigrations:
	python manage.py makemigrations --check --no-input --dry-run
```

Commands:

- `lint`: Runs [flake8](https://github.com/PyCQA/flake8) linter and [mypy](https://github.com/python/mypy) type checker.
- `black`: Automatic code formatting with [Black](https://github.com/psf/black). 
- `cleanimports`: runs [isort](https://github.com/PyCQA/isort) and removes unused imports with [Autoflake](https://github.com/PyCQA/autoflake). Be sure to set up `profile=black` in isort settings to avoid conflicts with Black. 
- `clean-lint`: run all the stuff above. You can run this command before committing to formatting your code properly.
- `checkmigrations`: prevents you from committing model changes without migrations. Really cool stuff!

Also, I use `make lint && make checkmigrations` in the CI pipeline and in the git pre-commit hook. You can also create a command for setting up such a hook: 

```makefile
install-hooks:
	echo "make lint && make checkmigrations" > .git/hooks/pre-commit && chmod 777 .git/hooks/pre-commit
```

### Running tests

```makefile
test:
	pytest -n 4 -x
```

Runs [pytest](https://docs.pytest.org/en/7.1.x/) in multiprocess mode using [pytest-xdist](https://github.com/pytest-dev/pytest-xdist).

### Compiling messages 

```makefile
messages:
	python manage.py makemessages --all --ignore=venv --extension html,py
	python manage.py translate_messages -s en -l uk -l es -u
	python manage.py compilemessages --ignore venv
```

Collects all string literals for [translation](https://docs.djangoproject.com/en/4.1/topics/i18n/translation/). Also, I use [Django Autotranslate](https://github.com/ankitpopli1891/django-autotranslate) to translate phrases using Google Translate automatically.

## The end

As long as the project goes on, new local commands appear. Maybe, you need to shell into the dev server to run some python code. Or download a fresh database dump (without sensitive data) for local bug reproducing. It's better to keep this knowledge in the codebase, not in your head. Also, you won't need to explain how to do exactly the same operation to your colleagues.

For more complex tasks, you can even create a shell script and call it from Makefile. So, it would be easier to find this new command for other devs.

