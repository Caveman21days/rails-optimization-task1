# Case-study оптимизации

## Актуальная проблема
В нашем проекте возникла серьёзная проблема.

Необходимо было обработать файл с данными, чуть больше ста мегабайт.

У нас уже была программа на `ruby`, которая умела делать нужную обработку.

Она успешно работала на файлах размером пару мегабайт, но для большого файла она работала слишком долго, и не было понятно, закончит ли она вообще работу за какое-то разумное время.

Я решил исправить эту проблему, оптимизировав эту программу.

## Формирование метрики
Для того, чтобы понимать, дают ли мои изменения положительный эффект на быстродействие программы я решил использовать такую метрику: `количество итераций в секунду`

## Гарантия корректности работы оптимизированной программы
Программа поставлялась с тестом. Выполнение этого теста в фидбек-лупе позволяет не допустить изменения логики программы при оптимизации.

## Feedback-Loop
Для того, чтобы иметь возможность быстро проверять гипотезы я выстроил эффективный `feedback-loop`, который позволил мне получать обратную связь по эффективности сделанных изменений за *время, которое у вас получилось*

Вот как я построил `feedback_loop`:
* Сделал файл размером 1000 строк (код с таким файлом выполнялся за комфортное для меня время)
* Использовал профилировщик для поиска точки роста (каждый раз старался пользоваться разным, отчеты строил так же разные)
* Вносил изменения в код
* Проверял, что точка роста менялась
* Писал тест
* Прогонял через бенчмарки и радовался результатам

## Вникаем в детали системы, чтобы найти главные точки роста
Для того, чтобы найти "точки роста" для оптимизации я воспользовался профилировщиками `ruby_prof` и `stackprof`.
`rbspy` и меня по какой-то причине так и не завелся, решил не тратить драгоценное послерабочее время :)


Вот какие проблемы удалось найти и решить

### Находка №1
- `ruby_prof` `flat` и `graph` отчеты показали главную точку роста - метод `select` в методе `work` при выборе данных по `id` пользователя
- сделал группировку сессий по пользователю
- убрал лишний `split` строк в методах `parse_user` и `parse_session`
- капля рефакторинга
- в `report['allBrowsers']` убрал лишний `map`, поместив `#upcase` в первый `map`
- метрика увеличилась (уменьшилась - не знаю, насколько применимо словов к итерациям в секунду) с 0.267 i/s до 5.9 i/s

### Находка №2
- `stackprof` показал, что в методе `#collect_stats_from_users` дольше всего парсится время - основная точка роста
- убрал весь парсинг времени, дата уже была в нужном формате, опять же, немного рефакторинга
- метрика улучшилась до 9.5 i/s
- исправленная проблема перестала быть основной точкой роста.

### Находка №3
- `ruby_prof` показал главную точку роста - метод `map` в методе `#collect_stats_from_users`
- убрал метод `#collect_stats_from_users`, он показался мне рудиментальным
- убрал лишние `map` при обработках браузеров
- убрал класс `User`, в нем нет необходимости
- еще немного рефакторинга
- метрика улучшилась до 17.7 i/s
- исправленная проблема перестала быть основной точкой роста.

### Находка №4
- `ruby_prof` показал главную точку роста - метод `map` - вызывался слишком часто
- переписал обрабатываемую структуру, уменьшил количество вызовов map
- очередная порция рефакторинга
- метрика улучшилась до ~25 i/s
- время обработки большого файла стало 28-31 s

## Результаты
В результате проделанной оптимизации наконец удалось обработать файл с данными.
Удалось улучшить метрику системы с 0.267 итераций в секунду до ~25 и уложиться в заданный бюджет.

Могу отметить весьма нетривиальынй и интересный процесс оптимизации в целом. Даже в рамках ограниченного основной работой свободного времени, ковырял крайне вовлеченно :) 


## Защита от регрессии производительности
Для защиты от потери достигнутого прогресса при дальнейших изменениях программы написал три теста, которые проверяют время работы программы