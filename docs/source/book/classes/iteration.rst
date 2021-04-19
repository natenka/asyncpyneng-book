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

Пример асинхронного итератора:

.. code:: python

    import asyncio
    from pprint import pprint
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
                async with timeout(5):
                    async with AsyncScrapli(**device) as conn:
                        prompt = await conn.get_prompt()
                    return True, prompt
            except (ScrapliException, asyncio.exceptions.TimeoutError) as error:
                return False, f"{error} {ip}"

        async def __anext__(self):
            if self._current_device >= len(self.device_list):
                raise StopAsyncIteration
            device_params = self.device_list[self._current_device]
            self._current_device += 1
            scan_results = await self._scan_device(device_params)
            return scan_results

        def __aiter__(self):
            return self


    async def ssh_scan(devices):
        check = CheckConnection(devices)
        async for status, msg in check:
            if status:
                print(f"SSH. Подключение успешно: {msg}")
            else:
                print(f"SSH. Не удалось подключиться: {msg}")


    if __name__ == "__main__":
        with open("devices_async.yaml") as f:
            devices = yaml.safe_load(f)
        asyncio.run(ssh_scan(devices))

