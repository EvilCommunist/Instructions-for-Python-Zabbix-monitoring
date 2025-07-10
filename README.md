# Instructions-for-Python-Zabbix-monitoring
Содержит инструкцию к настройке мониторинга системы с помощью Zabbix через Python-скрипт [NTTData only].

1) <strong>Перенос скрипта на целевой хост:</strong><br>
   1.1) Взять исполняемый файл (*.py) из почты / корпоративной шары;<br>
   1.2) С помощью <i>WinSCP</i> перенести его с RDS на целевой хост в папку `.../zabbix_agentd.d`;<br>
   1.3) Сменить владельца на `root` или `zabbix` с правами `750` (владелец может всё, группа - читать и исполнять, остальные - ничего) [`sudo chown zabbix script.py`]; [`sudo chmod 750 script.py`].<br>
2) <strong>Подготовка системы к работе:</strong><br>
   2.1) Настроить заббикс-агента на хосте;<br>
Один из возможных вариантов (если файл не изменялся, то в районе 320 строчки будет происходить подключение дополнительных конфигурационных файлов):
```conf
# Default:
# Include=

Include=/etc/zabbix/zabbix_agentd.d/*.conf  # Здесь происходит подключение

# Include=/usr/local/etc/zabbix_agentd.userparams.conf
# Include=/usr/local/etc/zabbix_agentd.conf.d/
# Include=/usr/local/etc/zabbix_agentd.conf.d/*.conf
```
Содержание подключаемого файла:
```conf
####UserParameters for monitoring of SAP instance via OS utilities
##################################################################
#To monitor ABAP process status
UserParameter=abap_wp[*],python3 .../zabbix_agentd.d/script.py $1 $2 sapcontrol -f ABAPGetWPTable -un $3
#To monitor ABAP process queues fill level
UserParameter=queuelist[*],python3 .../zabbix_agentd.d/script.py $1 $2 sapcontrol -f GetQueueStatistic -un $3
#To monitor ABAP instance service status
UserParameter=proclist[*],python3 .../zabbix_agentd.d/script.py $1 $2 sapcontrol -f GetProcessList -un $3
```
После изменения конфигурации надо перезагрузить агента: `sudo systemctl restart zabbix-agent`<br>
   2.2) Настроить файл sudoers на хосте;<br>
Последний шаг настройки - изменение файла sudoers, для избежания передачи пароля через сервер zabbix.
```
### For SAP ###
%zabbix ALL=(SIDADM) NOPASSWD:/bin/csh -c <SAP_lib_addr>/sapcontrol -nr 00 -function GetProcessList
%zabbix ALL=(SIDADM) NOPASSWD:/bin/csh -c <SAP_lib_addr>/sapcontrol -nr 00 -function GetQueueStatistic*
%zabbix ALL=(SIDADM) NOPASSWD:/bin/csh -c <SAP_lib_addr>/sapcontrol -nr 00 -function ABAPGetWPTable*
%zabbix ALL=(SIDADM) NOPASSWD:/bin/csh -c <SAP_lib_addr>/sapcontrol -nr 00 -function ParameterValue*
%zabbix ALL=(SIDADM) NOPASSWD:/bin/csh -c <SAP_lib_addr>/msprot*
```
Стоит учесть, что адреса саповских библиотек зависят от версии ядра, для уточнения - обратиться к ответственному администратору.<br>
Таким образом, хост сконфигурирован и готов к мониторингу.

3) <strong>Создание шаблона на сервере заббикс:</strong><br>
