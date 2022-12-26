# Развертывание готового проекта Django с Postgres, Nginx и Gunicorn на чистую VPS Debian 9

Мы настроим с нуля нашу машину, создадим базу данных PostgreSQL, выполним установку и настройку сервера приложений 
Gunicorn. Он послужит интерфейсом нашего приложения и будет обеспечивать преобразование запросов клиентов по протоколу 
HTTP в вызовы Python, которые наше приложение сможет обрабатывать. Затем мы настроим Nginx в качестве обратного 
прокси-сервера для Gunicorn, чтобы воспользоваться высокоэффективными механизмами обработки соединений и удобными 
функциями безопасности.

Мы будем устанавливать Django в виртуальной среде. Установка Django в отдельную среду проекта позволит отдельно 
обрабатывать проекты и их требования.

1. Устанавливаем must-have пакеты, размещаем Python из исходников в одну папку. Это займет примерно 20 минут.

		apt-get update && apt-get upgrade -y; \
		apt-get install -y vim mosh tmux htop git curl wget unzip zip gcc build-essential make libssl-dev zlib1g-dev 
		libbz2-dev libreadline-dev libsqlite3-dev llvm libncurses5-dev libncursesw5-dev xz-utils tk-dev libffi-dev 
		liblzma-dev python3-dev python3-lxml libxslt-dev python-libxml2 python-libxslt1 libffi-dev libssl-dev gnumeric 
		libsqlite3-dev libpq-dev libxml2-dev libxslt1-dev libjpeg-dev libfreetype6-dev libcurl4-openssl-dev nginx 
		gunicorn supervisor ; \
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

2. Допишем в конфигурацию алиасы и патчи, чтобы вызывался Python.
	
		vim ~/.bashrc

		export PATH=$PATH:/home/www/.python/bin
		alias python='python3.8'
		export PATH=$PATH:/home/www/.python/bin/easy_install-3.8
		alias easy_install='easy_install-3.8'

		. ~/.bashrc
				
3. Устанавливаем и конфигурируем PostgreSQL. Делается под рутом.

	Сначала необходимо узнать кодовое имя дистрибутива командой:

		lsb_release -c

	    Output:
	    Codename:   stretch - это кодовое имя дистрибутива, запомните его.
		
	потом добавим репозиторий в список доступных. Для этого в файл

		vim /etc/apt/sources.list.d/pgdg.list
		
	надо добавить строку описания репозитория

		deb http://apt.postgresql.org/pub/repos/apt/stretch-pgdg main
		
	где stretch - кодовое имя дистрибутива.

	Потом следует добавить публичный ключ репозитория в список разрешенных:

		wget --quiet -O - http://apt.postgresql.org/pub/repos/apt/ACCC4CF8.asc wget --quiet -O 
		- http://apt.postgresql.org/pub/repos/apt/ACCC4CF8.asc | sudo apt-key add -

	Затем обновим индекс доступных пакетов

		apt-get update

	Теперрь установим последнюю версию СУБД

		apt-get install -y postgresql-12

    PostgreSQL-12 готов к работе. Нужно создать БД и пользователя. Выполните вход в интерактивный сеанс Postgres, 
    введя следующую команду:

		sudo -u postgres psql
		
	Вы увидите диалог PostgreSQL, где можно будет задать наши требования. Вначале создайте базу данных для своего 
	проекта:

		CREATE DATABASE myproject;
		
	Каждое выражение Postgres должно заканчиваться точкой с запятой.

	Затем создайте пользователя базы данных для проекта. Обязательно выберите безопасный пароль:

		CREATE USER myprojectuser WITH PASSWORD 'password';
		
	Затем мы изменим несколько параметров подключения для только что созданного нами пользователя. Это ускорит работу 
	базы данных, поскольку теперь при каждом подключении не нужно будет запрашивать и устанавливать корректные значения.
	Мы зададим кодировку по умолчанию UTF-8, чего и ожидает Django. Также мы зададим схему изоляции транзакций по 
	умолчанию «read committed», которая будет блокировать чтение со стороны неподтвержденных транзакций. В заключение 
	мы зададим часовой пояс. По умолчанию наши проекты Django настроены на использование времени по Гринвичу (UTC). 
	Все эти рекомендации взяты из проекта Django:

		ALTER ROLE myprojectuser SET client_encoding TO 'utf8';
		ALTER ROLE myprojectuser SET default_transaction_isolation TO 'read committed';
		ALTER ROLE myprojectuser SET timezone TO 'UTC';
	
	Теперь мы предоставим созданному пользователю доступ для администрирования новой базы данных:

		GRANT ALL PRIVILEGES ON DATABASE myproject TO myprojectuser;
		
	Для вновь созданных таблиц полезна команда:
	
		GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO myprojectuser;
	
	Завершив настройку, закройте диалог PostgreSQL с помощью следующей команды:

		\q
		
	Теперь настройка Postgres завершена, и Django может подключаться к базе данных и управлять своей информацией в 
	базе данных.
	
	Сделем импорт базы данных с вашего локального компьютера. Чтобы это сделать проще, создадим файл с данными для 
	соединения с БД:
	
	    vim ~/.pgpass
	    
	с содержимым:
	
	    localhost:5432:myproject:myprojectuser:password
	    
	дадим права:
	
	    chmod 600 ~/.pgpass
	    
	До импорта таблицы с БД нужно сперва создать эту таблицу под пользователем, через которое будет работатт ваше 
	приложение. Для этого войдите в Postgres под пользователем:
	
	    psql -h localhost -U myprojectuser myproject
	    
	И тут создайте таблицу. Команды которые могут быть полезны:
	
	    \с myproject - соединиться с БД myproject
	    \d - посмотреть сущетсвующие таблицы
	    \l - посмотреть существующие БД
	    \q - выйти
	    
	Создание таблицы в моем случае:
	
	    CREATE TABLE postindex ( id SERIAL PRIMARY KEY, post_index_value integer, post_name text, post_type text, 
	    post_sub text, region text, autonomy text, area text, city text, city_1 text, actual_date date, index_old 
	    integer );
	    
	Затем выйдите и под рутом сделайте импорт в созданную таблицу:
	
	    sudo -u postgres
		psql
		COPY postindex FROM '/home/django/postindex/project/12' DELIMITER ';' NULL '\N' CSV HEADER;
		
	На этапе импорта БД могут возникнуть проблемы, так как ваша БД может не соответствовать требованиям psql, придется 
	эскпериментировать при подготовке БД с разделителями, пустыми строками, кодировками и т.д. Лучший помощник при этом:
	
	    https://postgrespro.ru/docs/postgrespro/12/sql-copy

	Как сделать бэкап БД:
		
		sudo -u postgres -i
		pg_dump db > db.sql
		
	Как развернуть бэкап:
		
		sudo -u postgres -i
		psql -d db -f db.sql

4. Установим virtualenv для виртуализации, создадим пользователя для django-приложения, обязательно добавляем его в 
группу sudo.

		easy_install virtualenv
		adduser django
		adduser django sudo
		login django
		
	Повторим пункт 2 для пользователя django - допишем в конфигурацию алиасы и патчи, чтобы вызывался 
	Python 
	под профилем пользователя.

5. Под юзером django установим среду виртуализации и клонируем проект с гитхаб. Очень важно установить среду 
виртуализации такой же версии, что и ваш Python.

		python3.8 -m venv env		
		git clone https://github.com/trystep/postindex.git
		
	Сейчас у вас должна быть такая структура папок: в /home/django находятся 2 папки - env и папка с вашим проектом. 
	Проект в этом примере называется postindex.
	
	Активриуем среду виртуализации:

		source env/bin/activate
		cd postindex/project/
		
	Очень важно пользоваться последней версией pip. Обновим pip и установим uwsgi - это веб-сервер и сервер 
	веб-приложений, первоначально реализованный для запуска приложений Python через протокол WSGI.

		python3.8 -m pip install --upgrade pip
		pip install uwsgi
	
	Установим зависимости нашего проекта и необходимые пакеты. requirements.txt - это список зависимостей вашего 
	проекта. Он создается в PyCharm командой `pip freeze > requirements.txt`. Командой ниже мы установим все зависимости 
	для проекта.

		pip install django gunicorn psycopg2-binary ; \
		pip install -r requirements.txt
		
6. Изменение настроек проекта.

	Откройте файл настроек в текстовом редакторе:
	
		vim /home/django/postindex/project/project/settings.py
		
	Найдите директиву ALLOWED_HOSTS. В квадратных скобках перечислите IP-адреса или доменные имена, связанные с вашим 
	сервером Django. Каждый элемент должен быть указан в кавычках, отдельные записи должны быть разделены запятой. 
	Если вы хотите включить в запрос весь домен и любые субдомены, добавьте точку перед началом записи. Обязательно 
	используйте localhost как одну из опций, поскольку мы будем использовать локальный экземпляр Nginx как прокси-сервер.
	
		ALLOWED_HOSTS = ['your_server_domain_or_IP', 'second_domain_or_IP', . . ., 'localhost']
		
	Затем найдите раздел, который будет настраивать доступ к базе данных. Он будет начинаться со слова DATABASES. 
	Измените настройки, указав параметры базы данных PostgreSQL. Мы укажем Django использовать адаптер psycopg2, 
	который мы установили вместе с pip.
	
		DATABASES = {
		    'default': {
			'ENGINE': 'django.db.backends.postgresql_psycopg2',
			'NAME': 'myproject',
			'USER': 'myprojectuser',
			'PASSWORD': 'password',
			'HOST': 'localhost',
			'PORT': '',
		    }
		}
		
	Затем перейдите в конец файла и добавьте параметр, указывающий, где следует разместить статичные файлы. 
	Это необходимо, чтобы Nginx мог обрабатывать запросы для этих элементов. Следующая строка указывает Django, 
	что они помещаются в каталог static в базовом каталоге проекта:
	
		STATIC_URL = '/static/'
		STATIC_ROOT = os.path.join(BASE_DIR, 'static/')
		
	Сохраните файл и закройте его после завершения.
    
    Теперь мы можем перенести начальную схему базы данных для нашей базы данных PostgreSQL, используя скрипт управления:

        cd /home/django/postindex/project/
        python manage.py makemigrations
        python manage.py migrate
        
    Создадим административного пользователя проекта с помощью следующей команды:

        python manage.py createsuperuser
        
    Вам нужно будет выбрать имя пользователя, указать адрес электронной почты, а затем задать и подтвердить пароль.

    Теперь мы можем собрать весь статичный контент в заданном каталоге с помощью следующей команды:

        python manage.py collectstatic
        
    Статичные файлы будут помещены в каталог static в каталоге вашего проекта.
    
    Теперь вы можете протестировать ваш проект, запустив сервер разработки Django с помощью следующей команды:

        python manage.py runserver 0.0.0.0:8000
        
    Откройте в браузере доменное имя или IP-адрес вашего сервера с суффиксом :8000:

        http://server_domain_or_IP:8000
        
    Вы увидите страницу вашего проекта.

7. Запускаем uWSGI. Если у вас нет файла test_uwsgi_deploy.py, то вы его можете создать автоматически набрав команду в 
PyCharm `python manage.py check --deploy`.

		uwsgi --http-socket :9090 --wsgi-file test_uwsgi_deploy.py	

    Сейчас перейдя на сайт по порту 9090, то есть пройдя в браузере по адресу ваш_домен:9090, вы должны увидеть 
    "Hello world".

8. Тестирование способности Gunicorn обслуживать проект.

	Перед выходом из виртуальной среды нужно протестировать способность Gunicorn обслуживать приложение. Для этого нам 
	нужно войти в каталог нашего проекта и использовать Gunicorn для загрузки модуля WSGI проекта, дав предварительно права на испольнение файлу wsgi.py:
		
		sudo chmod u+x /home/django/postindex/project/project/wsgi.py
		cd /home/django/postindex/project/
		gunicorn --bind 0.0.0.0:9090 project.wsgi
		
	Gunicorn будет запущен на том же интерфейсе, на котором работал сервер разработки Django. Теперь вы можете вернуться 
	и снова протестировать приложение, перейдя на сайт по порту 9090.

	Мы передали модуль в Gunicorn, указав относительный путь к файлу Django wsgi.py, то есть написав выше `project.wsgi`, 
	который представляет собой точку входа в наше приложение, где project - название нашего проекта. Для этого мы 
	использовали синтаксис модуля Python. В этом файле определена функция application, которая используется для 
	взаимодействия с приложением.

	После завершения тестирования нажмите CTRL+C в окне терминала, чтобы остановить работу Gunicorn.

	Мы завершили настройку нашего приложения Django. Теперь мы можем выйти из виртуальной среды:

		deactivate
		
	Индикатор виртуальной среды будет убран из командной строки.
	
9. Создание файлов сокета и служебных файлов systemd для Gunicorn.

    Мы убедились, что Gunicorn может взаимодействовать с нашим приложением Django, но теперь нам нужно реализовать более 
    надежный способ запуска и остановки сервера приложений. Для этого мы создадим служебные файлы и файлы сокета systemd.

    Сокет Gunicorn создается при загрузке и прослушивает подключения. При подключении systemd автоматически запускает 
    процесс Gunicorn для обработки подключения.

    Создайте и откройте файл сокета systemd для Gunicorn с привилегиями sudo:

        sudo vim /etc/systemd/system/gunicorn.socket
        
    В этом файле мы создадим раздел [Unit] для описания сокета, раздел [Socket] для определения расположения сокета и 
раздел [Install], чтобы обеспечить установку сокета в нужное время:

        [Unit]
        Description=gunicorn socket
        
        [Socket]
        ListenStream=/run/gunicorn.sock
        
        [Install]
        WantedBy=sockets.target

    Сохраните и закройте файл.

    Теперь создадим и откроем служебный файл systemd для Gunicorn в текстовом редакторе с привилегиями sudo. Имя файла 
    службы должно соответствовать имени файла сокета за исключением расширения:

        sudo vim /etc/systemd/system/gunicorn.service
        
    Начните с раздела [Unit], предназначенного для указания метаданных и зависимостей. Здесь мы разместим описание 
    службы и предпишем системе инициализации запускать ее только после достижения сетевой цели. Поскольку наша служба 
    использует сокет из файла сокета, нам потребуется директива Requires, чтобы задать это отношение:

        [Unit]
        Description=gunicorn daemon
        Requires=gunicorn.socket
        After=network.target

    Теперь откроем раздел [Service]. Здесь указываются пользователь и группа, от имени которых мы хотим запустить 
    данный процесс. Мы сделаем владельцем процесса учетную запись обычного пользователя, поскольку этот пользователь 
    является владельцем всех соответствующих файлов. Групповым владельцем мы сделаем группу django, чтобы Nginx мог 
    легко взаимодействовать с Gunicorn.

    Затем мы составим карту рабочего каталога и зададим команду для запуска службы. В данном случае мы укажем полный 
    путь к исполняемому файлу Gunicorn, установленному в нашей виртуальной среде. Мы привяжем процесс к сокету Unix, 
    созданному в каталоге /run, чтобы процесс мог взаимодействовать с Nginx. Мы будем регистрировать все данные на 
    стандартном выводе, чтобы процесс journald мог собирать журналы Gunicorn. Также здесь можно указать любые 
    необязательные настройки Gunicorn. Например, в данном случае мы задали 3 рабочих процесса:
    
        [Unit]
        Description=gunicorn daemon
        Requires=gunicorn.socket
        After=network.target
        
        [Service]
        User=django
        Group=www-data
        WorkingDirectory=/home/django/postindex/project
        ExecStart=/home/django/env/bin/gunicorn \
                  --access-logfile - \
                  --workers 3 \
                  --bind unix:/run/gunicorn.sock \
                  project.wsgi:application
          
    Наконец, добавим раздел [Install]. Это покажет systemd, куда привязывать эту службу, если мы активируем ее запуск 
    при загрузке. Нам нужно, чтобы эта служба запускалась во время работы обычной многопользовательской системы:

        [Unit]
        Description=gunicorn daemon
        Requires=gunicorn.socket
        After=network.target
        
        [Service]
        User=django
        Group=www-data
        WorkingDirectory=/home/django/postindex/project
        ExecStart=/home/django/env/bin/gunicorn \
                  --access-logfile - \
                  --workers 3 \
                  --bind unix:/run/gunicorn.sock \
                  project.wsgi:application
        
        [Install]
        WantedBy=multi-user.target

    Теперь служебный файл systemd готов. Сохраните и закройте его. Теперь мы можем запустить и активировать сокет 
    Gunicorn. Файл сокета /run/gunicorn.sock будет создан сейчас и будет создаваться при загрузке. При подключении к 
    этому сокету systemd автоматически запустит gunicorn.service для его обработки:

        sudo systemctl start gunicorn.socket
        sudo systemctl enable gunicorn.socket
        
    Успешность операции можно подтвердить, проверив файл сокета. Проверьте состояние процесса, чтобы узнать, удалось ли 
    его запустить:

        sudo systemctl status gunicorn.socket
        
    Затем проверьте наличие файла gunicorn.sock в каталоге /run:

        file /run/gunicorn.sock
        
        Output:
        /run/gunicorn.sock: socket
        
    Если команда systemctl status указывает на ошибку, или если в каталоге отсутствует файл gunicorn.sock, это означает, 
    что сокет Gunicorn не удалось создать. Проверьте журналы сокета Gunicorn с помощью следующей команды:

        sudo journalctl -u gunicorn.socket
        
    Еще раз проверьте файл /etc/systemd/system/gunicorn.socket и устраните любые обнаруженные проблемы, прежде чем 
    продолжить.

    Тестирование активации сокета. Если вы запустили только gunicorn.socket, служба gunicorn.service не будет активна в 
    связи с отсутствием подключений к совету. Для проверки можно ввести следующую команду:

        sudo systemctl status gunicorn
        
        Output
        ● gunicorn.service - gunicorn daemon
           Loaded: loaded (/etc/systemd/system/gunicorn.service; disabled; vendor preset: enabled)
           Active: inactive (dead)
           
    Чтобы протестировать механизм активации сокета, установим соединение с сокетом через curl с помощью следующей команды:

        curl --unix-socket /run/gunicorn.sock localhost
        
    Выводимые данные приложения должны отобразиться в терминале в формате HTML. Это показывает, что Gunicorn запущен и 
    может обслуживать ваше приложение Django. Вы можете убедиться, что служба Gunicorn работает, с помощью следующей 
    команды:

        sudo systemctl status gunicorn
        
        Output
        ● gunicorn.service - gunicorn daemon
           Loaded: loaded (/etc/systemd/system/gunicorn.service; disabled; vendor preset: enabled)
           Active: active (running) since Mon 2018-07-09 20:00:40 UTC; 4s ago
         Main PID: 1157 (gunicorn)
            Tasks: 4 (limit: 1153)
           CGroup: /system.slice/gunicorn.service
                   ├─1157 /home/sammy/myprojectdir/myprojectenv/bin/python3 /home/sammy/myprojectdir/myprojectenv/bin/gunicorn --access-logfile - --workers 3 --bind unix:/run/gunicorn.sock myproject.wsgi:application
                   ├─1178 /home/sammy/myprojectdir/myprojectenv/bin/python3 /home/sammy/myprojectdir/myprojectenv/bin/gunicorn --access-logfile - --workers 3 --bind unix:/run/gunicorn.sock myproject.wsgi:application
                   ├─1180 /home/sammy/myprojectdir/myprojectenv/bin/python3 /home/sammy/myprojectdir/myprojectenv/bin/gunicorn --access-logfile - --workers 3 --bind unix:/run/gunicorn.sock myproject.wsgi:application
                   └─1181 /home/sammy/myprojectdir/myprojectenv/bin/python3 /home/sammy/myprojectdir/myprojectenv/bin/gunicorn --access-logfile - --workers 3 --bind unix:/run/gunicorn.sock myproject.wsgi:application
        
        Jul 09 20:00:40 django1 systemd[1]: Started gunicorn daemon.
        Jul 09 20:00:40 django1 gunicorn[1157]: [2018-07-09 20:00:40 +0000] [1157] [INFO] Starting gunicorn 19.9.0
        Jul 09 20:00:40 django1 gunicorn[1157]: [2018-07-09 20:00:40 +0000] [1157] [INFO] Listening at: unix:/run/gunicorn.sock (1157)
        Jul 09 20:00:40 django1 gunicorn[1157]: [2018-07-09 20:00:40 +0000] [1157] [INFO] Using worker: sync
        Jul 09 20:00:40 django1 gunicorn[1157]: [2018-07-09 20:00:40 +0000] [1178] [INFO] Booting worker with pid: 1178
        Jul 09 20:00:40 django1 gunicorn[1157]: [2018-07-09 20:00:40 +0000] [1180] [INFO] Booting worker with pid: 1180
        Jul 09 20:00:40 django1 gunicorn[1157]: [2018-07-09 20:00:40 +0000] [1181] [INFO] Booting worker with pid: 1181
        Jul 09 20:00:41 django1 gunicorn[1157]:  - - [09/Jul/2018:20:00:41 +0000] "GET / HTTP/1.1" 200 16348 "-" "curl/7.58.0"
        
    Если результат вывода curl или systemctl status указывают на наличие проблемы, поищите в журналах более подробные 
    данные:

        sudo journalctl -u gunicorn
        
    Проверьте файл /etc/systemd/system/gunicorn.service на наличие проблем. 
    Если вы внесли изменения в файл /etc/systemd/system/gunicorn.service, перезагрузите демона, чтобы заново считать 
    определение службы, и перезапустите процесс Gunicorn с помощью следующей команды:

        sudo systemctl daemon-reload
        sudo systemctl restart gunicorn
        
    Обязательно устраните вышеперечисленные проблемы, прежде чем продолжить.
    
    Важно, если вы запускаете несколько проектов на одном сервере. Для меня это было неочевидно, но, оказывается все просто - нужно повторить этот пункт для другого вашего сайта, только надо выбрать другие названия для файлов, например gunicorn2.socket и gunicorn2.service.

10. Настройка Nginx как прокси для Gunicorn.

    Мы настроили Gunicorn, и теперь нам нужно настроить Nginx для передачи трафика в процесс. Для начала нужно создать и 
    открыть новый серверный блок в каталоге Nginx sites-available.
    
    Создадим новую конфигурацию
    
        vim /etc/nginx/sites-available/project
		        
	с содержимым:
			
        server {
                server_name ваш-домен.ru;
                
                listen ваш-IP:80;
                
                location = /favicon.ico {
                    access_log off; log_not_found off;
                }
                
                location /static/ {
                    root /home/django/postindex/project;
                }
            
                location / {
                    include proxy_params;
                    proxy_pass http://unix:/run/gunicorn.sock;
                }
        }
            
        server {
                server_name IP-сервера;
                listen ваш-IP:80;
            
                location / {
                    return 301 $scheme://ваш-домен.ru$request_uri;
                }
        }
		
	Создадим симлинк
	
	    ln -s /etc/nginx/sites-available/project /etc/nginx/sites-enabled/project
	    
	Протестируем конфигурацию Nginx на ошибки синтаксиса:
	
	    sudo nginx -t
            
    Если ошибок не будет найдено, перезапустите Nginx с помощью следующей команды:

        sudo systemctl restart nginx
	    
	Если будете менять конфигурацию, то чтобы применились новые изменения, наберите команду:
	
	    nginx -s reload

# Вот и весь деплой!)

Следующие журналы могут быть полезными:

    Проверьте журналы процессов Nginx с помощью команды: sudo journalctl -u nginx
    Проверьте журналы доступа Nginx с помощью команды: sudo less /var/log/nginx/access.log
    Проверьте журналы ошибок Nginx с помощью команды: sudo less /var/log/nginx/error.log
    Проверьте журналы приложения Gunicorn с помощью команды: sudo journalctl -u gunicorn
    Проверьте журналы сокета Gunicorn с помощью команды: sudo journalctl -u gunicorn.socket

При обновлении конфигурации или приложения вам может понадобиться перезапустить процессы для адаптации к изменениям.

Если вы обновите свое приложение Django, вы можете перезапустить процесс Gunicorn для адаптации к изменениям с помощью 
следующей команды:

    sudo systemctl restart gunicorn

Если вы измените файл сокета или служебные файлы Gunicorn, перезагрузите демона и перезапустите процесс с помощью 
следующей команды:

    sudo systemctl daemon-reload
    sudo systemctl restart gunicorn.socket gunicorn.service

Если вы измените конфигурацию серверного блока Nginx, протестируйте конфигурацию и Nginx с помощью следующей команды:

    sudo nginx -t && sudo systemctl restart nginx

Эти команды помогают адаптироваться к изменениям в случае изменения конфигурации.

Для добавления нового пользователя БД с правами только на чтение:
1) Создание группы:
CREATE ROLE user_group;

2) Создание пользователя:
CREATE ROLE user_db WITH LOGIN ENCRYPTED PASSWORD 'passdb';

3) Добавление пользователя в группу:
GRANT user_group TO user_db;

4) Выдача прав на подключение к БД:
GRANT CONNECT ON DATABASE server_DB TO user_group;

5) Выдача права на чтение (только на выполнение SELECT):
GRANT SELECT ON all tables IN schema public TO user_group;
