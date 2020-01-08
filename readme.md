#### Запуск виртуалок
Для запуска выполнить 
```
vagrant up
vagrant provision
```
Что бы получить ssh нужно выполнить `vagrant ssh jenkins`. Ожидается, что в папке с Vagrantfile 
существует папка с именем sync, она будет замаплена на каталог /vagrant на виртуалках.

После установки UI доступен через localhost:18080.
При первом входе будет запрошен пароль, который можно посмотреть командой
```
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
Затем вероятно возникнет сообщение "Offline. This Jenkins instance appears to be offline."
причиной которого являются просроченные сертификаты и как следствие неработающий https.
Лечится заменой https на http в теге url в файле /var/lib/jenkins/hudson.model.UpdateCenter.xml.
После это правки нужно рестартовать дженкинс `sudo service jenkins restart` и снова открыть интерфейс.

Появятся 2 кнопки выбор плагинов по умолчанию и избирательно, можно установить плагины по умолчанию. 
После нажатия нужно дождаться завершения загрузки.

Затем будет предложено создать пользователя - можно отказаться и продолжить под учеткой админа (паролем будет код подтвержения см.выше).
Что бы установить пароль, нужно перейти в раздел "Пользователи", выбрать в списке "admin", в меню "Настроить", 
ввести новый пароль и подтверждение и нажать сохранить - это потребует перелогиниться.


#### Настройка ssh на слейв-машине.
Дженскинс будет подключаться к целевой системе по ssh и выполнять в ней определенные действия.
Для этого ему нужно предоставить ssh-доступ по ключу.
Сначала нужно сгенерировать ключ, войти на машину с дженкинсом `vagrant ssh jenkins` и выполнить последовательность команд:
```
sudo cat /etc/passwd
```
Проверить имя пользователя под учеткой которого работает jenkins. Зайти под этим пользователем, и сгенерировать ssh-ключ:
```
sudo su jenkins
ssh-keygen
```
принять имя файла и пустую контрольную фразу <ENTER>, <ENTER>.
Затем можно выйти из под учетки jenkins, "забрать" сгененрированный открытый ключ, и выйти из шелла:
```
exit
cp /var/lib/jenkins/.ssh/id_rsa.pub /vagrant
logout
```

Теперь нужно подключиться к слейву `vagrant ssh slave`.\
Предполагается, что на слейве дженкинс будет работать под root.
Нужно перейти в root `sudo su root`, в домашнем каталоге создать папку `cd ~ && mkdir .ssh`, если её нет.
И скопировать в неё файл с открытым ключем из расшареной папки `cp /vagrant/id_rsa.pub ~/.ssh/authorized_keys`,
копия должна называться `authorized_keys`.\
Можно выйти из учетки и из шелла:
```
exit
exit
```

Для проверки подключаемся к другой виртуалке `vagrant ssh jenkins`. Перейти под учетку jenkins 
и попытаться подключиться по ssh:
```
sudo su jenkins
ssh root@192.168.33.20
```
При подключении возникнет запрос на подтверждение fingerprint, нужно ответить yes.
После этого должно произойти подключение с правами root. Это показатель успешной настройки.


#### Тестовый джоб.
Сейчас всё готово, что бы можно было создать "hello world" джоб через интерфейс дженкинса.\
Нужно зайти под админом, на основной вкладке будет отображаться предложение создать новый джоб.
После клика по кнопке отобразится экран на котором нужно ввести имя Item'a (джоба) например test-job,
и из вариантов выбрать "Создать задачу со свободной конфигурацией" и нажать "Ок".\
Отобразится экран со множеством панелей, на панели "Сборка" нужно добавить шаг "Выполнить команду shell".\
В поле ввести `ssh 192.168.33.20 'hostname'` - войти в систему slave под root и выполнить команду hostname.
Нажать "Сохранить", что переключит экран в раздел данного джоба. Здесть нужно выбрать "Собрать сейчас".\
Джоб начнет выполняться, на что указывает прогрессбар в левой части экрана. Через некоторое время "шарик" станет красным -
джоб завершился с ошибкой. Это произошло т.к. дженкинс пытался подключиться к удаленной машине под учеткой jenkins.\
Нужно исправить команду: `ssh root@192.168.33.20 'hostname'`, для чего перейти в меню "Настройки" и повторить 
шаги аналогиные созданию джоба.\
Повторить "Соборку" - теперь шарик станет синим, это говорит о том, что задача выполнена без ошибок.

**PS:** Шарик синий т.к. в японии вместо зеленого используется синий цвет светофора (что можно исправить плагином "Green balls").
Для установки плагина надо открыть меню "Настроить Jenkins" - "Управление плагинами" - "Доступные" и в списке отметить галочкой "Green Balls" (можно использовать фильтр),
нажать "Установить без перезагрузки", на следующем экране отметив "Перезапустить Jenkins по окончанию установки и отсутствии активных задач".
Во время установки интерфейс может зафризиться, поэтом через некоторое время можно попробовать нажать F5. Шарики должны стать зелеными.
