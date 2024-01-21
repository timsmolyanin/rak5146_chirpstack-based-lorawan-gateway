# Запуск Chirpstack LoRaWAN Network сервера на WirenBoard на базе концентратора RAK5146 USB
Общая структура необходимых сервисов для поддержки LoRaWAN выглядит так:
![[lorawan_chirpstack_architecture.png]]
Ниже будет приведено что нужно ставить и как все это запускать
## UDP Packet Forwarder
### Сборка сервиса из исходников
Для удобства в корне создается папка *lorawan*, куда будут складываться все необходимые исходники перечисленных сервисов для их дальнейшей установки.
С данного [репозитория](https://github.com/Lora-net/sx1302_hal.git) необходимо забрать исходники сервиса:
```bash
git clone https://github.com/Lora-net/sx1302_hal.git
```

Затем, если еще не установлены инструменты для сборки приложений, то надо установить их:
```bash
apt install build-essential zlib1g-dev libncurses5-dev libgdbm-dev libnss3-dev libssl-dev libreadline-dev libffi-dev
```
Переходим в директорию с исходниками сервиса *sx1302_hal* и вводим команду:
```bash
make
```
Должен начаться процесс сборки, в результате в директории *sx1302_hal/packet_forwarder* должен появиться исполняемый файл *lora_pkt_fwd*.
Если во время сборки случились ошибки или еще что-то, то действовать в зависимости от полученных ошибок, т.к. пока на текущий момент никаких ошибок получено не было на этом этапе.

### Запуск
Если модуль концентратор еще не вставлен в USB разъем - необходимо вставить его. Далее необходимо создать файл конфигурации *global_conf.json* В директории *sx1302_hal/packet_forwarder* лежат некотрое кол-во примеров такого файла для разных частот, интерфейсов (USB, SPI), на основе которых можно создать файл конфигурации. Конфигурация лежит на гитхаб. 
**Есть вопросы по созданию файла, по нескольким пунктам, особенно где узнать, найти, или как создать *GATEWAY_ID*?**
После создания конфигурационного файла его надобно положить в директорию *sx1302_hal/packet_forwarder*.
Далее можно пробовать запустить сервис *lora_pkt_fwd*:
```bash
./lora_pkt_fwd
```
Если запускать без наличия конфигурационного файла в указанной директории, то сервис будет ругаться:
```bash
ERROR: [main] failed to find any configuration file named global_conf.json
```
Ну а если никаких ошибок не возникло, то сервис будет выводить разного рода отладочную информацию, типа:
```bash
### Concentrator temperature: 28 C ###
##### END #####

JSON up: {"stat":{"time":"2024-01-20 17:43:15 GMT","rxnb":1,"rxok":1,"rxfw":1,"ackr":0.0,"dwnb":0,"txnb":0,"temp":27.6}}
```
Если такого рода сообщения появились, то значит все прошло успешно.
Можно завернуть данный сервис в *systemd*, чтобы он запускался автоматически с запуском системы.
```bash
[Unit]
Description=LoRa UDP Packet Forwarder Service
After=multi-user.target

[Service]
User=root
Type=simple
Restart=always
ExecStart=/root/lorawan/sx1302_hal/packet_forwarder/lora_pkt_fwd -c /root/lorawan/sx1302_hal/packet_forwarder/global_conf.json

[Install]
WantedBy=multi-user.target
```
На гитхаб лежит данный файл. Его надо положить в /etc/systemd/system
Затем выполнить ряд команд:
```bash
systemctl daemon-reload
```
```bash
systemctl enable lora_pkt_forwarder
systemctl start lora_pkt_forwarder
```
Ну и проверить, что сервис запустился:
```bash
systemctl status lora_pkt_forwarder
```
Если все хорошо, должно быть в выводе что-то такое:
```bash
lora_pkt_forwarder.service - LoRa UDP Packet Forwarder Service
     Loaded: loaded (/etc/systemd/system/lora_pkt_forwarder.service; enabled; vendor preset: enabled)
     Active: active (running) since Sat 2024-01-20 17:54:50 UTC; 4min 12s ago
   Main PID: 29435 (lora_pkt_fwd)
      Tasks: 6 (limit: 2354)
     Memory: 308.0K
        CPU: 8.067s
     CGroup: /system.slice/lora_pkt_forwarder.service
             └─29435 /root/lorawan/sx1302_hal/packet_forwarder/lora_pkt_fwd -c /root/lorawan/sx1302_hal/packet_forwarder/global_conf.json
```
На этом установка LoRa UDP Packet Forwarder сервиса завершается.

## Chirpstack MQTT Forwarder
Данный сервис необходим для перенаправления данных из LoRa UDP Packet Forwarder сервиса в MQTT брокер. На Wirenboard уже установлен MQTT брокер Mosquito, поэтому нет необходимости его ставить.

Первым делом скачиваем архив:
```bash
wget https://artifacts.chirpstack.io/downloads/chirpstack-mqtt-forwarder/chirpstack-mqtt-forwarder_4.1.3_linux_armv7hf.tar.gz
```
Архив разархивируем:
```bash
tar -xvzf chirpstack-mqtt-forwarder_4.1.3_linux_armv7hf.tar.gz
```
В результате получаем исполняемый файл *chirpstack-mqtt-forwarder*. Для его запуска ему нужен конфигурационный файл *chirpstack-mqtt-forwarder.toml*. Получить информацию об нем, как его заполнять можно [здесь](https://www.chirpstack.io/docs/chirpstack-mqtt-forwarder/configuration.html). Конфигурационный файл можно также получить на гитхаб.
Этот файл нужно положить в папку с исполняемым файлом и затем попробовать запустить:
```bash
./chirpstack-mqtt-forwarder
```
На выходе в случае успеха сервис будет выводить отладочную инфу, где можно будет увидеть, что сервис получает ответы от LoRa UDP Packet Forwarder, и что он смог подключиться к MQTT брокеру.
Его также можно добавить в *systemd*, файл сервиса лежит на гитхаб.
После добавления и запуска сервиса, если при проверке статуса все хорошо, то значит сервис работает и на этом его установка и запуск закончены.
## Chirpstack LoRaWAN Network Server
Для начала нужно поставить несколько пакетов (mqtt инструменты нам не нужны, они уже есть):
```bash
sudo apt install redis-server redis-tools postgresql
```
### PostgreSQL setup
Заходим в PostgreSQL:
```bash
sudo -u postgres psql
```

И далее вводим следующее:
```sql
-- create role for authentication
create role chirpstack with login password 'chirpstack';

-- create database
create database chirpstack with owner chirpstack;

-- change to chirpstack database
\c chirpstack

-- create pg_trgm extension
create extension pg_trgm;

-- exit psql
\q
```
### Setup software repository
Далее нужно установить какую-то штуку:
```bash
apt install apt-transport-https dirmngr
```
Установить ключ для репозитория ChirpStack:
```bash
apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 1CE2AFD36DBCCA00
```
Добавить репозиторий в список:
```bash
echo "deb https://artifacts.chirpstack.io/packages/4.x/deb stable main" | sudo tee /etc/apt/sources.list.d/chirpstack.list
```
Обновить кэш apt:
```bash
apt update
```

### Install ChirpStack
Собственно, устанавливаем Chirpstack из репозитрия:
```bash
apt install chirpstack
```
Запускаем Chirpstack:
```bash
# start chirpstack
systemctl start chirpstack

# start chirpstack on boot
systemctl enable chirpstack
```
Поглядеть логи:
```bash
journalctl -f -n 100 -u chirpstack
```
_______
# Materials
https://docs.loriot.io/display/LNS/Packet+Forwarder+Semtech
https://www.chirpstack.io/docs/getting-started/debian-ubuntu.html