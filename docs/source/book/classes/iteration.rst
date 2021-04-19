Асинхронная итерация
====================

При работе с асинхронным кодом, могут использоваться как обычные итераторы
и итерируемые объекты, так и асинхронные. Асинхронные итераторы и итерируемые
объекты в целом работают так же, как и синхронные.
Для асинхронных итераторов и итерируемых объектов созданы отдельные методы
``__aiter__`` и ``__anext__``, плюс для перебора асинхронных итерируемых объектов
есть цикл ``async for``.

.. note::

    ``async for`` можно использовать только внутри сопрограммы.

**Асинхронный итерируемый объект (asynchronous iterable)** - это объект, который можно
использовать в ``async for``. В асинхронном итерируемом объекте должен быть метод
``__aiter__``, который возвращает асинхронный итератор.

**Асинхронный итератор (asynchronous iterator)** - это объект, у которого есть методы
``__anext__`` и ``__aiter__``. Метод ``__anext__`` должен возвращать awaitable объект.
При завершении итерации генерируется исключение ``StopAsyncIteration``.

Асинхронные итераторы можно создавать с помощью классов или асинхронных генераторов.
Тут рассматривается вариант через классы, а в следующем разделе рассматриваются
асинхронные генераторы.

Пример асинхронного итератора, который на каждой итерации пытается подключиться к
одному устройству с помощью scrapli:

.. code:: python

    import asyncio
    from datetime import datetime
    from scrapli import AsyncScrapli
    from scrapli.exceptions import ScrapliException
    from async_timeout import timeout
    import yaml


    class CheckConnection:
        def __init__(self, device_list):
            self.device_list = device_list
            self._current_device = 0

        async def _scan_device(self, device):
            ip = device["host"]
            try:
                async with timeout(5): # для asynctelnet
                    async with AsyncScrapli(**device) as conn:
                        prompt = await conn.get_prompt()
                    return True, prompt
            except (ScrapliException, asyncio.exceptions.TimeoutError) as error:
                return False, f"{error} {ip}"

        async def __anext__(self):
            if self._current_device >= len(self.device_list):
                raise StopAsyncIteration
            device_params = self.device_list[self._current_device]
            scan_results = await self._scan_device(device_params)
            self._current_device += 1
            return scan_results

        def __aiter__(self):
            return self

Смысл этого итератора в том чтобы проверить получится ли подключиться к оборудованию
по SSH с помощью scrapli. Если подключиться получилось, ``__anext__`` возвращает
True и приглашение устройства, если нет - False и исключение.

.. note::

    Обратите внимание на то, что метод ``__anext__`` сопрограмма, а ``__aiter__`` нет.

Для перебора асинхронного итератора надо использовать ``async for`` и соответственно
перебор надо делать в сопрограмме:

.. code:: python

    async def ssh_scan(devices):
        check = CheckConnection(devices)
        async for status, msg in check:
            if status:
                print(f"{datetime.now()} SSH. Подключение успешно: {msg}")
            else:
                print(f"{datetime.now()} SSH. Не удалось подключиться: {msg}")


    if __name__ == "__main__":
        with open("devices_asyncssh.yaml") as f:
            devices = yaml.safe_load(f)
        asyncio.run(ssh_scan(devices))

Содержимое файла devices_asyncssh.yaml (первые два устройства доступны и с правильными
параметрами, третье доступно, но указан неправильный пароль, четвертое недоступно):

.. code:: yaml

    - host: 192.168.100.1
      auth_username: cisco
      auth_password: cisco
      auth_secondary: cisco
      auth_strict_key: false
      timeout_socket: 5
      timeout_transport: 10
      platform: cisco_iosxe
      transport: asyncssh
    - host: 192.168.100.2
      auth_username: cisco
      auth_password: cisco
      auth_secondary: cisco
      auth_strict_key: false
      timeout_socket: 5
      timeout_transport: 10
      platform: cisco_iosxe
      transport: asyncssh
    - host: 192.168.100.3
      auth_username: cisco
      auth_password: ciscoe
      auth_secondary: cisco
      auth_strict_key: false
      timeout_socket: 5
      timeout_transport: 10
      platform: cisco_iosxe
      transport: asyncssh
    - host: 192.168.100.11
      auth_username: cisco
      auth_password: cisco
      auth_secondary: cisco
      auth_strict_key: false
      timeout_socket: 5
      timeout_transport: 10
      platform: cisco_iosxe
      transport: asyncssh

Вызов скрипта:

::

    $ python ex05_async_iterator_ssh.py
    2021-04-19 11:25:40.775305 SSH. Подключение успешно: R1#
    2021-04-19 11:25:41.334456 SSH. Подключение успешно: R2#
    2021-04-19 11:25:43.638459 SSH. Не удалось подключиться: all authentication methods failed 192.168.100.3
    2021-04-19 11:25:48.647160 SSH. Не удалось подключиться: timed out opening connection to device 192.168.100.11


Итератор CheckConnection проверяет устройства последовательно, но вместе с этим
итератором, параллельно можно запускать что-то еще, например, паралелльно проверять
подключение telnet (файл ex06_async_iterator_telnet_ssh.py):

.. code:: python

    async def scan(devices, protocol):
        check = CheckConnection(devices)
        async for status, msg in check:
            if status:
                print(f"{datetime.now()} {protocol}. Подключение успешно: {msg}")
            else:
                print(f"{datetime.now()} {protocol}. Не удалось подключиться: {msg}")


    async def main():
        with open("devices_asyncssh.yaml") as f:
            devices_ssh = yaml.safe_load(f)
        with open("devices_asynctelnet.yaml") as f:
            devices_telnet = yaml.safe_load(f)
        await asyncio.gather(scan(devices_ssh, "SSH"), scan(devices_telnet, "Telnet"))


    if __name__ == "__main__":
        asyncio.run(main())

Вызов скрипта:

::

    $ python ex06_async_iterator_telnet_ssh.py
    2021-04-19 11:30:14.820195 SSH. Подключение успешно: R1#
    2021-04-19 11:30:14.983307 Telnet. Подключение успешно: R1#
    2021-04-19 11:30:15.296011 SSH. Подключение успешно: R2#
    2021-04-19 11:30:15.449338 Telnet. Подключение успешно: R2#
    2021-04-19 11:30:17.599259 SSH. Не удалось подключиться: all authentication methods failed 192.168.100.3
    2021-04-19 11:30:19.775107 Telnet. Не удалось подключиться: username/login prompt seen more than once, assuming auth failed 192.168.100.3
    2021-04-19 11:30:22.603411 SSH. Не удалось подключиться:  192.168.100.11
    2021-04-19 11:30:24.777987 Telnet. Не удалось подключиться:  192.168.100.11

