# README
Данный плейбук выполняется на **master-машине**, на которой установлен ансибл из под пользователя **ansible**. Для корректной работы плейбука у пользователя должен быть ssh ключ по пути `/home/ansible/.ssh/ansible_rsa.pub` (далее ключ для ansible-master).

Далее покупка и настройка сервера на примере РегРу.

# Покупка сервера

## Тип сервера
При покупке сервера выбираем категорию **Облачные серверы (KVM)**. Это сервера с почасовой оплатой, предварительно закидываем деньги на ЛС.

## Тарифы
* Cloud-2 20ГБ SSD 2 ядра 1024мб
* Cloud-3 40ГБ SSD 2 ядра 2048мб (для более ресурсозатратных проектов)

## Покупка
1. В личном кабинете выбираем **Все услуги**/**Облачные серверы (KVM)**
2. Нажимаем кнопку **Добавить сервер**
3. Выбираем вкладку **Только ОС** и операционную систему **Ubuntu 16.04 LTS**
4. Выбираем тариф **Cloud-2** или **Cloud-3**
5. Указываем название сервера, желательно информативное, например, **ProjectName Staging** или **ProjectName Production**
6. Включаем **Бекапы**. Резервная копия от провайдера.
7. Выбираем **Ansible-master** SSH ключ (загорается зеленым цветом). Если ключ отсутсвует, то нажимамем кнопку **Добавить ключ** и вводим следующие значения
**Имя SSH-ключа** `Ansible-master`, **SSH-ключ** 
> Нужно обязательно убедиться, что ключ выбран для добавления, а не просто появился в списке доступных
{.is-warning}
8. Нажимаем кнопку **Создать сервер**

# Настройка сервера

## Коротко о сути, инструментах и терминах
Инфраструктура накатавыется на сервак с помощью Ansible ([Официальный сайт](https://www.ansible.com) и [Википедия](https://ru.wikipedia.org/wiki/Ansible)).
Для Ansible написан специальный [ansible playbook](https://docs.ansible.com/ansible/latest/user_guide/playbooks.html).
Инфраструктура для всех серверов будет накатываться по единому алгоритму, прописаному в playbook'е. Все уникальные данные для каждого проекта хранятся в соотвествующих inventory (подробнее [здесь](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html) и [здесь](https://docs.ansible.com/ansible/2.3/intro_inventory.html)).

## Необходимые данные
* Доступ на Master-Ansible машину;
* Доступ к настройкам репозитория проекта (права админа);
* IP-адрес сервера;
* Окружение вашего приложения (staging, production и тд.);
* Домен, для подключения SSL HTTPS (если не указан, то доступ будет только по IP);
* Bitbucket Pipelines SSH ключ;
* Данные для БД (название, админ и пароль);
* Тип базы данных (mongo, postgresql), если присутствует база;
* Секретные данные (пароль, соль) для расшифровки зашифрованных файлов проекта с КВД;
* Название проекта.
 
### Машина Master-Ansible
Это наша основная машина, с которой мы накатываем инфраструктуру для серверов из под пользователя **ansible**. 
> Требуется убедится что у вас есть доступ туда (спрашивайте у главных)
{.is-warning}

Конфиг для подключения по SSH:
```
Host master-ansible 192.168.13.13
    HostName 192.168.13.13
    IdentityFile <путь до вашего ключа>
    User ansible
```

Заходим на машину под пользователем **ansible**. В папке пользователя находится папка **playbooks** в которой лежит наш playbook **bazuka.yml** и все inventory проектов.

В папке **.ssh** пользователя лежит ssh ключ **ansible_rsa.pub**, который требуется прокидывать на купленные серваки (желательно сразу в момент покупки, у рег.ру есть такая возможность).

Также в файле **/home/ansible/.ssh/config** находятся конфиги подключения по ssh для всех серверов.

## Начинаем
### 1. Bitbucket Pipelines SSH ключ
В репозитории вашего сервера, нужно перейти в **Настройки (Settings)**. Далее переходим в раздел **Pipelines/Settings** и включаем тумблер **Enable Pipelines**.
Далее переходим в **Pipelines/SSH keys** и генерим ssh ключ, нажимаем на кнопку **Generate keys**. Полученный ключ сохраняем на hansapp-ansible машину в папку **/home/ansible/.ssh** c названием `pipeline-<project_name>.pub`. Далее нам потребуется указать данный ключ в invenotry проекта (более подробно будет в следующих пунктах).

### 2. Создаем новую inventory для проекта
Клоним себе **локально** проект репозиторий с плейбуком и заходим в него.
В папке **inventories** создаем новую папку под inventrory нового проекта. Всего потребуется создать **3 файла**.

Также для локальной работы с Ansible нам потребуется установить его.
* [Installation Guide](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html);
* [Install Ansible on Mac OSX](https://hvops.com/articles/ansible-mac-osx/).

Для macOS с установленным homebrew выполняем команду
```
brew install ansible
```

#### 2.1 Файл hosts
В созданной папке создаем файл **hosts**
> Файл **hosts** должен быть без расширения
{.is-warning}

В этом файлике указывается сервер на который мы будем накатывать инфраструктуру для определенного окружения (staging, production) нового сервера, а также домен (если имеется). Пример содержимого:
```
[hosts]
staging ansible_host=113.113.133.133 ansible_python_interpreter=/usr/bin/python3 application_env=staging domain_name=staging.example.ru
production ansible_host=144.144.144.144 ansible_python_interpreter=/usr/bin/python3 application_env=production domain_name=example.ru
```
> Домены указываются без **https://**
{.is-danger}

> **domain_name** не является обязательным параметром, и если не будет указан, то будет пропущена установка SSL HTTPS
{.is-info}

Берем код из примера выше и добавляем в созданный нами файл **hosts**. Если у вас отсутсвует staging или production окружение, просто удаляем соотвествующую строчку целиком.
Заменяем значения из примера на наши реальные значения.
```
ansible_host=<IP сервера>
```

```
domain_name=<Домен>
```

#### 2.2 Файл vars.yml
В созданной inventory папке проекта создаем папку **group_vars**, а в ней создаем папку **all**. Внутри папки **all** создаем файл **vars.yml**.

В этом файлике указываются уникальные данные проекта (ключ bitbucket pipelines, данные по БД, название проекта). Пример содержимого:
```
# Ansible superuser and superuser
superuser: ansible
supergroup: ansible

# Base vars
local_ssh_pub_key: /home/ansible/.ssh/ansible_rsa.pub
pipeline_ssh_pub_key: /home/ansible/.ssh/pipeline-example.pub

# Database initial vars
initial_db_name: example
initial_db_user: root
initial_db_user_password: "{{ vault_initial_db_user_password }}"

# Nginx inventory vars
# domain_name: specified in hosts # if not define - nginx apply default config (and no certbot)
server_name: example
letsencrypt_email: example@example.ru

# Switch between database type (to exec specific db role and proxy in nginx config)
# Values: mongodb/postgresql/none
database_type: mongodb

# Server application 
application_name: example
# application_env: specified in hosts
application_base_url: "{% if domain_name is defined %}{{ domain_name }}{% else %}http://localhost:3000{% endif %}"

enckey_pswd: "{{ vault_enckey_pswd }}"
enckey_salt: "{{ vault_enckey_salt }}"
```
Берем код из примера выше и добавляем в созданный нами файл **vars.yml**.
Заменяем значения из примера на наши реальные значения.
```
pipeline_ssh_pub_key: <Путь до ключа из шага 1.>
```

```
initial_db_name: <Название вашей БД>
initial_db_user: <Админ вашей БД>
```
> переменную **initial_db_user_password** оставляем как есть
{.is-danger}
```
server_name: <Название для вашего .conf файла с виртуальным хостом для nginx>
```
> переменную **letsencrypt_email** менять не обязательно
{.is-info}
```
database_type: <Тип вашей БД>
```
> если переменную **database_type** задать в **none** или вовсе не указывать, то БД не будет инициализироваться.
{.is-warning}
```
application_name: <Название проекта>
```
> переменные **enckey_pswd** и **enckey_salt** оставляем как есть
{.is-danger}

#### 2.3 Файл vault.yml
Рядом с файлом **vars.yml** создаем файл **vault.yml**.
В этом файлике хранятся все данные, которые не стоит хранить в открытом виде. Для этого они шифруются специальным инструментом под названием **vault** (подробнее [здесь](https://docs.ansible.com/ansible/latest/user_guide/vault.html)). Пример содержимого:
```
vault_initial_db_user_password: <Пароль админа БД>
vault_enckey_pswd: <Секрет для дешифрации КВД в проекте>
vault_enckey_salt: <Соль для дешифрации КВД в проекте>
```
Берем код из примера выше и добавляем в созданный нами файл **vault.yml**.
Заменяем значения из примера на наши реальные значения.

Далее нам требуется зашифровать данный файл, для этого выполняем команду
```
ansible-vault encrypt <путь до нашего файла vault.yml>
```
пример
```
ansible-vault encrypt inventories/myproject/group_vars/all/vault.yml
```
Командная строка попросит ввести вас пароль для того, чтобы зашифровать файл.

> Если вы забудите пароль, то нужно будет заного создавать файл и шифровать его с новым паролем.
{.is-danger}

Чтобы отредактировать ранее зашифрованный файл, выполняем команду:
```
ansible-vault edit <путь до нашего файла vault.yml>
```
и вводим пароль.

#### 2.4 Коммитим
Коммитим новые изменения в **develop** и потом в **master** (под чутким руководством старшего брата).
Для тестовых прогонов можно использовать vagrant.

### 3. Загружаем на Master-Ansible
Заходим на Master-Ansible машину и переходим в папку **/home/ansible/playbooks**. Забираем созданную инвентори командой
```
git pull --rebase
```

### 4. Подготавливаем подключение по SSH
Открываем файл **/home/ansible/.ssh/config** на редактирование и добавляем в конец следующий код:
```
Host <проект>-<staging/production/env> <IP адрес>
    HostName <IP адрес>
    IdentityFile /home/ansible/.ssh/ansible_rsa
    User root
```
пример
```
Host myproject-staging 113.113.113.113
    HostName 113.113.113.113
    IdentityFile /home/ansible/.ssh/ansible_rsa
    User root
```
сохраняем изменения в файле.

Пробуем подключиться. Для примера выше выполняем следующую команду:
```
ssh myproject-staging
```
вводим **yes** на подтверждение fingerprint.

Если вы успешно зашли по SSH, то все хорошо.
Если подключится не удалось, значит на сервер не был прокинуть ssh ключ **ansible_rsa.pub**. Требуется прокинуть и попробовать заного.

### 5. Запускаем
Для запуска процесса накатывания инфраструктуры переходим в папку **playbooks**
```
cd /home/ansible/playbooks
```
В папке выполняем команду
```
ansible-playbook -i inventories/<инвентори проекта>/ bazuka.yml --ask-vault-pass
```
пример
```
ansible-playbook -i inventories/myproject/ bazuka.yml --ask-vault-pass
```

Ansible спросит вас пароль от **vault.yml** файла (который вводили на шаге 2.3). Вводим пароль, после чего начинается процесс накатывания инфраструктуры.

> Последний шаг выполнится с ошибкой и тегом **ignoring**
**ЭТО НОРМАЛЬНО, ТАК И ЗАДУМАНО** :-)
(скачивание докер образов, установка первых слоев из докерфайла, установка npm и прочие штуки, чтобы не тратить ограниченные минуты битбакет пайплайна)
{.is-warning}

### 6. Настраиваем доступ с сервера к репотзиторию
После отработки playbook'а в папке **/home/ansible/playbooks/remote_keys** будет создана папка с проектом. В ней будет лежать ключи под конкретную среду (staging/production и тд). 

Например для **myproject** проекта и окружения **staging** будет создан ключ **/home/ansible/playbooks/remote_keys/myproject/staging.pub**.

Берем наш ключ, и прописываем его в репозитории bitbucket в разделе **Настройка(Settings)/Access keys**.

### 7. Bitbucket Pipelines SSH fingerprint
В настройках репозитория снова переходим в **Pipelines/SSH keys**, в секции **Known hosts** добавляем IP адрес сервера и нажимаем кнопку **Fetch**.

> Если не был выполнен шаг 1, то будет ошибка
{.is-warning}

Если все прошло успешно, то добавляем полученный fingerprint - в поле для ввода **Fingerprint** автоматически подставится полученное значение, а кнопка **Fetch** заменится на кнопку **Add host**, при нажатии на которую сохранится полученный fingerprint. 

### 8. Bitbucket Pipelines переменные
В настройках репозитория переходим в раздел **Pipelines/Deployments** и вводим следующие переменные (**для каждого окружения Staging/Production свои переменные**):
* **HOST** `<IP сервера>`
* **HOST_USER** `xander` *(позже возможно под каждое окружение будет свой пользователь)*

## Добавление нового окружения (production)
Если у вас уже есть staging окружение, которое работает, и вы хотите добавить еще и production, то требуется выполнить только следующие пукнты
* Добавить новый сервер в **существующий файл hosts** в inventory проекта, с соотвествующим IP адресом, доменом и **production** окружением (шаг 2.1);
* Закомитить в репозиторий и залить на машину Master-Ansible (шаг 2.4 и шаг 3);
* Добавляем новый конфиг для подключения по SSH для нового сервера (шаг 4);
* Заного запускаем накат инфраструктуры (шаг 5);
> На предыдущее окружение инфраструктура повторно накатываться не будет, просто пройдет проверка что на сервере инфраструктура соотвествует шаблону. Ничего поломаться не должно :-)
{.is-info}
* Настраиваем доступ с нового добавленного сервера к репотзиторию (шаг 6);
* Добавляем Bitbucket Pipelines SSH fingerprint для нового сервера (шаг 7);
* Добавляем переменные **HOST** и **HOST_USER** в раздел **Pipelines/Deployments** в настройках репозитория для нового окружения (шаг 8).

# Bitbucket-Pipeline
[Что такое Bitbucket Pipelines](https://confluence.atlassian.com/bitbucket/get-started-with-bitbucket-pipelines-792298921.html) - это сервис непрерывной интеграции и деплоя, встроенный в Bitbucket. Позволяет автоматически собирать, тестировать и деплоить код на основании файла конфигурации, который хранится в репозитории.
## Подключаем
В репозитории вашего сервера, нужно перейти в **Настройки (Settings)**. Далее переходим в раздел **Pipelines/Settings** и включаем тумблер **Enable Pipelines**.

> Кнопку **Configure bitbucket-pipelines.yml** нажимать не обязательно, мы будем создавать файл вручную с нужной нам конфигурацией
{.is-info}

## Создаем файл конфигурации
В корне вашего проекта создаем файл **bitbucket-pipelines.yml**.
Копируем в созданный файл следующий код конфигурации
```
pipelines:
  branches:
    develop:
      - step:
          deployment: staging
          script:
            - ssh $HOST_USER@$HOST "bash ~/deploy.sh $BITBUCKET_GIT_SSH_ORIGIN $BITBUCKET_BRANCH"
    master:
      - step:
          deployment: production
          script:
            - ssh $HOST_USER@$HOST "bash ~/deploy.sh $BITBUCKET_GIT_SSH_ORIGIN $BITBUCKET_BRANCH"
```
Что содержит в себе данный конфиг:
* пайплайн будет запускаться только при коммитах/мерджах в ветки **develop** и **master**;
* для каждой ветки переменные HOST и HOST_USER будут иметь свое уникальное значение (тег **deployment**, настраивается в **Settings/Pipelines/Deployments**);
* пайплайн запускает выполнение деплой скрипта на указаном в **Deployments** удаленном хосте из под указанного пользователя с параметрами url репозитория и git ветки, которая запустила пайплайн.