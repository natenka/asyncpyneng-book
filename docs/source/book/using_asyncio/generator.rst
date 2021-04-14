Асинхронные генераторы
======================


.. code:: python

    import csv
    import asyncio
    import aiofiles


    async def open_csv(filename):
        async with aiofiles.open(filename) as f:
            headers = await f.readline()
            headers = list(csv.reader([headers]))[0]
            async for line in f:
                print("open_csv line")
                yield dict(list(csv.DictReader([line], fieldnames=headers))[0])


    async def filter_prefix_next_hop(async_iterable, nexthop):
        async for line in async_iterable:
            if line["nexthop"] == nexthop:
                yield line


    async def parse_data(filename):
        data = open_csv(filename)
        nhop_45 = filter_prefix_next_hop(data, "200.219.145.45")
        async for line in nhop_45:
            print("parse_data line")
            print(line)


    async def main():
        result = await parse_data("rib.table.lg.ba.ptt.br-BGP.csv")


    if __name__ == "__main__":
        asyncio.run(main())
