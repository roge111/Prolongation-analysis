# Prolongation-analysis

Привет! Тут я представляю программу, решаю задачу по составлении статистики о пролонгации менеджеров. Приятного просмотра!😉

![](https://media1.tenor.com/m/3BsGAeAG7R4AAAAd/confused-thinking.gif)

### Постановка задачи
---

Руководитель отдела сопровождения клиентов хочет получить информацию о том, насколько хорошо сотрудники его отдела (аккаунт-менеджеры) справляются с одной из своих основных задач – пролонгацией договоров с клиентами. От аналитика он хочет получить отчет о пролонгациях сотрудников за 2023 год. 

В компание используется два коэффициента пролонгации:
Для проектов пролонгированных в первый месяц – отношении суммы отгрузки проектов пролонгированных в первый месяц после завершения к сумме отгрузки последнего месяца реализации всех завершившихся в прошлом месяце проектов.
Для проектов, пролонгированных во второй месяц – отношение суммы отгрузки проектов, пролонгированных во второй месяц к сумме отгрузки последнего месяца проектов, не пролонгированных в первый. 

То есть, если нам нужно понять, насколько хорошо менеджер пролонгировал в мае, необходимо посчитать:
Сумму отгрузки проектов, завершившихся в апреле (за апрель) и сумму отгрузки тех проектов завершившихся в апреле, у которых есть отгрузка в мае (за май). Коэффициент – отношение второй суммы к первой. 
Сумму проектов, завершившихся в марте, у которых нет отгрузки в апреле (за март) и сумму отгрузки тех проектов, завершившихся в марте, у которых нет отгрузки в апреле но есть в мае (за май). Коэффициент – отношение второй суммы к первой. 

Имеются два набора данных:

prolongations.csv
- id – id проекта
- month – последний месяц реализации проекта
- AM – ФИО ответственного аккаунт-менеджера (данные первичны по отношению к financial_data)

financial_data.csv:
- id – id проекта
- Причина дубля – причина, почему строки с одним и тем же id встречаются несколько раз
- Колонки с названием месяца – сумма отгрузки проекта в данный месяц
- Account – ФИО ответственного аккаунт-менеджера

Необходимо: 
Рассчитать коэффициенты пролонгации для каждого менеджера и для всего отдела в целом
  - за каждый месяц
  - за год

Сформировать аналитический отчет в гугл-таблицах или excel, на основании которого руководитель отдела будет принимать управленческие решения (обязательно присутствие коэффициентов пролонгации, в остальном отчет можно дополнить на свое усмотрение – дополнительные метрики и визуализация приветствуются)

`Пролонгация` — это по сути продление договора с клиентом. Естественно, что чем больше продлений договоров у менеджера, тем он лучше. Но не всё так просто. Для этого в задании используются два коэффициента. Их вычисления описаны выше, но поясню еще раз. 

Представим, что первый коэффициент — K1. В K1 есть знаменатель и числитель. `Знаменатель` — отгрузка за месяц завершения. Например, месяц завершения — май, то берем май. 
`Числитель` — отгрузка этого же проекта в следующем месяце. Например, в июне.

Однако сложность к задаче добавляет еще то, что в таблице с отгрузками могут быть значения `стоп` или `end` — такие проекты мы игнорируем. Также может быть `в ноль` — отгрузка проекта в данном месяце равна 0, значит для коэффициента пролонгации нужно взять отгрузку предыдущего месяца (только если все части оплаты равны 0).

Теперь второй коэффициент — K2. 

`Знаменатель` — отгрузка проектов, которые завершились два месяца назад. Например, в феврале. 
`Числитель` — отгрузка этих же проектов через два месяца. Но это при условии, что в промежуточном месяце не было отгрузки. Например, если мы смотрим апрель, то в мае не должно быть отгрузки, но должна быть в июне. 

Теперь можно продолжить и перейти к реализации.

### Общий алгоритм
---

Перед тем как вообще писать код, надо представить алгоритм обработки данных.

1) Импорт необходимых библиотек
2) Чтение данных из файлов
3) Слияние данных
4) Вычисление коэффициентов
5) Формирование в отчеты

Однако пункт 4 имеет свои нюансы. Для корректного вычисления необходимо: 
  - Правильно парсить числа. В таблице они находятся не в стандартно представлении. Например, могут быть разделены запятой вместо точки.
  - Переформатировать месяци в удобный формат datetime для вычисления соседних месяцев.
  - Корректная обработка значений: `в ноль`, `стоп`, `end`

Теперь можно переходить к реализации.

### Импорт библиотек
---

Подключим все нужные нам библиотеки для работы.
```
import pandas as pd
from datetime import datetime
from dateutil.relativedelta import relativedelta
```

`pandas` — эта библиотека основа для составления статистики и анализа данных. Она содержит в себе инструмент для форматирования таблиц csv в формат Data Frame, что позволяет использовать большой набор инструментов для обработки. Например, удаления столбцов по определенному условию. Или же очистка от значения "NaN". 

`datetime` и `relativedelta` — это две библиотеки, которые необходимы для обработки дат. Тут они используються для вычисления соседних месяцев. 

### Чтение данных
---
```
df_financial = pd.read_csv('financial_data.csv')
df_prolong = pd.read_csv('prolongations.csv')
```
Тут мы считываем данные в формат Data Frame из файлов `financial_data.csv` и `prolongations.csv` в `df_financial` и `df_prolong ` соотвественно.

### Парсер чисел отгрузки
---

Как я уже писал выше, для вычисления коэффициентов необходимо привести числа в стандартный вид. Также мы можем частично обработать значения `стоп`, `end` и `в ноль`.

```
def parse_number(x):
    if isinstance(x, str):
        x = x.strip().lower()
        if x in ['стоп', 'end']:
            return 'стоп'
        if x in ['в ноль']:
            return 'в ноль'
        x = x.replace(',', '.')
        try:
            return float(x)
        except:
            return 0
    elif pd.isna(x):
        return 0
    return x
```

Функция `isinstance` — это встроенная функция Python, которая позволяет узнать, принадлежит ли объект определенному типу данных. И в данном случае `instance(x, str)` позволяет проверить, что x — это тип str.

Далее приводим `x` к единому формату, удаляя все пробелы по краям и на всякий случай делаем все символы строчными. Хотя и по заданию там и нет заглавных букв, но нам ничего и не мешает сделать их строчными)

Если мы `x` встречаем как `стоп` или `end`, то мы возвращаем `стоп`. Аналогично для `в ноль`. Это поможет нам привести некоторые значения в удобный и гарантированный для нас формат, что уменьшает вероятность возникновения ошибки. Ну а все остальные значения — это числа. В них мы заменяем запятые на точки и удаляем все пробелы внутри. В случае работы без ошибок возвращаем вещественный тип данных, иначе возвращаем ноль. Также вернем ноль, если значение `NaN`. Но если это не строка и не `NaN`, то просто возвращаем x. 


Эта функция справляется со своей задачей — приведение всех значений таблицы к единому формату. Самое главное — приводит числа к стандарту. 

 ### Переформатирование чисел

 ```
month_cols = [col for col in df_financial.columns if re.match(r'[А-Яа-я]+\s\d{4}', col)]
for col in month_cols:
    df_financial[col] = df_financial[col].apply(parse_number)
```
Этот участок кода позволяет нам запарсить все числа.

В `month_cols` мы полачем список стоблцов с результатами отгрузки. Например, `Апрель 2023`.
А `apply` - позволяет нам у Data Frame применить функцию к колонке.

### Слияние данных
---

Для удобства дальнейшей обработки сделаем оба Data Frame в одну таблицу, используя `merge`.

```
merged = df_prolong.merge(df_financial, on='id', how='left')
```

### Обработка месяцев
---

Как я упомянул в начале, но для удобства вычисления соседних месяцев хорошо было бы привести к удобному формату.

```
months = [
    'Январь', 'Февраль', 'Март', 'Апрель', 'Май', 'Июнь',
    'Июль', 'Август', 'Сентябрь', 'Октябрь', 'Ноябрь', 'Декабрь'
]
month_name_to_number = {name.lower(): i + 1 for i, name in enumerate(months)}
month_number_to_name = {i + 1: name for i, name in enumerate(months)}
```

`months` — тут понятно, просто список месяцев. 

`month_name_to_number` — функция, которая превращает месяц в число.

`month_number_to_name` — функция, которая превращает число в месяц.


### Обаботка  ключевых слов
---


```
def get_adjusted_value(row, target_month, later_months, previous_month):
    value = row.get(target_month)
    if value == 'стоп':
        return None  # не учитываем проекты со статусом "стоп"
    if value == 'в ноль':
        only_zeros = all(
            row.get(m, 0) in [0, 'в ноль'] for m in later_months
        )
        if only_zeros:
            return row.get(previous_month, 0)
        else:
            return 0
    return value if value != 'в ноль' else 0
```

`get_adjusted_value` - это функция, которая обрабатывает значения `стоп` и `в ноль`. Помним, что такие значения мы можем получить и `parser_number`. 
На вход функция принимает три параметра:
 - `row` - строка из таблицы (Data Frame)
 - `target_noth` - текущий месяц
 - `later_month` - все месяцы, идещие после текущего (нужно для случае `в ноль`)
 - `previous_month` - предыдущий месяц.

И если мы стречаем `стоп`, то для проекта возвращаем `None`. Если встречам `в ноль`, то пути разветвляються. `only_zeros` позволяет собрать значения в строке в последующих месяцах. Если там везде `0` или `в ноль`, то проект вполне реально `в ноль`. Если отгрузка пошла в ноль, то мы возвращаем, чтобы отобразить это при расчете в коэфициенте. Иначе просто возвращает ноль, что говорит, что отгрузка не состоялась. Во всех отсальных случаях просто возвращаем значение, если это не `в ноль`, иначе 0. Да да, я понимаю, что нет смысла писать `else 0`. Но мы должны обеспечить в любом случае возврат значения. 

### Расчет коэффициентов

Теперь когда все для работы готово, приступим к сложному.

```


def calculate_coefficients(df, current_month):
    coefficients = []

    try:
        month_str, year_str = current_month.strip().split()
        current_dt = datetime(int(year_str), month_name_to_number[month_str.lower()], 1)
    except Exception as e:
        print(f"Ошибка парсинга месяца {current_month}: {e}")
        return pd.DataFrame()

    # Определение нужных месяцев
    prev_dt = current_dt - relativedelta(months=1)  # апрель, если текущий — май
    prev2_dt = current_dt - relativedelta(months=2)  # март, если текущий — май

    prev_month = f"{month_number_to_name[prev_dt.month]} {prev_dt.year}"
    prev2_month = f"{month_number_to_name[prev2_dt.month]} {prev2_dt.year}"
    current_month_str = f"{month_number_to_name[current_dt.month]} {current_dt.year}"

    later_months_from_prev = [
        f"{month_number_to_name[(prev_dt.month + i - 1) % 12 + 1]} {prev_dt.year + ((prev_dt.month + i - 1) // 12)}"
        for i in range(1, 13)
    ]
    later_months_from_prev2 = [
        f"{month_number_to_name[(prev2_dt.month + i - 1) % 12 + 1]} {prev2_dt.year + ((prev2_dt.month + i - 1) // 12)}"
        for i in range(1, 13)
    ]

    for account, group in df.groupby('Account'):
        k1_num = 0
        k1_den = 0
        k2_num = 0
        k2_den = 0

        for _, row in group.iterrows():
            # Пролонгация в 1-й месяц после окончания
            end_val = get_adjusted_value(row, prev_month, later_months_from_prev, '')
            next_val = get_adjusted_value(row, current_month_str, [], prev_month)
            if end_val is not None and isinstance(end_val, (int, float)) and end_val > 0:
                k1_num += next_val if isinstance(next_val, (int, float)) else 0
                k1_den += end_val

            # Пролонгация во 2-й месяц после окончания
            end2_val = get_adjusted_value(row, prev2_month, later_months_from_prev2, '')
            skip_val = get_adjusted_value(row, prev_month, later_months_from_prev, prev2_month)
            second_val = get_adjusted_value(row, current_month_str, [], prev_month)

            if end2_val is not None and isinstance(end2_val, (int, float)) and end2_val > 0 and (not skip_val or skip_val == 0):
                k2_num += second_val if isinstance(second_val, (int, float)) else 0
                k2_den += end2_val

        k1 = round(k1_num / k1_den, 2) if k1_den else None
        k2 = round(k2_num / k2_den, 2) if k2_den else None

        coefficients.append({
            'Account': account,
            'K1': k1,
            'K2': k2
        })

    return pd.DataFrame(coefficients)
```

На вход функции приходит наша таблица на одного менеджера - `df`, и `current_month` - месяц, для которого анализируем пролонгацию.

Для нашего месяца мы парсим его отдельно на месяц, отдельно на год - `month_str` и `year_str`.

```
month_str, year_str = current_month.strip().split()
```

Далее мы приводим месяц и год к формату `datetime`, сохраняя результат в `current_dt` — что значит `current_datetime` (текущая дата). Это просто расшифровка названия 😊.
```
current_dt = datetime(int(year_str), month_name_to_number[month_str.lower()], 1)
```

Но наша основная цель — сделать код, который не будет прерываться в случае ошибки, поэтому всю нашу конвертацию в `datetime` оборачиваем в `try-except`. Вот как выглядит в итоговом формате:

```
try:
        month_str, year_str = current_month.strip().split()
        current_dt = datetime(int(year_str), month_name_to_number[month_str.lower()], 1)
    except Exception as e:
        print(f"Ошибка парсинга месяца {current_month}: {e}")
        return pd.DataFrame()
```

Далее определим предыдущие месяцы. Почему так, если в задаче у нас приводится пример, где мы берем месяц завершения апрель, а месяц отгрузки — май? Так всё правильно, я скажу) Ведь мы берем `current_dt` как месяц отгрузки, значит нам нужно смотреть назад. Поэтому мы определяем предыдущий месяц и предпредыдущий.

```
prev_dt = current_dt - relativedelta(months=1)
prev2_dt = current_dt - relativedelta(months=2)
```

Теперь у нас `prev_dt` — это предыдущий месяц, а `prev2_dt` — предпредыдущий. 

После вычисления всех месяцев мы переводим их в обратный формат. 

```
prev_month = f"{month_number_to_name[prev_dt.month]} {prev_dt.year}"
prev2_month = f"{month_number_to_name[prev2_dt.month]} {prev2_dt.year}"
current_month_str = f"{month_number_to_name[current_dt.month]} {current_dt.year}"
```

Но тут хочу сделать заметку насчет `current_month_str`. По идее, данная переменная будет равна `current_month`. Но я лично делаю преобразования и над ней, чтобы все месяцы у меня были в едином формате. Хотя можно подумать и над ее удалением в принципе. 

Теперь для дальнейшей обработки данных надо бы вычислить список месяцев для предыдущего и предпредыдущего месяцев. Такое нам необходимо для обработки `в ноль`. 

```
later_months_from_prev = [
        f"{month_number_to_name[(prev_dt.month + i - 1) % 12 + 1]} {prev_dt.year + ((prev_dt.month + i - 1) // 12)}"
        for i in range(1, 13)
    ]
    later_months_from_prev2 = [
        f"{month_number_to_name[(prev2_dt.month + i - 1) % 12 + 1]} {prev2_dt.year + ((prev2_dt.month + i - 1) // 12)}"
        for i in range(1, 13)
    ]
```
В итоге `later_months_from_prev` содержит месяцы, последующие после предыдущего, а `later_months_from_prev2` — после предпредыдущего. С помощью `month_number_to_name` - мы првращаем номер месяца в слово, а тут `{prev_dt.year + ((prev_dt.month + i - 1) // 12)}` - вычисляем сам год. С помощью `%12` - мы гарантируем, что наши номера месяцев будут в диапазоне от 1 до 12. Ну и само `i` мы тоже перебираем в диапазоне от 1 до 13. Если что, то мы ставим 13 там, просто потому что у нас `range` особенный - он не берет врехнюю границу дипазона, а берет последнее значение перед границей. 

Далее мы перебираем все наши аккаунты, сгрупировав Data Frame по колонке `Account`.

```
for _, row in group.iterrows():
            # Пролонгация в 1-й месяц после окончания
            end_val = get_adjusted_value(row, prev_month, later_months_from_prev, '')
            next_val = get_adjusted_value(row, current_month_str, [], prev_month)
            if end_val is not None and isinstance(end_val, (int, float)) and end_val > 0:
                k1_num += next_val if isinstance(next_val, (int, float)) else 0
                k1_den += end_val

            # Пролонгация во 2-й месяц после окончания
            end2_val = get_adjusted_value(row, prev2_month, later_months_from_prev2, '')
            skip_val = get_adjusted_value(row, prev_month, later_months_from_prev, prev2_month)
            second_val = get_adjusted_value(row, current_month_str, [], prev_month)

            if end2_val is not None and isinstance(end2_val, (int, float)) and end2_val > 0 and (not skip_val or skip_val == 0):
                k2_num += second_val if isinstance(second_val, (int, float)) else 0
                k2_den += end2_val
```

Сам по себе цикл собирает отгрузки и прочее. 

```
end_val = get_adjusted_value(row, prev_month, later_months_from_prev, '')
next_val = get_adjusted_value(row, current_month_str, [], prev_month)
```

Где `end_val` и `next_val` - это отгрузки в месяц завершения проекта и в месяц анализа. А `get_adjusted_value` испльзуется для обработки значений `стоп` и `в ноль`.  И далее если у нас отгрузка в месяц завершения не нулевая, то тогда мы к `k1_den` и `k1_val` прибавляем `end_val` и `next_val` соответсвенно. То есть `k1_den` - это сумма отгрузок в месяц заврешния (для K1, учитыая что мы считаем от месяца отгрузки это предыдщий месяц). Аналогчино для K2, но тут же есть нюанс. В промежуточный месяц отгрузок не должно быть. 

```
end2_val = get_adjusted_value(row, prev2_month, later_months_from_prev2, '')
skip_val = get_adjusted_value(row, prev_month, later_months_from_prev, prev2_month)
second_val = get_adjusted_value(row, current_month_str, [], prev_month)
```

`end2_val` и `second_val` - это отгрузки в месяц заверешния и во второй месяц пролонгации. А воот `skip_val` позволит нам понять, есть ли в первый месяц пролонгации отгрузка

```
if end2_val is not None and isinstance(end2_val, (int, float)) and end2_val > 0 and (not skip_val or skip_val == 0):
                k2_num += second_val if isinstance(second_val, (int, float)) else 0
                k2_den += end2_val
```

И если у нас `skip_val` нулевой, а `end2_val` (то есть отгрузка в месяц заывершения) не нулевой, то тогда мы считаем суммы отгрузки в месяц завершения и во второй месяц пролонгации. `k2_num`  и `k2_den` - zэто аналогично `k1_num` и `k2_val` соотвественно. 

Далее вычисляем наши коэффициенты для менеджера

```
k1 = round(k1_num / k1_den, 2) if k1_den else None
k2 = round(k2_num / k2_den, 2) if k2_den else None
```
Но я тут от себя добавил на всякий случай проверку, что `k1_den` и `k2_den` не пустые, что не было ошибок деления на 0.

```
 coefficients.append({
            'Account': account,
            'K1': k1,
            'K2': k2
        })
```

Закидывую это в результирующий формат для каждого менеджера, чтобы потом превратить в Data Frame

```
 return pd.DataFrame(coefficients)
```

Вот и весь основно алогритм. Тут главное было понять в самой задаче, как вычислять эти K1 и K2.


Ну дальше нам остается выгрузить список месяцев из таблицы пролонгации `df_prolong` И уже перебирая их подставлять в нашу функцию вычисления. А там если результат не пустов, то записываем в массив result.

```
# Список месяцев
months_to_analyze = sorted(set(df_prolong['month']))

# Расчёт коэффициентов
results = []
for month in months_to_analyze:
    coeffs_df = calculate_coefficients(merged, month)
    if not coeffs_df.empty:
        results.extend(coeffs_df.to_dict(orient='records'))
```
### Формирование отчетов

Отчеты я формирую в 4-х таблицах: `overall_report.csv` - отчет по каждому менеджеру за весь период, а `yearly_report.csv` - годовой отчет по каждому менеджеру, сгруппированный по годам, `prolongation_report_fixed.csv` - это основной отчет по каждому месяцу и менеджеру, а `department_summary.csv` - это общий отчет по всему отделу.


```
final_df = pd.DataFrame(results)

# Группировки
final_df['Год'] = final_df['Месяц'].str.extract(r'(\d{4})')
yearly_report = final_df.groupby(['Год', 'Менеджер'])[['K1', 'K2']].mean().reset_index()
overall_report = final_df.groupby('Менеджер')[['K1', 'K2']].mean().reset_index()

overall_summary = pd.DataFrame({
    'Менеджер': ['Весь отдел'],
    'K1': [final_df['K1'].mean(skipna=True)],
    'K2': [final_df['K2'].mean(skipna=True)]
})


# Сохранение
final_df.to_csv('prolongation_report_fixed.csv', index=False)
yearly_report.to_csv('yearly_report.csv', index=False)
overall_report.to_csv('overall_report.csv', index=False)
overall_summary.to_csv('department_summary.csv', index=False)

print("Все отчёты успешно сформированы и сохранены в CSV!")
print("Пример данных:")
print(final_df.head())
```



