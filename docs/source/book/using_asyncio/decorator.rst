Декораторы для сопрограмм
========================

Декораторы для сопрограмм в целом создаются так же, как и 
`декораторы для обычных функций <https://advpyneng.readthedocs.io/ru/latest/book/08_decorators/basics.html>`__.
Основное отличие в том, что если декоратор должен подменять функцию
(это делается в большинстве декораторов функций), надо чтобы декоратор подменял
сопрограмму сопрограммой.

Пример декоратора timecode, который замеряет время выполнения сопрограммы:

.. code:: python

    import asyncio
    from datetime import datetime
    from functools import wraps
    import yaml
    from scrapli import AsyncScrapli
    from scrapli.exceptions import ScrapliException


    def timecode(function):
        @wraps(function)
        async def wrapper(*args, **kwargs):
            start_time = datetime.now()
            result = await function(*args, **kwargs)
            print(">>> Функция выполнялась:", datetime.now() - start_time)
            return result
        return wrapper


    @timecode
    async def send_show(device, command):
        try:
            async with AsyncScrapli(**device) as conn:
                result = await conn.send_command(command)
                await asyncio.sleep(2)
                return result.result
        except ScrapliException as error:
            print(error, device["host"])


    if __name__ == "__main__":
        with open("devices_async.yaml") as f:
            devices = yaml.safe_load(f)
            r1 = devices[0]
        result = asyncio.run(send_show(r1, "sh clock"))
        print(result)

Для того чтобы замерить время выполнения send_show, надо дождаться выполнения
сопрограммы - сделать ``await``, а ``await`` можно писать только внутри сопрограммы,
соответственно функция wrapper должна быть сопрограммой.

