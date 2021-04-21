Основы scrapli
--------------

`scrapli <https://github.com/carlmontanari/scrapli>`__ поддерживает разные варианты
подключения: system, paramiko, ssh2, telnet, asyncssh, asynctelnet.
Тут рассматривается только asyncssh и asynctelnet.

Доступные варианты асинхронного транспорта:

* asyncssh
* asynctelnet (реализация telnet на основе asyncio)

.. note::

    Рассматривается scrapli версии 2021.1.30.

    
Для асинхронного транспорта, как и для синхронного, в scrapli есть два варианта
подключения: используя общий класс AsyncScrapli, который выбирает нужный driver
по параметру platform или конкретный driver, например, AsyncIOSXEDriver.
При этом параметры передаются те же самые и конкретному драйверу и Scrapli.

В целом методы и параметры методов такие же как и в синхронном варианте scrapli,
отличается транспорт, драйверы и то, что методы являются сопрограммами.

Параметры подключения
~~~~~~~~~~~~~~~~~~~~~

Основные параметры подключения:

* host - IP-адрес или имя хоста
* auth_username - имя пользователя
* auth_password - пароль
* auth_secondary - пароль на enable
* auth_strict_key - контролирует проверку SSH ключей сервера, а именно разрешать
  ли подключаться к серверам ключ которых не сохранен в ssh/known_hosts.
  False - разрешить подключение (по умолчанию значение True)
* platform - нужно указывать при использовании AsyncScrapli
* transport - для async варианта transport указывать обязательно: asyncssh или asynctelnet
* transport_options - опции для конкретного транспорта

Процесс подключения немного отличается в зависимости от того используется
асинхронный менеджер контекста или нет. При подключении без менеджера контекста,
сначала надо передать параметры драйверу или AsyncScrapli, а затем вызвать метод open:

.. code:: python

    In [1]: from scrapli import AsyncScrapli

    In [2]: r1 = {
       ...:    "host": "192.168.100.1",
       ...:    "auth_username": "cisco",
       ...:    "auth_password": "cisco",
       ...:    "auth_secondary": "cisco",
       ...:    "auth_strict_key": False,
       ...:    "platform": "cisco_iosxe",
       ...:    "transport": "asyncssh",
       ...: }

    In [3]: ssh = AsyncScrapli(**r1)

    In [5]: await ssh.open()

После этого можно отправлять команды:

.. code:: python

    In [6]: await ssh.get_prompt()
    Out[6]: 'R1#'

    In [7]: await ssh.close()

При использовании асинхронного менеджера контекста, open вызывать не надо:

.. code:: python

    In [8]: async with AsyncScrapli(**r1) as ssh:
       ...:     print(await ssh.get_prompt())
    R1#


.. note::

    Отличия в подключении с менеджером контекста и без это особенность
    классов для работы с асинхронным подключением.

Использование драйвера
~~~~~~~~~~~~~~~~~~~~~~

Доступные async драйверы:

+--------------+-------------------+-------------------+
| Оборудование | Драйвер           | Параметр platform |
+==============+===================+===================+
| Cisco IOS-XE | AsyncIOSXEDriver  | cisco_iosxe       |
+--------------+-------------------+-------------------+
| Cisco NX-OS  | AsyncNXOSDriver   | cisco_nxos        |
+--------------+-------------------+-------------------+
| Cisco IOS-XR | AsyncIOSXRDriver  | cisco_iosxr       |
+--------------+-------------------+-------------------+
| Arista EOS   | AsyncEOSDriver    | arista_eos        |
+--------------+-------------------+-------------------+
| Juniper JunOS| AsyncJunosDriver  | juniper_junos     |
+--------------+-------------------+-------------------+

Пример подключения с использованием драйвера IOSXEDriver (технически
подключение выполняется к Cisco IOS):

.. code:: python

    In [10]: from scrapli.driver.core import AsyncIOSXEDriver

    In [11]: r1_driver = {
        ...:    "host": "192.168.100.1",
        ...:    "auth_username": "cisco",
        ...:    "auth_password": "cisco",
        ...:    "auth_secondary": "cisco",
        ...:    "auth_strict_key": False,
        ...:    "transport": "asyncssh",
        ...: }

    In [12]: async with AsyncIOSXEDriver(**r1_driver) as ssh:
        ...:     print(await ssh.get_prompt())
    R1#

Пример базового использования scrapli
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

В остальном, принципы работы те же, что и с синхронным вариантом.

Пример подключения к одному устройству с помощью asyncssh и AsyncScrapli:

.. code:: python

    import asyncio
    from scrapli import AsyncScrapli
    from scrapli.exceptions import ScrapliException

    r1 = {
        "host": "192.168.100.1",
        "auth_username": "cisco",
        "auth_password": "cisco",
        "auth_secondary": "cisco",
        "auth_strict_key": False,
        "timeout_socket": 5,  # timeout for establishing socket/initial connection in seconds
        "timeout_transport": 10,  # timeout for ssh|telnet transport in seconds
        "platform": "cisco_iosxe",
        "transport": "asyncssh",
    }


    async def send_show(device, command):
        try:
            async with AsyncScrapli(**device) as conn:
                result = await conn.send_command(command)
                return result.result
        except ScrapliException as error:
            print(error, device["host"])


    if __name__ == "__main__":
        output = asyncio.run(send_show(r1, "show ip int br"))
        print(output)

