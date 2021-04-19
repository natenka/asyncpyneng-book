4. Использование модуля asyncio
###############################

В этом разделе рассматриваются асинхронные версии привычного функционала Python:

* list/dict/set comprehensions
* декораторы для сопрограмм
* асинхронные генераторы
* asyncio subprocess

.. note::

    Синхронная версия этих тем рассматривается в `базовой книге <https://pyneng.readthedocs.io/ru/latest/>`__
    (`list comprehensions <https://pyneng.readthedocs.io/ru/latest/book/08_python_basic_examples/x_comprehensions.html>`__
    и `subprocess <https://pyneng.readthedocs.io/ru/latest/book/12_useful_modules/subprocess.html>`__)
    и в `advanced книге <https://advpyneng.readthedocs.io/ru/latest/>`__
    (`декораторы <https://advpyneng.readthedocs.io/ru/latest/book/Part_II.html>`__ и
    `генераторы <https://advpyneng.readthedocs.io/ru/latest/book/Part_IV.html>`__).

И некоторые особенности работы:

* работа с синхронными модулями при использовании asyncio
* использование semaphore для ограничения количества параллельных подключений


.. toctree::
   :maxdepth: 2

   list_comprehensions
   decorator
   generator
   subprocess
   to_thread
   semaphore

