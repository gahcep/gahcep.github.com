---
layout: post
title: Quality Assurance in Python - работаем с unittest
category: QA, Python
tags: qa, python, unittest
author: gaHcep
year: 2013
month: 2
day: 10
published: true
summary: Знакомимся с тестированием в Python. Анонс цикла статей 'QA in Python'. Изучаем модуль unittest, принципы написания тестов, классов и модулей, а также механизмы работы с тестами.
---
 
# Quality Assurance in Python - работаем с unittest

Эта статья будет посвящена работе с тестированием (Quality Assurance - QA) в Python. Как я рассчитываю, сегодняшний пост один из первых в цикле по работе с QA в Python. Планирую в цикле затронуть разные unit test framework'и (**unittest**, **nose**), а также рассмотреть некоторые интересные библиотеки (**proboscis**). Мы узнаем основной функционал этих библиотек, копнем их поглубже, рассмотрим вопросы тестирования сложных систем с наличием зависимостей между тестами (зло? возможно, но в некоторых случаях оно неизбежно), напишем свой фреймворк поверх unittest для реализации более продвинутой системы отчета, а также других фич: настройка порядка выполнения тестов и возможность задачи зависимостей между ними (test order and test dependency).

Сегодня рассмотрим работу с основой юнит тестирования в Python'е - модулем unittest. Работать будем с Python 2.7

## Терминология

- [**Test Case**](http://en.wikipedia.org/wiki/Test_case) – тестовый сценарий (набор проверяемых условий, переменных, состояний системы или режимов). Обычно является логически неделимым и может содержать одну или более проверок (asserts). Здесь и далее, под тестом я понимаю именно Test Case, но в контексте Python документации, часто Test Case - это класс, отнаследованный от unittest.TestCase(). Так вот, в данной статье, клас есть класс, а Test Case - метод этого класса (начинающийся с **test_***)
- [**Test Suite**](http://en.wikipedia.org/wiki/Test_suite) – набор Test Case в рамках одного класса либо в рамках модуля. Группировка в тестовые наборы (Test Suite) обычно происходит по функциональным или логическим признакам. 
- [**Test Fixture**](http://en.wikipedia.org/wiki/Test_fixture) – совокупность функций или средств для консистентного прогона тестового сценария
- **setUp** – функция, реализующая предварительную подготовку (создания экземпляров классов, открытие соединений) для прогона тестов. Может относится к конкретному тесту (**setUp**), к набору тестов (**setUpClass**) или к модулю (**setUpModule**).
- **tearDown** – функция, реализующая окончательную очистку данных и закрытия всех ресурсов


## unittest - начало работы

Рассмотрим простой пример:

[**`pyexample_1.py`**](https://gist.github.com/gahcep/4749593)
{% highlight python %}
import unittest

class BaseTestClass(unittest.TestCase):

    def test_add(self):
        self.assertEquals(120, 100 + 20)
        self.assertFalse(10 > 20)
        self.assertGreater(120, 100)

    def test_sub(self):
        self.assertEquals(100, 140 - 40)

if __name__ == '__main__':
 unittest.main()
{% endhighlight %}

**Запуск файла:**

{% highlight bash %}
D:\Projects\PyUnitTesting\pyunittest>pyexample_1.py -v
test_add (__main__.BaseTestClass) ... ok
test_sub (__main__.BaseTestClass) ... ok

----------------------------------------------------------------------
Ran 2 tests in 0.001s
{% endhighlight %}

Перед нами:

 - **`import unittest`** – подключение unittest модуля
 - **`class BaseTestClass(unittest.TestCase)`** - объявление Test Class'а. Для распознавания функций в классе и интерпретации их как тесты, необходимо чтобы ***класс наследовался от unittest.TestCase*** и ***тесты в классе должны начинаться с префикса `test`***
 - **`def test_add(self):`** и **`def test_sub(self):`** – определения тестов
 - **`unittest.main()`** - запуск всех тестов в текущем модуле. 

> Почему **`main()`**? Под вызовом **`main()`** мы вызываем главный класс **`unittest`** модуля **`TestProgram()`**, который в свою очередь прописывает дефолтные инстансы классов загрузки (**`testLoader`**) и прогона (**`testRunner`**) тестов. Ну а после идет вызов **`TestProgram.runTests()`**.

> Вот как выглядит определение **`main`** в файле **`unittest.main.py::233`**:

> **`main = TestProgram`**

## unittest - модуль с классами

Рассмотрим теперь пример посложнее:

[**`pyexample_2.py`**](https://gist.github.com/gahcep/4749595)
{% highlight python %}
import unittest

class FirstTestClass(unittest.TestCase):

    def test_add(self):
        self.assertEquals(120, 100 + 20)

class SecondTestClass(unittest.TestCase):

    def test_sub(self):
        self.val = 210
        self.assertEquals(210, self.val)
        self.val = self.val - 40
        self.assertEquals(170, self.val)

    def test_mul(self):
        self.val = 210
        self.assertEquals(420, self.val * 2)


if __name__ == '__main__':
    unittest.main()
{% endhighlight %}

**Запуск файла:**

{% highlight bash %}
D:\Projects\PyUnitTesting\pyunittest>pyexample_2.py -v
test_add (__main__.FirstTestClass) ... ok
test_mul (__main__.SecondTestClass) ... ok
test_sub (__main__.SecondTestClass) ... ok

----------------------------------------------------------------------
Ran 3 tests in 0.002s
{% endhighlight %}

Имеем два обычных класса. Тесты запускаются как обычно. 
Однако, в классе **`SecondTestClass`** в начале каждого теста заново инициализируется переменная **`val`**. Нехорошая ситуация, которая решается просто: добавлением метода **`setUp`** в тело класса. Данный метод выполняется перед каждым тестом (Test Case) в наборе (Test Suite):

[**`pyexample_2_1.py`**](https://gist.github.com/gahcep/4749598)
{% highlight python %}
class SecondTestClass(unittest.TestCase):

    def setUp(self):
        self.val = 210

    def test_sub(self):
        self.val = 210
        self.assertEquals(210, self.val)
        self.val = self.val - 40
        self.assertEquals(170, self.val)

    def test_mul(self):
        self.val = 210
        self.assertEquals(420, self.val * 2)
{% endhighlight %}

## unittest - методы test fixture

Как видно, писать простые тесты просто. Но что если перед каждым тестом в общем тест-плане надо выполнить предварительную инициализацию какого-либо ресурса либо создать экземпляр другого класса (или, к примеру, открыть файл)? А раз создали в начале, то, вестимо, надо и удалить в конце. Как быть тогда?

Для этого существует ряд методов, уже реализованных в unittest модуле:

- [**`setUp`**](http://docs.python.org/2/library/unittest.html#unittest.TestCase.setUp) – подготовка прогона теста; вызывается перед каждым тестом.
- [**`tearDown`**](http://docs.python.org/2/library/unittest.html#unittest.TestCase.tearDown) – вызывается после того, как тест был запущен и результат записан. Метод запускается даже в случае исключения (exception) в теле теста.
- [**`setUpClass`**](http://docs.python.org/2/library/unittest.html#unittest.TestCase.setUpClass) – метод вызывается перед запуском всех тестов **класса**.
- [**`tearDownClass`**](http://docs.python.org/2/library/unittest.html#unittest.TestCase.tearDownClass) – вызывается после прогона всех тестов **класса**.
- [**`setUpModule`**](http://docs.python.org/2/library/unittest.html#setupmodule-and-teardownmodule) – вызывается перед запуском всех классов **модуля**.
- [**`tearDownModule`**](http://docs.python.org/2/library/unittest.html#setupmodule-and-teardownmodule) – вызывается после прогона всех тестов **модуля**.

**`setUpClass`** и **`tearDownClass`** требуют применения вкупе со [**`@staticmethod`**](http://docs.python.org/2/library/functions.html#staticmethod) - декоратором позволяющим объявить функцию внутри класса, таким образом, что ей доступ к классу в котором она находится не особо и нужен. Кроме того, данную функцию можно вызвать как используя класс (**`Class.f()`**), так и его экземпляр (**`Class().f()`**).

**`setUpModule`** и **`tearDownModule`** - реализуются в виде отдельных функций в модуле и не входят ни в один класс модуля.

Пример:

[**`pyexample_3.py`**](https://gist.github.com/gahcep/4749600)
{% highlight python %}
import unittest

def setUpModule():
    print "In setUpModule()"

def tearDownModule():
    print "In tearDownModule()"

class FirstTestClass(unittest.TestCase):
    '''
    Invoked without setUp and tearDown methods
    '''
    def test_add(self):
        print "In test_add()"
        self.assertEquals(120, 100 + 20)

class SecondTestClass(unittest.TestCase):
    '''
    Invoked with setUpClass/setUp and tearDown/tearDownClass methods
    '''
    @staticmethod
    def setUpClass():
        print "In setUpClass()"

    def setUp(self):
        print "In setUp()"

    def test_sub(self):
        print "In test_sub()"
        self.assertEquals(210, 110 * 2 - 10)
        self.assertEquals(170, 140 - (-30))

    def test_mul(self):
        print "In test_mul()"
        self.assertEquals(420, 210 * 2)
        self.assertEquals(420, 210 * 2.0000000000000000000001)

    def tearDown(self):
        print "In tearDown()"

    @staticmethod
    def tearDownClass():
        print "In tearDownClass()"


if __name__ == '__main__':
    unittest.main()
{% endhighlight %}

**Запуск файла:**

{% highlight bash %}
D:\Projects\PyUnitTesting\pyunittest>pyexample_3.py
In setUpModule()
In test_add()
.In setUpClass()
In setUp()
In test_mul()
In tearDown()
.In setUp()
In test_sub()
In tearDown()
.In tearDownClass()
In tearDownModule()

----------------------------------------------------------------------
Ran 3 tests in 0.002s
{% endhighlight %}

Видно, что в самом начале, до методов **`setUp()`**/**`tearDown()`** идут **`setUpClass()`**/**`tearDownClass()`**, а перед ними **`setUpModule()`**/**`tearDownModule()`**.

## unittest - статусы тестов

Теперь о возможных статусах, присваиваемых тестам.

Они могут быть следующих типов (слева - при обычном выводе (default verbosity), справа - при детализированном (verbosity > 1)):

        +----+--------------------+
		| .  | ok                 |
		| F  | FAIL               |
		| E  | ERROR              |
		| x  | expected failure   |
		| s  | skipped 'msg'      |
		| u  | unexpected success |
		+-------------------------+

Посмотрим наглядно:

[**`pyexample_4_status.py`**](https://gist.github.com/gahcep/4749602)
{% highlight python %}
import unittest

class BaseTestClass(unittest.TestCase):

    def test_ok(self):
        self.assertEquals(210, 110 * 2 - 10)

    @unittest.skip('not supported')
    def test_skip(self):
        self.assertEquals(1000, 10 * 10 * 10)

    def test_fail(self):
        self.assertEquals(420, 210 * 2.1)

    def test_error(self):
        raise ZeroDivisionError('Error! Division by zero')

    @unittest.expectedFailure
    def test_expected(self):
        raise ZeroDivisionError('Error! Division by zero')

    @unittest.expectedFailure
    def test_unexpected_ok(self):
        self.assertEquals(1, 1)

if __name__ == '__main__':
    unittest.main()
{% endhighlight %}

**Запуск файла:**

{% highlight bash %}
D:\Projects\PyUnitTesting\pyunittest>pyexample_4_status.py
ExF.su
....

----------------------------------------------------------------------
Ran 6 tests in 0.001s

FAILED (failures=1, errors=1, skipped=1, expected failures=1, unexpected successes=1)
{% endhighlight %}

- Со стасусами **ok**, **FAIL**, **ERROR** все понятно: тест успешно пройден, тест завалился на верификации условия (assert), тест вызвал исключение (exception).
- Статусы **expected failure**, **skipped** и **unexpected success** интереснее.

От теста можно ожидать исключения (exception). Описывается это поведение декоратором **`@unittest.expectedFailure`**. Например, тест рассмотренный в **pyexample\_4\_status.py**:

{% highlight python %}
def test_div(self): 
    raise ZeroDivisionError('Error! Division by zero')
{% endhighlight %}

вызывает **ERROR** (E) с запуском по-умолчанию, но с декоратором **`@unittest.expectedFailure`**:

{% highlight python %}
@unittest.expectedFailure
    def test_div(self):
        raise ZeroDivisionError('Error! Division by zero')
{% endhighlight %}

у нас уже другой статус - **expected failure** (**x**).

Если же тест, помеченный этим декоратором, исключения не кидает и пройден успешно, ему присваивается статус - **unexpected success** (**u**).

Также тест можно пропустить. Делается это с использованием декоратора **`@skip`** (читайте о [разных типах](http://docs.python.org/2/library/unittest.html#skipping-tests-and-expected-failures) этого декоратора на официальном сайте). 

## Наследование (inheritance)

Представьте, что вдруг мы захотели получить такое поведение, при котором ряд классов должны в начале своей работы прогонять общий набор тестов (Test Suite). При этом писать дублирующий код не хочется. Как быть в таком случае? Ответ - наследование:

[**`pyexample_5_inheritance.py`**](https://gist.github.com/gahcep/4749700)
{% highlight python %}
import unittest

class BaseTestClass(unittest.TestCase):

    def test_base_1(self):
        self.assertEquals(210, 110 * 2 - 10)

    def test_base_2(self):
        self.assertTrue(False is not None)

class DerivedTestClassA(BaseTestClass):

    def test_derived_a(self):
        self.assertEquals(100, 10 * 10)

class DerivedTestClassB(BaseTestClass):

    def test_derived_b(self):
        self.assertEquals(45, 46 - 1)

if __name__ == '__main__':
    unittest.main()
{% endhighlight %}

**Запуск файла:**

{% highlight bash %}
D:\Projects\PyUnitTesting\pyunittest>pyexample_5_inheritance.py -v
test_base_1 (__main__.BaseTestClass) ... ok
test_base_2 (__main__.BaseTestClass) ... ok
test_base_1 (__main__.DerivedTestClassA) ... ok
test_base_2 (__main__.DerivedTestClassA) ... ok
test_derived_a (__main__.DerivedTestClassA) ... ok
test_base_1 (__main__.DerivedTestClassB) ... ok
test_base_2 (__main__.DerivedTestClassB) ... ok
test_derived_b (__main__.DerivedTestClassB) ... ok

----------------------------------------------------------------------
Ran 8 tests in 0.005s
{% endhighlight %}

Не порядок, помимо тестов наших наследуемых классов (Derived), мы также видим и тесты базового класса. Это происходит, потому что родитель (parent) базового класса – **`unittest.TestCase`**.
Чтобы описанной выше ситуации не произошло, зависимость от **`unittest`** модуля надо убрать. 

Но как, в таком случае, сделать чтобы **Derived** классы все-таки грузились автоматически? Ответ – через множественное наследование (multiple inheritance):

[**`pyexample_5_multiple_inher.py`**](https://gist.github.com/gahcep/4749703)
{% highlight python %}
import unittest

class BaseTestClass(object):

    def test_base_1(self):
        self.assertEquals(210, 110 * 2 - 10)

    def test_base_2(self):
        self.assertTrue(False is not None)

class DerivedTestClassA(unittest.TestCase, BaseTestClass):

    def test_derived_a(self):
        self.assertEquals(100, 10 * 10)

class DerivedTestClassB(unittest.TestCase, BaseTestClass):

    def test_derived_b(self):
        self.assertEquals(45, 46 - 1)

if __name__ == '__main__':
    unittest.main()
{% endhighlight %}

**Запуск файла:**

{% highlight bash %}
D:\Projects\PyUnitTesting\pyunittest>pyexample_5_multiple_inher.py -v
test_base_1 (__main__.DerivedTestClassA) ... ok
test_base_2 (__main__.DerivedTestClassA) ... ok
test_derived_a (__main__.DerivedTestClassA) ... ok
test_base_1 (__main__.DerivedTestClassB) ... ok
test_base_2 (__main__.DerivedTestClassB) ... ok
test_derived_b (__main__.DerivedTestClassB) ... ok

----------------------------------------------------------------------
Ran 6 tests in 0.002s
{% endhighlight %}


## Базовые блоки unittest

Прежде чем вы продолжим, немного об устройстве unittest.

Можно выделить несколько основных функциональных блоков пакета:

1. **TestResult** и отнаследованный от него **TextTestResult**:  
хранит результаты выполнения тестов. Для реализации своего генератора отчета, обычная практика - отнаследовать от этого класса.

2. **TextTestRunner**:  
запускает тесты на прогон и работает с **TextTestResult** в части касаемо оповещения об успехе/провале прогона.

3. **TestLoader**:  
класс предназначен для создания коллекций тестов (**Test Suite**)

Если кратко и по сути, 

 - тесты грузятся через **TestLoader** в **TestSuite**,
 - **TestRunner** принимает на вход сформированный на первом шаге **TestSuite** и запускает тесты,
 - далее, по факту прогона тестов, заполняется **TestResult**.

Можно реализовать свой **TestRunner** и подключить вместо дефолтного. Единственное условие: ваш класс должен реализовывать метод **`run().`** Также можно отнаследовать от unittest'овского **TestResult** и вызвав родительский **`__init__()`** по другому отслеживать выполнение тестов (переопределение методов **`addSuccess`**, **`startTest`**, прочее). 

Более подробно о создании своего **TestRunner** в следующей статье.

## unittest - работа с модулями

Ладно, глянули немного архитектуру, время вернуться к тестам.

Мы уже знаем как работать с тестами находящимися в одном файле - модуле. Но обычно тестовая система состоит из огромного количества файлов, содержащих не меньшее количество тестов. Как быть?

Рассмотрим синтетический пример:

[**`pyexample_6_module_a.py`**](https://gist.github.com/gahcep/4749768)
{% highlight python %}
import unittest

class ClassA(unittest.TestCase):

    def test_add_a(self):
        self.assertEquals(120, 100 + 20)

    def test_sub_a(self):
        self.assertEquals(210, 230 - 20)

    def test_mul_a(self):
        self.assertEquals(420, 105 * 4)


if __name__ == '__main__':
    unittest.main()
{% endhighlight %}


[**`pyexample_6_module_b.py`**](https://gist.github.com/gahcep/4749770)
{% highlight python %}
import unittest

class ClassB(unittest.TestCase):

    def test_add_b(self):
        self.assertEquals(120, 100 + 20)

    def test_sub_b(self):
        self.assertEquals(210, 230 - 20)

    def test_mul_b(self):
        self.assertEquals(420, 105 * 4)


class ClassC(unittest.TestCase):

    def test_add_c(self):
        self.assertEquals(120, 100 + 20)

    def test_sub_c(self):
        self.assertEquals(210, 230 - 20)

    def test_mul_c(self):
        self.assertEquals(420, 105 * 4)


if __name__ == '__main__':
    unittest.main()
{% endhighlight %}

Оба модуля находятся в одной директории.

Для прогона тестов из обоих файлов, придется вручную создать набор тестов (Test Suite) и внести в него тесты из классов. Для парсинга и занесения тестов с разных модулей мы заюзаем класс **`Testloader()`**, имеющий несколько полезных методов:

 - [loadTestsFromModule](http://docs.python.org/2/library/unittest.html#unittest.TestLoader.loadTestsFromModule)
 - [loadTestsFromName](http://docs.python.org/2/library/unittest.html#unittest.TestLoader.loadTestsFromName)
 - [loadTestsFromNames](http://docs.python.org/2/library/unittest.html#unittest.TestLoader.loadTestsFromNames)
 - [loadTestsFromTestCase](http://docs.python.org/2/library/unittest.html#unittest.TestLoader.loadTestsFromTestCase)

Для загрузки тестов из наших двух файлов, мы используем первый метод:

[**`pyexample_6.py`**](https://gist.github.com/gahcep/4749765)
{% highlight python %}
import unittest

import pyexample_6_module_a
import pyexample_6_module_b

loader = unittest.TestLoader()

suite = loader.loadTestsFromModule(pyexample_6_module_a)
suite.addTests(loader.loadTestsFromModule(pyexample_6_module_b))

runner = unittest.TextTestRunner(verbosity=2)
result = runner.run(suite)
{% endhighlight %}

Вначале мы импортируем наши модули, затем создаем экземпляр **`unittest.TestLoader()`** и грузим с помощью метода **`loadTestsFromModule()`** тесты. В конце создаем раннер для тестов.

Для пояснения последней и самой важной строчки кода, приведу наглядную иллюстрацию с [voidspace.org.uk](http://voidspace.org.uk), статья ["Introduction to unittest"](http://www.voidspace.org.uk/python/articles/introduction-to-unittest.shtml#loaders-runners-and-all-that-stuff).

![](/images/qa-in-python-unittest/unittest.png)

### Методы формирования коллекции тестов (Test Suite)

#### **`loadTestsFromModule()`**  

Этот метод мы уже использовали в **pyexample_6.py.** Он грузит все тесты со всех классов указанного модуля (в нашем случае классы ClassA, ClassB и ClassC с двух файлов-модулей):

> suite = loader.loadTestsFromModule(pyexample\_6\_module\_a)  
  suite.addTests(loader.loadTestsFromModule(pyexample\_6\_module\_b))

**Результат:**

{% highlight bash %}
D:\Projects\PyUnitTesting\pyunittest>pyexample_6.py
test_add_a (pyexample_6_module_a.ClassA) ... ok
test_mul_a (pyexample_6_module_a.ClassA) ... ok
test_sub_a (pyexample_6_module_a.ClassA) ... ok
test_add_b (pyexample_6_module_b.ClassB) ... ok
test_mul_b (pyexample_6_module_b.ClassB) ... ok
test_sub_b (pyexample_6_module_b.ClassB) ... ok
test_add_c (pyexample_6_module_b.ClassC) ... ok
test_mul_c (pyexample_6_module_b.ClassC) ... ok
test_sub_c (pyexample_6_module_b.ClassC) ... ok

----------------------------------------------------------------------
Ran 9 tests in 0.004s
{% endhighlight %}


#### **`loadTestsFromName()`** & **`loadTestsFromNames()`**

**`loadTestsFromName`** принимает строку содержащую:

- ***имя\_модуля***, и тогда метод грузит все тесты из каждого класса указанного модуля:

> suite.addTests(loader.loadTestsFromName('pyexample_6_module_b'))

**Результат:**

{% highlight bash %}
D:\Projects\PyUnitTesting\pyunittest>pyexample_6.py
test_add_b (pyexample_6_module_b.ClassB) ... ok
test_mul_b (pyexample_6_module_b.ClassB) ... ok
test_sub_b (pyexample_6_module_b.ClassB) ... ok
test_add_c (pyexample_6_module_b.ClassC) ... ok
test_mul_c (pyexample_6_module_b.ClassC) ... ok
test_sub_c (pyexample_6_module_b.ClassC) ... ok

----------------------------------------------------------------------
Ran 6 tests in 0.004s
{% endhighlight %}

- ***имя\_модуля.имя\_класса***, и тогда метод грузит все тесты только указанного класса указанного модуля:

> suite.addTests(loader.loadTestsFromName('pyexample_6_module_b.ClassB'))

**Результат:**

{% highlight bash %}
D:\Projects\PyUnitTesting\pyunittest>pyexample_6.py
test_add_b (pyexample_6_module_b.ClassB) ... ok
test_mul_b (pyexample_6_module_b.ClassB) ... ok
test_sub_b (pyexample_6_module_b.ClassB) ... ok

----------------------------------------------------------------------
Ran 3 tests in 0.002s
{% endhighlight %}

- ***имя\_модуля.имя\_класса.имя\_теста***, и тогда метод грузит только указанный тест класса

> suite.addTests(loader.loadTestsFromName('pyexample_6_module_b.ClassC.test_add_c'))

**Результат:**

{% highlight bash %}
D:\Projects\PyUnitTesting\pyunittest>pyexample_6.py
test_add_c (pyexample_6_module_b.ClassC) ... ok

----------------------------------------------------------------------
Ran 1 test in 0.001s
{% endhighlight %}

**`loadTestsFromNames`** от **`loadTestsFromName`** отличается только тем, что принимает на вход последовательность строк:

> suite.addTests(loader.loadTestsFromNames(  
  ['pyexample_6_module_b.ClassB', 'pyexample_6_module_b.ClassC.test_add_c']))

**Результат:**

{% highlight bash %}
D:\Projects\PyUnitTesting\pyunittest>pyexample_6.py
test_add_b (pyexample_6_module_b.ClassB) ... ok
test_mul_b (pyexample_6_module_b.ClassB) ... ok
test_sub_b (pyexample_6_module_b.ClassB) ... ok
test_add_c (pyexample_6_module_b.ClassC) ... ok

----------------------------------------------------------------------
Ran 4 tests in 0.003s
{% endhighlight %}

#### **`loadTestsFromTestCase()`**

Этот метод лишь грузит набор тестов (Test Case) из указанного класса (Test Suite). Пусть вас не смущает то что метод имеет в свое названии сочетание TestCase (тест), на деле он грузит не единичный тест, а все тесты с класса. В данном случае Test Case обозначает именно класс, наследуемый от **`unittest.TestCase()`**.

**Замечание:** разные **assert** функции описывать я не буду - по ним море информации и в официальных источниках и в разных блогах.

## Automatic test recovery

**Замечание:** для примеров ниже, предполагается, что все рассмотренные файлы выше находятся в одной дректории.

Допустим, все тесты написаны. Как прогнать их все разом, если они не находятся в одном файле/модуле? Вообще, тут два варианта. Для обоих необходимо создать некий стартовый (bootstrap) файл, запускающий весь процесс прогона. 

- Использовать встроенную возможность unittest - метод **`TestLoader.discover()`** - данная фича появилась в unittest [совсем недавно](http://www.voidspace.org.uk/python/articles/introduction-to-unittest.shtml):

**`[launcher.py:](https://gist.github.com/gahcep/4749800)`**

{% highlight python %}
import unittest

loader = unittest.TestLoader()

suite = loader.discover(start_dir='.', pattern='pyexample_*.py')

runner = unittest.TextTestRunner(verbosity=2)
result = runner.run(suite)
{% endhighlight %}

Для того, что авто discovery работал, необходимо, чтобы все файлы, содержащие тест кейсы представляли собой модули (modules) или пакеты (packages) в питоновском понимании.

- Использовать специальные пакеты, например, [**discover.py**](http://pypi.python.org/pypi/discover). Предварительно, убедитесь, что пакет установлен:

> pip install discover

Затем, находясь в директории с рассматриваемыми файлами, в консоли напишите: 

> python -m discover -p "pyexample_*.py"

**`-p "pyexample_*.py"`** в данном случае лишь смена шаблона по которому ищатся файлы с тестами.

В результате выполнения как первого варианта, так и второго, в исполнение запустятся все тесты всех классов всех модулей.

На этом пока все. В следующей статье цикла - создание своего **TestRunner**'а.

---
## Ссылки

1. [Официальная документация unittest](http://docs.python.org/2/library/unittest.html)
2. [PythonTestingToolsTaxonomy](http://wiki.python.org/moin/PythonTestingToolsTaxonomy)
3. [Introduction to unittest](http://www.voidspace.org.uk/python/articles/introduction-to-unittest.shtm)
4. Статья "[unittest — Unit testing framework](http://docs.python.org/2/library/unittest.html)
