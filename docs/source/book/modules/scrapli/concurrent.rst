Подключение к нескольким устройствам
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Пример подключения к нескольким устройствам:

.. code:: python

    from pprint import pprint
    import asyncio

    import yaml
    from scrapli import AsyncScrapli
    from scrapli.exceptions import ScrapliException


    async def send_show(device, show_commands):
        cmd_dict = {}
        if type(show_commands) == str:
            show_commands = [show_commands]
        try:
            async with AsyncScrapli(**device) as ssh:
                for cmd in show_commands:
                    reply = await ssh.send_command(cmd)
                    cmd_dict[cmd] = reply.result
            return cmd_dict
        except ScrapliException as error:
            print(error, device["host"])


    async def send_command_to_devices(devices, commands):
        coroutines = [send_show(device, commands) for device in devices]
        result = await asyncio.gather(*coroutines)
        return result


    if __name__ == "__main__":
        with open("devices_async.yaml") as f:
            devices = yaml.safe_load(f)
        result = asyncio.run(send_command_to_devices(devices, "sh ip int br"))
        pprint(result, width=120)

Файл devices_async.yaml:

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
      auth_password: cisco
      auth_secondary: cisco
      auth_strict_key: false
      timeout_socket: 5
      timeout_transport: 10
      platform: cisco_iosxe
      transport: asyncssh

