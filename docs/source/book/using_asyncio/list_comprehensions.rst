list/dict/set comprehensions
============================

В list/dict/set comprehensions можно использовать await и, при переборе
асинхронного итератора - ``async for``.

Пример использования list comprehensions в сопрограмме:

.. code:: python

    from pprint import pprint
    import asyncio
    import yaml
    from scrapli import AsyncScrapli
    from scrapli.exceptions import ScrapliException


    async def send_show(device, command):
        print(f">>> connect to {device['host']}")
        try:
            async with AsyncScrapli(**device) as conn:
                result = await conn.send_command(command)
                return result.result
        except ScrapliException as error:
            print(error, device["host"])


    async def send_command_to_devices(devices, commands):
        tasks = [asyncio.create_task(send_show(device, commands)) for device in devices]
        result = [await task for task in tasks]
        return result


    if __name__ == "__main__":
        with open("devices_async.yaml") as f:
            devices = yaml.safe_load(f)
        result = asyncio.run(send_command_to_devices(devices, "sh clock"))
        pprint(result, width=120)


Тут с помощью обычного list comprehensions (без async/await) создается список задач:

.. code:: python

    tasks = [asyncio.create_task(send_show(device, commands)) for device in devices]

А в этом используется await чтобы получить результаты каждой задачи:

.. code:: python

    result = [await task for task in tasks]


Также в list comprehensions может использоваться ``async for``, при переборе
асинхронного итератора.

.. code:: python

    import asyncio                                                                                                                                                                                                                              from pprint import pprint
    from datetime import datetime
    from scrapli import AsyncScrapli
    from scrapli.exceptions import ScrapliException
    import yaml


    class CheckConnection:
        def __init__(self, device_list):
            self.device_list = device_list
            self._current_device = 0

        async def _scan_device(self, device):
            ip = device["host"]
            try:
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


    async def ssh_scan(devices):
        check = CheckConnection(devices)
        results = [out async for out in check]
        return results


    if __name__ == "__main__":
        with open("devices_asyncssh.yaml") as f:
            devices = yaml.safe_load(f)
        result = asyncio.run(ssh_scan(devices))
        pprint(result)
        for status, msg in result:
            if status:
                print(f"{datetime.now()} SSH. Подключение успешно: {msg}")
            else:
                print(f"{datetime.now()} SSH. Не удалось подключиться: {msg}")


В примере выше в list comp используется именно ``async for`` потому что выполняется
перебор асинхронного итератора:

.. code:: python

    results = [out async for out in check]


