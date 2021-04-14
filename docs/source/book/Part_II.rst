II. Асинхронные модули
######################

Почти все модули, которые работали в многопоточном варианте, не будут работать
с asyncio, так как они будут блокировать работу и не будут отдавать управление
циклу событий.

В этом разделе рассматриваются модули:

* asyncssh
* scrapli
* netdev


.. toctree::
   :maxdepth: 1

   async_modules/2_1_asyncssh/index.rst
   async_modules/2_2_scrapli/index.rst
   async_modules/2_3_netdev/index.rst
   async_modules/2_4_async_http/index.rst

