# deploy-django

# Развертывание готового Django проекта c Github по шагам на чистую VPS Debian 9 с Nginx и uWSGI.

1. Устанавливаем must-have пакеты, размещаем последний Python из исходников в одну папку. Это займет примерно 20 минут.

		apt-get update && apt-get upgrade -y; \
		apt-get install -y vim mosh tmux htop git curl wget unzip zip gcc build-essential make libssl-dev zlib1g-dev libbz2-dev libreadline-dev libsqlite3-dev llvm libncurses5-dev libncursesw5-dev xz-utils tk-dev libffi-dev liblzma-dev python3-dev python3-lxml libxslt-dev python-libxml2 python-libxslt1 libffi-dev libssl-dev gnumeric libsqlite3-dev libpq-dev libxml2-dev libxslt1-dev libjpeg-dev libfreetype6-dev libcurl4-openssl-dev nginx supervisor ; \
		mkdir ~/code ; \
		cd /code ; \
		wget https://www.python.org/ftp/python/3.8.2/Python-3.8.2.tgz ; \
		tar xvf Python-3.8.* ; \
		cd Python-3.8.2 ; \
		mkdir ~/.python ; \
		./configure --enable-optimizations --prefix=/home/www/.python ; \
		make -j8 ; \
		sudo make altinstall ; \
		sudo /home/www/.python/bin/python3.8 -m pip install -U pip

	
2. Допишем в конфигурацию алиасы и патчи, чтобы вызывался последний Python.
	
		vim ~/.bashrc

		export PATH=$PATH:/home/www/.python/bin
		alias python='python3.8'
		export PATH=$PATH:/home/www/.python/bin/easy_install-3.8
		alias easy_install='easy_install-3.8'

		. ~/.bashrc
				
3. Cоздаем пользователя для старта django-приложения, обязательно добавляем его в группу sudo.

		easy_install virtualenv
		adduser django
		adduser django sudo
		login django
		
	Повторим пункт 2 для этого пользователя - допишем в конфигурацию алиасы и патчи, чтобы вызывался последний Python.

4. Под юзером django установим среду виртуализации и клонируем проект с гитхаб. Очень важно установить среду виртуализации такой же версии, что и ваш Python.

		python3.8 -m venv env		
		git clone https://github.com/trystep/postindex.git
		
	Сейчас у вас должна быть такая структура папок: в /home/django находятся 2 папки - env и папка с вашим проектом. Проект в этом примере называется postindex.

		source env/bin/activate
		cd postindex/project/
		
	Очень важно пользоваться последней версией pip

		python3.8 -m pip install --upgrade pip
		pip install uwsgi
		
	requirements.txt - это список зависимостей вашего проекта. Он создается в PyCharm командой `pip freeze > requirements.txt`. Командой ниже мы установим все зависимости для проекта.

		pip install django gunicorn psycopg2-binary ; \
		pip install -r requirements.txt
		
5. Запускаем uWSGI. Если у вас нет файла test_uwsgi_deploy.py, то вы его можете создать автоматически набрав команду в PyCharm `manage.py check --deploy`.

		uwsgi --http-socket :9090 --wsgi-file test_uwsgi_deploy.py	

6. Сейчас перейдя на сайт по порту 9090, то есть пройдя в браузере по адресу ваш_домен:9090, вы должны увидеть "Hello world".
Если все работает, нажимаем ctrl-c, завершая процесс и продолжаем дальше.
						
##########################################################################################

7. Устанавливаем и конфигурируем последний PostgreSQL.

	Сначала необходимо узнать кодовое имя дистрибутива

		lsb_release -c

		Codename:   stretch
		
	потом добавить репозиторий в список доступных. Для этого в файл

		vim /etc/apt/sources.list.d/pgdg.list
		
	надо добавить строку описания репозитория

		deb http://apt.postgresql.org/pub/repos/apt/ stretch-pgdg main
		
	где stretch - кодовое имя дистрибутива.

	Потом следует добавить публичный ключ репозитория в список разрешенных

		wget --quiet -O - http://apt.postgresql.org/pub/repos/apt/ACCC4CF8.asc wget --quiet -O - http://apt.postgresql.org/pub/repos/apt/ACCC4CF8.asc | sudo apt-key add -

	Затем обновим индекс доступных пакетов

		apt-get update

	Теперрь установим последнюю версию СУБД

		apt-get install -y postgresql-12

	
	PostgreSQL-12 готов к работе. Нужно создать БД и пользователя.
	
	Перейдите на пользователя root. Затем выполните вход в интерактивный сеанс Postgres, введя следующую команду:

		sudo -u postgres psql
		
	Вы увидите диалог PostgreSQL, где можно будет задать наши требования. Вначале создайте базу данных для своего проекта:

		CREATE DATABASE myproject;
		
	Примечание. Каждое выражение Postgres должно заканчиваться точкой с запятой. Если с вашей командой возникнут проблемы, проверьте это.

	Затем создайте пользователя базы данных для нашего проекта. Обязательно выберите безопасный пароль:

		CREATE USER myprojectuser WITH PASSWORD 'password';
		
	Затем мы изменим несколько параметров подключения для только что созданного нами пользователя. Это ускорит работу базы данных, поскольку теперь при каждом подключении не нужно будет запрашивать и устанавливать корректные значения.

	Мы зададим кодировку по умолчанию UTF-8, чего и ожидает Django. Также мы зададим схему изоляции транзакций по умолчанию «read committed», которая будет блокировать чтение со стороны неподтвержденных транзакций. В заключение мы зададим часовой пояс. По умолчанию наши проекты Django настроены на использование времени по Гринвичу (UTC). Все эти рекомендации взяты из проекта Django:

		ALTER ROLE myprojectuser SET client_encoding TO 'utf8';
		ALTER ROLE myprojectuser SET default_transaction_isolation TO 'read committed';
		ALTER ROLE myprojectuser SET timezone TO 'UTC';
	
	Теперь мы предоставим созданному пользователю доступ для администрирования новой базы данных:

		GRANT ALL PRIVILEGES ON DATABASE myproject TO myprojectuser;
		
	Завершив настройку, закройте диалог PostgreSQL с помощью следующей команды:

		\q
		
	Теперь настройка Postgres завершена, и Django может подключаться к базе данных и управлять своей информацией в базе данных.
		
##########################################################################################

8. Настроим nginx

		Создадим vim /etc/nginx/sites-available/project с содержимым:
			
			server {
				listen 80;
				server_name localhost;

				location / {
					uwsgi_pass unix:///tmp/main.sock;
					include uwsgi_params;
				}

				location /static/ {
					alias /var/www/postindex/project/static_final/;
				}
			}
		
		Создадим симлинк
			ln -s /etc/nginx/sites-available/project /etc/nginx/sites-enabled/project
		
##########################################################################################

9. Настроим uwsgi

		Создадим с vim /etc/uwsgi/apps-available/project.ini c содержанием:
		
		[uwsgi]
		vhost = true
		plugins = python
		socket = /tmp/main.sock
		master = true
		enable-threads = true
		processes = 4
		wsgi-file = /var/www/postindex/project/project/wsgi.py
		virtualenv = /var/www/venv/env
		chdir = /var/www/postindex/project
		touch-reload =- /var/www/postindex/project/relood
		env = DJANGO_ENV=production
		env DJANGO_ENV=production
		env DJANGO_NAME=mydb
		env DJANGO_USER=django
		env DJANGO_PASSWORD=password1
		env DJANGO_HOST=localhost
		env DJANGO_PORT=5432
		env ALLOWED_HOST=*
		
		cd /var/www
		chown -R www-data ./
		
		Сделаем симлинк
			ln -s /etc/uwsgi/apps-available/project.ini /etc/uwsgi/apps-enabled/
			
10. Перезапускаем uwsgi и nginx:

		service uwsgi restart
		service nginx restart

# Вот и все!)	
