# deploy-django
Развертывание проекта Django c Github по шагам c нуля

1. Вы арендовавали VPS и первый раз к ней подключаетесь по SSH. Я использую Ubuntu 18.04 LTS.
2. Обновляем и апгрейдим списки репозиториев:

        apt-get update && apt-get upgrade

3. Генерируем локали

        locale-gen en_US en_US.UTF-8 ru_RU.UTF-8

4. Устанавливаем Nginx, запускаем, смотрим статус

        apt-get install nginx
        service nginx start
        service nginx status

5. Проверяем на наличие ошибок локали

        

6. 
