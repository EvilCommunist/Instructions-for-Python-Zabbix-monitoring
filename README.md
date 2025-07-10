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
#To monitor ABAP system parameters
UserParameter=proclist[*],python3 .../zabbix_agentd.d/script.py $1 $2 sapcontrol -f ParameterValue -p $3 -un $4
#To monitor hardware key
UserParameter=proclist[*],python3 .../zabbix_agentd.d/script.py $1 $2 msprot -un $3
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
   3.1) Создание нового шаблона;<br>
![image](https://github.com/user-attachments/assets/d1e9080a-7441-40ef-a56f-670dc189494c)
   3.2) Создание в шаблоне новых айтемов согласно юзерпараметрам, прописанным на хосте/хостах;<br>
Для начала надо зайти в айтемы шаблона:<br>
![image](https://github.com/user-attachments/assets/ae61fae8-0bd2-492d-83b0-efb431864b7b)<br>
После чего необходимо создать новый айтем:<br>
![image](https://github.com/user-attachments/assets/de469696-fbeb-42ca-a857-72a2910620bb)<br>
Здесь есть два ключевых момента:<br>
I. Аргументы передаются позиционно согласно юзерпараметру. Для того, чтобы узнать как правильно вызывать скрипт можно вызвать к нему справку с помощью команды `sudo python3 script.py --help` (совет для понимания);<br>
II. Для унификации шаблона все данные будут передаваться в макросах (записи вида `{$MACROS_NAME}` - все буквы пишутся капсом, условность заббикса).<br>
II.I) Подпункт - все макросы необходимо инициализировать в шаблоне:<br>
![image](https://github.com/user-attachments/assets/c1075169-6b76-4a80-9eb2-ed72526d75e6)<br>
<strong><i>ВАЖНО:</i></strong> значения, получаемые из `UserParameter` и `msprot` являются конечными значениями и их можно сразу же поставить на мониторинг, в то время как остальные три команды выдают в качестве результата JSON-значение, содержащее значимую информацию. Таким образом, для `ABAPGetWPTable`; `GetPocessList`; `GetQueueStatistics` необходимо провести извлченеие информации из полученного JSON-а. Для этого необходимо использовать low-level disovery.
   3.3) Создание low-level discovery;<br>
   lld должны создаваться согласно виду возвращаемых данных:<br>
   <ul>
   <li>GetQueueStatistics: {"TYP": "ABAP/NOWP", "NOW": "0", "HIGH": "13"}</li>
   <li>GetProcessList: {"DESCRIPTION": "Dispatcher", "DISPSTATUS": "GREEN"}</li>
   <li>ABAPGetWPTable: {"TYP": "DIA", "STATUS": "Wait", "NUMBER": 10}</li>
   </ul>
   
   На шаблоне надо нажать на `Discovery`, а затем - `Create discovery rule`.<br>
   ![image](https://github.com/user-attachments/assets/3792f398-5f04-4886-a476-5ccea2758ff1)<br>
   ![image](https://github.com/user-attachments/assets/45a22ec2-0ab0-4ade-9f26-8ee5b5b24db0)<br>
   Далее настройка:<br>
   ![image](https://github.com/user-attachments/assets/ccbe7144-077f-4752-86da-a7192af68a36)<br>
   ![image](https://github.com/user-attachments/assets/2176ad6b-b7d9-4822-ac56-59f60ffcdefa)<br>
   И создание прототипа айтема для работы внутри lld:<br>
   ![image](https://github.com/user-attachments/assets/e96bfbb8-b727-4b44-853b-70567130d81d)<br>
   ![image](https://github.com/user-attachments/assets/4770a1bc-bf70-4bb6-8b2f-0f8d909a08d2)<br>
   ![image](https://github.com/user-attachments/assets/61f9c3be-b329-434d-bf88-807e790a6d4f)<br>
   Препроцессинг позволяет извлечь из строки JSON нужные значения.<br>
   ABAPWP - `$.[?(@.TYP=='{#TYP}' && @.STATUS=='{#STATUS}')].NUMBER.first()`
   








