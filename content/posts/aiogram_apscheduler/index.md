---
title: 'Отправка сообщения по таймеру в Telegram-Боте на Aiogram3'
date: 2024-01-22T12:20:25+03:00
draft: false
tags: ["Telegram", "Aiogram3", "Apscheduler"]
categories: ["VPS"]
description: "Разберем как отправлять отложенные сообщения на Aiogram 3"
---

Разберем как отправлять отложенные сообщения на Aiogram 3

<!--more-->

## Введение
Бывают ситуации, когда нам может понадобиться отправлять сообщения с помощью Бота раз в какое-то время.
Например:
* Окончание подписки
* Напоминание/уведомление пользователя
* Размещение постов

И вообще для чего угодно, на что хватит фантазии.

* https://t.me/S2Vel_bot - Пример моего бота на webhook с сервисом VPN. Где происходит отправка уведомлений о окончании подписки или долгой настройке.

### Apsheduler
Это планировщик задач на python который поддерживает следущие триггеры для выполнения:
* `Interval` - запускается раз в какой-то промежуток времени. Поддерживает следущие аргументы `seconds`, `minutes`, `hours`, `day`, `week`, `month` и `year`.

* `Date` - запускается в установленную дату и время.

* `Cron` - запускается с определенной периодичностью в установленную дату. Является популярным способом планирования задач.

### Echo Bot.
Теперь давайте напишем простого Бота, который будет отправлять в ответ наши сообщения, но используя `Apsheduler`:


```python
import asyncio
import logging
import sys
from os import getenv

from aiogram import Bot, Dispatcher, Router, types
from aiogram.enums import ParseMode
from aiogram.filters import CommandStart
from aiogram.types import Message
from aiogram.utils.markdown import hbold
from apscheduler.schedulers.asyncio import AsyncIOScheduler


from typing import Any, Awaitable, Callable, Dict
from aiogram import BaseMiddleware
from aiogram.types import TelegramObject
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from datetime import datetime, timedelta


# Bot token can be obtained via https://t.me/BotFather
TOKEN = "YOUR_TOKEN"

# All handlers should be attached to the Router (or Dispatcher)
dp = Dispatcher()


# Add middleware Scheduler
class SchedulerMiddleware(BaseMiddleware):
    def __init__(self, scheduler: AsyncIOScheduler):
        self.scheduler = scheduler

    async def __call__(
        self,
        handler: Callable[[TelegramObject, Dict[str, Any]], Awaitable[Any]],
        event: TelegramObject,
        data: Dict[str, Any],
    ) -> Any:
        # add apscheduler to data
        data["apscheduler"] = self.scheduler
        return await handler(event, data)


async def send_message_scheduler(bot: Bot, message: str, user_id: int):
    await bot.send_message(chat_id=int(user_id), text=message)


@dp.message(CommandStart())
async def command_start_handler(message: Message) -> None:
    """
    This handler receives messages with `/start` command
    """
    # Most event objects have aliases for API methods that can be called in events' context
    # For example if you want to answer to incoming message you can use `message.answer(...)` alias
    # and the target chat will be passed to :ref:`aiogram.methods.send_message.SendMessage`
    # method automatically or call API method directly via
    # Bot instance: `bot.send_message(chat_id=message.chat.id, ...)`
    await message.answer(f"Hello, {hbold(message.from_user.full_name)}!")


@dp.message()
async def echo_handler(
    message: types.Message, bot: Bot, apscheduler: AsyncIOScheduler
) -> None:
    """
    Handler will forward receive a message back to the sender

    By default, message handler will handle all message types (like a text, photo, sticker etc.)
    """
    try:
        # Send a copy of the received message
        dt = datetime.now() + timedelta(seconds=5)
        print(dt, bot, message.chat.id, message.text)
        apscheduler.add_job(
            send_message_scheduler,
            trigger="date",
            run_date=dt,
            kwargs={
                "bot": bot,
                "message": message.text,
                "user_id": message.chat.id,
            },
        )
    except TypeError:
        # But not all the types is supported to be copied so need to handle it
        await message.answer("Nice try!")


async def main() -> None:
    # Initialize Bot instance with a default parse mode which will be passed to all API calls
    bot = Bot(TOKEN, parse_mode=ParseMode.HTML)
    # And the run events dispatching
    scheduler = AsyncIOScheduler()
    timezone="Europe/Moscow"

    dp.update.middleware(
        SchedulerMiddleware(scheduler=scheduler),
    )
    scheduler.start()
    await dp.start_polling(bot)


if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO, stream=sys.stdout)
    asyncio.run(main())
```

Теперь более подробно посмотрим что мы написали.
Первым делом мы создаем объект `scheduler = AsyncIOScheduler()` внути которого можно например указать таймзону для вашего удобства `AsyncIOScheduler(timezone="Europe/Moscow")`. Затем мы создаем `middleware`, что бы мы могли в полученных сообщениях использовать объект `apscheduler`. Также создали метод `send_message_scheduler` который в нашем случае используется просто для отправки текста сообщения, которые мы же и отправили, но вы можете добавлять какие нибудь проверки, подключаться к БД и тд, все что угодно под вашу задачу. Ловим сообщения мы через `echo_handler` в котором как раз и настраиваем наш `apscheduler`. В данном примере мы используем триггер `date` в котором указываем дату срабатываения нашего расписания с методом `send_message_scheduler` как текущая дата + 5 секунд т.е. через 5 секунд после отправки нашего сообщения боту мы и получим от него ответ.

Примеры использования с другими триггерами:

```python
scheduler.add_job(send_message_scheduler,trigger="interval",hours=1, kwargs={
                "bot": bot,
                "message": message.text,
                "user_id": message.chat.id,
            },)
```


```python
scheduler.add_job(send_message_scheduler, trigger="cron", hour=15,minute=15,start_date=datetime.now(), kwargs={
                "bot": bot,
                "message": message.text,
                "user_id": message.chat.id,
            },)
```


Также возможно использовать `apscheduler` отдельно от хендлеров. Для этого не нужно создавать `middleware` для этого вам просто надо указать в методе  `main()` например:
```python
async def main() -> None:
    # Initialize Bot instance with a default parse mode which will be passed to all API calls
    bot = Bot(TOKEN, parse_mode=ParseMode.HTML)
    # And the run events dispatching
    scheduler = AsyncIOScheduler()
    timezone="Europe/Moscow"

    scheduler.add_job(send_message, trigger="cron", hour=15,minute=15,start_date=datetime.now(), kwargs={
                    "bot": bot,
                    "message": message.text,
                    "user_id": message.chat.id,
                },)
    scheduler.start()
    await dp.start_polling(bot)
```

Где мы получим запуск каждый день в 15 часов 15 минут запуск данного задания, которое в свою очередь будет запускать `send_message`.

P.S. Для полноценной отправки отложенных сообщений понадобиться использовать FSM про который напишу позднее.