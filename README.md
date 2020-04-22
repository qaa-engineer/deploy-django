# deploy-django
Мой вариант. Полный список действий. Это действительно работает. Проверено.

# Развертывание Django проекта c Github по шагам на чистую VPS c Debian 10.

1. Обновляем и апгрейдим списки репозиториев.

        apt-get update && apt-get upgrade

2. Устанавливаем must-have пакеты.

		apt-get install -y vim mosh tmux htop git curl wget unzip zip gcc build-essential make zsh tree redis-server nginx  libssl-dev zlib1g-dev libbz2-dev libreadline-dev libsqlite3-dev llvm libncurses5-dev libncursesw5-dev xz-utils tk-dev libffi-dev liblzma-dev python3-dev python3-pip python3-lxml libxslt-dev python-libxml2 python-libxslt1 libffi-dev libssl-dev python-dev gnumeric libsqlite3-dev libpq-dev libxml2-dev libxslt1-dev libjpeg-dev libfreetype6-dev libcurl4-openssl-dev supervisor

3. Устанавливавам Python3 из исходников.

		mkdir ~/code ; \
		wget https://www.python.org/ftp/python/3.8.2/Python-3.8.2.tgz ; \
		tar xvf Python-3.8.* ; \
		cd Python-3.8.2 ; \
		mkdir ~/.python ; \
		./configure --enable-optimizations --prefix=/home/www/.python ; \
		make -j8 ; \
		sudo make altinstall

4. Пишем алис и путь для Python3.

		vim ~/.bashrc
		
		export PATH=$PATH:/home/www/.python/bin
		alias python='python3.8'
		
		source ~/.bashrc

5. Обновляем pip.

		python -m pip install -U pip





В данный момент я редактирую этот документ.....
 
16. Установим из исходников среду виртуализации
 
 		/home/www/.python/bin/easy_install-3.8 virtualenv
 
17. Cоздадим пользователя для старта django-приложения. И создадим из-под него виртуальное окружение.

		adduser django
		login django
		cd /home/django
		virtualenv venv

 <h3>Загружаем проект с Github</h3>
	
17. Клонируем проект под рутом
 	
		su - root
		cd /home/django/
		git clone https://github.com/trystep/postindex.git
	
18. Удалим символическую ссылку
 
		rm /etc/nginx/sites-enabled/default
 
19. Логинимся под юзером django. Активируем виртуальное окружение. Устанавливаем Django3.* в наше виртуальное окружение c pip3: 

		login django
		source venv/bin/activate
		pip3 install django
	
31. Переходим в корневую папку проекта.

		cd my_best_site_from_github/project

32. Установим uWSGI с pip3

		pip3 install uwsgi

33. Проверка. Создаем файл test.py, пишем в него функцию для проверки. Если не создается, создайте от рута.

		vim test.py

		def application(env, start_response):
			start_response('200 OK', [('Content-Type','text/html')])
			return [b"Hello World"]

34. Если все работает, нажимаем ctrl-c, завершая процесс и продолжаем дальше. Переходим на пользователя root.
35. Начинаем конфигурировать БД. В моем случае - PostgreSQL.
	
		su - postgres
		createdb my_site
		createuser -P django
		psql
		postgres=# GRANT ALL PRIVILEGES ON DATABASE my_site TO django
		\c my_site
		GRANT ALL ON ALL TABLES IN SCHEMA public to django
		GRANT ALL ON ALL SEQUENCES IN SCHEMA public to django
		GRANT ALL ON ALL FUNCTIONS IN SCHEMA public to django
		\q
		exit

36. Тут вы должны оказаться под рутом. Устанавливаем интерфейс psycopg2 для работы с PostgreSQL, чтобы нам не писать на чистом SQL.

		login django
		source /home/django/venv/bin/activate
		pip3 install psycopg2

<h3>Загружаем дамп базы данных</h3>

36. Обращаю внимание, что тут вы должны находиться в среде виртуализации venv под пользователем django. dump - это ваш файл дампа/экспорта БД, формат может быть разный, cvs, txt и т.д.

		psql -h localhost my_site django  < dump
	
37. Скоро допишу, застрял на этом шаге, БД не импортится...
