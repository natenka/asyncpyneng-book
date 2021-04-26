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

Аналогичный обычный декоратор:

.. code:: python

    def timecode(function):
        @wraps(function)
        def wrapper(*args, **kwargs):
            start_time = datetime.now()
            result = function(*args, **kwargs)
            print('>>> Функция выполнялась:', datetime.now() - start_time)
            return result
        return wrapper

.. note::

    Измерение времени выполнения сопрограммы неоднозначное занятие, так как оно
    зависит не только от самой сопрограммы, но и от того что еще работает в цикле
    событий.

Встроенные декораторы
---------------------

Многие встроенные декораторы работают с функциями/методами, которые являются
сопрограммами: functools.wraps, classmethod, staticmethod.

.. note::

    Сопрограммы можно декорировать и property, но по смыслу property обычно
    применяется к чему-то, что выполняется быстро, поэтому стоит, как минимум,
    подумать о том надо ли делать property метод, который выполняет операции ввода-вывода.

Пример использования classmethod:

.. code:: python

    from pprint import pprint
    import asyncio
    import asyncssh


    class ConnectAsyncSSH:

        @classmethod
        async def connect(cls, host, username, password, enable_password, connection_timeout=5):
            self = cls()

            self.host = host
            self.username = username
            self.password = password
            self.enable_password = enable_password
            self.connection_timeout = connection_timeout
            self._ssh = await asyncio.wait_for(
                asyncssh.connect(
                    host=self.host,
                    username=self.username,
                    password=self.password,
                    encryption_algs="+aes128-cbc,aes256-cbc",
                ),
                timeout=self.connection_timeout,
            )
            self.writer, self.reader, _ = await self._ssh.open_session(
                term_type="Dumb", term_size=(200, 24)
            )
            await self.reader.readuntil(">")
            self.writer.write("enable\n")
            await self.reader.readuntil("Password")
            self.writer.write(f"{self.enable_password}\n")
            await self.reader.readuntil("#")
            self.writer.write("terminal length 0\n")
            await self.reader.readuntil("#")
            return self

        async def get_prompt(self):
            self.writer.write("\n")
            output = await self.reader.readuntil("#")
            return output

Пример использования staticmethod:

.. code:: python

    import asyncio

    class PingIP:
        def __init__(self, ip_list):
            self.ip_list = ip_list

        @staticmethod
        async def _ping(ip):
            reply = await asyncio.create_subprocess_shell(
                f"ping -c 3 -n {ip}",
                stdout=asyncio.subprocess.PIPE,
                stderr=asyncio.subprocess.PIPE,
            )
            stdout, stderr = await reply.communicate()
            ip_is_reachable = reply.returncode == 0
            return ip, ip_is_reachable

        async def scan(self):
            ping_ok = []
            ping_not_ok = []
            coroutines = [self._ping(ip) for ip in self.ip_list]
            result = await asyncio.gather(*coroutines)
            for ip, status in result:
                if status:
                    ping_ok.append(ip)
                else:
                    ping_not_ok.append(ip)
            return ping_ok, ping_not_ok


    if __name__ == "__main__":
        ip_list = ["192.168.100.1", "192.168.100.2", "192.168.100.3", "192.168.100.11"]
        scanner = PingIP(ip_list)
        results = asyncio.run(scanner.scan())
        print(results)

