2. Асинхронные модули
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

   modules/asyncssh/index
   modules/scrapli/index
   modules/netdev/index
   modules/further_reading

