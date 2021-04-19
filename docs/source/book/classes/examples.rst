Примеры классов
===============

Класс для подключения по SSH с помощью asyncssh
-----------------------------------------------

.. code:: python

    from pprint import pprint
    import asyncio
    import asyncssh


    class ConnectAsyncSSH:
        def __init__(self, host, username, password, enable_password, connection_timeout=5):
            self.host = host
            self.username = username
            self.password = password
            self.enable_password = enable_password
            self.connection_timeout = connection_timeout

        async def connect(self):
            self.ssh = await asyncio.wait_for(
                asyncssh.connect(
                    host=self.host,
                    username=self.username,
                    password=self.password,
                    encryption_algs="+aes128-cbc,aes256-cbc",
                ),
                timeout=self.connection_timeout,
            )
            self.writer, self.reader, _ = await self.ssh.open_session(
                term_type="Dumb", term_size=(200, 24)
            )
            await self._read_until(">")
            self.writer.write("enable\n")
            await self._read_until("Password")
            self.writer.write(f"{self.enable_password}\n")
            await self._read_until([">", "#"])
            self.writer.write("terminal length 0\n")
            await self._read_until()

        async def _read_until(self, prompt="#", timeout=3):
            try:
                return await asyncio.wait_for(self.reader.readuntil(prompt), timeout)
            except asyncio.TimeoutError as error:
                output = ""
                while True:
                    try:
                        output += await asyncio.wait_for(self.reader.read(1000), 0.1)
                    except asyncio.TimeoutError as error:
                        break
                message = (
                    f"TimeoutError while executing self.reader.readuntil('{prompt}')\n"
                    f"Last output from device:\n{output}"
                )
                raise asyncio.TimeoutError(message)

        async def send_show_command(self, command):
            self.writer.write(command + "\n")
            output = await self._read_until()
            return output

        async def send_config_commands(self, commands):
            cfg_output = ""
            if type(commands) == str:
                commands = ["conf t", commands, "end"]
            else:
                commands = ["conf t", *commands, "end"]
            for cmd in commands:
                self.writer.write(cmd + "\n")
                cfg_output += await self._read_until()
            return cfg_output

        async def close(self):
            self.ssh.close()
            await self.ssh.wait_closed()

        async def __aenter__(self):
            await self.connect()
            return self

        async def __aexit__(self, exc_type, exc_val, exc_tb):
            await self.close()


    async def main():
        r1 = {
            "host": "192.168.100.1",
            "username": "cisco",
            "password": "cisco",
            "enable_password": "cisco",
        }
        config_commands = ["logging buffered 20010", "ip http server"]
        async with ConnectAsyncSSH(**r1) as ssh:
            print(await ssh.send_show_command("sh ip int br"))
            print(await ssh.send_config_commands(config_commands))


    if __name__ == "__main__":
        asyncio.run(main())

Пример использования класса:

.. code:: python

    from pprint import pprint
    import asyncio
    import asyncssh

    import yaml
    from ex11_asyncssh_basic_class import ConnectAsyncSSH


    async def send_show(device, command):
        host = device["host"]
        try:
            async with ConnectAsyncSSH(**device) as ssh:
                output = await ssh.send_show_command("sh ip int br")
                return output
        except asyncio.TimeoutError:
            print(f"Connection Timeout on {host}")
        except asyncssh.PermissionDenied:
            print(f"Authentication Error on {host}")
        except asyncssh.Error as error:
            print(f"{error} on {host}")


    async def send_command_to_devices(devices, command):
        coroutines = [send_show(device, command) for device in devices]
        result = await asyncio.gather(*coroutines)
        return result


    if __name__ == "__main__":
        with open("devices.yaml") as f:
            devices = yaml.safe_load(f)
        result = asyncio.run(send_command_to_devices(devices, "sh ip int br"))
        pprint(result, width=120)

Класс итератор для проверки доступности устройств ping'ом
----------------------------------------------------------

.. code:: python

    import asyncio
    from datetime import datetime
    from scrapli import AsyncScrapli
    from scrapli.exceptions import ScrapliException
    from async_timeout import timeout
    import yaml

    from ex06_async_iterator_telnet_ssh import CheckConnection, scan


    class CheckConnectionPing(CheckConnection):
        def __init__(self, device_list):
            self.device_list = device_list
            self._current_device = 0

        async def _scan_device(self, device):
            reply = await asyncio.create_subprocess_shell(
                f"ping -c 3 -n {device}",
                stdout=asyncio.subprocess.PIPE,
                stderr=asyncio.subprocess.PIPE,
            )

            stdout, stderr = await reply.communicate()
            output = (stdout + stderr).decode("utf-8")

            if reply.returncode == 0:
                return True, output
            else:
                return False, output


    async def scan(devices, protocol):
        protocol_class_map = {
            "ssh": CheckConnection,
            "telnet": CheckConnection,
            "icmp": CheckConnectionPing,
        }
        ConnectionClass = protocol_class_map.get(protocol.lower())
        if ConnectionClass:
            check = ConnectionClass(devices)
            async for status, msg in check:
                if status:
                    print(f"{datetime.now()} {protocol}. Подключение успешно: {msg}")
                else:
                    print(f"{datetime.now()} {protocol}. Не удалось подключиться: {msg}")
        else:
            raise ValueError(f"Для протокола {protocol} нет соответствующего класса")


    async def main():
        with open("devices_asyncssh.yaml") as f:
            devices_ssh = yaml.safe_load(f)
        with open("devices_asynctelnet.yaml") as f:
            devices_telnet = yaml.safe_load(f)
        ip_list = ["192.168.100.1", "8.8.8.8", "10.1.1.1"]
        await asyncio.gather(
            scan(devices_ssh, "SSH"), scan(devices_telnet, "Telnet"), scan(ip_list, "ICMP")
        )



    if __name__ == "__main__":
        asyncio.run(main())

