## Напишем сервис, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова. Файл и слово должны задаваться в /etc/sysconfig

- Для начала создаём файл с конфигурацией для сервиса в директории /etc/sysconfig - из неё сервис будет брать необходимые переменные

    vi /etc/sysconfig/watchlog
  
![](https://github.com/dimkaspaun/systemd/blob/main/screen/1.png)


- Затем создаем /var/log/watchlog.log и пишем туда строки на своё усмотрение, плюс ключевое слово **ALERT**
  
![](https://github.com/dimkaspaun/systemd/blob/main/screen/2.png)  

- Создадим скрипт vi /opt/watchlog.sh

![](https://github.com/dimkaspaun/systemd/blob/main/screen/3.png)
![](https://github.com/dimkaspaun/systemd/blob/main/screen/3_1.png)

> Команда logger отправляет лог в системный журнал

- Создадим юнит для сервиса watchlog - vi /etc/systemd/system/watchlog.service

![](https://github.com/dimkaspaun/systemd/blob/main/screen/4.png)

- Создадим юнит для таймера - vi /etc/systemd/system/watchlog.timer

![](https://github.com/dimkaspaun/systemd/blob/main/screen/5.png)
![](https://github.com/dimkaspaun/systemd/blob/main/screen/6.png)

- Затем достаточно только стартануть timer

```bash
systemctl start watchlog.timer
```
- И убедиться в результате
  
![](https://github.com/dimkaspaun/systemd/blob/main/screen/18.png)


## Из epel установим spawn-fcgi и перепишем init-скрипт на unit-файл. Имя сервиса должно также называться

- Устанавливаем spawn-fcgi и необходимые для него пакеты

```bash
yum install epel-release -y && yum install spawn-fcgi php php-cli mod_fcgid httpd -y
```
![](https://github.com/dimkaspaun/systemd/blob/main/screen/19.png)


> /etc/rc.d/init.d/spawn-fcg - cам Init скрипт, который будем переписывать  
> Но перед этим необходимо раскомментировать строки с переменными в /etc/sysconfig/spawn-fcgi

Он должен получится следующего вида:

![](https://github.com/dimkaspaun/systemd/blob/main/screen/20.png)

- А сам юнит файл будет следующего вида - vi /etc/systemd/system/spawn-fcgi.service

![](https://github.com/dimkaspaun/systemd/blob/main/screen/21.png)

- Убеждаемся что все успешно работает

```bash
systemctl start spawn-fcgi
systemctl status spawn-fcgi
```
![](https://github.com/dimkaspaun/systemd/blob/main/screen/22.png)
![](https://github.com/dimkaspaun/systemd/blob/main/screen/23.png)

## Дополним юнит-файл apache httpd возможностью запустить несколько инстансов сервера с разными конфигами

- Для запуска нескольких экземпляров сервиса будем использовать шаблон в конфигурации файла окружения - vi /etc/systemd/system/httpd@second.service /etc/systemd/system/httpd@first.service

```bash
[Unit]
Description=The Apache HTTP Server
After=network.target remote-fs.target nss-lookup.target
Documentation=man:httpd(8)
Documentation=man:apachectl(8)

[Service]
Type=notify
EnvironmentFile=/etc/sysconfig/httpd-%I
ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND
ExecReload=/usr/sbin/httpd $OPTIONS -k graceful
ExecStop=/bin/kill -WINCH ${MAINPID}
KillSignal=SIGCONT
PrivateTmp=true
[Install]
WantedBy=multi-user.target
```

![](https://github.com/dimkaspaun/systemd/blob/main/screen/29.png)

> Добавим параметр %I к EnvironmentFile=/etc/sysconfig/httpd

- В самом файле окружения (которых будет два) задается опция для запуска веб-сервера с необходимым конфигурационным файлом

/etc/sysconfig/httpd-first

```bash
OPTIONS=-f conf/first.conf
```

/etc/sysconfig/httpd-second

```bash
OPTIONS=-f conf/second.conf
```
![](https://github.com/dimkaspaun/systemd/blob/main/screen/25.png)

- Соответственно в директории с конфигами httpd должны лежать два конфига, в нашем случае это будут first.conf и second.conf

```bash
cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/first.conf                              
cp /etc/httpd/conf/httpd.conf /etc/httpd/conf/second.conf
```

> Для удачного запуска, в конфигурационных файлах должны быть указаны уникальные для каждого экземпляра опции **Listen** и **PidFile**.  

- Конфиги можно скопировать и поправить только второй, в нем должна быть следующие опции

```bash
PidFile /var/run/httpd-second.pid
Listen 8080
```
![](https://github.com/dimkaspaun/systemd/blob/main/screen/27.png)
- Запустим

```bash
systemctl start httpd@first
systemctl start httpd@second
```

- Проверить можно несколькими способами, например посмотреть какие порты слушаются

```bash
ss -tnulp | grep httpd
```
![](https://github.com/dimkaspaun/systemd/blob/main/screen/30.png)
