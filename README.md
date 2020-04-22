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
		
