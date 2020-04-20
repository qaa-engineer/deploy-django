# deploy-django
Развертывание проекта Django c Github по шагам c нуля

<h3>Подготовка сервера</h3>

1. Вы арендовавали VPS и первый раз к ней подключаетесь по SSH. Я использую Ubuntu 18.04 LTS.
2. Обновляем и апгрейдим списки репозиториев:

        apt-get update && apt-get upgrade

3. Генерируем локали

        locale-gen en_US en_US.UTF-8 ru_RU.UTF-8

4. Устанавливаем Nginx, запускаем, смотрим статус

        apt-get install nginx
        service nginx start
        service nginx status

5. Если все ок, и он запущен можете открыть страничку вашего сайта и вы увидите стартовую страницу Nginx. Если же сервер не запустился, проверте, не запущен ли у вас Apache

        service apache2 status

6. Если он запущен стопайте его. Его необходимо полностью удалить.

        service apache2 stop

7. Деинсталируем апач и все его конфиги такой вот не хитрой командой:

        apt-get purge apache2 apache2-utils apache2.2-bin apache2-common

8. Если какие то пакеты, которые собираемся удалить отсутствуют, просто уберите их из команды. Далее:

        apt-get autoremove

9. Наконец, надо проверить наличие конфигурационных файлов или мануалов, связанных с Apache2, но до сих пор не удаленных.

        whereis apache2

10. Удаляете все эти директории вручную. Например, вот так:

        rm -rf /etc/apache2

11. Если все удалено, запускаем Nginx. И смотрим статус:

        service nginx start
        service nginx status
 
12. Ставим must-have packages

        sudo apt-get install -y vim mosh tmux htop git curl wget unzip zip gcc build-essential make zsh tree redis-server libssl-dev zlib1g-dev libbz2-dev libreadline-dev libsqlite3-dev llvm libncurses5-dev libncursesw5-dev xz-utils tk-dev libffi-dev liblzma-dev python3-dev python3-lxml python3-setuptools python3-pip libxslt-dev python-libxml2 python-libxslt1 libffi-dev libssl-dev python-dev gnumeric libsqlite3-dev libpq-dev libxml2-dev libxslt1-dev libjpeg-dev libfreetype6-dev libcurl4-openssl-dev virtualenv supervisor

13. Ставим Python3 из исходников

        mkdir ~/code ; \
        cd code ; \
        wget https://www.python.org/ftp/python/3.8.2/Python-3.8.2.tgz ; \
        tar xvf Python-3.8.* ; \
        cd Python-3.8.2 ; \
        mkdir ~/.python ; \
        ./configure --enable-optimizations --prefix=/home/www/.python ; \
        make -j8 ; \
        sudo make altinstall

14. Обновляем pip

        /home/www/.python/bin/python3.8 -m pip install -U pip
 
15. Прописываем путь до свежего Python3 и алис. Перезапускаем файлик, чтобы изменения применились.

        vim ~/.bashrc

        export PATH=$PATH:/home/www/.python/bin
        alias python='python3.8'

        . ~/.bashrc
        
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
34.
