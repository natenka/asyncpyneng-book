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

   async_modules/asyncssh/index
   async_modules/scrapli/index
   async_modules/netdev/index
   async_modules/async_http/index
   async_modules/further_reading

