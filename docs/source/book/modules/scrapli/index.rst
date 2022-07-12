.. raw:: latex

   \newpage

scrapli
=======

.. note::

    Основы scrapli с использованием синхронных вариантов транспорта рассматриваются
    в книге `Python для сетевых инженеров <https://pyneng.readthedocs.io/ru/latest/book/18_ssh_telnet/scrapli.html>`__.

scrapli - это модуль, который позволяет подключаться к сетевому оборудованию
используя Telnet, SSH или NETCONF.

`scrapli <https://github.com/carlmontanari/scrapli>`__ поддерживает разные варианты
подключения: system, paramiko, ssh2, telnet, asyncssh, asynctelnet.
Тут рассматривается только asyncssh и asynctelnet.

.. toctree::
   :maxdepth: 1
   :hidden:

   basics
   textfsm
   concurrent
   telnet
   further_reading
