Semaphore
=========

.. code:: python

    import asyncio


    async def connect(ip, semaphore):
        async with semaphore:
            print(f"Подключаюсь к {ip}")
            await asyncio.sleep(1)
            print(f"Ответ от {ip}")


    async def main():
        sem = asyncio.Semaphore(20)
        coroutines = [connect(i, sem) for i in range(500)]
        await asyncio.gather(*coroutines)


    asyncio.run(main())


.. code:: python

    from pprint import pprint
    import asyncio

    import yaml
    from scrapli import AsyncScrapli
    from scrapli.exceptions import ScrapliException


    async def send_show(device, show_commands):
        print(f'>>> Подключаюсь к {device["host"]}')
        cmd_dict = {}
        if type(show_commands) == str:
            show_commands = [show_commands]
        try:
            async with AsyncScrapli(**device) as ssh:
                for cmd in show_commands:
                    reply = await ssh.send_command(cmd)
                    cmd_dict[cmd] = reply.result
                print(f'<<< Получен результат от {device["host"]}')
            return cmd_dict
        except ScrapliException as error:
            print(error, device["host"])


    async def connect_ssh_with_semaphore(semaphore, function, *args, **kwargs):
        async with semaphore:
            return await function(*args, **kwargs)


    async def send_command_to_devices(devices, commands, max_workers=50):
        semaphore = asyncio.Semaphore(max_workers)
        coroutines = [
            connect_ssh_with_semaphore(semaphore, send_show, device, commands)
            for device in devices
        ]
        result = await asyncio.gather(*coroutines)
        return result


    if __name__ == "__main__":
        with open("devices_async.yaml") as f:
            devices = yaml.safe_load(f)
        result = asyncio.run(send_command_to_devices(devices, "sh ip int br"))
        pprint(result, width=120)

