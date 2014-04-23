---
layout: post
title: Особенности работы с SQLS Stored Procedure из C# кода.
category: SQL Server
tags: sqls, c#, stored procedure
author: gaHcep
year: 2014
month: 4
day: 23
published: true
summary: Часто приходится работать с ORM? Все хорошо и довольно просто пока идет работа с таблицами, индексами и внешними ключами? Что насчет хранимых процедур? Вот этот вопрос мы и рассмотрим сегодня.
---

# Особенности работы с SQLS Stored Procedure с C# кода, 1 из 2

Суть сегодняшнего топика в работе с хранимыми процедурами (их вызов и обработка результатов) с C# кода.

Мы посмотрим как работать с: 

- результатом **`SELECT`** запроса (result set)
- параметрами типа **`OUTPUT`**
- **`RETURN`** значением

а также каким образом с помощью **`NHibernate`** можно полученный **`result set`** сразу перевести в массив экземпляров определенного класса.

## Получение информации о размере базы данных

Для работы нам понадобится хранимая процедура. К примеру, посчитаем размер конкретной базы данных на сервере. 

Входные данные:

- Имя процедуры должно начинаться с **`sp_`** (это даст серверу понять что мы хотим создать процедуру в системной базе **`master`** и запускать ее можно будет откуда угодно)
- Процедура может иметь параметры двух типов: **`INPUT`** и **`OUTPUT`**
- Процедура может возвращать скалярное значение посредством **`RETURN`**
- Процедура может возвращать множественный набор данных

Наша задача - получить значение **`OUTPUT`** переменной, **`RETURN`** переменной (значение в случае выполнения **`SCALAR`** типа запроса, например, **`SELECT 1`** ), а также результирующего массива данных (**`result set`**).

Получим информацию о размере базы данных используя следующий код:

{% highlight sql %}
CREATE PROCEDURE [dbo].[sp_Database_Size]
	@DBName     VARCHAR(200),
	@Error      VARCHAR(250) OUTPUT
AS
-- ========================================================
-- Author:      Your Name
-- Create date: mm/dd/yy
-- Description: Retrieves different metrics of a given database:
--              * overall size for all LOG records
--              * overall size for all ROWs
--              * overall size (LOG plus ROWs)
-- ========================================================
BEGIN
	
	-- Prevents DONE_IN_PROC messages from being sent 
	-- back to the client
	SET NOCOUNT ON;

	-- Check if the given database already exists
	IF (NOT EXISTS (
		SELECT name 
		FROM master.dbo.sysdatabases 
		WHERE (''['' + name + '']'' = @DBName OR name = @DBName))
	   )
		BEGIN
			SET @Error = ''Database '' + @DBName + '' doesn''''t exist'';
			RETURN -1;
		END

	-- Get different size values for the given database
	SELECT
		LogDataSize = CAST(SUM(CASE WHEN type_desc = ''LOG'' THEN size END) * 8. / 1024 AS DECIMAL(8,1)),
		RowDataSize = CAST(SUM(CASE WHEN type_desc = ''ROWS'' THEN size END) * 8. / 1024 AS DECIMAL(8,1)),
		OverallSize = CAST(SUM(size) * 8. / 1024 AS DECIMAL(8,2))
	FROM sys.master_files WITH(NOWAIT)
	WHERE database_id = DB_ID(@DBName)
	GROUP BY database_id;

	RETURN 0;
END
{% endhighlight %}

[ссылка на gist](https://gist.github.com/gahcep/11135326)

Процедура принимает в качестве аргумента имя базы данных и возвращает три числа (значения в Mb):

- Размер всех логов (таблица **`sys.master_files`**, строки с типом `LOG`) 
- Размер данных в базе (таблица **`sys.master_files`**, строки с типом `ROWS`)
- Суммарный объем

В случае корректного выполнения (передано имя реально существующей базы данных), **OUTPUT** параметр будет пустой, а **RETURN** по-умолчанию будет равен 0, так что ничего интересного. 

Вот пример ответа по одной из моих баз:

![](/images/sqls-stored-procedure-from-csharp/sp_exec_success.png)

Если же передать неверное значение в качестве имени базы, то результат будет немного более интересным:

![](/images/sqls-stored-procedure-from-csharp/sp_exec_failure.png)

Вот все эти значения и надо получить в коде. Как это сделать? С помощью каких инструментов? К сожалению (если я не прав, прошу в комменты), все выходные параметры можно получить только с помощью **ADO.NET**, однако не всегда и не всем требует получать значения и `**RETURN**` и `**OUTPUT**` параметра, а вот вызов и обработка обычного результата нужна во всех случаях. 

Рассмотрим как работать с процедурой используя следующие подходы/инструменты:

 * **NHibernate** и **Fluent NHibernate**
 * **ADO.NET**
 * **SQL Server Management Objects (SMO)**

Пост будет состоять из двух частей - этот про Fluent NHibernate, следующий про ADO.NET и SMO.

## Работаем с хранимыми процедурами используя Fluent NHibernate

Прежде всего давайте вспомним классический способ вызова процедуры используя TSQL:

{% highlight sql %}
DECLARE @ReturnValue INT,
		@Error VARCHAR(250)

EXEC	@ReturnValue = [dbo].[sp_Database_Size]
		@DBName = N'GAHCEPDB',
		@Error = @Error OUTPUT

SELECT	@Error as N'@Error'

SELECT	'Return Value' = @ReturnValue

GO
{% endhighlight %}

Результат вы уже видели на скриншотах выше.

Первый способ работы с процедурами включает использование инструмента создания ORM классов для вашего **persistance** (база данных) слоя.

Для использования примеров и работы с NHibernate, в проекте необходимо установить ряд библиотек (используя **NuGet Package Console**):

**NHibernate**

> Install-Package NHibernate -Version 3.3.3.4001  

Либо последнюю версию (учтите, она может быть альфой):

> Install-Package NHibernate
 
**Fluent NHibernate**

> Install-Package FluentNHibernate

Также убедитесь что процедура уже создана, соединение настроено и все необходимые шаги для инициализации **Fluent NHibernate** сделаны.

Как было сказано, получить значения **`RETURN`** и **`OUTPUT`** аргументов используя известный фреймворк не получится. Тем не менее, все же давайте посмотрим, что предлагает нам Fluent NHibernate для работы с процедурами. 

OOP моделью тут и близко не пахнет, однако вызов процедуры можно осуществить с помощью **createSQLQuery** запроса. Так как для почти всего прочего есть море других вещей более высокого уровня - **HQL**, **ICriteria**, **QueryOver**, для работы с процедурами приходится использовать низкоуровневые механизмы. По сути мы работаем на уровне обычного SQL.

Результат полученный от вызова SP может быть как **managed** (иметь существующий мэппинг - entity), так и быть **unmanaged**. **Unmanaged** означает что парсить и преобразовывать результат мы должны вручную либо с помощью вспомогательных методов фреймворка. **Managed** означает вовлечение сущностей (описанных классов - entity). Работать мы будем с первым типом результата, а по второму есть много дополнительных методов Fluent NHibernate - советую обратить внимание на методы **`AddEntity`**, **`AddJoin`** и прочие.

Так как **`OUTPUT`** параметры NHibernate возвращать не будет (а передать процедуре все равно надо все **`INPUT`** и **`OUTPUT`** параметры), вырежем из тела процедуры **`OUTPUT`** параметр.

Фрагмент C# кода позволяющий получить результат:

{% highlight csharp %}
var nhelper = new NHibernateHelper(SERVER, CATALOG, PORT, GAHCEDB, USER, PASS);
var result = nhelper.OpenSession().GetCurrentSession()
    .CreateSQLQuery("EXEC sp_Database_Size ?")
    .AddScalar("LogDataSize", NHibernateUtil.Decimal)
    .AddScalar("RowDataSize", NHibernateUtil.Decimal)
    .AddScalar("OverallSize", NHibernateUtil.Decimal)
    .SetParameter(0, "CCSD", NHibernateUtil.String)
    .SetResultTransformer(Transformers.AliasToBean<DatabaseSizePayload>()).List<DatabaseSizePayload>();
{% endhighlight %}

Теперь рассмотрим подробнее код выше.

{% highlight csharp %}
var nhelper = new NHibernateHelper(SERVER, CATALOG, PORT, GAHCEDB, USER, PASS);
nhelper.OpenSession().GetCurrentSession()
{% endhighlight %}

Ничего особенного, просто создание соединения и получения объекта сессии (**`ISession`**).

А вот далее начинаются интересные методы (причем они могут быть легко связаны - **chained**):

{% highlight csharp %}
CreateSQLQuery("EXEC sp_Database_Size ?")
{% endhighlight %}

Функция содержащая тело SQL запроса. Так как мы вызываем процедуру с одним **`INPUT`** параметром (помните, мы **`OUTPUT`** пока убрали?), то это надо отразить в запросе. Вопросительные знаки - один из способов задачи аргументов (вообще есть [два способа](https://docs.jboss.org/hibernate/orm/3.3/reference/en-US/html/querysql.html) - позиционная (как у нас) и именованная). 

Количество вопросов равно количеству входных аргументов процедуры.

{% highlight csharp %}
AddScalar("LogDataSize", NHibernateUtil.Decimal)
AddScalar("RowDataSize", NHibernateUtil.Decimal)
AddScalar("OverallSize", NHibernateUtil.Decimal)
{% endhighlight %}

Эти три строчки - явное определение колонок (имени и типа) в возвращаемом результате. Использование метода опционально. Для указания типа используйте пространство имен **`NHibernateUtil`**.

{% highlight csharp %}
SetParameter(0, "GAHCEPDB", NHibernateUtil.String)
{% endhighlight %}

Задача значения для **параметра**. Первый аргумент - позиционный **номер** аргумента. Так как у нас один и единственный параметр, то тут у нас **0**. Позиция задается в случае использования **позиционной** схемы задания параметров. Как быть в случае с **именованной** рассмотрено чуть ниже.

{% highlight csharp %}
SetResultTransformer(Transformers.AliasToBean<DatabaseSizePayload>()).List<DatabaseSizePayload>();
{% endhighlight %}

Это формат задачи преобразования **выходного** результата. **`AliasToBean`** метод является факторным методом над статичным классом **`Transformers`**. 

В качестве контейнера для хранения результата выступает класс **`DatabaseSizePayload`**. Это одна из интересных функций **NHibernate** - преобразование он делает за вас, но для того, чтобы это преобразование было возможным и не вызвало исключений, необходимо, чтобы методы класса:

 * именовались также как и параметры процедуры
 * имели публичные сеттеры
 * были строго совместимого типа с параметрами процедуры.

Для запуска преобразования полученного результата в список вызываем метод **`List<DatabaseSizePayload>()`**.

Кстати, можно писать свои кастомные классы трансформации (через реализацию **`IResultTransformer`** интерфейса) полученного результата.

**Класс DatabaseSizePayload:**

{% highlight csharp %}
public class DatabaseSizePayload
{
    public Decimal LogDataSize { get; set; }
    
    public Decimal RowDataSize { get; set; }

    public Decimal OverallSize { get; set; }
}
{% endhighlight %}


Мы не рассмотрели как быть в случае **именованной**, а не **позиционной** схемы задачи параметров.

В этом случае меняется формат представления аргументов в строке SQL запроса и в функции определения входных параметров **номер позиции** меняется на **имя аргумента**:

**Позиционная схема:**

{% highlight csharp %}
...
CreateSQLQuery("EXEC sp_Database_Size ?")
...
SetParameter(0, "GAHCEPDB", NHibernateUtil.String)
...
{% endhighlight %}

**Именованная схема:**

{% highlight csharp %}
...
CreateSQLQuery("EXEC sp_Database_Size :DBName")
...
SetParameter("DBName", "GAHCEPDB", NHibernateUtil.String)
...
{% endhighlight %}


Довольно просто.

---

## Ссылки

1. [Официальный сайт Fluent NHibernate](http://www.fluentnhibernate.org/)
2. [Fluent NHibernate: Getting started](https://github.com/jagregory/fluent-nhibernate/wiki/Getting-started)
3. [Как задавать имена в процедуре?](https://docs.jboss.org/hibernate/orm/3.3/reference/en-US/html/querysql.html)


