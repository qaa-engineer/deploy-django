# deploy-django

# Развертывание Django проекта c Github по шагам на чистую VPS c Debian 10 с Nginx и uWSGI.

1. Обновляем списки репозиториев.

        	apt-get update

2. Пишем алис для Python3.

		vim ~/.bashrc
		
		alias python='python3.7'
		
		source ~/.bashrc

3. Устанавливаем необходимые пакеты:

		apt-get install -y python3-setuptools libpython3-dev python3-dev git nginx uwsgi uwsgi-plugin-python3 virtualenv python3-pip

4. Настроим среду вирутализации.

		cd /var/www
		mkdir venv
		cd venv/
		virtualenv env

		vim env/bin/activate
		и добавим в конец файла:
			export DJANGO_ENV=production
			export DJANGO_ENV=production
			export DJANGO_NAME=mydb
			export DJANGO_USER=django
			export DJANGO_PASSWORD=password1
			export DJANGO_HOST=localhost
			export DJANGO_PORT=5432
			export ALLOWED_HOST=*
		
		Активируем виртуализацию:
			source env/bin/activate
			cd ..
			git clone https://github.com/trystep/postindex.git
			cd postindex/project
			pip3 install -r requirements.txt
			apt-get install python3-psycopg2
			pip3 install django
			python manage.py syncdb
			python manage.py migrate
			python manage.py collectstatic
						
5. Устанавливаем и конфигурируем PostgreSQL.
		
		Обратите внимание, мы должны находиться сейчас в среде виртуализации.


		wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - ; \
		RELEASE=$(lsb_release -cs) ; \
		echo "deb http://apt.postgresql.org/pub/repos/apt/ ${RELEASE}"-pgdg main | sudo tee  /etc/apt/sources.list.d/pgdg.list ; \
		sudo apt update ; \
		sudo apt -y install postgresql-11 ; \
		sudo localedef ru_RU.UTF-8 -i ru_RU -fUTF-8 ; \
		export LANGUAGE=ru_RU.UTF-8 ; \
		export LANG=ru_RU.UTF-8 ; \
		export LC_ALL=ru_RU.UTF-8 ; \
		sudo locale-gen ru_RU.UTF-8 ; \
		sudo dpkg-reconfigure locales
		
		
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

6. Загрузим дамп вашей базы данных с локальной машины. Зайдите в программу pgAdmin - правая клавиша над нужной таблицей - импорт, загрузите этот файл на сервак, затем наберите команду.

		psql -h localhost dbms_db dbms  < my_import_db

7. Настроим nginx

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

8. Настроим uwsgi

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
			
9. Перезапускаем uwsgi и nginx:

		service uwsgi restart
		service nginx restart

10. Вот и все!)	
