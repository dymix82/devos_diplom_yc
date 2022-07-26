# Дипломное задание по курсу «DevOps-инженер»
## Этапы выполнения:

### 1. Регистрация доменного имени

Зарегистрирован домен gavva.site  
![img.png](img.png)

### 2. Создание инфраструктуры
Описание инфраструктуры как код создано и выложено в git репозиторий   
  https://github.com/dymix82/devops_diploma_iac   
В качестве backend используется S3 Yandex storage:  
![img_12.png](img_12.png)
### 3. Установка Nginx и LetsEncrypt
1. Настроены все A-записи на внешний адрес этого сервера:  
![img_3.png](img_3.png)
2. Настроены все upstream для выше указанных URL
3. При разворачивании инфраструктуры автоматически создается сертификат:    
![img_7.png](img_7.png)
4. Стейджинг сертификатов задается переменной letsencrypt_staging: true/false
### 4. Установка кластера MySQL
1. MySQL работает в режиме репликации Master/Slave.
2. В кластере автоматически создаётся база данных c именем wordpress.
```bash
mysql> show replica status \G;
*************************** 1. row ***************************
             Replica_IO_State: Waiting for source to send event
                  Source_Host: db01.gavva.site
                  Source_User: repl
                  Source_Port: 3306
                Connect_Retry: 60
              Source_Log_File: mysql-bin.000002
          Read_Source_Log_Pos: 548903
               Relay_Log_File: db02-relay-bin.000005
                Relay_Log_Pos: 549119
        Relay_Source_Log_File: mysql-bin.000002
           Replica_IO_Running: Yes
          Replica_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Source_Log_Pos: 548903
              Relay_Log_Space: 549497
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Source_SSL_Allowed: No
           Source_SSL_CA_File:
           Source_SSL_CA_Path:
              Source_SSL_Cert:
            Source_SSL_Cipher:
               Source_SSL_Key:
        Seconds_Behind_Source: 0
Source_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Source_Server_Id: 1
                  Source_UUID: 1b93d15e-0aae-11ed-83d7-d00d6145a327
             Source_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
    Replica_SQL_Running_State: Replica has read all relay log; waiting for more updates
           Source_Retry_Count: 86400
                  Source_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Source_SSL_Crl:
           Source_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name:
           Source_TLS_Version:
       Source_public_key_path:
        Get_Source_public_key: 0
            Network_Namespace:
1 row in set (0.00 sec)

ERROR:
No query specified

mysql>
```
### 5. Установка WordPress
![img_4.png](img_4.png)
### 6. Установка Gitlab CE и Gitlab Runner
Интерфейс Gitlab доступен по https.  
В вашей доменной зоне настроена A-запись на внешний адрес reverse proxy:
https://gitlab.gavva.site (Gitlab)
![img_11.png](img_11.png)
  
При любом коммите в репозиторий с WordPress и создании тега (например, v1.0.0) происходит деплой на виртуальную машину.  
Для настройки gitlab-runner используется отдельный playbook  
https://github.com/dymix82/devops_diploma_iac/tree/main/gitrunner-playbook  
Создаем проект в Гитлаб и заливаем в него содержимое папки /var/www/gavva.site/wordpress
![img_17.png](img_17.png)

Cоздаем файлик test.txt
![img_16.png](img_16.png)
Файлик появляется  
![img_15.png](img_15.png)  
Без создания тега:  
![img_14.png](img_14.png)  
При создании тега:  
![img_13.png](img_13.png)  
Текст .gitlab-ci.yml
```yaml
stages:          # List of stages for jobs, and their order of execution
  - deploy

deploy-job:
  stage: deploy
  script:
    - echo "Some changes in repo detected..."
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add
    - if [ "$CI_COMMIT_TAG" = "" ] ; then echo "No tag - no deploy!";
      else cd ~/wp && rm -rf wordpress && git clone -b master git@gitlab.gavva.site:gitlab-instance-77f2ac07/wordpress.git; 
      bash ~/deploy.sh;
      fi
    - echo "The End"

```
Текст скрипта по deploy.sh
```bash
#!/usr/bin/env bash
sudo systemctl stop apache2
sudo rm -rf /var/www/gavva.site/wordpress
rm -rf ~/wp/wordpress/.git
sudo mv ~/wp/wordpress /var/www/gavva.site/wordpress
sudo chown -R www-data:www-data /var/www
sudo systemctl start apache2
```
###  7. Установка Prometheus, Alert Manager, Node Exporter и Grafana
1. На всех серверах установлен Node Exporter и его метрики доступны Prometheus
![img_10.png](img_10.png)
2. У Alert Manager есть необходимый набор правил для создания алертов.
![img_6.png](img_6.png)
![img_8.png](img_8.png)
3. В Grafana есть дашборд отображающий метрики из Node Exporter по всем серверам.
![img_9.png](img_9.png)