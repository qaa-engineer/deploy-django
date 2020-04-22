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

6. Извлекаем проект из репозитория Git, создаем и активируем виртуальную среду Python:

		cd code
		git clone https://github.com/trystep/postindex.git
		cd postindex/project
		python -m venv env
		. ./env/bin/activate
			
7. Устанавливаем и конфигурируем PostgreSQL. Обратите внимание, мы должны находиться сейчас в среде виртуализации.

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
			
		Поздравляю, соединение с БД прошло успешно. Далее самое главное - как сделать дамп с вашей локалки.

8. Загрузим дамп вашей базы данных с локальной машины.
		
