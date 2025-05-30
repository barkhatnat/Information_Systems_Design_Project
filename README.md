# ML System Design Doc - [RU]

## Дизайн ML системы - Прогнозирование спроса для сети супермаркетов MVP

### 0. Состав команды

- Бархатова Наталья (Product Owner, Software Architect)
- Ребров Сергей (Data Scientist, Data Engineer)

### 1. Цели и предпосылки 

#### 1.1. Зачем идем в разработку продукта?  

- Бизнес-цель:
	- Оптимизировать управление товарными запасами и логистикой за счет точного прогнозирования спроса для каждой точки продаж, снизить потери из-за избыточных и дефицитных запасов, увеличить выручку
- Почему станет лучше, чем сейчас, от использования ML:
	- Отказ от интуитивного планирования заказов приведет к снижению количества ситуаций дефицита/избытка, а следовательно вырастет прибыль.
    - ML-модели смогут учитывать сезонность, тренды и региона точки.
- Что будем считать успехом итерации с точки зрения бизнеса:
	- Процент просроченных товаров уменьшился на 20%
	- Дефицит уменьшился на 15%

#### 1.2. Бизнес-требования и ограничения  

- Краткое описание БТ и ссылки на детальные документы с бизнес-требованиями:
	- Прогноз спроса на 14 дней вперед
	- Гранулярность: SKU × магазин × день
	- Обновление прогноза — ежедневно
	- Интеграция с системой заказов (через API)
	- Интерфейс отчетности для менеджеров
- Бизнес-ограничения: 
	- Нельзя увеличивать штат
	- Ограниченный бюджет на вычисления
	- Ограниченный доступ к облачным сервисам
- Что мы ожидаем от конкретной итерации: 
	- В этой итерации мы проверяем возможность автоматизированного прогноза спроса на уровне SKU × магазин с горизонтом 14 дней, и интегрируем его в реальный процесс принятия решений на ограниченной группе магазинов и товаров. Были определены следующие цели интеграции:
		- Получить работающий MVP-прогноз, сравнимый по качеству или лучше, чем прогнозы менеджеров
		- Проверить пригодность данных, их достаточность и стабильность
		- Протестировать процесс ежедневного обновления прогноза и передачи результата в ERP-систему
		- Проверить реакцию и восприятие бизнес-пользователей (менеджеров) на прогнозы от модели
		- Провести сравнительный анализ (A/B-группы) и подтвердить бизнес-эффект
- Описание бизнес-процесса пилота, насколько это возможно - как именно мы будем использовать модель в существующем бизнес-процессе?    
	- Сейчас менеджеры вручную формируют заявки на закупку товаров, опираясь на интуицию, сезон и прошлый опыт. Заказы передаются в ERP-систему, оттуда — на склады и поставки. Как будет работать во время пилота:
      	- Модель ежедневно прогнозирует продажи на 14 дней вперёд по каждому товару
      	- Результат прогнозов выгружается через API
        - Менеджеры в пилотных магазинах получают рекомендованный заказ, основанный на прогнозе (например: прогноз + safety stock – остатки)
        - Человеческая корректировка возможна, но сравниваем: что заказано по прогнозу, что заказано по ручной корректировке
        - Далее идет стандартный процесс — поставка, продажа, мониторинг остатков
        - После 2 недель сбора данных — сравниваем реальный спрос с прогнозом, и с заказом, оформленным вручную и по ML
- Что считаем успешным пилотом? Критерии успеха и возможные пути развития проекта:
	
| Метрика                  | Целевое значение                          | Обоснование                               |
| ------------------------ | ----------------------------------------- | ----------------------------------------- |
| Снижение излишков        | -20% по сравнению с контрольной           | Меньше списаний товаров с коротким сроком |
| Снижение дефицита        | -15%                                      | Меньше out-of-stock ситуаций              |
| WAPE прогноза            | ≤ 50% по 80% SKU                          | Прогноз достаточно точный                 |
| Приемлемость прогноза    | ≥ 70% менеджеров считают прогноз полезным | Анкетирование после пилота                |
| Доля предсказанных пиков | ≥ 60%                                     | Прогноз не пропускает резкие скачки       |

- Возможные пути развития проекта:
    - Расширение на всю сеть: 500 магазинов × 10,000 SKU
    - Автоматическое формирование заказов (без участия менеджера)
    - Учет внешних факторов (погода, праздники, акции)
    - Рекомендательная система по оптимизации запасов (optimizer поверх прогноза)
    - Перевод моделей на online-learning / auto-retraining

#### 1.3. Что входит в скоуп проекта/итерации, что не входит   

- На закрытие каких БТ подписываемся в данной итерации:
	- Прогноз продаж на горизонте 14 дней вперёд для уровня гранулярности SKU × магазин × день
	- Ежедневный пересчёт прогноза и выгрузка результата в API
	- Построение MVP-модели с адекватной точностью (WAPE ≤ 50% по 80% SKU в пилоте)
	- Подготовка витрин данных и реализация пайплайна модели
	- Интеграция с интерфейсом для менеджеров пилотных магазинов
	- Проведение A/B-теста с участием менеджеров
- Что не будет закрыто:
    - Учёт внешних факторов (погода, акции, праздники), будет отложено до следующих итераций
    - Обучение онлайн и auto-retrain моделей 
    - Полноценная интеграция в ERP-систему всей сети 
    - Полная автоматизация процесса заказа, пока только рекомендации
    - Обработка новых товаров, пока предполагаем наличие исторических данных
- Описание результата с точки зрения качества кода и воспроизводимости решения:
	- Код написан с учетом возможности расширения, включает конфигурируемые параметры модели, расширяемую архитектуру моделей
	- Используются unit-тесты для ключевых компонентов
    - Для моделей и результатов прогнозирования ведётся управление версиями
    - Документация на пайплайн, модели, формат данных и API-интерфейсы
- Описание планируемого технического долга (что оставляем для дальнейшей продуктивизации):
    - Упрощенная предобработка данных, пока без полной нормализации категориальных фичей
    - Использование одной модели на все магазины и товары без кластеризации и сегментации
    - MVP-модель пока не настроена на быструю работу при прогнозировании
	- Функция замены прогноза в случае ошибки пока не реализована

#### 1.4. Предпосылки решения  

| Блок          | Предпосылка                                                                    | Обоснование                                                   |
|---------------|--------------------------------------------------------------------------------| ------------------------------------------------------------- |
| Данные        | Используем исторические данные продаж по SKU × магазин × день за последний год | Достаточный период для учёта сезонности и трендов             |
| Гранулярность | SKU × магазин × день                                                           | Соответствует уровню принятия решений и точности заказов      |
| Горизонт      | Прогноз на 14 дней вперёд                                                      | Обоснован логистическим циклом и бизнес-требованиями          |
| Обновление    | Ежедневный пересчёт                                                            | Позволяет учитывать новые данные и адаптироваться к трендам   |
| Метрика       | Продажи в штуках (или литрах/килограммах — в зависимости от категории SKU)     | Единицы измерения, по которым происходит заказ и доставка     |
| Стратегия     | Прогнозирование с использованием отдельных моделей или кластеров SKU           | Обосновано различием поведения категорий товаров              |
| Ограничения   | Ограниченные вычислительные ресурсы и доступ к облакам                         | Будет использоваться on-premise решение или минимальный клауд |

### 2. Методология   

#### 2.1. Постановка задачи  

- **Тип задачи:** прогнозирование временных рядов на уровне SKU × магазин × день
- **Цель:** построить модель, которая на горизонте 14 дней будет выдавать прогноз спроса по каждому товару и каждой торговой точке
- **Ключевые компоненты решения:** подготовка и чистка данных, инженерия признаков, выбор и обучение моделей, валидация, управление версиями, метрики качества

#### 2.2. Блок-схема решения  

- СХЕМА ТУТ !!!!!!

#### 2.3. Этапы решения задачи 

- И ЧТО-ТО НЕПОНЯТНОЕ ЕЩЕ ЗДЕСЬ !!!!!!

### 3. Подготовка пилота  
  
#### 3.1. Способ оценки пилота  

- Выбираем 20 магазинов × 500 SKU
- A/B-группа:
    - A: менеджеры работают по старому (ручной заказ)
    - B: получают рекомендации модели

#### 3.2. Что считаем успешным пилотом  

- WAPE модели в группе B < WAPE в A
- На 15% меньше out-of-stock ситуаций
- Визуальный отчет, подтверждающий бизнес-выгоду

#### 3.3. Подготовка пилота
  
- Развернуть сервиc, настроить ежедневный запуск и интеграцию с интерфейсом
- Провести краткий вебинар по работе с рекомендациями модели и сбору обратной связи
- Ежедневно сохранять прогнозы, сформированные заказы, фактические продажи и остатки
- Обеспечить техподдержку на время пилота, отслеживать корректность работы пайплайна
- Анкета для менеджеров после 2 недель работы
- По завершении пилота собрать и проанализировать метрики, сформировать отчёт и рекомендации по дальнейшим шагам

### 4. Внедрение  

#### 4.1. Архитектура решения   
  
- [Ссылка на архитектуру схемы](https://viewer.diagrams.net/?tags=%7B%7D&lightbox=1&highlight=0000ff&edit=_blank&layers=1&nav=1&title=C4.drawio&dark=auto#R%3Cmxfile%20pages%3D%223%22%3E%3Cdiagram%20name%3D%22Context%22%20id%3D%22NY88W09RBWIob4Yexyum%22%3E7Vpbc5s4FP41foxHXI0fDbaTbdKtW3dn277sEFBsGmyxgGM7v34lJMQR4Fviup1uMi1BB%2BlcpHP5JKVjeIvNdeon8%2FckxHFHR%2BGmYww7uq5rPYv%2BYpQtpzi6IMzSKOQkrSJMo2csiEhQV1GIM6VjTkicR4lKDMhyiYNcoflpStZqtwcSq1ITf4YbhGngx03q31GYz0srehX9BkezeSlZs%2Fv8y8IvOwtLsrkfkjUgGaOO4aWE5PxtsfFwzCavnBc%2Bbrzjq1Qsxcu8ZQC5%2F87mg%2FaI%2FQDPqd04rU2Z%2Bae%2FEGZ2hqjT99jTtYrnELybxdMuKVrHoZLsjm7QRTJc5ZXz%2FbxNBN8JFUqWJX2IsyCNkjyipEqsU7DUCvbj4jkqnm5B50r1incDqMA4goGSSfGp0t0TA1lnzr9Qv5JiAHNHpSzJEPbkbBHoz%2FuMVU1qIiQdlTqYwLraWAvo74BP0vaxsgjCQK6VpAwarEr9i2WI%2FXsao3z6dTumvuM%2BkCXzlCzfCre3%2F12R8sNVVgTlgHbQ7GRTfaRvM%2FG74HJfESzhWyzY%2BUfql%2Ff1AZTGJavkMHqSJIszKxyKMrOGYCzsV6qQ7mN2qrXaPmsFl4DEJFW40GAIip%2B2sZYaBMr8tM1F3UwdZIDw0%2BaOBtf643R4%2B40Yq8nHd%2B6VJjOLzBiVofo8X7Cl14T%2BIt1qrB362RwzziyQ1%2FMox9OEJg5KWNPMzgZEcexJaw3kGGOa3A03y1PyiOEX29AsT4gA9Ifih42Y%2B0V%2BWGxmrGh0A7ObFHmCau76cTRj2SGgaQ3Tke4C5%2F4ojHKhd0KiZV4kMou5B%2BrSKfRQ8Z95h0dpGmsJukrrqURB0GrENpreQmxlqcqm%2Fww3xdSp%2FPtiEao0CTO3yMhPOM3xBpDEQl5jQmch3dIu4uuVVpYVUVetvmivqyqlI0GbwwrlCKIvqsFMMq%2BKB30R7lM2RTl5QWkZfZpcHZeoeZKsl5ApecjXdK4oebrNcrzYW00GR0pBalKVyROBVO826mEt4SOQaWWW5nSQ9iu5TlsZgjUW0ktZsCII6V7Jhyd2raG%2FFM3H0qfbKGGwjMqqByuIpBfDhdBjJo2rfVBEragNdk%2BXfHEbhbu50A7gxnv2wYzJpdcb9pYARwoV615zIaOxRnBaLDDnWr3EH4tYoOZnxwC7QAh0J%2BjVNYm9vUEBnaEmGioJV6oH6DagtwbLCHSDcadBG7tvQOd%2FAHTQXqSTktUyLPCMtgPPACxUeIrrB4%2BzYliJWZZkiZvAx%2FEcs2%2FvgTcNBOOnQYm0UBtg6hl2z9GbUKeOGxrQR29inzY8JEFKExDpbSjpl0BOgKarcOoM8EmiJYGebKsFPZlt6Em%2FMHqizfd3J%2BGnX2Oj%2B0oY19eBdJ7oYT0et%2BlzcGYkCBgAnXXwLgtYs54NwbvbqF67cYCOdugGJTpqZZUWDYAgS63cUKshQHhWVct3egLkYAOTayJq4ODHeRQqIaxRSpdIuq8CL2nXYSd8AwG%2FFwh4vvm8IJNre4aep%2B710%2BTz92x74LTjx2EADdmai86JAdDQQn3tDQNcDgPolooBTO1yGKBxmj%2BOPiTvrnB6uzbyj%2F94JrlBwZUwqdWjmS%2BQNJ%2BTGVn68R0hiXCY7zjPt8LV%2FFVOVC%2Bn05Vuv7DxXatsfoXfhhvBnLe2sDXBaUQNZf483HuOlfvpDOeH4lZHOFQuX5pLleLYz6Mn9S6mbdrF0AmLjmqJjV4N5pk9lUVGVmmAxSh42VJjpNUYGbbTRY7Kixvd4FU4gjSpzTeOcwZxbfbkxyt4Z%2FPiy5OaZ7GluOO1EvpLI4%2Bx%2BIoCPx6ID4soDBmPA4mqPWb3eX0jkOUdnxCiXKO1BThNGqYjFumljlR2IQ8PGf4h62q0LsVUNEX5eX3cb6L8SxnL9P1rlQJoqwp61tiCRj3kZf7QYPYAyeSU%2FJHRkMkH7JKWeVjsZ1kUlORxFFc5Kyw7ienYmXh4PB88NVAz1DuX%2FPHw%2BO0v6%2FHG%2B7C9XVnb6eJKP3eG2lEYGk6SzB%2BjyDLv4lsDX8fhJnERkjEBgl9Apizxl0fBO53CuwJp1VGayCIHzzprJ3QcvUvcxvWAwO2yyWUHfKPIalz8HJ9%2FjH3J5hSAgbrIMnpK3SgX8meko9P%2FIOC0u5TL3WC%2F4ljBABt4eGsxBDrITSmMCrOxaffaokKqOtzB9tTTiuPvW%2BhzBPbMNaHqNNYnHJ4g1O5e4KXNGAzcc7v1%2BusaaOAINGtuJg8sPKCMAd57QARcC2uP074dHPxeBwe7y%2Fvb5cHbwcEZDg4ueXnQgIut3n2hPYU8SzjDXqDcoGhdhExlk0Ld%2B%2BRtyis2CL%2F6%2FqBVG%2FPMhwO2v2BZbnmfJT8JzzdXcJ%2BnnwWsI9RXwbrxOrBesjHVES%2FB7rRZ%2FaEw7179ubUx%2Bg8%3D%3C%2Fdiagram%3E%3Cdiagram%20id%3D%22nEQKmDmmrMPcoJwDYCde%22%20name%3D%22Containers%22%3E7V1dl5s2E%2F4t74XPaS%2FsA%2BL70l%2Bb9M0m3WZzmrY3PdjGNl0MLrDxbn99JSFgJATG2Ov1pk5bFwZJ6GP0zMNopPS08ebpXexu1x%2BjhRf0kLJ46mmTHkIasmz8PyJ5ziSqoSiZZBX7CyYrBff%2BPx4T5ske%2FYWXcAnTKApSf8sL51EYevOUk7lxHO34ZMso4N%2B6dVdeRXA%2Fd4Oq9Ku%2FSNeZ1EZWKX%2Fv%2Bat1%2FmbVdLInGzdPzFqSrN1FtAMibdrTxnEUpdnV5mnsBaT38n7J8t3UPC0qFnth2ibD4y%2FLB21%2Ba4U3M339%2BcH%2BWw3MPivlmxs8sgazyqbPeQ%2FE0WO48EghSk8b7dZ%2B6t1v3Tl5usODjmXrdBPgOxVfLv0gGEdBFOP7MApxotHCTdY0u8pu7tw09eIQS7ByKDaWVpuS18uLU%2B8JiFjT3nnRxkvjZ5yEPe3rJutnpmnF%2Fa4cNsNhsjUYMlNnQpepyqoovOxNfME6VN654W%2BT9x92X4110N%2FNb5fju8397ffTuaot9K1e7VtT1rcOeqm%2BRZK%2BNQP82tE29rg%2BNv9%2BJFNstIzCtL90N36AWzHEST56YRD1EH4%2FrqTpbkhvs7Sfo1mURiRNFEbiQ3mWcfQY%2B16Mn3zydvIsRRJys8ElJ3So6e1PYZo%2FSdww6Sde7C9pTiXCA7UMCHSQWruPuGJUvnUXCz9cZWJnoOr69glfqdrAUk18SRPhEU77RJX6GwzOWVqmWeQqihf9Wey5D9kTetl3yVCMKKhSZSNP4tXsB0vLKlj%2B%2F0fwEjfwV2GWOEndOGs2hsF45Yf9NNpmjxRax%2F%2F5m20Up27Ip8KdnkabhoRm0Z%2FmimX9eNvvYe231R6eQiM7v8a%2FiEoM%2BjumvwoZN3JxQxNk11P6q9HfCZBYoECYErGiyvRZmhtaeFET4RWFnNThh4%2B%2F3v2Y6ytWf6qyeYsEhCB9y8MA6%2BjJ3KM6o43IRPaxyRqyBxt%2FsSDZpXjCIw6ZFMzoqicCC00RwMKogoVuS8BCOxwrotlfxOjjGge4kWts3L1Y4AX6J3eTIwRusjMGWjEB1zr9NXMJHTtk9pCmKpnClZdZuV%2Bet6zcO%2FzSKMzlEy%2BZx%2F429aMQvNbmVaPQphGVj3ONw9caqALidapBu2nGy9Bu0J0jWV4D1N8Gj4q233CDAKdvKRlWisrrT4chcGdewNsFour1hiGhk4AAD8VOCdJkpcxKgcF0CxlgKs%2FEDFiWvZkXL%2FxvhcjICqMKhQszJiAvTJdXIW4q7NDWqk2tZaUwOzCBCbU5%2FSPLa%2FCTgOsfWV%2BIzUTA%2BP%2BapNrH6W754P3%2B%2BZMxWUbpu6QPAYiRBYCYkDEBeMsZUiPbghQLt1GxtRvLwPIkjaMHDz4xNdUYs1cA%2BZL%2BITnWLsWHzdOKfBoN5vpgS3ECySAcA507Xfgpq%2Fc28sOUAplB1EMZ4C4cK%2FQ%2Foh1jLFPJHZPzMosXMoEqCGUyJBFKi%2BTfjf%2FFdsXDSuXO6CCUMHkcp7c03pSYElOCFIkpKQjrAbYE3zJz0sG0TD%2FfVe3CfbRMdy6lo%2FfPSeptak0EN12UaIl%2FsNnHPN6lul2Wk9ByBleQ%2B%2F5BDjWCXEnj1P0fjlRRRu78YUWzCd%2BQAubZY1t3zAZkq4CXG89zkFVkWGlppmWjKsqJkFFBPVSFPRkUFvhUxUIkA8iLAE0gQzySngI5xS92ZEmQU5chZ4cv9qOQk7JjFbBOS0YkISGdUNKXUUujnvPCjGYjfx9jRXf9kHx%2Fs0fefB1i%2FV2x3ruLknQVe%2Fe%2F3DaS%2FCGkrFUKjVo0iXJalDejZdtKblzktcsL8ZNhCkrLnlqAS6uAXdugPll2aessIBnSNCbIbr99W0Vqwfw54SzZ5u8CWvJ2DNrUJP%2B8ikHTGg1azpfnz4GPLVusEUvCrIqx38TNMnt4O8sFvJ%2FjIAuItCGaOMLnAzrEIlY%2FF6bWZDg5guh3N2SnNk8vQfRVgzdXhd%2F4DOaq4m%2BWKq%2Bxx3Ufxek6WkWYsN9GxNdJR%2FcvL02fmQIxvy1QWdxf8fNvJP9AUQrB71SgUb5EBZMn9ors7jm%2Fe%2FLTLLNj6%2Bw%2By2vaDrsvs5IbmPPOi33cT0RZy7JFYe3IJtFjPPf2faAjJXXjlceySpeftCydt%2BAW4ap6EnuBm%2Frf%2BDU52ZCzrHdkWgH9EtRL09SBg0zV0Gyd%2FFo6X2LWPlYIXFITyy2WKfMPVFEdsx6olEQ1smhfq1UQaT%2BbvcoqyFHORkHFybjcZoa7mwO6meHLyW7T9KtASrHwy17Sg2urMqjBcKbbjJMcq1WqxQ1%2BXyghWi4T70WG3ZKO1D27ZUbtaHyiqzhDsoZOxjpwk8Sf5%2BIbPyhhbJEnYm8GwKYZiAc2RdkLbDIoKtHOAFinHoxz3SENtYQ0%2FUyQ5vDIY5jtkKdakMUXpCtCQTVg2EGXo5vJH9Fu9%2FMX7Y%2Bvc9v49P7r3G1YyE22btiKDCO22FnhtAwLiw8suGai8V8z8KslW7IoWG5Wj5rluTNAZA13xazyhv5pj6LWqVC0rwxUI7eaOfxpx6Hqy8OmfRbYhLTOsnj0M9Vu6CeB2AWA4LZIzUA0v%2F6dA9RzgKhWBdE%2F%2F5x8%2B%2F%2FwRvM%2FTAJ9Nvp18W7i9FXzPChqGeoAKeUfjdNo2zEGVjdUFRYubFUfIEd3NMNWDMdR2%2FHNU%2Bm9U8sSuyHjxbJE%2B4T4ppiI14bcUr0Gvh3uUm3yRWZfAsWIOrzbcb%2FD9YfJ1%2Fc%2F5q89wI063Lj%2F0AWuz94iWftL0iDSCUh5F0WrgKxvjfzVL48eHYrsyX0Y7ZaB%2B%2BDlhco9rybQ30mNtxJ%2B%2FcBQiMLB2aHLxr0yfkHwzsLSoI%2FWAVVt0%2BnVZDCqQsilVKqKQBNyt%2B7VI%2Fuf88jKzSxEwatL9uqSLS2gYfPODVWxL3oFERIYtV9ZNoMQPgWQDOMA7RZYe0j0YPku1MFYfvZc0gXEwHyKFt7gr6TRAsrZXFYP%2FKuDujrMYDW1E1VW9mzQfsHMgTLLtxd25yTdp4A6twz2nbKKMRtdVGkI2jsF658K7wqFWWDIJCzc5LkErAxlGvL126L%2BYyDh6crBgZywoxQQVqlVyoHtytkFNwfgeAlLu4q4Rl%2B24koqrqSCkYrm6ExJXOTOm43ITi0vLlARvTC9qLABVbWH40kD8ajPAZ4ggVp0ZitXTiLhJEjY4mVItiHJKYnoHD%2FBKrFc952zuBhbOeAaHWv7PXDKqT1wbbd7SauTc0rRh1Vd3dT4kKZmdrc3rOzQD25gGy%2FCTdaopadZB1Ccy1kHOOzDYXj3U%2FVL4YQbZg7m%2FHfPGArCjLHeBG7yUE%2F5M1V0YM0hlZ5WqLHA%2B2gNJYR3b1cIxbaPIGyedDCZ3WKiVT1bcAdUnqWIpCwrBscUie89KlayihsN8HIly1eyXNjjvWR57T25KzL3c%2F%2BbQQ0AWJdjKYqluhYEeuk%2FefmJCzx%2FPnjPwNU998ruOUsW3y%2FdGWW9hneuzm2UL8m87MZV0Q5fTpUa7P8XL0yi%2BIbs%2B2%2F0%2B7VsB9xBmwfmc8ZJaN8U2E78W2zPrePWWbNgtwm1yovljKtS74magArf9JroePshUnj%2F4d5NzBM%2BV7XaxY5katqvRv1q1KlRt69G%2FWrUO%2Fi3EO%2FfUlXlfFa9pR8mdzSdLYZORXqPi6FDehFUd0gM3RE%2BM8luhjNFrR3nM5OE1hLGABdvZCawi1m9WF9X7SzsFBOGLIOboZcU8ipvvnbu%2BarT8zK4iP9O87Xz7oMjJnpr57h9WRNdl0%2F0zs7x9lO%2FwWm%2Bx%2Fl1mWihnQotsOEyOah4xejRllokH5gXhArb1gTTrnWCinJzENLNHoxtdyzUe5Ho9kvDD3kUuoFsTgedfPIdutFRM0Vmihy%2BpNNtdJQrZ81OR%2BhPqItyyQNX%2FvPgdsJNlG8O3M6zZVK%2Bw8Y6fIsNQEkjw7Q3Saje6pdTvpGkSqhaQMcFI8DJNgDS%2FX%2F8EsQlfQt9eFjvNs8f4qeZ9sf4fbq2%2BuOHftV1cUIyw09Ro8sELcBjoPD4YVv6y7CYpjMZ9s9Z9TwkRjwDUAwvb8thVMG7ZrXcMY3R0H0GydjsewEFrfHXvOnTHJpmohxzDvGYnuI0h5MCj3RGScb1elb5f%2B%2Bscjxnhu1PKVfPdVzy93kKed8Q7IYuiZB4xWPIr%2FuVwLfzdb9Skfe6X6nBk3Pdr3SN1jgmWqP%2Be%2Be6Xem6Xenkp1rq4henKgnneKH9Sh0oSefzYEpjJz0Gfz91mN5%2FwdW%2F%2F3mIcyvDu5%2FqmUO57UEwYdUtEJAkDYHREUIiDWAZdbCLoMq%2Fs49taoVPz8JLO4j7sP3faFSQhr3blVDlrbB%2BcC%2BFBeTCSc51%2FAAyIVUWSVk9Ixr22pTJX48NfAd%2Fo8KVDpyEDuit6MA1dvNq7IUjrIWjgA0kMfbnit2UavbZ4zsUze7xyyLFSubJIzfbLmWc58zWtquP0trUhDrsDdu8BHd%2Fk%2BKdwt2vGHkUw%2BW6%2B88UagBCpHWHn2eW1WmiVZf%2BxeCAmhgCcBa9o%2FXg0qWanYD6ekuXb2C%2B1wQbwNAm6CC9kIW9JtU%2FyUzPo1rzYwX0S5%2F35zn7oy7kAHWKOTjpXGs68vdC5lpO0jruiXgTwX9Nynn8zCTbIDQNvbG5qcr50bkmp9UtIOh4e3yGvy3hstBBHvvTF%2F9SaqtrFFFfdfjQV9t%2BsUhouSbXRAkV574odLe24CuzOScmt7252M%2BsgoNYZL4f3re0Bw5hMTIfzEWCZI4S3x9K4ts4ilKYnCxnfYwWpNun%2FwI%3D%3C%2Fdiagram%3E%3Cdiagram%20id%3D%2227XsrQHmJQv-H2r86Rup%22%20name%3D%22Components%22%3E7V1bd5tItv4t50FrdR6kRXHnURc7yYyTduKc09PzMgtLWCKRhBrh2J5ff6qAQrsuIEAIoTROty0VRVHUZe9vX2ugTTev70N3t%2FoULLz1QFUWrwNtNlBVQ1MV%2FIeUvCUlqmWkJcvQXyRl6FDw4P%2FXSwtptWd%2F4e2ZilEQrCN%2FxxbOg%2B3Wm0dMmRuGwQtb7SlYs0%2FduUtPKHiYu2ux9A9%2FEa2SUlu1DuUfPH%2B5ok9GppNc2bi0cvom%2B5W7CF5AkXYz0KZhEETJp83r1FuT0aPjktx3m3M161jobaMyN3h3%2Fw62Xz95j8E%2FfqD37p9%2FBF%2F%2BOVTNpJmf7vo5feO0t9EbHYIweN4uPNKKMtAmLys%2F8h527pxcfcGzjstW0WaNvyH88clfr6fBOgjx922wxZUmC3e%2Fim9H6Zd7N4q8cItLbPwkG5eK75K%2B3k8vjLxXUJS%2B23sv2HhR%2BIarpFc1Ox3ndKUZdAW9HKYNGXT1rZg5o1XddLEss9YP44k%2FpEMqH9516P%2B1%2B%2Fgf8%2BPLn7ubL%2B%2BXd7cf3SE64%2BgOVM1UXAdZuHwfhcEPD1xRZ5apkAafgm0Eyp%2FiH1we4Gf4ERkrTWlmApCC2BlAiiZOgW2p4gwYVgMTIF3fhn3eGVjYimJpshkYG4qitzsDOtLZLWCI4%2B9opjj%2Bun6u8ad0khl%2Fc42fO9mFHjMP5l%2FPhBTG4zV8cjf%2BGr%2FGGFf55G3XwUDFHcC9NN0NmZG07tfgMYgCUifYBvxF%2BS3T4Dn0vRBf%2Bey9yG%2FJqpAvG9zyPl4O8deP24he2bvb%2FXDvhf5TfKcS4Jl6WhMST3rtPuOOxeU7d7Hwt8uk2BkhXd%2B94k9IG1nIxB%2FjSniKoyFZbsMNZqJJ3XT1kU9BuBg%2Bhp77I7kSfxy6ZC4mMfOLlxe5Ei4ff8MLMu7g4e878BB37S%2B3SeV95IbJa2N2FS797TAKdsklJe7j%2F%2FibXRBG7pathQc9CjYFFc1sPM1leuunu%2BEA7yAbDfA2m9j0M%2F6txiVG%2FHsa%2F1bIvJEPt3GF5PNN%2FFuLf89AiQUahDXVtKlD%2FaTObdx41hPuEVk56cNvn%2F7v%2Fh1dr3j9x0uWvhFHRcjYsqQiHejZ3IvXjDYhO9nH0GKcXtj4iwW5XUpzWKpENkUKjlBDDBMpynGOqdsShqlVpxbB43eCznCX1%2FgtVxiFeSEH4PTP7oaSCPzODgLzZ7HLQhemdhbP3CT%2BbeSvHngjpoP4Pw0diO9c%2F%2Fa2S7swxQPu%2Bluy19NL3ny1xftsmY7WfbCPlqH38OWO1ph5%2B3no7yI%2F2B5eY2yAToiLUS3xSvHmUOlrlHy39DMcDvvwgd98N6C15KpFN2W2Icb0ataf5Hbp21mgZBzXMcHtKVNeu4%2FemmUKZJ3nc4V9vAMI1YkJp4TMJK08HgqMdF2pBtjHj%2FwNuCx5Mlu88H9mRUbSWLxCcGOkFynv2D7ud%2FRZYJXgSsYMPAA2RvsZFj2x6pCgoiFJW5lnQORQUbsxyT%2FZvQa7qplBlA0Y%2F5rqMXygKYBGpYACEFUsM8Ubcv629jE1DGOklZJB4zhYe0xo6N0jLWCJarz%2BJu78xzIu58QWDulhRKfOHI4UqwX4TiD%2FAkRUbqzZeIavYOrp3iz8KO3lLvC3UUwdDbLulBEe9qkS%2F09W1BSXIfJtpBqSQlmZxRYmlRBbJq0oaVHhHm2QNwg9PC3uYzxvTckTjs3LE7bIoFRdJtGp1TkU%2FpoyqQrwVjMEHOAtlt5D%2BjVdSeyqwzBpFSyDrbu%2BCwjaiqf8uxdFb%2BmqSpEjWMd4EMO3f5EvIwwbacGfpMGRZui0YPaaPiL59ga%2F3WOcioeALMS0cDEm6pFDL3HJrb%2Bmj4yhIa0xX7v7vT%2BnxaBa7jzvMYaee0VDl04RbnDpRUUiXCrUkIEtXDeht3Yj%2FyeruJGtgPTWe7LNwHrTFYtdb0jhllHS1%2FQ%2BbiVlHakvLGkSZQxBQjbAAxAsayzzhUw2wbmSpXmX8Nx6OJXf5SyhmknXQ%2BG%2BEahBpshLnzKAujIZlRgqI8VUNWbihqmMX3dB0CrB09PeqzjT1YFuEULEvydgSh0WDB6Hwb%2FN%2Fvjwrga4HW%2Fc%2FxJ%2Br3z1FvuV%2FxTFgjSRnN4HwXJNBPeJv%2Fzy7MVTkVx52AYvT2v3h1eMh02wgGc5GDJ5gakg6mWws8aQTVmxD2Jm2BpEzg7oaplBF6tNgMTJ3aUIXVXBK1Cw3ePkHifHBNPqcXKPk%2BU42TBZnGxL1O4XhslOOzD51Y%2F%2BFYNiQ6ffE5CsEEoUfz9gZPLlDXzhEXJ9aGtoJaGtZrUFbTVn5Ni2pSiGqTsWMlmgK%2Bj%2Fk1c8G9Clj%2BOB7i1gzQnr1FhlFMfoKyChWkwZah67i6WdBrE0UhzWhjTUrglLO1MZuAOI8rx2BR5nd6dLvPL6DRPXbQKKv3nbfRDeEsNVEXAv%2Bx7UgkMKqbaX2Zbc%2B2Ua7aQQHdv2yWvBYeN6RZtltNvZIE2ENmegw7eFlKL8FCmssFFkY2L6cNCqi3ObtBATtV4q6KWCmI%2BqpaSClffqLslepjKBETMcgHfSGhkEUo%2FLDE%2F%2Bq0f9tViRAfUiQ8dFBp3XrKumRGRQZCJDDU%2BdWiKD3rJmXRnZZMSgZt02M1V7Fc06ozlP%2BwnV5pz2%2FaBbz8QXRnSxqksurCb%2FBDnGKCnH6GpLcoxm6iNTs1RFszUTmQYnxiBtZNuaqpsG%2FmIrRrtCTY72vrZQUx5uFAg7BTJRZyUauvdPl2iwQMOKuhe0DJRcRVbbdM%2BxbZbuGYZVh%2B7JyVcNxYtomuStjjnGyV%2BK0lnOCDmHH5sjdZoxUjXh8hFSJzzG0TkgoCkO29KZTZ66xD%2BXEE0ovkE6qACZaMzKUH9fckl9bP%2BO5LIlzfKBXCYELaOWJyPEfN8KiYtGpuC2LR2QWTRSFLVFBbdJI3GO%2Bm6glsgl0jgbCG%2FaKEsRY1V5AeFFjokvt0ojjXxteQmC12G61ZjiGtMtw2Kdd65LbT0GUzYV9JozVlKAbsNQaQo5ZObkkHHRzKOA9aguUk9SBaSg154Gmx2mSOlyl%2BiXbzFR%2B0EvytXJxT2H3sUTQcdtsIOgUp8QqAmOFbcHL46DbPXh27f7YbkxSJStCjOWKoQgyTPRUGgCOqIkXeL8Xg7K7NHgl1DnKl3V4J5dC2tAniZqYVnF6BGtaiW9qalNbm4r6UndcJ5iHFOmNVUV25w6ota0mORPMpWlqAjly6zm9a2WROEqU8Imj2hSM5sLoKoE4WicEgvpElXsmbw32uRUJX3fCkJPkkc4J7C%2BynyM2kkLGNlYB8%2BDJtDMTJto%2FES2VYn9jOmHAjMs5O%2Fni5xTWHMlhxlmQjtVnTQ4RiuqT6eg86KNFDJmaOx1ejZ75Wy22NjZs9mezeazWZPTD2Ax%2F9dks6rAWuxCtuEIsaCQFYNYaVWZ%2FfGhBgt9%2BHI3Xs9X3uYQKbBztwt3P1w%2B%2FnUo2r%2FNg91SzWe1PfW%2Bbuqt9dS7p941qbdmcNQb9dS7KvUGshEUPzihqpQVrBL990bLEb7wsAtJDhT8wQt%2F%2BvP8IDGGDCnBU6wNzp6hhAGJPrvFmw0X7f1Hf%2B1Hb70K7dq5g95zh547NIXtkSTv1a%2FAHWrEgWcqIcgFjnnyk983NUh92UgCqJuCVhtRl2fla8%2BkRh%2FIr0qqnmA1lQ5Vnk6PqphUadiGDfRseV3iOC0XYXBJ5V7PRK%2BdiRo9E%2B2ZaFN2KL3bTFTKCzN7i0xGasaeIA%2F0O8oav3ruPCrmimLEmsgPTfDZKAEOygyIlBGdQxBtavAhmkHlEkqmUm7P4q6dxTk9i%2BtZXD0Wp3NKxMxFtDNpMmgQXp8s%2BZzJki%2BYBPkIleXSOBO%2FMjJ0SvJHNd%2FJmyXgoRl3RFFklsqn8C4DVM7aFETOo2wgXt%2BdyKwcf08fjkSCf2uSf81QJE1nfZYNGobUjQTMUE8lZmq7AUtqwk78IUdBJYz8Ge%2Fl0ff9UZQsRcZH3YZjnVZN%2FysIl8eMQobZDtmj87zWUH7WZTjMmdNXpvBpBI4jGnkKxyF9tfH9xx4cXzk4Nns%2F5B4c12RFtsOCY8vptPaHo3uZjYDLRaQAoqfKaHWmONABUR2DeyVZ6yvxtN%2FHz9FKHSmFTG2sCTwIkn2KwSRKH5FtQQyGADOi5qYj1hNFYD0IjBtU09yCUeJcfznWA80Wp2p%2FRgO63no%2BdbV8qnfk7flUbTsFp8UxWmRUJbU4miDBnjPZKZvotE62jSy0XUHOgMkEYttZQZX49tx5Pp6HI43SOBpYTiPQmwssP%2BmYM1MXNXcEp1QK8WnMMtXJaG%2B6LRpJU6oqJpvyP22pu4kqaK6q1hJVsDvZqLOPD%2BkmTNMZMOkm9Mp0pj5R6F62CY1mqMrS5ljWyNFz0%2FGcOUGEmZN57Bj1qexN1OU0yHSDNZMG2WIn%2BILZJIQlYCv3%2F4i%2Bz%2FTJRL3d%2Fvjz%2Fb%2Ff6x498aTFNDgOky2RHExUK2tYC2TBVC%2BFFaRTRfkgt1kb1FU07nHRhU1ftOzlG7yKXEGOEeJdoIYngopX6Q2N0ADpYdMtyR1NIowTJIXSoOBiu186STUFhU5swaJVd%2FoWxDtQN5vdgeffc22j%2BpFuWuy%2Bc2y15Z2nlt15Wqd23mkgObM7cyA54ZoOy7CVnNQWV5FjsmidN7LLVYfd5UaHNvl%2FPj4ut%2Fsv90%2Fu59fF5%2Bg1enG1oZz8nnGTW%2BwhnyS9I2p5k%2BslN3nzRxiV3eTSqVLlm1wX9jA0sGUpiKDtsGTIED0lSIU5jW5YbM5BbyhXZ%2BFAKH1oJ0hA0S5ohAQYVHjrYJ5Z6cu3jK3R4KLYujSHNzq1%2BXOwNbScw6DDbIcjmnNMKbH5RX5e%2B5SgSpm3%2BONyOksnmhQITA4qdEkekL58%2B9n7FZOlFZZyWvb%2BkYKocj9pEOmXVvbfjb9uN9vVh%2B3z7e9f7yfP%2Fxv8wMCjcblerr7JFi71jDI5O3Jzyn3pispJkF%2BDqEFJB2ITVhXYXcJiNUVYiLJPs7prQJS%2BfeuJ7hUOhCDDPoWwsH4KNU41OgHPXE5XmKMSVjl3S4QcjqqUzldvUkKY2SLtdgkUyklPD%2BNZYUBql0mM0yB26bQyUz6TLSk6Dsc0Q5pQmSBAhQlzKEctXUmJQznkhxOdQJe6r0mVr5McLUttHakixJ0V6F06SzxQYxoSDFA0y2CDw6hq60TWM%2BTuOCM1uaw90jrJ5emSaMXoGloREmwZ1FJeGa2onGMNMoyW0UqhkuispzQqV6TEQX9rLQ6S2Ar7QPy%2FbSA%2BSgPxD3%2Flkfjj%2B49i6H2TKuA%2BiF4WEcIp6JAiOUGlqTB60SH3596aL8Mfq%2FHbX19UZftP6%2FuLxGDcLberDOWAcwOT5szzAB25Bre0WsZpB%2BgIql5%2BgZSFOSp3vrrNa3dyQA6WLN03UC1l36dxOOkKzZHgsixaxZ6jSMjJ2QUgU7QVm8AxmIjYHUIu0j2V0U%2BIXApp0ZEASxhCmYpsC3e%2FyuIzyZd7N8LTSeaPhPrazdD1oWaxm0i3RLKeRf9Bsm7q1el67Rj0crlwVAwoNZSQS%2FAxafcQTo5p6%2F7IcVP1t2jJbHnxjec1ZSvsK4hRcaIpXkj%2BB18tafxwKXt30UgG0%2FhCSxhsirGEXXOYeSfCyZlW5hkGPFTU5vHPuUPR5eRSBTRIDEWHdBAgXEr3CmkoF3uu2NptrEYSgsgVU0PGVATJWVT6fuXGBGLzusRNr0ZzfbSLCYUqY6lcKLoQep4XUN6J2PGYs%2FO8vwlmYnIH3ZumyEziTGs8M0G8Ne3c%2BU1uvt6LjOEheIpe3Fj18PC2j7xNLo%2BQHJ%2BBJT%2FMnd14bR%2Fa2cft%2FAJJn3oqd5TKXersJXtq645ZQNoKEm4gRUYsLc204oCHPuNGSxk3hpyrQibbdjMzlHjCUVWnThH0whvNQgA%2Fxevc9bdE15pe4g%2BUDfbRMvQevtwVovwxxKwihlZLvBK0UdyUfbcDOAYO7gdP94L8hcVH90KbCWLzSMLOcx5q8vxb18yrSC94G1A3s0AdZ2g3Jvl3GYZWfFwUBczzt7WPOVtIgmn2KVcxjrO4x4Qf3j3SAlZhUokDqtpYnTmc%2FFApBZUoL9xYs%2FHsBKRfn5E1zZ7OgvRZduW0eFJgSRWd6MPSqNsm77epHQ8QzRyyHFsfMOYAm%2BZ9qeiW1ZyVgCowjzuEt5T9hZMkNR2NHNVEhmbr5Lelsw2WtRkMEWd9MHnrQ3OeEfKBzgmQvWqbQOEO7IZRIJt%2Fzt%2FrHI5a8uFoJ2ilhINlQrl4B0tA3DRDZYlbar%2BsH%2B0CXVBRZVp3AlnTypI19cxkjUZhW%2BXIjWjs5Mw0iGsnhwA2tnolATIprtzv3G0pDKym%2FizSoxVqnLrIuWsk%2FchxzGiBLOYgVuKIEf9UoJxNxuAgg%2FJKSvK6lGZL%2Fv7OL8gjGzORI9scKbpqKSpybMsyKKKg7hTIHJGzWXXb0gzdoF7kp3jfyClmf4xOC957nI8cCacjHTz8fQcechEvP6JHK3M4nDJo1arcQXe9XJpQgZ7rnFykS%2FTEFzzMpusHJFXWJKcnSpK9VurgnKs4H7L28MFDesp4uWT3OoPDWc4q2LrJ%2B94wcdQM54a3QE8Y2LgJXoQ7FwGmmsnzgZmCElZlXtk%2FBw4UpGtHT54Q9wCcr%2BKQcjinvfK%2BV95DPd1R5T3wdnnxHjEDe8GwJ6OKag6fa0qNL2jdEbLHU4Kf8xT8%2BXeAK%2BqAVeHXtgr0un%2BpyygXqGapZZX%2FfFKPy51hXUazcOB2Um%2Bh49jh5uEb7v7D72N8Nzn7LR86JIT%2BEB6X8TAx6QhESfDsu4zPQe5u0hszf1MRgCeic8yGm4fhB0aIx3BYHTUUgKOMhXJPhf1TwWBZoJwzeOcBhLxjnvKSvXCjRhPUXQ4O%2FAKOZz0eaAYPqKXwwMp7dZeELFFTvhHrt4BCPq2R6ehLYIQn%2F9VbUAbMQITKvm69pb9tbs8ZHgyZpV%2Fq0stbOs5l6dfazoAwUrQsR1MaHphlh2s8dWTpmL6WzFqn6YyvOS974eJrJl9rp5I0y9%2B39VTsSOeOS7OsWrtNtHjzNvEc0znw23G0AZOz0bIGlw3kvYpNn%2BPx0vHMZYXLv5kM7ZxdVu%2F85m877yq78dXzJWg%2BbcPRiezKhstJZlryDIarOD6hcH02lLuUS%2Fl4Bfuz7eylHGOulyXjdL58Ke%2ByCxKIHO9GFXGq2bq5NYZIYd1KbMdhWzqznyzVHzdwxgS006ls4I5MF8Tqmo5QRNiMTCfTTTrZWALWrtFJ%2FDUMgghWJ%2BatT8GCDPvN%2FwM%3D%3C%2Fdiagram%3E%3C%2Fmxfile%3E#%7B%22pageId%22%3A%2227XsrQHmJQv-H2r86Rup%22%7D)
  
#### 4.2. Описание инфраструктуры и масштабируемости 
  
- Какая инфраструктура выбрана:
    - **Kubernetes-кластер** для оркестрации контейнеров всех сервисов.
	- **Docker-контейнеры:**
        - Web-приложение для менеджеров (React + Node.js)
        - API-слой прогноза (Python + Flask)
        - Модель (Python + TensorFlow)
    - **Хранилища данных:**
        - PostgreSQL для транзакционных и оперативных данных о продажах 
        - DWH (Amazon Redshift / Google BigQuery / Snowflake) для исторических витрин

- Плюсы и минусы выбора:

| Технология                 | Плюсы                                                                                           | Минусы                                                   |
|----------------------------| ----------------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| **Kubernetes**             | • Низкая латентность между сервисами и локальным хранилищем<br>• Полный контроль и безопасность | • Необходима внутренняя поддержка и обновление кластера  |
| **Docker-контейнеризация** | • Повторяемость окружений<br>• Быстрый rollout и rollback<br>• Изоляция зависимостей            | • Дополнительный слой сложности (надстройка и CI/CD)     |
| **Облачный DWH**           | • Практически неограниченный объём хранения<br>• Высокая скорость агрегаций для отчётов         | • Затраты на хранение и запросы в облаке                 |

- Почему финальный выбор лучше других альтернатив:
    - Контейнеры в Kubernetes обеспечивают повторяемость и изолированность сервисов, что упрощает CI/CD и rollback
    - Облачный DWH дает баланс стоимости и производительности: ядро аналитики в облаке, оперативные сервисы рядом с данными 
    - При росте нагрузки достаточно добавить ноды или включить облачный burst — минимальные изменения в архитектуре
  
#### 4.3. Требования к работе системы  
  
- **SLA:** 99% успешных ежедневных запусков
- **Пропускная способность:** 10.000 прогнозных точек в минуту
- **Задержка:** ≤ 1с на запрос SKU × магазин
  
#### 4.4. Безопасность системы  
  
- **Аутентификация:** JWT‑токены
- **Изоляция:** Kubernetes + лимиты ресурсов  
  
#### 4.5. Безопасность данных   

- **Шифрование данных:** все данные шифруются как при хранении, так и при передаче по сети
- **Дополнительная защита:** анонимизация персональных данных, соответсвие GDPR
  
#### 4.6. Издержки  
  
- **Бюджет:** 2.000 USD/мес (серверы + БД)
- **Хранение:** 100 GB прогнозов ≈ 20 USD/мес
  
#### 4.5. Точки интеграции
  
- **API-запрос:**
```
POST /recommendations

Headers:
Authorization: Bearer <TOKEN>
Content-Type: application/json

Request:
{
    "sku_id": 1,                                // id товара в БД
    "store_id": 2,                              // id магазина в БД
    "start_date": "30.05.2025",                 // день начала рекомандаций
    "end_date": "03.06.2025"                    // день конца рекомандаций (проверка на лимит 14 дней)
}

Response:
{
    "recommendation": [100, 209, 120, 23, 106]  // количество товара на каждый день
}
```
  
#### 4.6. Риски  

- Сбой передачи данных → отсутствие новых прогнозов
- Сбой сервиса → задержка рекомендаций
- Рост нагрузки → падение SLA
