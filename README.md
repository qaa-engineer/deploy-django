# deploy-django

# Развертывание готового Django проекта c Github по шагам на чистую VPS Debian 9 с Nginx и uWSGI.

1. Устанавливаем must-have пакеты, размещаем последний Python из исходников в одну папку. Это займет примерно 30 минут.

		apt-get update ; \
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

	
2. Заходим на наш сайт и видим стартовую страницу nginx. Cоздаем пользователя для старта django-приложения, обязательно добавляем его в группу sudo.

		easy_install virtualenv
		adduser django
		adduser django sudo
		cd /home/django

3. Допишем в конфигурацию алиасы и патчы, чтобы вызывался последний Python. Это надо сделать для рута и пользователя django.
Заходим под рутом, делаем команды ниже. Затем заходим под пользователем django и делаем эти же команды. Дело в том, что у каждого отдельные файлы ~/.bashrc.
	
		vim ~/.bashrc

		export PATH=$PATH:/home/www/.python/bin
		alias python='python3.8'
		export PATH=$PATH:/home/www/.python/bin/easy_install-3.8
		alias easy_install='easy_install-3.8'

		. ~/.bashrc
		
4. Создаем файл test.py для проверки uwsgi. Это делается под рутом.

		vim /home/django/postindex/project/test.py

	c таким содержимым:

		def application(env, start_response):
			start_response('200 OK', [('Content-Type','text/html')])
			return [b"Hello World"]

5. Под юзером django установим среду виртуализации и клонируем проект с гитхаб.

		login django
		
Очень важно установить среду виртуализации такой же версии, что и ваш Python

		python3.8 -m venv env		
		git clone https://github.com/trystep/postindex.git
		
Сейчас у вас должна быть такая структура папок: в /home/django находятся 2 папки - env и папка с вашим проектом. Проект в этом примере называется postindex.

		source env/bin/activate
		cd postindex/project/
		
Очень важно пользоваться последней версией pip

		python3.8 -m pip install --upgrade pip
		pip install uwsgi
		
requirements.txt - это список зависимостей вашего проекта. Он создается в PyCharm командой pip freeze > requirements.txt. Командой ниже мы установим все зависимости для проекта.

		pip install -r requirements.txt
		
6. Запускаем uWSGI:

		uwsgi --http-socket :9090 --wsgi-file test.py	

7. Сейчас перейдя на сайт по порту 9090, то есть пройдя в браузере по адресу ваш_домен:9090, вы должны увидеть "Hello world".
Если все работает, нажимаем ctrl-c, завершая процесс и продолжаем дальше.
						
##########################################################################################

8. Устанавливаем и конфигурируем PostgreSQL.
		
		Обратите внимание, мы должны находиться сейчас в среде виртуализации.


		wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add - ; \
		RELEASE=$(lsb_release -cs) ; \
		echo "deb http://apt.postgresql.org/pub/repos/apt/ ${RELEASE}"-pgdg main | tee  /etc/apt/sources.list.d/pgdg.list ; \
		apt update ; \
		apt -y install postgresql-12 ; \
		localedef ru_RU.UTF-8 -i ru_RU -fUTF-8 ; \
		export LANGUAGE=ru_RU.UTF-8 ; \
		export LANG=ru_RU.UTF-8 ; \
		export LC_ALL=ru_RU.UTF-8 ; \
		locale-gen ru_RU.UTF-8 ; \
		dpkg-reconfigure locales
		
		
		vim /etc/profile
		Добавим следующие строки:
			export LANGUAGE=ru_RU.UTF-8
			export LANG=ru_RU.UTF-8
			export LC_ALL=ru_RU.UTF-8
		    
		    
		Изменим пароль для postges, создадим чистую базу данных с именем dbms_db под пользователем postgres:
			sudo passwd postgres
			su - postgres
			export PATH=$PATH:/usr/lib/postgresql/12/bin
			createdb --encoding UNICODE dbms_db --username postgres
			exit
		
		
		Тут мы опять перешли в среду виртуализации.
		Создадим пользователя БД с именем dbms и предоставим ему большие привилегии.
		psql — это интерактивный терминал PostgreSQL, в консоли будет так - postgres=#
		
			sudo -u postgres psql
			postgres=# ...
			create user dbms with password 'some_password';
			ALTER USER dbms CREATEDB;
			grant all privileges on database dbms_db to dbms;
			\c dbms_db
			GRANT ALL ON ALL TABLES IN SCHEMA public to dbms;
			GRANT ALL ON ALL SEQUENCES IN SCHEMA public to dbms;
			GRANT ALL ON ALL FUNCTIONS IN SCHEMA public to dbms;
			\q
			deactivate
			
		Тут мы оказались под рутом. Теперь мы можем проверить соединение.
		Создайте файл ~/.pgpass с логином и паролем для БД для быстрого подключения:
			vim ~/.pgpass
				localhost:5432:dbms_db:dbms:some_password
				
		Дайте права на выполнение и запустите соединение с БД.
			chmod 600 ~/.pgpass
			psql -h localhost -U dbms dbms_db
		
		Тут у вас должен получится примерно такой вывод на консоли:
			SSL-соединение (протокол: TLSv1.3, шифр: TLS_AES_256_GCM_SHA384, бит: 256, сжатие: выкл.)
			Введите "help", чтобы получить справку.
			dbms_db=>
			
		Выйдите на рута командой \q
		Поздравляю, соединение с БД прошло успешно. Далее самое главное - как сделать дамп с вашей локалки.

9. Загрузим дамп вашей базы данных с локальной машины. Зайдите в программу pgAdmin - правая клавиша над нужной таблицей - импорт, загрузите этот файл на сервак, затем наберите команду.

		psql -h localhost dbms_db dbms  < my_import_db

10. Настроим nginx

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

11. Настроим uwsgi

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
			
12. Перезапускаем uwsgi и nginx:

		service uwsgi restart
		service nginx restart

# Вот и все!)	
