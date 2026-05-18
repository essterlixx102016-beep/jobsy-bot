import requests
import time

TOKEN = "8908985115:AAEpuegY5EWAoSZTdfki8ibXBRsCy3oZ6co"
URL = f"https://api.telegram.org/bot{TOKEN}"

users = {}
workers = {}
orders = {}
active_chat = {}

last_update = 0
order_id = 1


def send(chat_id, text, keyboard=None):

    data = {
        "chat_id": chat_id,
        "text": text
    }

    if keyboard:
        data["reply_markup"] = {
            "keyboard": keyboard,
            "resize_keyboard": True
        }

    requests.post(URL + "/sendMessage", json=data)


while True:

    try:

        r = requests.get(
            URL + f"/getUpdates?offset={last_update + 1}"
        ).json()

        for update in r.get("result", []):

            last_update = update["update_id"]

            if "message" not in update:
                continue

            msg = update["message"]

            chat_id = msg["chat"]["id"]
            user_id = msg["from"]["id"]

            text = msg.get("text", "")

            # START
            if text == "/start":

                keyboard = [
                    ["1 👷 Рабочий", "2 🧑‍💼 Заказчик"],
                    ["3 ℹ️ Как это работает"]
                ]

                send(
                    chat_id,
                    "Добро пожаловать в Jobsy.\n\n"
                    "Биржа фриланса внутри Telegram.\n\n"
                    "Выберите роль:",
                    keyboard
                )

                continue

            # INFO
            if text == "3 ℹ️ Как это работает":

                send(
                    chat_id,
                    "1. Рабочие создают анкеты\n"
                    "2. Заказчики создают задания\n"
                    "3. Рабочие откликаются\n"
                    "4. Вы общаетесь внутри бота"
                )

                continue

            # WORKER
            if text == "1 👷 Рабочий":

                users[user_id] = "worker_profile"

                send(
                    chat_id,
                    "Отправьте анкету:\n\n"
                    "Имя | Услуга | Цена"
                )

                continue

            # CLIENT
            if text == "2 🧑‍💼 Заказчик":

                users[user_id] = "client"

                keyboard = [
                    ["/order"],
                    ["📋 Заказы"]
                ]

                send(
                    chat_id,
                    "Вы стали заказчиком.\n\n"
                    "Вы можете создать заказ "
                    "или искать рабочих по словам.\n\n"
                    "Например:\n"
                    "дизайн\n"
                    "монтаж\n"
                    "тексты",
                    keyboard
                )

                continue

            # SAVE PROFILE
            if users.get(user_id) == "worker_profile":

                if "|" in text:

                    workers[user_id] = text

                    users[user_id] = "worker"

                    keyboard = [
                        ["📋 Заказы"]
                    ]

                    send(
                        chat_id,
                        "Анкета сохранена.",
                        keyboard
                    )

                else:

                    send(
                        chat_id,
                        "Неверный формат.\n\n"
                        "Пример:\n"
                        "Иван | Дизайн | 500"
                    )

                continue

            # CREATE ORDER
            if text == "/order":

                users[user_id] = "waiting_order"

                send(
                    chat_id,
                    "Напишите описание заказа:"
                )

                continue

            if users.get(user_id) == "waiting_order":

                orders[order_id] = {
                    "client_id": user_id,
                    "text": text,
                    "worker_id": None
                }

                send(
                    chat_id,
                    f"Заказ создан.\nID: {order_id}"
                )

                order_id += 1

                users[user_id] = "client"

                continue

            # SEARCH WORKERS
            if users.get(user_id) == "client":

                found = []

                for wid, profile in workers.items():

                    if text.lower() in profile.lower():

                        found.append(profile)

                if found:

                    result = "Найденные рабочие:\n\n"

                    for f in found:

                        result += f"{f}\n\n"

                    send(chat_id, result)

                continue

            # VIEW ORDERS
            if text == "📋 Заказы":

                if not orders:

                    send(chat_id, "Заказов пока нет.")

                else:

                    result = "Доступные заказы:\n\n"

                    for oid, order in orders.items():

                        if order["worker_id"] is None:

                            result += (
                                f"{oid}. "
                                f"{order['text']}\n\n"
                            )

                    result += (
                        "Напишите номер заказа "
                        "чтобы откликнуться."
                    )

                    send(chat_id, result)

                continue

            # ACCEPT ORDER
            if text.isdigit():

                oid = int(text)

                if oid in orders:

                    order = orders[oid]

                    if order["worker_id"] is None:

                        order["worker_id"] = user_id

                        client_id = order["client_id"]

                        active_chat[user_id] = client_id
                        active_chat[client_id] = user_id

                        send(
                            client_id,
                            "Рабочий откликнулся.\n"
                            "Теперь вы можете общаться."
                        )

                        send(
                            chat_id,
                            "Вы откликнулись на заказ."
                        )

                continue

            # CHAT
            if user_id in active_chat:

                partner = active_chat[user_id]

                send(
                    partner,
                    f"Сообщение:\n{text}"
                )

                continue

    except Exception as e:
        print(e)

    time.sleep(1)
