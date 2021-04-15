Создание экземпляра
===================

В синхронном варианте, при создании класса который подключается
по SSH, само подключение обычно будет выполняться в методе ``__init__``
(напрямую или через вызов другого метода).
В асинхронных классах так сделать не получится, так как для выполнения
подключения в ``__init__``, надо сделать init сопрограммой. Так как init
должен возвращать None, если сделать init сопрограммой, возникнет ошибка:

.. code:: python

    In [3]: class ConnectSSH:
       ...:     async def __init__(self):
       ...:         await asyncio.sleep(0.1)
       ...:

    In [5]: r1 = ConnectSSH()
    ---------------------------------------------------------------------------
    TypeError                                 Traceback (most recent call last)
    <ipython-input-5-ceb9b33fd87d> in <module>
    ----> 1 r1 = ConnectSSH()

    TypeError: __init__() should return None, not 'coroutine'


Пожалуй, самый распространенный вариант решения проблемы - создание отдельного
метода для подключения (вместе с добавлением асинхронного менеджера контекста).

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

        async def get_prompt(self):
            self.writer.write("\n")
            output = await self.reader.readuntil("#")
            return output

В этом случае для подключения надо сначала создать экземпляр класса, а потом
вызвать метод connect где выполняется само подключение:

.. code:: python

    async def main():
        r1 = {
            "host": "192.168.100.1",
            "username": "cisco",
            "password": "cisco",
            "enable_password": "cisco",
        }
        ssh = ConnectAsyncSSH(**r1)
        await ssh.connect()
        print(await ssh.get_prompt())


    if __name__ == "__main__":
        asyncio.run(main())

Как правило, вместе с таким вариантом используется асинхронный менеджер контекста.
В этом случае, метод connect вызывается в методе ``__aenter__`` (аналог ``__enter__``).
Асинхронный менеджер контекста рассматривается позже.

classmethod
-----------

Еще один распространенный вариант - использование classmethod для создания экземпляра.
В примере ниже classmethod connect единственный способ создания экземлпяра:

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

Создание экземпяра выглядит так:

.. code:: python

    async def main():
        r1 = {
            "host": "192.168.100.1",
            "username": "cisco",
            "password": "cisco",
            "enable_password": "cisco",
        }
        ssh = await ConnectAsyncSSH.connect(**r1)
        print(await ssh.get_prompt())


    if __name__ == "__main__":
        asyncio.run(main())

Второй вариант - оставить init и обычный метод connect и добавить отдельный classmethod:

.. code:: python

    class ConnectAsyncSSH:
        def __init__(self, host, username, password, enable_password, connection_timeout=5):
            self.host = host
            self.username = username
            self.password = password
            self.enable_password = enable_password
            self.connection_timeout = connection_timeout

        async def connect(self):
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

        @classmethod
        async def create_connection(cls, host, username, password, enable_password, connection_timeout=5):
            self = cls(host, username, password, enable_password, connection_timeout)
            await self.connect()
            return self

        async def get_prompt(self):
            self.writer.write("\n")
            output = await self.reader.readuntil("#")
            return output


Создание экземпяра через classmethod выглядит так, но при этом можно создавать
экземпляр и через init + connect:

.. code:: python

    async def main():
        r1 = {
            "host": "192.168.100.1",
            "username": "cisco",
            "password": "cisco",
            "enable_password": "cisco",
        }
        ssh = await ConnectAsyncSSH.create_connection(**r1)
        print(await ssh.get_prompt())

