async with
----------

До сих пор подключение выполнялось не в менеджере контекста. Модуль asynssh
может выполнять подключение в менеджере контекста, но не в обычном блоке
``with``, а в ``async with``. Для асинхронного менеджера контекста созданы
отдельные специальные методы ``__aenter__``, ``__aexit__``, которые равнозначны
синхронным вариантам по смыслу, но при этом являются сопрограммами.

Пример подключения с помощью асинхронного менеджера контекста:

.. code:: python

    async def send_show(host, username, password, enable_password, command):
        print(f"Подключение к {host}")
        async with asyncssh.connect(
            host=host,
            username=username,
            password=password,
            encryption_algs="+aes128-cbc,aes256-cbc",
        ) as ssh:
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
            return output



