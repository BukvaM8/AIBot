# AIBot

GigaChat Bot - это проект, который позволяет взаимодействовать с GigaChat API для получения ответов на запросы пользователей, включая текстовые и графические ответы. Проект реализован на языке Python с использованием библиотеки Streamlit для создания интерфейса.

Функциональные возможности
Аутентификация и получение токена доступа для взаимодействия с GigaChat API.
Прием текстовых запросов от пользователей через интерфейс Streamlit.
Отправка текстовых запросов в GigaChat API и получение ответов.
Обработка и отображение изображений, полученных в ответ на запросы.
Сохранение и отображение истории сообщений в чате.
Установка и настройка
Шаг 1: Клонирование репозитория
Склонируйте репозиторий на свой локальный компьютер:

https://github.com/BukvaM8/AIBot.git

Шаг 2: Установка зависимостей
Создайте виртуальное окружение и установите все необходимые зависимости:

python -m venv venv
source venv/bin/activate  # Для Windows: venv\Scripts\activate
Шаг 3: Настройка секретов
Создайте директорию .streamlit в основнйо директории проекта.
Создайте файл secrets.toml в директории .streamlit и добавьте в него ваши ключи доступа, которые можно получить на сайте https://developers.sber.ru/portal/products/gigachat-api:


CLIENT_ID = "your_client_id"
SECRET = "your_secret"

Шаг 4: Запуск приложения
Запустите приложение Streamlit:
streamlit run main.py
Структура проекта
main.py - основной файл приложения, содержит логику интерфейса и взаимодействия с GigaChat API.
gigachat_api.py - модуль для взаимодействия с GigaChat API: получение токена доступа, отправка запросов и получение ответов.
utils.py - вспомогательный модуль для обработки ответов, включая обработку изображений.
.streamlit/secrets.toml - файл конфигурации, содержащий секреты для аутентификации, его нужно создать в основной директории.

Основные файлы:
main.py
import streamlit as st

from gigachat_api import get_access_token, send_prompt, sent_prompt_and_get_response

st.title("Чат бот")

if "access_token" not in st.session_state:
    try:
        st.session_state.access_token = get_access_token()
        st.toast("Получил токен")
    except Exception as e:
        st.toast(f"Не получилось получить токен: {e}")

if "messages" not in st.session_state:
    st.session_state.messages = [{"role": "ai", "content": "С чем вам помочь?"}]

for msg in st.session_state.messages:
    if msg.get("is_image"):
        st.chat_message(msg["role"]).image(msg["content"])
    else:
        st.chat_message(msg["role"]).write(msg["content"])

if user_prompt := st.chat_input():
    st.chat_message("user").write(user_prompt)
    st.session_state.messages.append({"role": "user", "content": user_prompt})

    with st.spinner("В процессе..."):
        response, is_image = sent_prompt_and_get_response(user_prompt, st.session_state.access_token)
        if is_image:
            st.chat_message("ai").image(response)
            st.session_state.messages.append({"role": "ai", "content": response, "is_image": True})
        else:
            st.chat_message("ai").write(response)
            st.session_state.messages.append({"role": "ai", "content": response})
gigachat_api.py
import json
import uuid

import streamlit as st
import requests
from requests.auth import HTTPBasicAuth

from utils import get_file_id

CLIENT_ID = st.secrets["CLIENT_ID"]
SECRET = st.secrets["SECRET"]

def get_access_token() -> str:
    url = "https://ngw.devices.sberbank.ru:9443/api/v2/oauth"
    headers = {
        'Content-Type': 'application/x-www-form-urlencoded',
        'Accept': 'application/json',
        'RqUID': str(uuid.uuid4()),
    }
    payload = {"scope": "GIGACHAT_API_PERS"}
    res = requests.post(
        url=url,
        headers=headers,
        auth=HTTPBasicAuth(CLIENT_ID, SECRET),
        data=payload,
        verify=False,
    )
    access_token = res.json()["access_token"]
    return access_token

def get_image(file_id: str, access_token: str):
    url = f"https://gigachat.devices.sberbank.ru/api/v1/files/{file_id}/content"

    payload = {}
    headers = {
        'Accept': 'application/jpg',
        'Authorization': f'Bearer {access_token}'
    }

    response = requests.get(url, headers=headers, data=payload, verify=False)
    return response.content

def send_prompt(msg: str, access_token: str):
    url = "https://gigachat.devices.sberbank.ru/api/v1/chat/completions"

    payload = json.dumps({
        "model": "GigaChat-Pro",
        "messages": [
            {
                "role": "user",
                "content": msg,
            }
        ],
    })
    headers = {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
        'Authorization': f'Bearer {access_token}'
    }

    response = requests.post(url, headers=headers, data=payload, verify=False)
    return response.json()["choices"][0]["message"]["content"]

def sent_prompt_and_get_response(msg: str, access_token: str):
    res = send_prompt(msg, access_token)
    data, is_image = get_file_id(res)
    if is_image:
        data = get_image(file_id=data, access_token=access_token)
    return data, is_image

utils.py
import re
from typing import Any

def get_file_id(data: str) -> tuple[Any, bool]:
    match = re.search(r'src="([^"]+)"', data)
    if match:
        return match.group(1), True
    else:
        return data, False


