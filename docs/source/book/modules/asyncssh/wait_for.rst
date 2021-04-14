asyncio.wait_for
----------------

У метода readuntil есть одна проблема - у него нет параметра timeout, в итоге,
если указанная строка не найдена, метод зависает, пока соединение не прервется.
Исправить это можно с помощью функции ``asyncio.wait_for``:

.. code:: python

    asyncio.wait_for(aw, timeout)

Функция wait_for запускает awaitable и ждет его выполнение указанный timeout.
Если сопрограмма не выполнилась за timeout, генерируется исключение asyncio.TimeoutError.

С использованием ``asyncio.wait_for`` функция send_show будет выглядеть так:

.. code:: python

    async def send_show(host, username, password, enable_password, command):
        print(f"Подключение к {host}")
        ssh = await asyncssh.connect(
            host=host,
            username=username,
            password=password,
            encryption_algs="+aes128-cbc,aes256-cbc",
        )
        writer, reader, stderr = await ssh.open_session(
            term_type="Dumb", term_size=(200, 24)
        )
        try:
            await asyncio.wait_for(reader.readuntil(">"), timeout=3)
            writer.write("enable\n")
            await asyncio.wait_for(reader.readuntil("Password"), timeout=3)
            writer.write(f"{enable_password}\n")
            await asyncio.wait_for(reader.readuntil([">", "#"]), timeout=3)
            writer.write("terminal length 0\n")
            await asyncio.wait_for(reader.readuntil("#"), timeout=3)

            print(f"Отправка команды {command} на {host}")
            writer.write(f"{command}\n")
            output = await asyncio.wait_for(reader.readuntil("#"), timeout=3)
            ssh.close()
            return output
        except asyncio.TimeoutError as error:
            print("TimeoutError при выполнении reader.readuntil")


Так как писать ``asyncio.wait_for`` для каждого вызова ``reader.readuntil``
не очень удобно, можно сделать отдельную сопрограмму, которая выполняет эти
операции, а также выводит последний вывод, который был получен с устройства,
если readuntil не дождался нужного символа:

.. code:: python

    async def read_until(reader, line, timeout=3):
        try:
            return await asyncio.wait_for(reader.readuntil(line), timeout)
        except asyncio.TimeoutError as error:
            output = ""
            while True:
                try:
                    output += await asyncio.wait_for(reader.read(1000), 0.1)
                except asyncio.TimeoutError as error:
                    break
            print(
                f"TimeoutError при выполнении reader.readuntil('{line}')\n"
                f"Последний вывод:"
            )
            print(output)

Функция read_until сначала пытается выполнить ``reader.readuntil`` с указанным
разделителем, при этом ``asyncio.wait_for`` ждет выполнения ``reader.readuntil``
только указанный timeout. Если разделитель line не найден, функция ``read_until``
пытается считать доступный вывод с помощью метода ``reader.read``.

Теперь функция send_show выглядит так:

.. code:: python

    async def send_show(host, username, password, enable_password, command):
        print(f"Подключение к {host}")
        ssh = await asyncssh.connect(
            host=host,
            username=username,
            password=password,
            encryption_algs="+aes128-cbc,aes256-cbc",
        )
        writer, reader, stderr = await ssh.open_session(
            term_type="Dumb", term_size=(200, 24)
        )
        await read_until(reader, ">")
        writer.write("enable\n")
        await read_until(reader, "Password")
        writer.write(f"{enable_password}\n")
        await read_until(reader, [">", "#"])
        writer.write("terminal length 0\n")
        await read_until(reader, "#")

        print(f"Отправка команды {command} на {host}")
        writer.write(f"{command}\n")
        output = await read_until(reader, "#")
        ssh.close()
        return output

И если указанная строка не была найдена в выводе, функция read_until
выведет такую информацию на stdout:

::

    TimeoutError при выполнении reader.readuntil('>')
    Последний вывод:
    sh ip int br
    Interface                  IP-Address      OK? Method Status                Protocol
    Ethernet0/0                192.168.100.3   YES NVRAM  up                    up
    Ethernet0/1                10.100.23.3     YES NVRAM  up                    up
    Ethernet0/2                unassigned      YES NVRAM  administratively down down
    Ethernet0/3                unassigned      YES NVRAM  administratively down down
    Loopback9                  unassigned      YES unset  up                    up
    R3#

Соответственно будет видно при выполнении какой команды возникло исключение
``asyncio.TimeoutError`` и какой вывод был получен с устройства.

