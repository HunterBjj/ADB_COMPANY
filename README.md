# <p align="center"> ![image](https://github.com/user-attachments/assets/1d8f232a-4fcd-4757-9c4f-62e51a826186) </p>
# Тестовое задание. Компания АДВ. 
# Описание задачи
https://docs.google.com/document/d/14eteN_WxFdtAqgPg2TKQNoULKOrBxomMo9mvjijkgcM/edit?tab=t.0 


```
import requests
import json
from datetime import timedelta, date

token = 'XXX'

counter_id = 1234567
url = f'https://api-metrika.yandex.net/management/v1/counter/{counter_id}/logrequests'

source = 'visit'

visit_fields = [
   "ym:pv:visitID",
   "ym:pv:date",
   "ym:pv:pageViews"
]
fields = ",".join(visit_fields)  

start_date = (date.today() - timedelta(days=1)).strftime("%Y-%m-%d")  
end_date = date.today().strftime("%Y-%m-%d")

params = {
   "source": source,
   "fields": fields,
   "date1": start_date,  
   "date2": end_date
}

headers = {'Authorization': f'OAuth {token}'}  

r = requests.post(url, json=params, headers=headers)  

if r.status_code == 200:
   response_data = r.json()
   if 'log_request' in response_data:
       request_id = response_data['log_request']['request_id']
   else:
       print('Unexpected response format:', response_data)
else:
   print(f'Error {r.status_code}: {r.text}')
```
Исправления:

Неправильный URL (не форматировался counter_id).

Неправильный разделитель полей (; → ,).

Ошибка в датах (start_date был позже end_date).

API использует date1 и date2, а не start_date и end_date.

Ошибка в заголовке авторизации (OAut → OAuth).

Запрос должен быть POST, а не GET.

```
Решение проблемы зависших процессов Python
Так как мы не можем изменить код Программы X, необходимо создать механизм, который будет отслеживать зависшие Python-процессы и завершать их.

Подход 1: Использование psutil для отслеживания зависших процессов
Python-скрипт можно модифицировать так, чтобы он проверял, есть ли его родительский процесс (Программа X) в системе. Если родительский процесс завершился, то скрипт завершает сам себя.

Как это сделать:

В начале выполнения Python-скрипт получает PID родительского процесса (os.getppid()).

Периодически проверяет, существует ли этот PID в системе.

Если родительский процесс исчез, скрипт сам себя завершает.

Пример кода:

import os
import psutil
import time

PARENT_PID = os.getppid()  # Получаем PID родителя

def is_parent_alive():
    return psutil.pid_exists(PARENT_PID)

try:
    while True:
        if not is_parent_alive():
            print("Parent process is gone. Exiting...")
            break
        time.sleep(5)  # Проверяем каждые 5 секунд
except Exception as e:
    print(f"Error: {e}")
finally:
    os._exit(0)  # Завершаем процесс
✅ Плюсы: Решение работает изнутри скрипта, легко реализовать.
❌ Минусы: Если скрипт зависнет до выполнения проверки, он не завершится.

Подход 2: Airflow Task с мониторингом зависших процессов
Airflow можно настроить так, чтобы он периодически проверял и убивал зависшие процессы Python.


Создать отдельный DAG в Airflow, который раз в 5-10 минут проверяет, есть ли процессы Python, у которых нет родителя (PPID = 1).

Завершать их при обнаружении.

Пример Bash-скрипта для Airflow Task:

#!/bin/bash
ps -eo pid,ppid,cmd | grep '[p]ython' | while read pid ppid cmd; do
    if [ "$ppid" -eq 1 ]; then  # Если процесс остался без родителя
        echo "Killing orphaned Python process $pid ($cmd)"
        kill -9 $pid
    fi
done
DAG в Airflow:

from airflow import DAG
from airflow.operators.bash import BashOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'airflow',
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
}

dag = DAG(
    'kill_orphaned_python_processes',
    default_args=default_args,
    schedule_interval='*/5 * * * *',  # Каждые 5 минут
    start_date=datetime(2024, 3, 1),
    catchup=False
)

kill_task = BashOperator(
    task_id='kill_orphans',
    bash_command='sh /path/to/kill_orphans.sh',
    dag=dag
)

kill_task
✅ Плюсы: Универсальное решение, убивает все зависшие Python-процессы без изменения кода.
❌ Минусы: Airflow будет убивать все зависшие процессы, даже если они критичны.

Вывод: Какой вариант предпочтительнее?
Лучше совместить оба метода:

Встроенная самопроверка в Python-скрипте (psutil) — убивает зависший процесс сразу.

Airflow-мониторинг (BashOperator) — как дополнительная страховка, если процесс завис до проверки.

🔹 Оптимальный вариант: сначала добавить самопроверку в Python-скрипт, а если проблема не решится, настроить очистку в Airflow.
```

```
В Airflow можно управлять приоритезацией DAG’ов и задач с помощью Pools (пулы), Priority Weights (приоритеты задач) и Concurrency (параллельность).

Решение

Ограничим количество одновременно выполняемых задач с высоким потреблением памяти с помощью Pools.

Зададим приоритет исполнения с помощью priority_weight.

Ограничим максимальное количество параллельно выполняемых DAG’ов с помощью dag_concurrency.

Код для dag_1
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

dag = DAG(
    dag_id="dag_1",
    default_args={"pool": "high_memory"},
    max_active_tasks=1,  # Только 1 активный таск одновременно
    dag_concurrency=1,    # Одновременно выполняется не более 1 таска из DAG
    start_date=datetime(2024, 3, 1),
)

main = PythonOperator(
    task_id='main',
    python_callable=some_function_1,
    priority_weight=10,  # Высокий приоритет выполнения
    dag=dag
)

main
Код для dag_2
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

dag = DAG(
    dag_id="dag_2",
    default_args={"pool": "high_memory"},
    max_active_tasks=1,  # Только 1 активный таск одновременно
    dag_concurrency=1,    # Одновременно выполняется не более 1 таска из DAG
    start_date=datetime(2024, 3, 1),
)

step1 = PythonOperator(
    task_id='step1',
    python_callable=some_function_2,
    priority_weight=5,  # Низкий приоритет
    pool="default",     # Используем стандартный пул (не требует памяти)
    dag=dag
)

step2 = PythonOperator(
    task_id='step2',
    python_callable=some_function_3,
    priority_weight=8,  # Средний приоритет
    pool="high_memory",
    dag=dag
)

step1 >> step2
Объяснение, как достигаются цели
✅ Не допускаем ошибок из-за нехватки памяти:

Введен пул "high_memory", который позволяет выполнять только один тяжёлый таск одновременно.

max_active_tasks=1 в каждом DAG’е гарантирует, что не будет выполняться больше одной тяжёлой задачи.

✅ Обе обработки выполняются максимально быстро:

priority_weight у main в dag_1 выше (10), чем у step2 в dag_2 (8). Поэтому dag_1 получит больше ресурсов, если оба DAG’а конкурируют.

step1 (dag_2) выполняется в другом пуле (default), поэтому он запускается параллельно и не ждёт dag_1.

dag_concurrency=1 предотвращает ситуацию, когда два тяжёлых таска выполняются одновременно.

📌 В результате:

dag_1 получает приоритетное исполнение.

dag_2 подготавливает данные (step1), пока dag_1 выполняется.

dag_2 (step2) запускается, когда освобождается память.

Исключена ошибка нехватки памяти.
```
```
Способы получения данных из Wordstat
Парсинг данных из HTML страницы

Открываем страницу в браузере и авторизуемся в Яндексе.

Используем Python (Selenium, BeautifulSoup) или JavaScript (document.querySelectorAll) для извлечения данных.

Преобразуем данные в CSV/Excel.

Плюсы: Можно быстро получить данные вручную или автоматически.

Минусы: Требуется авторизация, есть защита от ботов (CAPTCHA).

JavaScript-скрипт в консоли браузера

Открываем Wordstat в браузере.

Запускаем JS-скрипт в консоли для извлечения данных.

Сохраняем их в CSV/Excel.

Плюсы: Самый быстрый способ для разового сбора небольшого количества данных.

Минусы: Требуется ручной запуск, неудобно для массового сбора.

API Яндекс.Директа

Wordstat использует данные из Яндекс.Директа.

Можно использовать API Reports API или Keyword Statistics.

Плюсы: Официальный способ, нет CAPTCHA, удобен для массового сбора.

Минусы: Требуется регистрация в Директе, доступ к API, возможны ограничения по запросам.

Лучший способ для разового сбора 15 фраз
JavaScript-скрипт в консоли браузера.

Открываем страницу с данными.

Запускаем JS-код для парсинга данных.

Копируем результат в Excel.

Быстро, не требует сложной настройки, подходит для небольшого количества данных.

Лучший способ для автоматического сбора 2000 фраз ежедневно
API Яндекс.Директа.

Позволяет запрашивать данные автоматически без CAPTCHA.

Работает стабильно, есть документация и поддержка.

Можно написать Python-скрипт с requests или использовать pandas для обработки данных.

Надёжный и масштабируемый вариант для больших объёмов информации.
```
