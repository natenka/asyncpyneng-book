Асинхронная итерация
====================

Асинхронные итераторы 

.. code:: python

    import asyncio
    from pprint import pprint
    from scrapli import AsyncScrapli
    from scrapli.exceptions import ScrapliException
    from async_timeout import timeout


    class CheckConnection:
        def __init__(self, device_list, common_ssh_params):
            self.device_list = device_list
            self.common_ssh_params = common_ssh_params
            self._current_device = 0

        async def _scan_device(self, ip):
            params = {**self.common_ssh_params, "host": ip}
            try:
                async with timeout(5):
                    async with AsyncScrapli(**params) as conn:
                        prompt = await conn.get_prompt()
                    return True, prompt
            except (ScrapliException, asyncio.exceptions.TimeoutError) as error:
                return False, f"{error} {ip}"

        async def __anext__(self):
            if self._current_device >= len(self.device_list):
                raise StopAsyncIteration
            ip = self.device_list[self._current_device]
            self._current_device += 1
            scan_results = await self._scan_device(ip)
            return scan_results

        def __aiter__(self):
            return self


    async def ssh_scan():
        devices = ["192.168.100.1", "192.168.100.2", "192.168.100.3", "192.168.100.15"]
        params = {
            "auth_username": "cisco",
            "auth_password": "cisco",
            "auth_secondary": "cisco",
            "auth_strict_key": False,
            "timeout_socket": 5,  # timeout for establishing connection
            "timeout_transport": 10,  # timeout for ssh|telnet transport
            "platform": "cisco_iosxe",
            "transport": "asyncssh",
        }

        check = CheckConnection(devices, params)
        async for status, msg in check:
            if status:
                print(f"SSH. Подключение успешно: {msg}")
            else:
                print(f"SSH. Не удалось подключиться: {msg}")


    if __name__ == "__main__":
        asyncio.run(ssh_scan())

