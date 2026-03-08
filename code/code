from telethon import TelegramClient, events, Button
import logging
from datetime import datetime, timedelta
from telethon.tl.functions.channels import EditBannedRequest
from telethon.tl.types import ChatBannedRights
from telethon.tl.types import PeerUser
from telethon.tl.types import User
from telethon import types
from telethon.tl.types import Channel, ChatAdminRights, User
from telethon.tl.functions.channels import EditAdminRequest
from telethon.tl.types import InputPeerChat
from telethon.errors import ChatAdminRequiredError, UserNotParticipantError
from telethon.tl.types import ChatBannedRights, ChannelParticipantsAdmins
from telethon.tl.types import UserStatusRecently
from telethon.tl.custom import Button
import json
import os
import asyncio
import sqlite3
from threading import Lock
import requests
import random
import re
import hashlib
import uuid
from datetime import datetime, timedelta  # Правильный импорт
from collections import defaultdict
import functools
import time
import sys
import signal

user_scammers_count = {}
user_states = {}
checks_count = 0

# ============ ЗАЩИТА ОТ СПАМА ДЛЯ КНОПОК ============
button_cooldowns = {}
BUTTON_COOLDOWN_TIME = 2  # секунды между нажатиями кнопок
user_button_presses = {}  # Счетчик нажатий кнопок для каждого пользователя
MAX_BUTTON_PRESSES = 3  # Максимальное количество нажатий за период
BUTTON_PRESS_WINDOW = 10  # Окно времени в секундах
button_loading_messages = {}  # Хранит ID сообщений о загрузке для каждого пользователя

# ============ НОВЫЕ МИНИ-ИГРЫ ============
# Глобальное хранилище игр
games_in_progress = {}
mines_games = {}
dice_games = {}
keno_games = {}
wheel_games = {}

# Эмодзи для валюты
INFINIX_CURRENCY = "💎 Инфиниксов"

# Кэш для баланса (чтобы не спамить в БД)
balance_cache = {}

def format_balance(user_id):
    """Форматирует баланс для отображения"""
    balance = db.get_balance(user_id)
    return f"💎 **Б:** `{balance}`"

last_check_time = {}
last_button_click = {}
check_cooldown = 3  # секунды между запросами
button_cooldown = 2  # секунды между нажатиями кнопок

# Глобальный счетчик активных проверок
active_checks = {}

user_message_count = defaultdict(list)

# Настройки логирования
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)
logger = logging.getLogger(__name__)

# Конфигурация
API_ID = '27231812'
API_HASH = '59d6d299a99f9bb97fcbf5645d9d91e9'
BOT_TOKEN = '8502910736:AAFthLXDC6hS1zq2o1VdCgpjJSPJQeWn1Fo'
ADMINS = [262511724]  # ID администраторов
LOG_CHANNEL = 'https://t.me/+cnym32Oi-mJiMGNi'  # Ссылка на канал логов



OWNER_ID = [262511724]

# Добавьте после других глобальных переменных
APPEAL_CHAT_ID = -1003808268065  # Замените на реальный ID чата для апелляций

# Глобальная переменная для управления рассылкой
broadcast_active = False  # По умолчанию выключена
BROADCAST_CHAT_ID = -1002440915213  # ID чата для рассылки (InfinityAntiScam)
BROADCAST_INTERVAL = 300


# ФУНКЦИЯ ЗАЩИТЫ ОТ СПАМА ДЛЯ КНОПОК
def check_button_spam_protection(user_id):
    """Проверяет защиту от спама для кнопок"""
    current_time = time.time()

    # Инициализация данных пользователя
    if user_id not in user_button_presses:
        user_button_presses[user_id] = {
            'count': 0,
            'window_start': current_time,
            'last_press': 0
        }

    user_data = user_button_presses[user_id]

    # Сброс счетчика если окно времени истекло
    if current_time - user_data['window_start'] > BUTTON_PRESS_WINDOW:
        user_data['count'] = 0
        user_data['window_start'] = current_time

    # Проверка кулдауна между нажатиями
    if current_time - user_data['last_press'] < BUTTON_COOLDOWN_TIME:
        remaining = BUTTON_COOLDOWN_TIME - (current_time - user_data['last_press'])
        return False, f"⏳ Подождите {remaining:.1f} секунд перед повторным нажатием"

    # Проверка лимита нажатий в окне времени
    if user_data['count'] >= MAX_BUTTON_PRESSES:
        reset_time = BUTTON_PRESS_WINDOW - (current_time - user_data['window_start'])
        return False, f"🚫 Слишком много нажатий! Подождите {reset_time:.1f} секунд"

    # Обновляем данные пользователя
    user_data['count'] += 1
    user_data['last_press'] = current_time

    return True, "OK"


async def show_button_loading(event, button_name):
    """Показывает сообщение о загрузке для кнопки"""
    user_id = event.sender_id

    # Показываем сообщение о загрузке
    loading_text = f"🔄 Загружаем {button_name}..."
    try:
        if user_id in button_loading_messages:
            try:
                await bot.delete_messages(event.chat_id, button_loading_messages[user_id])
            except:
                pass

        loading_msg = await event.respond(loading_text)
        button_loading_messages[user_id] = loading_msg.id

        # Небольшая задержка для имитации загрузки
        await asyncio.sleep(0.5)

    except Exception as e:
        logging.error(f"Ошибка показа загрузки: {e}")

class Database:
    def __init__(self, db_name='Ice.db'):
        logging.info("Инициализация базы данных...")
        self.users = {}
        self.conn = sqlite3.connect(db_name, isolation_level=None)
        self.conn.execute("PRAGMA foreign_keys = ON")
        self.cursor = self.conn.cursor()
        self.conn.row_factory = sqlite3.Row
        self.lock = asyncio.Lock()
        self.create_tables()
        self.check_table_structure()
        self.check_and_fix_database()

    def is_connected(self):
        """Проверяет, открыто ли соединение с БД"""
        try:
            self.cursor.execute("SELECT 1")
            return True
        except sqlite3.Error:
            return False

    def create_tables(self):
        logging.info("Проверка и создание таблиц если не существуют...")

        self.cursor.execute('''CREATE TABLE IF NOT EXISTS user_ratings (
                       user_id INTEGER PRIMARY KEY,
                       total_rating REAL DEFAULT 5.0,
                       rating_count INTEGER DEFAULT 1,
                       average_rating REAL DEFAULT 5.0
                   )''')
        logging.info("Таблица user_ratings проверена/создана")

        self.cursor.execute('''CREATE TABLE IF NOT EXISTS user_votes (
                       vote_id INTEGER PRIMARY KEY AUTOINCREMENT,
                       voter_id INTEGER NOT NULL,
                       target_id INTEGER NOT NULL,
                       vote_type TEXT NOT NULL,
                       vote_date TEXT DEFAULT CURRENT_TIMESTAMP,
                       UNIQUE(voter_id, target_id)
                   )''')
        logging.info("Таблица user_votes проверена/создана")

        self.cursor.execute('''CREATE TABLE IF NOT EXISTS users (
                user_id INTEGER PRIMARY KEY,
                username TEXT,
                role_id INTEGER DEFAULT 0,
                check_count INTEGER DEFAULT 0,
                last_check_date TEXT,
                country TEXT,
                channel TEXT,
                custom_photo TEXT,
                custom_photo_url TEXT,
                premium_points INTEGER DEFAULT 0,
                description TEXT,
                scammers_count INTEGER DEFAULT 0,
                scammers_slept INTEGER DEFAULT 0,
                warnings INTEGER DEFAULT 0,
                role TEXT,
                custom_status TEXT,
                granted_by_id INTEGER,
                curator_id INTEGER,
                allowance INTEGER DEFAULT 0,
                FOREIGN KEY(curator_id) REFERENCES users(user_id)
            )''')
        logging.info("Таблица users проверена/создана")

        self.cursor.execute('''CREATE TABLE IF NOT EXISTS premium_users (
                user_id INTEGER PRIMARY KEY,
                expiry_date TEXT NOT NULL,
                FOREIGN KEY(user_id) REFERENCES users(user_id)
            )''')
        logging.info("Таблица premium_users проверена/создана")

        self.cursor.execute('''CREATE TABLE IF NOT EXISTS checks (
                check_id INTEGER PRIMARY KEY AUTOINCREMENT,
                checker_id INTEGER,
                target_id INTEGER,
                check_date TEXT,
                description TEXT,
                FOREIGN KEY(checker_id) REFERENCES users(user_id),
                FOREIGN KEY(target_id) REFERENCES users(user_id)
            )''')
        logging.info("Таблица checks проверена/создана")

        # ✅ ИСПРАВЛЕНО: Только создаем если не существует
        self.cursor.execute('''CREATE TABLE IF NOT EXISTS scammers (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER,
                reason TEXT,
                reported_by TEXT,
                description TEXT,
                reporter_id INTEGER,
                unique_id VARCHAR(255),
                proof_link TEXT,
                added_date TEXT DEFAULT CURRENT_TIMESTAMP
            )''')
        logging.info("Таблица scammers проверена/создана")

        # Проверяем и добавляем недостающие поля
        self.check_and_add_missing_columns('scammers')

        self.cursor.execute('''CREATE TABLE IF NOT EXISTS statistics (
                total_messages INTEGER DEFAULT 0
            )''')
        self.cursor.execute('INSERT OR IGNORE INTO statistics (total_messages) VALUES (0)')
        logging.info("Таблица statistics проверена/создана")

        self.cursor.execute('''CREATE TABLE IF NOT EXISTS reasons (
                user_id INTEGER PRIMARY KEY,
                reason TEXT
            )''')
        logging.info("Таблица reasons проверена/создана")

        self.cursor.execute('''CREATE TABLE IF NOT EXISTS trainees (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL,
                email TEXT NOT NULL UNIQUE
            )''')
        logging.info("Таблица trainees проверена/создана")

        self.cursor.execute('''CREATE TABLE IF NOT EXISTS messages (
                message_id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER NOT NULL,
                timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
                content TEXT,
                FOREIGN KEY(user_id) REFERENCES users(user_id)
            )''')
        logging.info("Таблица messages проверена/создана")

        self.cursor.execute('''CREATE TABLE IF NOT EXISTS trust (
                user_id INTEGER PRIMARY KEY,
                granted_by INTEGER,
                grant_date TEXT
            )''')
        logging.info("Таблица trust проверена/создана")

        self.conn.commit()
        logging.info("Все таблицы проверены/созданы")

        # Таблица для спонсоров (обязательная подписка)
        self.cursor.execute('''CREATE TABLE IF NOT EXISTS sponsors (
                        id INTEGER PRIMARY KEY AUTOINCREMENT,
                        chat_id INTEGER UNIQUE NOT NULL,
                        chat_title TEXT NOT NULL,
                        chat_link TEXT,
                        added_by INTEGER NOT NULL,
                        added_date TEXT DEFAULT CURRENT_TIMESTAMP,
                        is_mandatory INTEGER DEFAULT 1
                    )''')
        logging.info("Таблица sponsors создана")

        # Таблица для ежедневных заданий
        self.cursor.execute('''CREATE TABLE IF NOT EXISTS daily_tasks (
                        id INTEGER PRIMARY KEY AUTOINCREMENT,
                        task_name TEXT NOT NULL,
                        task_description TEXT,
                        reward INTEGER NOT NULL DEFAULT 10,
                        is_active INTEGER DEFAULT 1,
                        created_by INTEGER NOT NULL,
                        created_date TEXT DEFAULT CURRENT_TIMESTAMP
                    )''')
        logging.info("Таблица daily_tasks создана")

        # Таблица для выполнения заданий пользователями - ИСПРАВЛЕННАЯ ВЕРСИЯ
        self.cursor.execute('''CREATE TABLE IF NOT EXISTS user_tasks (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER NOT NULL,
            task_id INTEGER NOT NULL,
            completed_date TEXT DEFAULT CURRENT_TIMESTAMP,
            completed_day TEXT DEFAULT (date('now')),
            reward_claimed INTEGER DEFAULT 1,
            FOREIGN KEY(task_id) REFERENCES daily_tasks(id),
            UNIQUE(user_id, task_id, completed_day)
        )''')
        logging.info("Таблица user_tasks создана")

        # Таблица для проверки подписок
        self.cursor.execute('''CREATE TABLE IF NOT EXISTS user_subscriptions (
                        user_id INTEGER PRIMARY KEY,
                        last_check TEXT DEFAULT CURRENT_TIMESTAMP,
                        violations INTEGER DEFAULT 0
                    )''')
        logging.info("Таблица user_subscriptions создана")

        # Таблица для баланса инфиниксов
        self.cursor.execute('''CREATE TABLE IF NOT EXISTS infinix_balance (
                        user_id INTEGER PRIMARY KEY,
                        balance INTEGER DEFAULT 0,
                        total_earned INTEGER DEFAULT 0,
                        total_spent INTEGER DEFAULT 0,
                        last_daily TEXT
                    )''')
        logging.info("Таблица infinix_balance создана")

        self.conn.commit()


    def check_table_structure(self):
        logging.info("Проверка структуры таблицы users...")
        self.cursor.execute("PRAGMA table_info(users);")
        columns = self.cursor.fetchall()
        for column in columns:
            print(column)  # Вывод структуры таблицы

    def cooldown(seconds):
        """Декоратор для защиты от спама"""

        def decorator(func):
            last_called = {}

            @functools.wraps(func)
            async def wrapper(event, *args, **kwargs):
                user_id = event.sender_id
                current_time = time.time()

                if user_id in last_called:
                    elapsed = current_time - last_called[user_id]
                    if elapsed < seconds:
                        remaining = seconds - elapsed
                        await event.answer(f"⏳ Подождите {remaining:.1f} секунд", alert=False)
                        return

                last_called[user_id] = current_time
                return await func(event, *args, **kwargs)

            return wrapper

        return decorator

    def check_and_fix_database(self):
        """Проверяет и добавляет недостающие столбцы в таблицу users"""
        try:
            # Проверяем существующие столбцы
            self.cursor.execute("PRAGMA table_info(users)")
            existing_columns = [column[1] for column in self.cursor.fetchall()]

            # Столбцы, которые должны быть
            required_columns = [
                'scammers_count',
                'premium_points',
                'allowance'
            ]

            # Добавляем недостающие столбцы
            for column in required_columns:
                if column not in existing_columns:
                    if column == 'scammers_count':
                        self.cursor.execute(f"ALTER TABLE users ADD COLUMN {column} INTEGER DEFAULT 0")
                    elif column == 'premium_points':
                        self.cursor.execute(f"ALTER TABLE users ADD COLUMN {column} INTEGER DEFAULT 0")
                    elif column == 'allowance':
                        self.cursor.execute(f"ALTER TABLE users ADD COLUMN {column} INTEGER DEFAULT 0")
                    logging.info(f"Добавлен столбец {column} в таблицу users")

            self.conn.commit()
        except Exception as e:
            logging.error(f"Ошибка при проверке/исправлении базы данных: {e}")

    def check_and_add_missing_columns(self, table_name):
        """Проверяет и добавляет отсутствующие столбцы в таблицу"""
        try:
            # Требуемые поля для таблицы scammers
            required_columns = {
                'id': 'INTEGER PRIMARY KEY AUTOINCREMENT',
                'user_id': 'INTEGER',
                'reason': 'TEXT',
                'reported_by': 'TEXT',
                'description': 'TEXT',
                'reporter_id': 'INTEGER',
                'unique_id': 'VARCHAR(255)',
                'proof_link': 'TEXT',
                'added_date': 'TEXT DEFAULT CURRENT_TIMESTAMP'
            }

            # Получаем текущие столбцы
            self.cursor.execute(f"PRAGMA table_info({table_name})")
            existing_columns = {row[1]: row[2] for row in self.cursor.fetchall()}

            # Добавляем недостающие столбцы
            for column, column_type in required_columns.items():
                if column not in existing_columns:
                    try:
                        self.cursor.execute(f"ALTER TABLE {table_name} ADD COLUMN {column} {column_type}")
                        logging.info(f"Добавлен столбец {column} в таблицу {table_name}")
                    except Exception as e:
                        logging.error(f"Ошибка добавления столбца {column}: {e}")

            self.conn.commit()

        except Exception as e:
            logging.error(f"Ошибка проверки таблицы {table_name}: {e}")

    def get_user_rating(self, user_id):
        """Получает рейтинг пользователя с ограничением от 2.0 до 10.0"""
        try:
            self.cursor.execute('SELECT average_rating FROM user_ratings WHERE user_id = ?', (user_id,))
            result = self.cursor.fetchone()

            if result:
                rating = float(result[0]) if result[0] is not None else 5.0
                # Ограничиваем рейтинг в пределах 2.0-10.0
                rating = max(2.0, min(10.0, rating))
                logging.info(f"Рейтинг пользователя {user_id}: {rating}")
                return round(rating, 1)
            else:
                # Создаем запись с начальным рейтингом 5.0
                self.cursor.execute('''
                    INSERT INTO user_ratings (user_id, total_rating, rating_count, average_rating)
                    VALUES (?, 5.0, 1, 5.0)
                ''', (user_id,))
                self.conn.commit()
                logging.info(f"Создана новая запись рейтинга для пользователя {user_id}")
                return 5.0
        except Exception as e:
            logging.error(f"Ошибка получения рейтинга для {user_id}: {e}")
            return 5.0

    def can_user_vote(self, voter_id, target_id):
        """Проверяет, может ли пользователь голосовать за другого"""
        try:
            # Нельзя голосовать за себя
            if voter_id == target_id:
                return False, "❌ Нельзя голосовать за себя!"

            # Проверяем, голосовал ли уже пользователь
            self.cursor.execute('SELECT 1 FROM user_votes WHERE voter_id = ? AND target_id = ?',
                                (voter_id, target_id))
            if self.cursor.fetchone():
                return False, "❌ Вы уже оценили этого пользователя!"

            return True, "Можно голосовать"
        except Exception as e:
            logging.error(f"Ошибка проверки возможности голосования: {e}")
            return False, "Произошла ошибка"

    def add_user_vote(self, voter_id, target_id, vote_type):
        """Добавляет голос пользователя и обновляет рейтинг"""
        try:
            # Проверяем тип голоса
            if vote_type not in ['like', 'dislike']:
                return False, "Неверный тип голоса"

            # Проверяем, может ли пользователь голосовать
            can_vote, message = self.can_user_vote(voter_id, target_id)
            if not can_vote:
                return False, message

            # Добавляем запись о голосе
            self.cursor.execute('''
                INSERT INTO user_votes (voter_id, target_id, vote_type, vote_date)
                VALUES (?, ?, ?, datetime('now'))
            ''', (voter_id, target_id, vote_type))

            # Обновляем рейтинг
            success, rating_message = self.update_user_rating(target_id, vote_type)
            if not success:
                return False, rating_message

            self.conn.commit()
            logging.info(f"Пользователь {voter_id} проголосовал за {target_id}: {vote_type}")
            return True, "✅ Ваш голос учтен!"

        except sqlite3.IntegrityError:
            return False, "❌ Вы уже оценили этого пользователя!"
        except Exception as e:
            logging.error(f"Ошибка добавления голоса: {e}")
            return False, "❌ Произошла ошибка при обработке голоса"

    def update_user_rating(self, target_id, vote_type):
        """Обновляет рейтинг пользователя на основе голоса"""
        try:
            # Определяем изменение рейтинга
            rating_change = 1.0 if vote_type == 'like' else -1.0

            # Получаем текущий рейтинг
            self.cursor.execute('''
                SELECT total_rating, rating_count, average_rating 
                FROM user_ratings WHERE user_id = ?
            ''', (target_id,))
            result = self.cursor.fetchone()

            if result:
                total_rating, rating_count, average_rating = result
                # Преобразуем в float
                total_rating = float(total_rating) if total_rating is not None else 5.0
                rating_count = int(rating_count) if rating_count is not None else 1
                average_rating = float(average_rating) if average_rating is not None else 5.0

                # Обновляем значения
                new_total_rating = total_rating + rating_change
                new_rating_count = rating_count + 1
                new_average_rating = new_total_rating / new_rating_count

                # Ограничиваем от 2.0 до 10.0
                new_average_rating = max(2.0, min(10.0, new_average_rating))

                self.cursor.execute('''
                    UPDATE user_ratings 
                    SET total_rating = ?, rating_count = ?, average_rating = ?
                    WHERE user_id = ?
                ''', (new_total_rating, new_rating_count, new_average_rating, target_id))

                logging.info(f"Рейтинг пользователя {target_id} обновлен: {new_average_rating:.1f}")
                return True, f"Новый рейтинг: {new_average_rating:.1f}/10"
            else:
                # Создаем новую запись
                initial_rating = 6.0 if vote_type == 'like' else 4.0
                self.cursor.execute('''
                    INSERT INTO user_ratings (user_id, total_rating, rating_count, average_rating)
                    VALUES (?, ?, 1, ?)
                ''', (target_id, initial_rating, initial_rating))

                logging.info(f"Создан новый рейтинг для пользователя {target_id}: {initial_rating}")
                return True, f"Начальный рейтинг: {initial_rating:.1f}/10"

        except Exception as e:
            logging.error(f"Ошибка обновления рейтинга для {target_id}: {e}")
            return False, "Ошибка обновления рейтинга"

    def get_user_vote_history(self, user_id, limit=10):
        """Получает историю голосования пользователя"""
        try:
            self.cursor.execute('''
                SELECT v.vote_type, v.vote_date, u.username,
                       CASE 
                           WHEN v.vote_type = 'like' THEN '👍 Лайк'
                           ELSE '👎 Дизлайк'
                       END as vote_text
                FROM user_votes v
                LEFT JOIN users u ON v.target_id = u.user_id
                WHERE v.voter_id = ?
                ORDER BY v.vote_date DESC
                LIMIT ?
            ''', (user_id, limit))

            votes = self.cursor.fetchall()
            return votes
        except Exception as e:
            logging.error(f"Ошибка получения истории голосования: {e}")
            return []

    def has_user_voted(self, voter_id, target_id):
        """Проверяет, голосовал ли пользователь уже за целевого пользователя"""
        try:
            self.cursor.execute('SELECT 1 FROM user_votes WHERE voter_id = ? AND target_id = ?',
                                (voter_id, target_id))
            return self.cursor.fetchone() is not None
        except Exception as e:
            logging.error(f"Ошибка проверки голоса: {e}")
            return False

    def user_exists(self, user_id):
        self.cursor.execute("SELECT COUNT(*) FROM users WHERE user_id = ?", (user_id,))
        exists = self.cursor.fetchone()[0] > 0
        return exists

    def execute(self, query, params=()):
        """
        Выполняет SQL-запрос с передачей параметров.
        """
        try:
            self.cursor.execute(query, params)  # Выполнение запроса
            self.conn.commit()  # Сохранение изменений
        except sqlite3.Error as e:
            print(f"Ошибка при выполнении запроса: {e}")  # Обработка ошибок

    def increment_scammers_count_all_roles(self, user_id):
        """Увеличивает счетчик скамеров для ЛЮБОЙ роли пользователя"""
        try:
            self.cursor.execute('UPDATE users SET scammers_count = scammers_count + 1 WHERE user_id = ?', (user_id,))
            self.conn.commit()
            logging.info(f"Счетчик скамеров увеличен для пользователя {user_id} (любая роль)")
        except sqlite3.Error as e:
            logging.error(f"Ошибка увеличения счетчика скамеров: {e}")

    def update_total_messages(self, count):
        try:
            logging.info("Обновление количества сообщений...")
            self.cursor.execute('UPDATE statistics SET total_messages = total_messages + ?', (count,))
            self.conn.commit()
            current_count = self.get_total_messages()
            logging.info(f"Текущее количество сообщений в базе данных: {current_count}")
        except sqlite3.Error as e:
            logging.error(f"Ошибка обновления количества сообщений: {e}")

    def get_total_messages(self):
        self.cursor.execute('SELECT total_messages FROM statistics')
        result = self.cursor.fetchone()
        return result[0] if result is not None else 0

    def get_granted_by(self, user_id):
        """Получает ID гаранта для указанного user_id."""
        self.cursor.execute("SELECT granted_by_id FROM users WHERE user_id = ?", (user_id,))
        result = self.cursor.fetchone()
        if result:
            logging.info(f"Гарант найден для user_id {user_id}: {result[0]}")
        else:
            logging.warning(f"Гарант не найден для user_id {user_id}.")
        return result[0] if result else None

    def increment_scammers_count(self, user_id):
        """Увеличивает общий счетчик скамеров для любого пользователя"""
        try:
            # Проверяем, есть ли столбец
            self.cursor.execute("PRAGMA table_info(users)")
            columns = [col[1] for col in self.cursor.fetchall()]

            if 'scammers_count' not in columns:
                logging.error(f"Столбец scammers_count не найден!")
                return False

            # Получаем текущее значение
            current = self.get_user_reported_scammers_count(user_id)
            new_count = current + 1

            self.cursor.execute('UPDATE users SET scammers_count = ? WHERE user_id = ?', (new_count, user_id))
            self.conn.commit()

            logging.info(f"Общий счетчик скамеров для {user_id} увеличен: {current} -> {new_count}")
            return True
        except Exception as e:
            logging.error(f"Ошибка увеличения общего счетчика для {user_id}: {e}")
            return False

    def add_user(self, user_id, username, role_id=0):
        try:
            self.cursor.execute('''
                INSERT INTO users (user_id, username, role_id)
                VALUES (?, ?, ?)
            ''', (user_id, username, role_id))
            self.conn.commit()
            logging.info(f"Пользователь {username} с ID {user_id} добавлен с ролью {role_id}.")
        except Exception as e:
            logging.error(f"Ошибка при добавлении пользователя: {e}")

    def get_user_role(self, user_id):
        """Получает роль пользователя с проверкой соединения"""
        if not self.is_connected():
            self.reconnect()

        try:
            self.cursor.execute('SELECT role_id FROM users WHERE user_id = ?', (user_id,))
            result = self.cursor.fetchone()
            role = result[0] if result else 0
            logging.info(f"Роль пользователя {user_id}: {role}")
            return role
        except Exception as e:
            logging.error(f"Ошибка получения роли для {user_id}: {e}")
            self.reconnect()
            return 0

    def update_user(self, user_id, country=None, channel=None):
        logging.info(f"Обновление пользователя {user_id}: страна - {country}, канал - {channel}")

        # Явная проверка на None для страны
        if country is not None:
            logging.info(f"Обновляем страну на: {country}")
            self.cursor.execute('UPDATE users SET country = ? WHERE user_id = ?', (country, user_id))

        # Явная проверка на None для канала
        if channel is not None:
            logging.info(f"Обновляем канал на: {channel}")
            self.cursor.execute('UPDATE users SET channel = ? WHERE user_id = ?', (channel, user_id))

        # Выполнение коммита для сохранения изменений
        self.conn.commit()

        # Проверка обновленных данных
        self.cursor.execute('SELECT country, channel FROM users WHERE user_id = ?', (user_id,))
        user_data = self.cursor.fetchone()

        # Логирование обновленных данных
        if user_data:
            logging.info(
                f"Данные пользователя после обновления: id={user_id}, страна={user_data[0]}, канал={user_data[1]}")
        else:
            logging.warning(f"Пользователь с id={user_id} не найден после обновления.")

    def get_user_allowance(self, user_id):
        """Получает сумму ручения для указанного пользователя."""
        try:
            self.cursor.execute("SELECT allowance FROM users WHERE user_id = ?", (user_id,))
            result = self.cursor.fetchone()
            if result:
                allowance = result[0]
                logging.info(f"Сумма ручения для пользователя {user_id}: {allowance}")
                return allowance
            else:
                logging.warning(f"Пользователь с ID {user_id} не найден.")
                return None
        except sqlite3.Error as e:
            logging.error(f"Ошибка при получении суммы ручения для пользователя {user_id}: {e}")
            return None

    def get_user_custom_photo(self, user_id):
        logging.info(f"Attempting to retrieve custom photo for user_id: {user_id}")

        try:
            # Изменяем запрос на правильный столбец
            self.cursor.execute('SELECT custom_photo_url FROM users WHERE user_id = ?', (user_id,))
            result = self.cursor.fetchone()

            logging.info(f"SQL query executed for user_id {user_id}. Result: {result}")

            if result:
                custom_photo = result[0]
                logging.info(f"Retrieved custom photo for user {user_id}: {custom_photo}")
            else:
                logging.warning(f"No custom photo found for user_id: {user_id}. Result was None.")
                custom_photo = None

        except Exception as e:
            logging.error(f"Error retrieving custom photo for user_id {user_id}: {str(e)}")
            custom_photo = None

        if custom_photo is None:
            logging.info(f"Custom photo for user_id {user_id} is None or not found.")
        else:
            logging.info(f"Custom photo URL for user_id {user_id}: {custom_photo}")

        return custom_photo

    def get_user_custom_photo_url(self, user_id):
        """Получает URL кастомного фото пользователя"""
        try:
            self.cursor.execute('SELECT custom_photo_url FROM users WHERE user_id = ?', (user_id,))
            result = self.cursor.fetchone()
            return result[0] if result and result[0] else None
        except Exception as e:
            logging.error(f"Error getting custom photo for {user_id}: {e}")
            return None

    def get_user_curator(self, user_id):
        query = "SELECT curator_id FROM users WHERE user_id = ?"
        self.cursor.execute(query, (user_id,))
        result = self.cursor.fetchone()
        return result[0] if result else None

    def get_user_name(self, user_id):
        query = "SELECT username FROM users WHERE user_id = ?"
        self.cursor.execute(query, (user_id,))
        result = self.cursor.fetchone()
        return result[0] if result else "Не указано"

    def get_last_spin(self, user_id):
        """Получает время последнего использования команды рулетки для указанного пользователя."""
        self.cursor.execute('SELECT last_spin FROM users WHERE user_id = ?', (user_id,))
        result = self.cursor.fetchone()
        return result[0] if result else None

    def update_last_spin(self, user_id):
        """Обновляет время последнего использования команды рулетки для указанного пользователя."""
        self.cursor.execute('UPDATE users SET last_spin = ? WHERE user_id = ?', (datetime.now(), user_id))
        self.conn.commit()

    def add_grant(self, user_id, granted_by_id):
        """Добавляет запись о гарантии для пользователя."""
        try:
            self.cursor.execute('''
                INSERT INTO trust (user_id, granted_by, grant_date)
                VALUES (?, ?, ?)
            ''', (user_id, granted_by_id, datetime.now().isoformat()))
            self.conn.commit()
            logging.info(f"Запись о гарантии для user_id {user_id} добавлена. Granted by ID: {granted_by_id}.")
        except sqlite3.Error as e:
            logging.error(f"Ошибка при добавлении записи о гарантии для user_id {user_id}: {e}")

    def set_profile_checks_count(self, user_id, checks_count):
        # Устанавливаем количество проверок для пользователя
        logging.info(f"Устанавливаем количество проверок для пользователя {user_id}: {checks_count}")

        # Проверяем, существует ли пользователь
        if self.get_user(user_id) is None:
            logging.warning(f"Пользователь {user_id} не найден. Не удается установить количество проверок.")
            return

        self.cursor.execute("UPDATE users SET checks_count = ? WHERE user_id = ?", (checks_count, user_id))
        self.conn.commit()
        logging.info(f"Количество проверок для пользователя {user_id} успешно установлено на {checks_count}")

    def get_profile_checks_count(self, user_id):
        # Получаем количество проверок для пользователя
        logging.info(f"Запрос количества проверок для пользователя {user_id}")
        self.cursor.execute("SELECT checks_count FROM users WHERE user_id = ?", (user_id,))
        result = self.cursor.fetchone()

        if result is not None:
            logging.info(f"Количество проверок для пользователя {user_id}: {result[0]}")
        else:
            logging.warning(f"Пользователь {user_id} не найден в базе данных.")

        return result[0] if result else None

    def update_profile_checks_count(self, user_id, checks_count):
        # Обновляем количество проверок профиля
        if checks_count < 0:
            logging.warning(
                f"Попытка установить отрицательное количество проверок для пользователя {user_id}. Устанавливаем 0.")
            checks_count = 0

        logging.info(f"Обновляем количество проверок для пользователя {user_id} на {checks_count}")
        self.cursor.execute("UPDATE users SET checks_count = ? WHERE user_id = ?", (checks_count, user_id))
        self.conn.commit()
        logging.info(f"Количество проверок для пользователя {user_id} успешно обновлено на {checks_count}")

    def add_premium(self, user_id, expiry_date):
        """Добавляет пользователя в премиум с указанной датой окончания."""
        try:
            self.cursor.execute('''
                INSERT INTO premium_users (user_id, expiry_date)
                VALUES (?, ?)
            ''', (user_id, expiry_date))
            self.conn.commit()
            logging.info(f"Пользователь {user_id} добавлен в премиум до {expiry_date}.")
        except sqlite3.Error as e:
            logging.error(f"Ошибка при добавлении пользователя {user_id} в премиум: {e}")

    def is_premium_user(self, user_id):
        self.cursor.execute('SELECT expiry_date FROM premium_users WHERE user_id = ?', (user_id,))
        result = self.cursor.fetchone()
        if result:
            expiry_date = result[0]
            logging.info(f"Пользователь {user_id} имеет премиум статус до {expiry_date}.")
            return expiry_date
        else:
            logging.warning(f"Пользователь {user_id} не найден в таблице premium_users.")
            return None

    def remove_premium(self, user_id):
        # Удаляем премиум статус пользователя из таблицы users
        self.cursor.execute('UPDATE users SET premium = NULL, premium_expiry = NULL WHERE user_id = ?', (user_id,))
        # Удаляем запись из таблицы premium_users
        self.cursor.execute('DELETE FROM premium_users WHERE user_id = ?', (user_id,))
        self.conn.commit()

    def get_premium_expiry(self, user_id):
        """Возвращает дату истечения премиум статуса для пользователя."""
        self.cursor.execute('SELECT expiry_date FROM premium_users WHERE user_id = ?', (user_id,))
        result = self.cursor.fetchone()
        logging.info(f"Результат запроса для пользователя {user_id}: {result}")
        return result[0] if result else None

    def increment_check_count(self, user_id):
        """Увеличивает счетчик проверок для пользователя с указанным user_id, добавляя пользователя в базу, если он не найден."""
        try:
            # Проверяем, существует ли пользователь
            self.cursor.execute('SELECT COUNT(*) FROM users WHERE user_id = ?', (user_id,))
            user_exists = self.cursor.fetchone()[0] > 0

            if not user_exists:
                # Если пользователь не найден, добавляем его в базу данных
                self.cursor.execute('INSERT INTO users (user_id, check_count) VALUES (?, ?)', (user_id, 0))
                logging.info(f"Пользователь с ID {user_id} добавлен в базу данных.")

            # Увеличиваем счетчик
            self.cursor.execute('UPDATE users SET check_count = check_count + 1 WHERE user_id = ?', (user_id,))
            self.conn.commit()
            logging.info(f"Счетчик проверок для пользователя {user_id} увеличен.")
        except sqlite3.Error as e:
            logging.error(f"Ошибка обновления счетчика проверок для {user_id}: {e}")

    def update_warnings(self, user_id):
        try:
            self.cursor.execute('UPDATE users SET warnings = warnings + 1 WHERE user_id = ?', (user_id,))
            self.conn.commit()
            logging.info(f"Количество выговоров для пользователя {user_id} увеличено.")
        except sqlite3.Error as e:
            logging.error(f"Ошибка обновления выговоров для {user_id}: {e}")

    def get_warnings_count(self, user_id):
        result = self.cursor.execute('SELECT warnings FROM users WHERE user_id = ?', (user_id,)).fetchone()
        return result[0] if result is not None else 0

    def reset_warnings(self, user_id):
        """Сбрасывает количество выговоров до 0 для указанного пользователя."""
        self.cursor.execute('UPDATE users SET warnings = 0 WHERE user_id = ?', (user_id,))
        self.conn.commit()
        logging.info(f"Количество выговоров для пользователя {user_id} сброшено до 0.")

    def delete_old_description(self, user_id):
        """Удаляет старое описание."""
        self.cursor.execute("DELETE FROM reasons WHERE user_id = ?", (user_id,))
        self.conn.commit()


    def add_or_update_premium_user(self, user_id, expiry_date):
        try:
            existing_user = self.cursor.execute('SELECT * FROM premium_users WHERE user_id = ?', (user_id,)).fetchone()
            if existing_user:
                self.cursor.execute('UPDATE premium_users SET expiry_date = ? WHERE user_id = ?',
                                    (expiry_date, user_id))
                logging.info(f"Обновлена дата истечения для пользователя {user_id}: {expiry_date}")
            else:
                self.cursor.execute('INSERT INTO premium_users (user_id, expiry_date) VALUES (?, ?)',
                                    (user_id, expiry_date))
                logging.info(f"Добавлен пользователь {user_id} с премиум статусом до {expiry_date}")
            self.conn.commit()
        except sqlite3.Error as e:
            logging.error(f"Ошибка при добавлении/обновлении пользователя {user_id} в премиум: {e}")

    def update_description(self, user_id, new_description):
        try:
            # Обновление описания пользователя в базе данных
            self.cursor.execute("UPDATE users SET description = ? WHERE user_id = ?", (new_description, user_id))
            self.conn.commit()  # Зафиксировать изменения

            # Логирование успешного обновления
            logging.info(f"Описание для пользователя {user_id} обновлено на: {new_description}")

            # Вставка нового описания в статус
            self.update_status(user_id, new_description)
        except Exception as e:
            logging.error(f"Ошибка при обновлении описания: {str(e)}")

    def is_user_in_db(self, user_id):
        """Проверяет, есть ли пользователь в базе данных."""
        self.cursor.execute("SELECT user_id FROM users WHERE user_id = ?", (user_id,))
        return self.cursor.fetchone() is not None

    def get_user_info(self, user_id):
        self.cursor.execute('''
            SELECT user_id, username, role 
            FROM users 
            WHERE user_id = ?
        ''', (user_id,))
        return self.cursor.fetchone()  # Возвращает sqlite3.Row

    def update_status(self, user_id, new_description):
        try:
            # Обновление статуса с новым описанием
            status_message = f"Новое описание: {new_description}"
            self.cursor.execute("UPDATE users SET status = ? WHERE user_id = ?", (status_message, user_id))
            self.conn.commit()  # Зафиксировать изменения

            logging.info(f"Статус для пользователя {user_id} обновлен на: {status_message}")
        except Exception as e:
            logging.error(f"Ошибка при обновлении статуса: {str(e)}")

    def update_user_description(self, user_id, description):
        """Обновляет описание пользователя."""
        try:
            logging.info(f"Попытка обновления описания пользователя {user_id} на: {description}.")

            # Проверяем, существует ли пользователь перед обновлением
            existing_user = self.get_user(user_id)
            if not existing_user:
                logging.warning(f"Пользователь с ID {user_id} не найден. Описание не может быть обновлено.")
                return False

            # Обновляем описание
            self.cursor.execute('UPDATE users SET description = ? WHERE user_id = ?', (description, user_id))
            self.conn.commit()

            # Проверяем, обновилось ли описание
            updated_description = self.get_user_description(user_id)
            if updated_description == description:
                logging.info(f"Описание пользователя {user_id} успешно обновлено на: {description}.")
            else:
                logging.error(
                    f"Описание пользователя {user_id} не обновилось. Текущее значение: {updated_description}.")

            return True
        except sqlite3.Error as e:
            logging.error(f"Ошибка обновления описания для {user_id}: {e}")
            return False

    def get_user_description(self, user_id):
        try:
            self.cursor.execute('SELECT description FROM scammers WHERE user_id = ?', (user_id,))
            result = self.cursor.fetchone()
            if result and result[0]:
                logging.info(f"Описание для пользователя {user_id}: {result[0]}.")
                return result[0]
            else:
                logging.warning(f"Описание для пользователя {user_id} не найдено.")
                return "Описание отсутствует"
        except sqlite3.Error as e:
            logging.error(f"Ошибка при получении описания для пользователя {user_id}: {e}")
            return "Ошибка базы данных"

    def update_role(self, user_id, role_id, granted_by_id=None):
        """Обновляет роль пользователя"""
        try:
            # Обновляем роль
            self.cursor.execute('UPDATE users SET role_id = ? WHERE user_id = ?', (role_id, user_id))

            if granted_by_id is not None:
                self.cursor.execute('UPDATE users SET granted_by_id = ? WHERE user_id = ?', (granted_by_id, user_id))

            # ВСЕГДА делаем commit
            self.conn.commit()
            logging.info(f"Роль пользователя {user_id} обновлена на {role_id}. Granted by ID: {granted_by_id}.")
            return True
        except sqlite3.Error as e:
            logging.error(f"Ошибка обновления роли для {user_id}: {e}")
            return False

    def add_scammer(self, scammer_id, reason, reported_by, description, unique_id, proof_link=None, reporter_id=None):
        try:
            # Всегда проверяем структуру таблицы
            self.cursor.execute("PRAGMA table_info(scammers)")
            columns = [col[1] for col in self.cursor.fetchall()]

            logging.info(f"Добавление скамера: scammer_id={scammer_id}, reporter_id={reporter_id}")

            if 'reporter_id' in columns:
                # НОВАЯ версия с reporter_id
                self.cursor.execute('''
                    INSERT INTO scammers (user_id, reason, reported_by, description, scammer_id, unique_id, proof_link, reporter_id)
                    VALUES (?, ?, ?, ?, ?, ?, ?, ?)
                ''', (scammer_id, reason, reported_by, description, scammer_id, unique_id, proof_link, reporter_id))
                logging.info(f"Добавлено с reporter_id: {reporter_id}")
            else:
                # Старая версия
                self.cursor.execute('''
                    INSERT INTO scammers (user_id, reason, reported_by, description, scammer_id, unique_id, proof_link)
                    VALUES (?, ?, ?, ?, ?, ?, ?)
                ''', (scammer_id, reason, reported_by, description, scammer_id, unique_id, proof_link))
                logging.info("Добавлено без reporter_id (старая структура)")

            self.conn.commit()

            # УВЕЛИЧИВАЕМ СЧЕТЧИК В ТАБЛИЦЕ users
            if reporter_id:
                # Получаем текущее значение
                self.cursor.execute('SELECT scammers_count FROM users WHERE user_id = ?', (reporter_id,))
                result = self.cursor.fetchone()

                if result:
                    current_count = result[0] if result[0] is not None else 0
                    new_count = current_count + 1
                else:
                    new_count = 1
                    # Если пользователя нет в таблице users, добавляем
                    self.cursor.execute('INSERT INTO users (user_id, scammers_count) VALUES (?, ?)',
                                        (reporter_id, new_count))

                # Обновляем счетчик
                self.cursor.execute('UPDATE users SET scammers_count = ? WHERE user_id = ?', (new_count, reporter_id))
                self.conn.commit()

                logging.info(f"✅ Счетчик scammers_count для {reporter_id} увеличен до {new_count}")
            else:
                logging.warning("⚠️ reporter_id не указан, счетчик не увеличен")

            return True
        except Exception as e:
            logging.error(f"❌ Ошибка при добавлении скамера: {e}")
            self.conn.rollback()
            return False

            # Увеличиваем счетчик в таблице users
            if reporter_id:
                self.cursor.execute('''
                    UPDATE users 
                    SET scammers_count = COALESCE(scammers_count, 0) + 1 
                    WHERE user_id = ?
                ''', (reporter_id,))
                self.conn.commit()
                logging.info(f"Счетчик scammers_count для {reporter_id} увеличен")

            logging.info(f"Запись о скамере {scammer_id} добавлена. Занес: {reporter_id}")
            return True
        except Exception as e:
            logging.error(f"Ошибка при добавлении скамера: {e}")
            return False

    def get_scammer_details(self, user_id):
        """Получает все записи о скамере с доказательствами и причинами"""
        try:
            self.cursor.execute('''
                SELECT reason, proof_link, reported_by, added_date 
                FROM scammers 
                WHERE user_id = ? 
                ORDER BY added_date DESC
            ''', (user_id,))
            return self.cursor.fetchall()
        except Exception as e:
            logging.error(f"Ошибка получения данных скамера: {e}")
            return []

    def update_reason(self, user_id, reason):
        """Обновляет причину заноса для указанного пользователя."""
        self.cursor.execute('''
            INSERT INTO reasons (user_id, reason) VALUES (?, ?)
            ON CONFLICT(user_id) DO UPDATE SET reason=excluded.reason
        ''', (user_id, reason))
        self.conn.commit()

    def add_additional_reason(self, user_id, additional_reason):
        """Добавляет дополнительное описание для указанного пользователя."""
        # Предполагаем, что у вас есть отдельная таблица для дополнительных описаний
        self.cursor.execute('''
            INSERT INTO additional_reasons (user_id, additional_reason) VALUES (?, ?)
        ''', (user_id, additional_reason))
        self.conn.commit()

    def get_user_reported_scammers_count(self, user_id):
        """Получает количество СЛИТЫХ скамеров (сколько занес в базу scammers)"""
        try:
            # Вариант 1: Ищем по reporter_id если есть такое поле
            self.cursor.execute("PRAGMA table_info(scammers)")
            columns = [col[1] for col in self.cursor.fetchall()]

            if 'reporter_id' in columns:
                self.cursor.execute('SELECT COUNT(*) FROM scammers WHERE reporter_id = ?', (user_id,))
                result = self.cursor.fetchone()
                count = result[0] if result else 0

                if count > 0:
                    logging.info(f"Найдено {count} слитых скамеров для {user_id} (по reporter_id)")
                    return count

            # Вариант 2: Ищем по имени в reported_by
            # Получаем имя пользователя из базы
            self.cursor.execute('SELECT username, first_name FROM users WHERE user_id = ?', (user_id,))
            user_info = self.cursor.fetchone()

            if user_info:
                username = user_info[0] or user_info[1] or str(user_id)
                # Ищем упоминания имени в reported_by
                self.cursor.execute('SELECT COUNT(*) FROM scammers WHERE reported_by LIKE ?', (f'%{username}%',))
                result = self.cursor.fetchone()
                count = result[0] if result else 0

                logging.info(f"Найдено {count} слитых скамеров для {user_id} (по имени в reported_by)")
                return count

            return 0
        except Exception as e:
            logging.error(f"Ошибка получения количества слитых скамеров для {user_id}: {e}")
            return 0

    def update_user_scammers_count(self, user_id, new_count):
        """Обновляет количество слитых скаммеров для указанного пользователя."""
        try:
            self.cursor.execute('UPDATE users SET scammers_slept = ? WHERE user_id = ?', (new_count, user_id))
            self.conn.commit()
            logging.info(f"Количество слитых скаммеров для пользователя {user_id} обновлено на {new_count}.")
        except sqlite3.Error as e:
            logging.error(f"Ошибка при обновлении количества слитых скаммеров для {user_id}: {e}")

    def get_user(self, user_id):
        self.cursor.execute('SELECT * FROM users WHERE user_id = ?', (user_id,))
        result = self.cursor.fetchone()
        if result:
            logging.info(f"Пользователь найден: {result}")
        else:
            logging.info(f"Пользователь с ID {user_id} не найден.")
        return result

    def is_scammer(self, user_id):
        self.cursor.execute("SELECT * FROM scammers WHERE user_id = ?", (user_id,))
        return self.cursor.fetchone() is not None

    async def update_user_check_count(self, user_id):
        async with self.lock:
            try:
                self.cursor.execute('UPDATE users SET check_count = check_count + 1 WHERE user_id = ?', (user_id,))
                self.conn.commit()
                logging.info(f"Счетчик проверок для пользователя {user_id} обновлен.")
            except sqlite3.Error as e:
                logging.error(f"Ошибка при обновлении счетчика проверок для {user_id}: {e}")

    def get_check_count(self, user_id):
        try:
            self.cursor.execute('SELECT check_count FROM users WHERE user_id = ?', (user_id,))
            result = self.cursor.fetchone()
            count = result[0] if result else 0
            logging.info(f"Количество проверок для пользователя {user_id}: {count}")
            return count
        except Exception as e:
            logging.error(f"Ошибка базы данных в get_check_count: {e}")
            return 0

    def get_display_scammers_count(self, user_id):
        """Получает количество для отображения в чеке"""
        try:
            # 1. Получаем из поля scammers_count таблицы users
            self.cursor.execute('SELECT scammers_count FROM users WHERE user_id = ?', (user_id,))
            result = self.cursor.fetchone()

            if result:
                count_from_users = result[0] if result[0] is not None else 0
            else:
                count_from_users = 0
                # Если пользователя нет, добавляем запись
                self.cursor.execute('INSERT OR IGNORE INTO users (user_id, scammers_count) VALUES (?, 0)', (user_id,))
                self.conn.commit()

            logging.info(f"📊 Из поля scammers_count для {user_id}: {count_from_users}")

            # 2. Проверяем таблицу scammers
            real_count = 0

            # Проверяем есть ли поле reporter_id
            self.cursor.execute("PRAGMA table_info(scammers)")
            columns = [col[1] for col in self.cursor.fetchall()]

            if 'reporter_id' in columns:
                # Ищем по reporter_id
                self.cursor.execute('SELECT COUNT(*) FROM scammers WHERE reporter_id = ?', (user_id,))
                real_result = self.cursor.fetchone()
                real_count = real_result[0] if real_result else 0

                if real_count > 0:
                    logging.info(f"🔍 Найдено {real_count} записей в scammers для {user_id} по reporter_id")

            # 3. Если не нашли по reporter_id, ищем в reported_by
            if real_count == 0:
                try:
                    # Получаем username для поиска
                    self.cursor.execute('SELECT username FROM users WHERE user_id = ?', (user_id,))
                    user_result = self.cursor.fetchone()

                    if user_result and user_result[0]:
                        username = user_result[0]
                        self.cursor.execute('SELECT COUNT(*) FROM scammers WHERE reported_by LIKE ?',
                                            (f'%{username}%',))
                        real_result = self.cursor.fetchone()
                        real_count = real_result[0] if real_result else 0

                        if real_count > 0:
                            logging.info(f"🔍 Найдено {real_count} записей в scammers для {user_id} по username")
                except Exception as e:
                    logging.error(f"Ошибка поиска по username: {e}")

            # 4. Возвращаем большее значение
            final_count = max(count_from_users, real_count)
            logging.info(
                f"🎯 Финальное значение для {user_id}: users={count_from_users}, scammers={real_count}, итого={final_count}")
            return final_count

        except Exception as e:
            logging.error(f"❌ Ошибка в get_display_scammers_count для {user_id}: {e}")
            return 0

    def get_user_scammers_slept(self, user_id):
        """Получает количество слитых скаммеров для указанного пользователя."""
        logging.info(f"Запрос на получение количества слитых скаммеров для пользователя {user_id}.")
        query = 'SELECT scammers_slept FROM users WHERE user_id = ?'
        self.cursor.execute(query, (user_id,))
        result = self.cursor.fetchone()
        if result:
            logging.info(f"Пользователь {user_id} имеет {result[0]} слитых скаммеров.")
            return result[0]
        else:
            logging.warning(f"Пользователь {user_id} не найден, возвращаем 0.")
            return 0

    def update_user_scammers_slept(self, user_id, new_count):
        logging.info(f"Обновление количества слитых скаммеров для пользователя {user_id} на {new_count}.")
        try:
            self.cursor.execute('''
                UPDATE users SET scammers_slept = ? WHERE user_id = ?
            ''', (new_count, user_id))
            self.conn.commit()
            logging.info(f"Количество слитых скаммеров для пользователя {user_id} успешно обновлено на {new_count}.")
            return True
        except sqlite3.Error as e:
            logging.error(f"Ошибка обновления количества слитых скаммеров для пользователя {user_id}: {e}")
            return False

    def remove_scammer_status(self, user_id):
        try:
            # Проверка, есть ли пользователь в базе скаммеров
            if not self.is_scammer(user_id):  # Если пользователя уже нет в базе
                return False  # Возвращаем False, чтобы бот сообщил об ошибке

            # Удаление пользователя из таблицы скаммеров
            self.cursor.execute("DELETE FROM scammers WHERE user_id = ?", (user_id,))
            self.conn.commit()

            # Обновление роли пользователя на "Нет в базе" (0)
            self.cursor.execute("UPDATE users SET role_id = 0 WHERE user_id = ?", (user_id,))
            self.conn.commit()
            logging.info(f"Статус скамера для пользователя {user_id} успешно снят.")

            return True  # Возвращаем True, если всё прошло успешно
        except sqlite3.Error as e:
            logging.error(f"Ошибка при удалении статуса скамера для пользователя {user_id}: {e}")
            return False  # Возвращаем False, если произошла ошибка


    def set_user_allowance(self, user_id, amount):
        try:
            self.cursor.execute("UPDATE users SET allowance = ? WHERE user_id = ?", (amount, user_id))
            self.conn.commit()

            if self.cursor.rowcount == 0:
                logging.warning(f"Пользователь с ID {user_id} не найден.")
            else:
                logging.info(f"Сумма ручения для пользователя с ID {user_id} успешно обновлена на {amount}.")
        except sqlite3.Error as e:
            logging.error(f"Ошибка при обновлении суммы ручения: {e}")

    def add_premium_points(self, user_id, points):
        """Добавляет премиум очки пользователю."""
        try:
            self.cursor.execute('UPDATE users SET premium_points = premium_points + ? WHERE user_id = ?', (points, user_id))
            self.conn.commit()
            logging.info(f"Премиум очки пользователя {user_id} обновлены на {points}.")
        except sqlite3.Error as e:
            logging.error(f"Ошибка при обновлении премиум очков для {user_id}: {e}")

    def get_premium_points(self, user_id):
        """Получает количество премиум очков пользователя."""
        try:
            self.cursor.execute('SELECT premium_points FROM users WHERE user_id = ?', (user_id,))
            result = self.cursor.fetchone()
            return result[0] if result else 0
        except Exception as e:
            logging.error(f"Ошибка при получении премиум очков для {user_id}: {e}")
            return 0

    def add_check(self, checker_id, target_id):
        """Добавляет запись о проверке."""
        try:
            self.cursor.execute('''
                INSERT INTO checks (checker_id, target_id, check_date)
                VALUES (?, ?, ?)
            ''', (checker_id, target_id, datetime.now().isoformat()))
            self.conn.commit()
            logging.info(f"Запись о проверке добавлена: checker_id={checker_id}, target_id={target_id}")
        except sqlite3.Error as e:
            logging.error(f"Ошибка при добавлении записи о проверке: {e}")

    def ensure_start_balance(self, user_id):
        """Выдает стартовый капитал 500 💎 при первом обращении"""
        try:
            # Проверяем, есть ли у пользователя баланс
            self.cursor.execute('SELECT balance FROM infinix_balance WHERE user_id = ?', (user_id,))
            result = self.cursor.fetchone()

            if not result:
                # Первый раз - выдаем 500 💎
                self.cursor.execute('''
                    INSERT INTO infinix_balance (user_id, balance, total_earned, total_spent) 
                    VALUES (?, 500, 500, 0)
                ''', (user_id,))
                self.conn.commit()
                logging.info(f"Пользователь {user_id} получил стартовый капитал 500 💎")
                return 500
            else:
                return result[0]
        except Exception as e:
            logging.error(f"Ошибка при выдаче стартового капитала: {e}")
            return 0

    def can_claim_daily(self, user_id):
        """Проверяет, может ли пользователь получить ежедневный бонус"""
        try:
            self.cursor.execute('SELECT last_daily FROM infinix_balance WHERE user_id = ?', (user_id,))
            result = self.cursor.fetchone()

            if not result or not result[0]:
                return True, 0  # Никогда не получал

            last_daily = datetime.strptime(result[0], '%Y-%m-%d %H:%M:%S')
            now = datetime.now()

            # Прошло ли 24 часа?
            delta = now - last_daily
            if delta.total_seconds() >= 86400:  # 24 часа = 86400 секунд
                return True, int(delta.total_seconds() - 86400)
            else:
                remaining = 86400 - delta.total_seconds()
                hours = int(remaining // 3600)
                minutes = int((remaining % 3600) // 60)
                return False, f"{hours}ч {minutes}м"
        except Exception as e:
            logging.error(f"Ошибка проверки daily бонуса: {e}")
            return False, "Ошибка"

    def transfer_balance(self, from_id, to_id, amount):
        """Переводит инфиниксы от одного пользователя другому"""
        try:
            # Проверяем баланс отправителя
            from_balance = self.get_balance(from_id)
            if from_balance < amount:
                return False, f"Недостаточно средств! Баланс: {from_balance} 💎"

            # Списываем у отправителя
            new_from = from_balance - amount
            self.cursor.execute('''
                UPDATE infinix_balance 
                SET balance = ?, total_spent = total_spent + ? 
                WHERE user_id = ?
            ''', (new_from, amount, from_id))

            # Начисляем получателю
            to_balance = self.get_balance(to_id)
            new_to = to_balance + amount
            self.cursor.execute('''
                UPDATE infinix_balance 
                SET balance = ?, total_earned = total_earned + ? 
                WHERE user_id = ?
            ''', (new_to, amount, to_id))

            if self.cursor.rowcount == 0:
                self.cursor.execute('''
                    INSERT INTO infinix_balance (user_id, balance, total_earned) 
                    VALUES (?, ?, ?)
                ''', (to_id, amount, amount))

            self.conn.commit()
            return True, (new_from, new_to)
        except Exception as e:
            logging.error(f"Ошибка перевода: {e}")
            return False, str(e)

    def give_balance(self, admin_id, to_id, amount):
        """Выдает инфиниксы (только для OWNER_ID)"""
        try:
            # Проверяем, что админ - владелец
            if admin_id not in OWNER_ID:
                return False, "❌ Только владелец может выдавать монеты!"

            # Начисляем получателю
            to_balance = self.get_balance(to_id)
            new_to = to_balance + amount
            self.cursor.execute('''
                UPDATE infinix_balance 
                SET balance = ?, total_earned = total_earned + ? 
                WHERE user_id = ?
            ''', (new_to, amount, to_id))

            if self.cursor.rowcount == 0:
                self.cursor.execute('''
                    INSERT INTO infinix_balance (user_id, balance, total_earned) 
                    VALUES (?, ?, ?)
                ''', (to_id, amount, amount))

            self.conn.commit()
            return True, new_to
        except Exception as e:
            logging.error(f"Ошибка выдачи: {e}")
            return False, str(e)

    def claim_daily(self, user_id):
        """Получить ежедневный бонус (100 💎)"""
        try:
            can_claim, info = self.can_claim_daily(user_id)
            if not can_claim:
                return False, info

            # Добавляем 100 💎
            current = self.get_balance(user_id)
            new_balance = current + 100

            self.cursor.execute('''
                UPDATE infinix_balance 
                SET balance = ?, total_earned = total_earned + 100, last_daily = datetime('now')
                WHERE user_id = ?
            ''', (new_balance, user_id))

            if self.cursor.rowcount == 0:
                self.cursor.execute('''
                    INSERT INTO infinix_balance (user_id, balance, total_earned, last_daily) 
                    VALUES (?, 600, 600, datetime('now'))
                ''', (user_id,))  # 500 стартовых + 100 бонус = 600

            self.conn.commit()

            # Получаем новую дату для следующего бонуса
            self.cursor.execute('SELECT last_daily FROM infinix_balance WHERE user_id = ?', (user_id,))
            next_date = self.cursor.fetchone()[0]
            next_daily = datetime.strptime(next_date, '%Y-%m-%d %H:%M:%S') + timedelta(days=1)

            return True, (100, new_balance, next_daily.strftime('%d.%m.%Y %H:%M'))
        except Exception as e:
            logging.error(f"Ошибка получения daily бонуса: {e}")
            return False, str(e)



    async def __aenter__(self):
        await self.lock.acquire()
        logging.info("База данных открыта для асинхронного доступа.")
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        self.lock.release()
        logging.info("База данных закрыта для асинхронного доступа.")

    def close(self):
        try:
            self.conn.close()
            logging.info("Соединение с базой данных закрыто.")
            return True
        except sqlite3.Error as e:
            logging.error(f"Ошибка закрытия БД: {e}")
            return False



    def get_balance(self, user_id):
        """Получает баланс пользователя"""
        try:
            self.cursor.execute('SELECT balance FROM infinix_balance WHERE user_id = ?', (user_id,))
            result = self.cursor.fetchone()
            if result:
                return result[0]
            else:
                self.cursor.execute('INSERT INTO infinix_balance (user_id, balance) VALUES (?, 0)', (user_id,))
                self.conn.commit()
                return 0
        except Exception as e:
            logging.error(f"Ошибка получения баланса: {e}")
            return 0

    def add_balance(self, user_id, amount):
        """Добавляет инфиниксы на баланс"""
        try:
            current = self.get_balance(user_id)
            new_balance = current + amount
            self.cursor.execute('''
                UPDATE infinix_balance 
                SET balance = ?, total_earned = total_earned + ? 
                WHERE user_id = ?
            ''', (new_balance, amount, user_id))
            if self.cursor.rowcount == 0:
                self.cursor.execute('''
                    INSERT INTO infinix_balance (user_id, balance, total_earned) 
                    VALUES (?, ?, ?)
                ''', (user_id, amount, amount))
            self.conn.commit()
            return True
        except Exception as e:
            logging.error(f"Ошибка добавления баланса: {e}")
            return False

    def remove_balance(self, user_id, amount):
        """Снимает инфиниксы с баланса"""
        try:
            current = self.get_balance(user_id)
            if current < amount:
                return False
            new_balance = current - amount
            self.cursor.execute('''
                UPDATE infinix_balance 
                SET balance = ?, total_spent = total_spent + ? 
                WHERE user_id = ?
            ''', (new_balance, amount, user_id))
            self.conn.commit()
            return True
        except Exception as e:
            logging.error(f"Ошибка снятия баланса: {e}")
            return False


    def get_sponsors(self):
        """Получает список всех спонсоров"""
        try:
            self.cursor.execute(
                'SELECT id, chat_id, chat_title, chat_link, added_by, added_date, is_mandatory FROM sponsors WHERE is_mandatory = 1')
            return self.cursor.fetchall()
        except Exception as e:
            logging.error(f"Ошибка получения спонсоров: {e}")
            return []

    def add_sponsor(self, chat_id, chat_title, chat_link, added_by):
        """Добавляет обязательного спонсора"""
        try:
            self.cursor.execute('''
                INSERT OR REPLACE INTO sponsors (chat_id, chat_title, chat_link, added_by)
                VALUES (?, ?, ?, ?)
            ''', (chat_id, chat_title, chat_link, added_by))
            self.conn.commit()
            return True
        except Exception as e:
            logging.error(f"Ошибка добавления спонсора: {e}")
            return False

    def remove_sponsor_by_id(self, sponsor_id):
        """Удаляет спонсора по ID"""
        try:
            self.cursor.execute('DELETE FROM sponsors WHERE id = ?', (sponsor_id,))
            self.conn.commit()
            return self.cursor.rowcount > 0
        except Exception as e:
            logging.error(f"Ошибка удаления спонсора: {e}")
            return False

    # ============ МЕТОДЫ ДЛЯ РАБОТЫ С ЗАДАНИЯМИ ============
    def add_daily_task(self, task_name, task_description, reward, created_by):
        """Добавляет ежедневное задание"""
        try:
            self.cursor.execute('''
                INSERT INTO daily_tasks (task_name, task_description, reward, created_by)
                VALUES (?, ?, ?, ?)
            ''', (task_name, task_description, reward, created_by))
            self.conn.commit()
            return self.cursor.lastrowid
        except Exception as e:
            logging.error(f"Ошибка добавления задания: {e}")
            return None

    def remove_daily_task(self, task_id):
        """Удаляет задание"""
        try:
            self.cursor.execute('DELETE FROM daily_tasks WHERE id = ?', (task_id,))
            self.conn.commit()
            return self.cursor.rowcount > 0
        except Exception as e:
            logging.error(f"Ошибка удаления задания: {e}")
            return False

    def get_active_tasks(self):
        """Получает активные задания"""
        try:
            self.cursor.execute(
                'SELECT id, task_name, task_description, reward, is_active, created_by, created_date FROM daily_tasks WHERE is_active = 1')
            return self.cursor.fetchall()
        except Exception as e:
            logging.error(f"Ошибка получения заданий: {e}")
            return []

    def complete_task(self, user_id, task_id):
        """Отмечает задание как выполненное"""
        try:
            # Проверяем, не выполнял ли уже сегодня
            self.cursor.execute('''
                SELECT id FROM user_tasks 
                WHERE user_id = ? AND task_id = ? AND date(completed_date) = date('now')
            ''', (user_id, task_id))
            if self.cursor.fetchone():
                return False, "Задание уже выполнено сегодня"

            # Получаем награду
            self.cursor.execute('SELECT reward FROM daily_tasks WHERE id = ?', (task_id,))
            reward = self.cursor.fetchone()
            if not reward:
                return False, "Задание не найдено"

            # Добавляем запись о выполнении
            self.cursor.execute('''
                INSERT INTO user_tasks (user_id, task_id) VALUES (?, ?)
            ''', (user_id, task_id))

            # Начисляем награду
            self.add_balance(user_id, reward[0])
            self.conn.commit()

            return True, reward[0]
        except Exception as e:
            logging.error(f"Ошибка выполнения задания: {e}")
            return False, str(e)

    def get_user_completed_tasks(self, user_id):
        """Получает ID заданий, выполненных пользователем сегодня"""
        try:
            today = datetime.now().strftime('%Y-%m-%d')
            self.cursor.execute('''
                SELECT task_id FROM user_tasks 
                WHERE user_id = ? AND date(completed_date) = ?
            ''', (user_id, today))
            return [row[0] for row in self.cursor.fetchall()]
        except Exception as e:
            logging.error(f"Ошибка получения выполненных заданий: {e}")
            return []

    def complete_task_user(self, user_id, task_id):
        """Отмечает задание как выполненное и начисляет награду"""
        try:
            # Проверяем, не выполнял ли уже сегодня
            today = datetime.now().strftime('%Y-%m-%d')
            self.cursor.execute('''
                SELECT id FROM user_tasks 
                WHERE user_id = ? AND task_id = ? AND date(completed_date) = ?
            ''', (user_id, task_id, today))
            if self.cursor.fetchone():
                return False, "Задание уже выполнено сегодня"

            # Получаем награду
            self.cursor.execute('SELECT reward FROM daily_tasks WHERE id = ?', (task_id,))
            reward_row = self.cursor.fetchone()
            if not reward_row:
                return False, "Задание не найдено"

            reward = reward_row[0]

            # Добавляем запись о выполнении
            self.cursor.execute('''
                INSERT INTO user_tasks (user_id, task_id, completed_date) 
                VALUES (?, ?, datetime('now'))
            ''', (user_id, task_id))

            # Начисляем награду
            self.add_balance(user_id, reward)
            self.conn.commit()

            return True, reward
        except Exception as e:
            logging.error(f"Ошибка выполнения задания: {e}")
            return False, str(e)

    # После класса Database и до обработчиков событий

# ============ ИНИЦИАЛИЗАЦИЯ БОТА ============
bot = TelegramClient('sosot.session', API_ID, API_HASH).start(bot_token=BOT_TOKEN)
db = Database()


# ============ ПРОВЕРКА ПОДПИСКИ НА СПОНСОРОВ ============
async def check_subscription(user_id):
    """Проверяет, подписан ли пользователь на всех спонсоров"""
    sponsors = db.get_sponsors()

    if not sponsors:
        return True, []  # Нет спонсоров - всё ок

    not_subscribed = []

    for sponsor in sponsors:
        chat_id = sponsor[1]  # ID чата
        chat_title = sponsor[2]  # Название

        try:
            # Пытаемся получить информацию о пользователе в чате
            participant = await bot.get_permissions(chat_id, user_id)

            # Проверяем, является ли пользователь участником
            # Если есть права доступа - значит он в чате
            if participant is None:
                not_subscribed.append((chat_title, chat_id))
        except Exception as e:
            # Если ошибка - скорее всего пользователь не в чате
            logging.error(f"Ошибка проверки подписки для {user_id} в чате {chat_id}: {e}")
            not_subscribed.append((chat_title, chat_id))

    if not_subscribed:
        return False, not_subscribed
    else:
        # Обновляем время последней проверки
        db.cursor.execute('''
            INSERT OR REPLACE INTO user_subscriptions (user_id, last_check, violations) 
            VALUES (?, datetime('now'), 0)
        ''', (user_id,))
        db.conn.commit()
        return True, []


def subscription_required(func):
    """Декоратор для проверки подписки перед выполнением команды"""

    @functools.wraps(func)
    async def wrapper(event, *args, **kwargs):
        user_id = event.sender_id

        # Пропускаем владельца бота
        if user_id in OWNER_ID:
            return await func(event, *args, **kwargs)

        # Проверяем подписку
        subscribed, not_subscribed = await check_subscription(user_id)

        if not subscribed:
            # Формируем сообщение о необходимости подписки
            text = "🔒 **ДОСТУП ЗАБЛОКИРОВАН**\n\n"
            text += "Для использования бота необходимо подписаться на следующие каналы/чаты:\n\n"

            buttons = []
            for title, chat_id in not_subscribed:
                text += f"• {title}\n"
                # Пытаемся получить ссылку на чат
                try:
                    chat = await bot.get_entity(chat_id)
                    if hasattr(chat, 'username') and chat.username:
                        buttons.append([Button.url(f"📢 {title}", f"https://t.me/{chat.username}")])
                except:
                    pass

            text += "\nПосле подписки нажми кнопку '✅ Проверить подписку'"

            # Добавляем кнопку проверки
            if buttons:
                buttons.append([Button.inline("✅ Проверить подписку", "check_subscription")])
            else:
                buttons = [[Button.inline("✅ Проверить подписку", "check_subscription")]]

            # Отправляем сообщение и НЕ выполняем основную функцию
            await event.respond(text, buttons=buttons, parse_mode='md')
            return

        # Если подписка есть - выполняем функцию
        return await func(event, *args, **kwargs)

    return wrapper


@bot.on(events.CallbackQuery(pattern="check_subscription"))
async def check_subscription_handler(event):
    """Проверяет подписку пользователя после нажатия кнопки"""
    user_id = event.sender_id

    # Пропускаем владельца
    if user_id in OWNER_ID:
        await event.answer("✅ Вы владелец бота, подписка не требуется!", alert=True)
        await event.delete()
        return

    subscribed, not_subscribed = await check_subscription(user_id)

    if subscribed:
        await event.answer("✅ Подписка подтверждена! Доступ разблокирован.", alert=True)
        # Удаляем сообщение с предупреждением
        await event.delete()
        # Отправляем приветственное сообщение
        await event.respond(
            "🎉 **Доступ разблокирован!**\n\n"
            "Теперь вы можете пользоваться всеми функциями бота.\n"
            "Нажмите /start для начала работы.",
            buttons=get_main_buttons(user_id)
        )
    else:
        # Формируем список неподписанных каналов
        text = "❌ **Вы все еще не подписаны на:**\n\n"
        for title, chat_id in not_subscribed:
            text += f"• {title}\n"
        text += "\nПожалуйста, подпишитесь и нажмите кнопку снова."

        await event.edit(text, buttons=[Button.inline("✅ Проверить подписку", "check_subscription")])

# Роли пользователей
ROLES = {
    0: {"name": "Нет в базе 📝", "preview_url": "https://imgfy.ru/ib/NS5ly0KvlGnJ7TH_1768319364.jpg", "scam_chance": 31},
    1: {"name": "Гарант 🛡️", "preview_url": "https://imgfy.ru/ib/1GWpjFVMTDoAb8Q_1768319364.jpg", "scam_chance": 1},
    2: {"name": "Возможно скамер ⚠️", "preview_url": "https://imgfy.ru/ib/vgyGQVxXgTlD4su_1768319364.jpg",
        "scam_chance": 65},
    3: {"name": "Скамер ❌", "preview_url": "https://imgfy.ru/ib/YT6lXofT8fHsnA4_1768319364.jpg", "scam_chance": 99},
    4: {"name": "Петух 🐓", "preview_url": "https://imgfy.ru/ib/qF7jT8qDILL06Ni_1768319901.jpg", "scam_chance": 45},
    5: {"name": "Подозрение на скам ⚠️", "preview_url": "https://imgfy.ru/ib/fdnOeaUX2htvdkm_1768319365.jpg",
        "scam_chance": 51},
    6: {"name": "Стажёр 🎓", "preview_url": "https://imgfy.ru/ib/3ub4rh7JxOE3kno_1768319365.jpg", "scam_chance": 20},
    7: {"name": "Админ 👮", "preview_url": "https://imgfy.ru/ib/8vPp8tINWVPyYuE_1768319364.jpg", "scam_chance": 15},
    8: {"name": "Директор 👔", "preview_url": "https://imgfy.ru/ib/59y4upESFCONO2x_1768319364.jpg", "scam_chance": 10},
    9: {"name": "Президент 👑", "preview_url": "https://imgfy.ru/ib/6O81I764EZvEFFe_1768319364.jpg", "scam_chance": 5},
    10: {"name": "Создатель ⭐", "preview_url": "https://imgfy.ru/ib/HXkVyyIJl2xJ5l3_1768319364.jpg", "scam_chance": 1},
    11: {"name": "Кодер 💻", "preview_url": "https://i.ibb.co/pjYvHgP2/IMG-20250830-171539-780.jpg", "scam_chance": 3},
    12: {"name": "Проверен гарантом ✅", "preview_url": "https://imgfy.ru/ib/fDocPi2gjwsztYh_1768319365.jpg",
         "scam_chance": 5},
    13: {"name": "сын шлюхи⭐", "preview_url": "", "scam_chance": 50}
}

# Добавьте в начало файла после ROLES:

# Все роли персонала базы (кто защищен от заноса)
STAFF_ROLES = [1, 6, 7, 8, 9, 10, 11, 12]  # Все персоналы + гаранты

# Роли, которым разрешено заносить (все персоналы кроме некоторых)
CAN_ADD_SCAMMER_ROLES = [1, 6, 7, 8, 9, 10, 11]  # Все могут заносить, но...
# 12 (Проверен гарантом) не может заносить

async def check_user(event):
    global checks_count
    user_id = event.sender_id
    user = await event.get_sender()

    # ПРОВЕРКА НА СПАМ - первое что делаем
    current_time = time.time()

    # Проверяем, не выполняется ли уже проверка для этого пользователя
    if user_id in active_checks and active_checks[user_id]:
        await event.respond("⏳ Ваш предыдущий запрос еще обрабатывается...")
        return

    # Проверяем время последнего вызова команды
    if user_id in last_check_time:
        elapsed_time = current_time - last_check_time[user_id]
        if elapsed_time < check_cooldown:
            remaining_time = check_cooldown - elapsed_time
            await event.respond(f"⏳ Подождите {remaining_time:.1f} секунд перед повторной проверкой!")
            return

    # Устанавливаем блокировку
    active_checks[user_id] = True
    last_check_time[user_id] = current_time

    # Только теперь показываем загрузку
    try:
        loading_msg = await event.respond("🔍 Ищу данные о пользователе в базе...")

        await asyncio.sleep(0.5)  # Минимальная задержка

        user_to_check = None
        user_data = None

        # Определяем пользователя для проверки
        if event.reply_to_msg_id:
            replied = await event.get_reply_message()
            user_to_check = await event.client.get_entity(replied.sender_id)
            user_data = db.get_user(user_to_check.id) if db else None
        else:
            if "чек себя" in event.raw_text.lower() or "чек ми" in event.raw_text.lower():
                user_to_check = user
                user_data = db.get_user(user.id) if db else None
            else:
                try:
                    args = event.raw_text.split()[1:]
                    if args and args[0].isdigit():
                        user_id_to_check = int(args[0])
                        user_data = db.get_user(user_id_to_check) if db else None

                        if user_data:
                            # Если пользователь есть в базе, создаем минимальный объект
                            user_to_check = type('UserObject', (), {
                                'id': user_id_to_check,
                                'first_name': f"ID: {user_id_to_check}",
                                'username': user_data[1] if len(user_data) > 1 and user_data[1] else None
                            })()
                        else:
                            # ДОБАВЛЯЕМ ПОЛЬЗОВАТЕЛЯ С РОЛЬЮ 0, ЕСЛИ ЕГО НЕТ В БАЗЕ
                            db.add_user(user_id_to_check, str(user_id_to_check), 0)
                            user_data = db.get_user(user_id_to_check)
                            # Создаем объект пользователя
                            user_to_check = type('UserObject', (), {
                                'id': user_id_to_check,
                                'first_name': f"ID: {user_id_to_check}",
                                'username': None
                            })()

                    elif args:
                        # Для юзернейма
                        user_to_check = await event.client.get_entity(args[0])
                        user_data = db.get_user(user_to_check.id) if db else None
                except Exception as e:
                    logging.error(f"Ошибка при получении пользователя: {e}")
                    await loading_msg.delete()
                    active_checks[user_id] = False
                    return await send_response(event, "❌ | Не удалось найти пользователя.")

        if user_to_check is None:
            await loading_msg.delete()
            active_checks[user_id] = False
            return await send_response(event, "❌ | Не удалось определить пользователя.")

        # Получаем данные, если они еще не загружены
        if not user_data and db:
            # ДОБАВЛЯЕМ ПОЛЬЗОВАТЕЛЯ С РОЛЬЮ 0, ЕСЛИ ЕГО НЕТ В БАЗЕ
            username = getattr(user_to_check, 'username', None) or getattr(user_to_check, 'first_name',
                                                                           f"ID: {user_to_check.id}")
            db.add_user(user_to_check.id, username, 0)
            # Получаем обновленные данные
            user_data = db.get_user(user_to_check.id)

        # Увеличиваем счетчики проверок
        if db:
            db.increment_check_count(user_to_check.id)
        checks_count += 1

        # Формируем ответ
        response = await get_user_profile_response(event, user_to_check, user_data)

        if isinstance(response, tuple):
            message_text, buttons = response
        else:
            message_text = response
            buttons = []

        # Отправка результата
        try:
            await send_response(event, message_text[:4096] if len(message_text) > 4096 else message_text, buttons)
        except Exception as e:
            logging.error(f"Ошибка при отправке сообщения: {e}")

        # Уведомление для премиум-пользователей
        if db and db.is_premium_user(user_id) and event.raw_text.lower() in ('чек', '/check'):
            await bot.send_message(
                user_id,
                f'🔍 Пользователь [{user.first_name}](tg://user?id={user_id}) проверял вас в боте!',
                buttons=Button.inline("↩Скрыть", b"hide_message")
            )

        # Удаляем сообщение о загрузке
        await loading_msg.delete()

    except Exception as e:
        logging.error(f"Ошибка в check_user: {e}")
        try:
            await loading_msg.delete()
        except:
            pass
    finally:
        # Снимаем блокировку в любом случае
        active_checks[user_id] = False




def get_guarantors():
    """Получает список гарантов из базы данных"""
    try:
        db.cursor.execute('SELECT user_id FROM users WHERE role_id = 1')  # role_id = 1 - гарант
        guarantors = db.cursor.fetchall()
        return [guarantor[0] for guarantor in guarantors]
    except Exception as e:
        logging.error(f"Ошибка при получении гарантов: {e}")
        return []

def get_trainees():
    """Получает список стажеров из базы данных"""
    try:
        db.cursor.execute("SELECT * FROM trainees")
        trainees = db.cursor.fetchall()
        return trainees
    except Exception as e:
        logging.error(f"Ошибка при получении стажеров: {e}")
        return []


@bot.on(events.CallbackQuery(pattern="admin_tasks"))
async def admin_tasks(event):
    user_id = event.sender_id

    # Проверка прав - ТОЛЬКО OWNER_ID
    if user_id not in OWNER_ID:
        await event.answer("❌ У вас нет доступа!", alert=True)
        return

    # Получаем активные задания
    tasks = db.get_active_tasks()

    text = """📋 **УПРАВЛЕНИЕ ЗАДАНИЯМИ**
━━━━━━━━━━━━━━━━━━━━

📝 **Как добавить задание:**
Отправь сообщение в формате:
`Название | Описание | Награда`

Пример:
`Подпишись | Подпишись на наш канал | 25`

📋 **Текущие задания:**
"""

    if tasks:
        for task in tasks:
            task_id, task_name, task_description, reward, is_active, created_by, created_date = task
            # Форматируем дату
            try:
                date_obj = datetime.strptime(created_date, '%Y-%m-%d %H:%M:%S')
                date_str = date_obj.strftime('%d.%m.%Y')
            except:
                date_str = created_date[:10] if created_date else "??"

            text += f"• [{task_id}] **{task_name}** - {task_description}\n"
            text += f"  💰 Награда: {reward} 💎 | Создано: {date_str}\n"
    else:
        text += "• Список пуст\n"

    text += "\n━━━━━━━━━━━━━━━━━━━━"

    buttons = [
        [Button.inline("➕ Добавить задание", "add_task")],
        [Button.inline("❌ Удалить задание", "remove_task")],
        [Button.inline("◀ Назад в админ-панель", "admin_panel_back")]
    ]

    # Отправляем новое сообщение
    await event.respond(text, buttons=buttons, parse_mode='md')
    await event.answer()


@bot.on(events.CallbackQuery(pattern="add_task"))
async def add_task_prompt(event):
    user_id = event.sender_id

    # Проверка прав - ТОЛЬКО OWNER_ID
    if user_id not in OWNER_ID:
        await event.answer("❌ У вас нет доступа!", alert=True)
        return

    # Сохраняем состояние
    user_states[user_id] = {"state": "waiting_task_data"}

    text = (
        "📝 **ДОБАВЛЕНИЕ ЗАДАНИЯ**\n\n"
        "Отправь данные в формате:\n"
        "`Название | Описание | Награда`\n\n"
        "**Пример:**\n"
        "`Подпишись | Подпишись на наш канал | 25`\n\n"
        "**Правила:**\n"
        "• Название должно быть кратким (до 30 символов)\n"
        "• Описание - что нужно сделать (до 100 символов)\n"
        "• Награда - число от 10 до 1000\n\n"
        "Или отправь 'отмена' для выхода."
    )

    buttons = [Button.inline("❌ Отмена", "admin_tasks")]

    # Отправляем новое сообщение
    await event.respond(text, buttons=buttons, parse_mode='md')
    await event.answer()

@bot.on(events.CallbackQuery(pattern="add_sponsor"))
async def add_sponsor_prompt(event):
    user_id = event.sender_id

    # Проверка прав - ТОЛЬКО OWNER_ID
    if user_id not in OWNER_ID:
        await event.answer("❌ У вас нет доступа!", alert=True)
        return

    # Сохраняем состояние
    user_states[user_id] = {"state": "waiting_sponsor_data"}

    text = (
        "📝 **ДОБАВЛЕНИЕ СПОНСОРА**\n\n"
        "Отправь данные в формате:\n"
        "`ID_канала | Название | ссылка`\n\n"
        "Пример:\n"
        "`-100123456789 | Мой Канал | https://t.me/mychannel`\n\n"
        "Или отправь 'отмена' для выхода."
    )

    buttons = [Button.inline("❌ Отмена", "admin_sponsors")]

    # Отправляем новое сообщение, не пытаясь редактировать
    await event.respond(text, buttons=buttons)

    # Подтверждаем callback
    await event.answer()

@bot.on(events.CallbackQuery(pattern="remove_sponsor"))
async def remove_sponsor_prompt(event):
        user_id = event.sender_id

        # Проверка прав - ТОЛЬКО OWNER_ID
        if user_id not in OWNER_ID:
            await event.answer("❌ У вас нет доступа!", alert=True)
            return

        sponsors = db.get_sponsors()

        if not sponsors:
            await event.answer("❌ Нет спонсоров для удаления", alert=True)
            return

        text = "❌ **УДАЛЕНИЕ СПОНСОРА**\n\nВыбери спонсора для удаления:\n\n"
        buttons = []

        for sponsor in sponsors:
            sponsor_id = sponsor[0]
            chat_title = sponsor[2]
            buttons.append([Button.inline(f"❌ {chat_title}", f"delete_sponsor_{sponsor_id}")])

        buttons.append([Button.inline("◀ Назад", "admin_sponsors")])

        try:
            await event.edit(text, buttons=buttons)
        except:
            await event.respond(text, buttons=buttons)

@bot.on(events.NewMessage(pattern="👥 Состав базы"))
@subscription_required
async def members_menu(event):
    if not event.is_private:
        return

    user_id = event.sender_id

    # Проверка защиты от спама
    can_press, message = check_button_spam_protection(user_id)
    if not can_press:
        await event.respond(message)
        return

    # Показываем загрузку
    await show_button_loading(event, "состав базы")

    buttons = [
        [Button.text("✅ Гаранты базы", resize=True)],
        [Button.text("👨‍🎓 Волонтёры базы", resize=True)],
        [Button.text("↩ Назад", resize=True)]
    ]

    # Удаляем сообщение о загрузке если есть
    if user_id in button_loading_messages:
        try:
            await bot.delete_messages(event.chat_id, button_loading_messages[user_id])
            del button_loading_messages[user_id]
        except:
            pass

    await event.respond(
        "👥 **Меню состава базы**\n\n"
        "Выберите категорию участников для просмотра:",
        buttons=buttons,
        parse_mode='md'
    )


@bot.on(events.NewMessage(pattern="↩ Назад"))
async def back_to_main(event):
    if not event.is_private:
        return

    user_id = event.sender_id

    # Проверка защиты от спама (но менее строгая для кнопки Назад)
    can_press, message = check_button_spam_protection(user_id)
    if not can_press:
        await event.respond(message)
        return

    # Показываем загрузку
    await show_button_loading(event, "главное меню")

    # Удаляем сообщение о загрузке если есть
    if user_id in button_loading_messages:
        try:
            await bot.delete_messages(event.chat_id, button_loading_messages[user_id])
            del button_loading_messages[user_id]
        except:
            pass

    await event.respond(
        "Главное меню:",
        buttons=get_main_buttons(user_id)
    )


@bot.on(events.NewMessage(pattern=r'^(баланс|balance|б)$'))
@subscription_required
async def check_balance_command(event):
    """Проверка баланса"""
    user_id = event.sender_id

    # Убеждаемся, что есть стартовый капитал
    db.ensure_start_balance(user_id)

    balance = db.get_balance(user_id)

    # Проверяем ежедневный бонус
    can_claim, claim_info = db.can_claim_daily(user_id)
    daily_status = "✅ Доступен!" if can_claim else f"⏳ Через {claim_info}"

    text = f"""💰 **ТВОЙ БАЛАНС**
💎 **Баланс:** `{balance}` Инфиниксов
🎁 **Следующий бонус:** {daily_status}"""

    buttons = []
    if can_claim:
        buttons.append([Button.inline("🎁 Забрать бонус 100 💎", "claim_daily")])
    buttons.append([Button.inline("🎮 В меню игр", "games_menu")])

    await event.respond(text, buttons=buttons, parse_mode='md')

@bot.on(events.NewMessage(pattern="📊 Статистика базы"))
async def statistics(event):
    if not event.is_private:
        return

    user_id = event.sender_id

    # Проверка защиты от спама
    can_press, message = check_button_spam_protection(user_id)
    if not can_press:
        await event.respond(message)
        return

    # Показываем загрузку
    await show_button_loading(event, "статистику")

    try:
        # Получаем общую статистику
        total_scammers = db.cursor.execute('SELECT COUNT(*) FROM scammers').fetchone()[0] or 0
        total_users = db.cursor.execute('SELECT COUNT(*) FROM users').fetchone()[0]
        total_checks = db.cursor.execute('SELECT SUM(check_count) FROM users').fetchone()[0] or 0

        # Получаем топ персонала по количеству слитых скамеров
        top_staff = db.cursor.execute('''
            SELECT u.user_id, u.username, u.role_id, u.scammers_count 
            FROM users u 
            WHERE u.role_id IN (6, 7, 8, 9, 10, 11, 13) 
            AND u.scammers_count > 0
            ORDER BY u.scammers_count DESC 
            LIMIT 10
        ''').fetchall()

        # Формируем текст статистики
        text = f"""📊 **СТАТИСТИКА БАЗЫ INFINITY**
━━━━━━━━━━━━━━━━━━━━
[⠀](https://i.ibb.co/Fzpqd0K/IMG-3735.jpg)

🚫 **Скаммеров в базе:** `{total_scammers}`
👥 **Пользователей бота:** `{total_users}`
🔍 **Всего проверок:** `{total_checks}`

🏆 **ТОП ПЕРСОНАЛА ПО СЛИТЫМ СКАМЕРАМ:**
"""

        # Создаем кнопки для топ персонала
        buttons = []

        for i, (user_id_staff, username, role_id, scammers_count) in enumerate(top_staff, 1):
            # Получаем имя роли
            role_name = ROLES.get(role_id, {}).get('name', 'Неизвестно')
            role_emoji = get_role_emoji(role_id)

            # Формируем отображаемое имя
            if username:
                display_name = f"{username}"
            else:
                display_name = f"ID:{user_id_staff}"

            # Сокращаем слишком длинные имена
            if len(display_name) > 15:
                display_name = display_name[:12] + "..."

            # Добавляем кнопку
            button_text = f"{role_emoji} {display_name} - {scammers_count}🔪"
            buttons.append([Button.inline(button_text, f"staff_stats_{user_id_staff}")])

        # Если нет топ персонала
        if not top_staff:
            text += "\n📭 Пока нет активного персонала"

        # Добавляем разделитель
        text += "\n━━━━━━━━━━━━━━━━━━━━\n"

        # Кнопки навигации
        nav_buttons = [
            [Button.inline("🏆 Топ Стажеров", b"top_trainees")],
            [Button.inline("😎 Топ Активных", b"top_day")],
            [Button.inline("🔄 Обновить", b"refresh_stats")]
        ]

        buttons.extend(nav_buttons)

        # Добавляем кнопку чата
        buttons.append([Button.url("🎇 Наша База", 'https://t.me/Infinityantiscam')])

        # Удаляем сообщение о загрузке если есть
        if user_id in button_loading_messages:
            try:
                await bot.delete_messages(event.chat_id, button_loading_messages[user_id])
                del button_loading_messages[user_id]
            except:
                pass

        stat_message = await event.respond(text, parse_mode='md', link_preview=True, buttons=buttons)
        bot.stat_message_id = stat_message.id

    except Exception as e:
        logging.error(f"Ошибка в статистике: {e}")
        await event.respond(f"⚠️ Ошибка загрузки статистики: {str(e)}")


# Вспомогательная функция для эмодзи ролей
def get_role_emoji(role_id):
    emoji_map = {
        6: "👨‍🎓",  # Стажер
        7: "👮",  # Админ
        8: "🎩",  # Директор
        9: "👑",  # Президент
        10: "⭐",  # Владелец
        11: "💻",  # Кодер
        13: "🌟"  # Айдош
    }
    return emoji_map.get(role_id, "👤")


@bot.on(events.NewMessage(pattern=r'(?i)^/fullstats|/полнаястата'))
async def full_statistics(event):
    """Полная статистика по всем ролям"""
    if not event.is_private:
        return

    user_id = event.sender_id

    try:
        # Получаем статистику по каждой роли
        stats_by_role = {}

        for role_id in [6, 7, 8, 9, 10, 11, 13]:  # Все роли персонала
            role_name = ROLES.get(role_id, {}).get('name', 'Неизвестно')

            # Количество пользователей с этой ролью
            db.cursor.execute('SELECT COUNT(*) FROM users WHERE role_id = ?', (role_id,))
            count_users = db.cursor.fetchone()[0] or 0

            # Общее количество слитых скамеров для этой роли
            db.cursor.execute('SELECT SUM(scammers_count) FROM users WHERE role_id = ?', (role_id,))
            total_scammers = db.cursor.fetchone()[0] or 0

            # Топ 3 пользователя этой роли
            db.cursor.execute('''
                SELECT user_id, username, scammers_count 
                FROM users 
                WHERE role_id = ? AND scammers_count > 0
                ORDER BY scammers_count DESC 
                LIMIT 3
            ''', (role_id,))
            top_users = db.cursor.fetchall()

            stats_by_role[role_id] = {
                'name': role_name,
                'count_users': count_users,
                'total_scammers': total_scammers,
                'top_users': top_users
            }

        # Формируем сообщение
        text = "🏆 **ПОЛНАЯ СТАТИСТИКА ПЕРСОНАЛА**\n━━━━━━━━━━━━━━━━━━━━\n[⠀](https://i.ibb.co/Fzpqd0K/IMG-3735.jpg)\n"

        # Создаем кнопки по ролям
        role_buttons = []

        for role_id in [6, 7, 8, 9, 10, 11, 13]:
            stats = stats_by_role[role_id]
            emoji = get_role_emoji(role_id)
            button_text = f"{emoji} {stats['name']} - {stats['total_scammers']}🔪"
            role_buttons.append([Button.inline(button_text, f"role_stats_{role_id}")])

        # Кнопки навигации
        nav_buttons = [
            [Button.inline("📊 Общая статистика", b"general_stats")],
            [Button.inline("👥 Все пользователи", b"all_users_stats")],
            [Button.inline("🔄 Обновить", b"refresh_fullstats")]
        ]

        all_buttons = role_buttons + nav_buttons

        await event.respond(text, parse_mode='md', link_preview=True, buttons=all_buttons)

    except Exception as e:
        logging.error(f"Ошибка в полной статистике: {e}")
        await event.respond(f"⚠️ Ошибка: {str(e)}")

def get_role_emoji(role_id):
    """Возвращает эмодзи для роли"""
    emoji_map = {
        6: "👨‍🎓",  # Стажер
        7: "👮",   # Админ
        8: "🎩",   # Директор
        9: "👑",   # Президент
        10: "⭐",  # Владелец
        11: "💻",  # Кодер
        13: "🌟"   # Айдош
    }
    return emoji_map.get(role_id, "👤")

@bot.on(events.NewMessage(pattern='/check_my_photo'))
async def check_my_photo(event):
    user_id = event.sender_id

    # Проверяем структуру таблицы
    db.cursor.execute("PRAGMA table_info(users)")
    columns = db.cursor.fetchall()
    print("Структура таблицы users:")
    for i, col in enumerate(columns):
        print(f"{i}: {col}")

    # Проверяем конкретно нашу запись
    db.cursor.execute("SELECT * FROM users WHERE user_id = ?", (user_id,))
    user_data = db.cursor.fetchone()
    print(f"Все данные пользователя: {user_data}")

    if user_data:
        print(f"custom_photo_url: {user_data[8] if len(user_data) > 8 else 'NO COLUMN'}")

    await event.respond("Проверка завершена, смотрите консоль")





@bot.on(events.CallbackQuery(pattern=r'rate_user_(\d+)'))
async def rate_user_handler(event):
    """Обработчик оценки пользователя"""
    try:
        target_user_id = int(event.pattern_match.group(1))
        voter_id = event.sender_id

        # Проверяем, может ли пользователь голосовать
        can_vote, message = db.can_user_vote(voter_id, target_user_id)

        if not can_vote:
            await event.answer(message, alert=True)
            return

        # Получаем информацию о целевом пользователе
        try:
            target_user = await bot.get_entity(target_user_id)
            target_name = target_user.first_name
        except:
            target_name = f"Пользователь {target_user_id}"

        # Получаем текущий рейтинг
        current_rating = db.get_user_rating(target_user_id)
        rating_stars = "⭐" * int(current_rating / 2)

        # Создаем inline-клавиатуру для оценки
        buttons = [
            [
                Button.inline("👍 Лайк (+1)", f"vote_like_{target_user_id}"),
                Button.inline("👎 Дизлайк (-1)", f"vote_dislike_{target_user_id}")
            ],
            [
                Button.inline("📊 История голосов", f"vote_history_{target_user_id}"),
                Button.inline("❌ Отмена", "cancel_rate")
            ]
        ]

        # ВМЕСТО event.edit() используем event.respond() для нового сообщения
        await event.answer()  # Сначала отвечаем на callback
        await event.respond(
            f"📊 **Оцените пользователя**\n\n"
            f"👤 **Пользователь:** {target_name}\n"
            f"⭐ **Текущий рейтинг:** {current_rating:.1f}/10\n"
            f"{rating_stars}\n\n"
            f"Выберите оценку:",
            buttons=buttons
        )

    except Exception as e:
        logging.error(f"Ошибка в rate_user_handler: {e}")
        await event.answer("❌ Произошла ошибка", alert=True)


@bot.on(events.CallbackQuery(pattern=r'vote_(like|dislike)_(\d+)'))
async def vote_handler(event):
    """Обработчик голосования"""
    try:
        vote_type = event.pattern_match.group(1).decode('utf-8')  # Декодируем из bytes в str
        target_user_id = int(event.pattern_match.group(2))
        voter_id = event.sender_id

        # Добавляем голос
        success, message = db.add_user_vote(voter_id, target_user_id, vote_type)

        if success:
            # Получаем новый рейтинг
            new_rating = db.get_user_rating(target_user_id)
            rating_stars = "⭐" * int(new_rating / 2)

            vote_text = "👍 Лайк" if vote_type == 'like' else "👎 Дизлайк"

            # Редактируем сообщение вместо удаления
            await event.edit(
                f"✅ **Оценка принята!**\n\n"
                f"📊 **Новый рейтинг:** {new_rating:.1f}/10\n"
                f"{rating_stars}\n\n"
                f"📝 Ваша оценка: {vote_text}",
                buttons=None
            )

            # Уведомляем пользователя (если он не заблокировал бота)
            try:
                voter_user = await bot.get_entity(voter_id)
                voter_name = voter_user.first_name

                notification_text = (
                    f"📢 **Вас оценили!**\n\n"
                    f"Пользователь {voter_name} поставил вам {vote_text.lower()}.\n"
                    f"Ваш новый рейтинг: {new_rating:.1f}/10\n"
                    f"⭐ {'⭐' * int(new_rating / 2)}"
                )

                await bot.send_message(target_user_id, notification_text)
            except:
                pass  # Пользователь заблокировал бота

        else:
            await event.answer(message, alert=True)

    except Exception as e:
        logging.error(f"Ошибка в vote_handler: {e}")
        await event.answer("❌ Произошла ошибка при обработке голоса", alert=True)


@bot.on(events.CallbackQuery(pattern=r'vote_history_(\d+)'))
async def vote_history_handler(event):
    """Показывает историю голосования"""
    try:
        target_user_id = int(event.pattern_match.group(1))
        voter_id = event.sender_id

        # Получаем историю голосования
        votes = db.get_user_vote_history(voter_id, limit=5)

        if not votes:
            await event.answer("📭 У вас пока нет истории голосования", alert=True)
            return

        history_text = "📋 **Последние оценки:**\n\n"
        for vote in votes:
            vote_type, vote_date, username, vote_text = vote
            user_display = username or f"ID:{target_user_id}"

            # Форматируем дату
            try:
                vote_date_obj = datetime.strptime(vote_date, '%Y-%m-%d %H:%M:%S')
                formatted_date = vote_date_obj.strftime('%d.%m.%Y')
            except:
                formatted_date = vote_date

            history_text += f"• {vote_text} для {user_display} ({formatted_date})\n"

        # Добавляем статистику
        total_votes = len(votes)
        likes = sum(1 for v in votes if v[0] == 'like')
        dislikes = total_votes - likes

        history_text += f"\n📊 **Статистика:**\n"
        history_text += f"Всего оценок: {total_votes}\n"
        history_text += f"Лайков: {likes} | Дизлайков: {dislikes}"

        await event.edit(
            history_text,
            buttons=[
                [Button.inline("« Назад к оценке", f"rate_user_{target_user_id}")]
            ]
        )

    except Exception as e:
        logging.error(f"Ошибка в vote_history_handler: {e}")
        await event.answer("❌ Произошла ошибка", alert=True)


@bot.on(events.CallbackQuery(pattern='cancel_rate'))
async def cancel_rate_handler(event):
    """Отмена оценки"""
    await event.delete()
    await event.answer("❌ Оценка отменена", alert=False)


@bot.on(events.CallbackQuery(pattern=r'appeal_(\d+)'))
async def appeal_handler(event):
    target_user_id = int(event.pattern_match.group(1))
    sender_id = event.sender_id

    # Сохраняем информацию о том, на кого подается апелляция
    user_states[sender_id] = {'appeal_target': target_user_id, 'waiting_for_appeal': True}

    try:
        await bot.send_message(
            sender_id,
            f"📝 Вы начали процесс апелляции на пользователя с ID {target_user_id}.\n\n"
            "Пожалуйста, напишите текст вашей апелляции. Опишите подробно причины, "
            "по которым считаете, что пользователь не должен быть в базе скамеров.\n\n"
            "❌ Отправьте 'отмена' для отмены процесса."
        )
        await event.answer("📨 Инструкции по апелляции отправлены вам в личные сообщения", alert=True)
    except Exception as e:
        await event.answer("❌ Не удалось отправить сообщение. Убедитесь, что у бота есть доступ к вашим ЛС", alert=True)
        logging.error(f"Ошибка отправки сообщения в ЛС: {e}")

@bot.on(events.NewMessage)
async def handle_appeal_text(event):
    user_id = event.sender_id

    # Проверяем, что это личное сообщение и пользователь ожидает ввода апелляции
    if event.is_private and user_id in user_states and user_states[user_id].get('waiting_for_appeal'):
        appeal_text = event.raw_text.strip()

        # Проверка на отмену
        if appeal_text.lower() in ['отмена', 'cancel', 'отменить']:
            if user_id in user_states:
                del user_states[user_id]
            await event.respond("❌ Процесс апелляции отменен.")
            return

        # Проверяем, что текст не пустой
        if not appeal_text:
            await event.respond("❌ Текст апелляции не может быть пустым. Пожалуйста, напишите вашу апелляцию.")
            return

        target_user_id = user_states[user_id]['appeal_target']

        try:
            # Получаем информацию о пользователях
            target_user = await bot.get_entity(target_user_id)
            sender_user = await event.get_sender()

            # Формируем сообщение для чата апелляций
            appeal_message = (
                f"🚨 **Новая апелляция**\n\n"
                f"👤 **На пользователя:** {target_user.first_name} (ID: {target_user_id})\n"
                f"📝 **От пользователя:** {sender_user.first_name} (ID: {user_id})\n"
                f"📄 **Текст апелляции:**\n{appeal_text}\n\n"
                f"⏰ **Время подачи:** {datetime.now().strftime('%d.%m.%Y %H:%M')}"
            )

            # ID группы апелляций (ЗАМЕНИТЕ НА РЕАЛЬНЫЙ ID!)
            APPEAL_CHAT_ID = -1003808268065   # Замените на реальный ID группы

            # Пытаемся отправить в группу апелляций
            try:
                await bot.send_message(
                    APPEAL_CHAT_ID,
                    appeal_message,
                    parse_mode='md'
                )
                logging.info(f"Апелляция успешно отправлена в группу {APPEAL_CHAT_ID}")

                # Подтверждаем пользователю
                await event.respond(
                    "✅ Ваша апелляция успешно отправлена на рассмотрение!\n\n"
                    "Мы рассмотрим ваше обращение в ближайшее время. "
                    "О результате уведомим вас личным сообщением."
                )

            except Exception as e:
                logging.error(f"Ошибка отправки в группу апелляций: {e}")
                await event.respond(
                    f"❌ Ошибка при отправке апелляции в группу. "
                    f"Пожалуйста, сообщите администраторам об ошибке: {str(e)}"
                )

        except Exception as e:
            logging.error(f"Ошибка при обработке апелляции: {e}")
            await event.respond(
                "❌ Произошла ошибка при обработке апелляции. "
                "Пожалуйста, попробуйте позже или свяжитесь с администраторами."
            )

        # Очищаем состояние в любом случае
        if user_id in user_states:
            del user_states[user_id]


async def get_user_profile_response(event, user, user_data):
    user_id = user.id
    role_id = db.get_user_role(user_id)

    print(f"User ID: {user_id}, Role: {role_id}")

    custom_image_url = db.get_user_custom_photo_url(user_id)
    print(f"Custom image URL: {custom_image_url}")

    logging.info(f"Проверка профиля для user_id: {user_id}, role_id: {role_id}")

    # Получаем данные пользователя из user_data или базы
    if user_data:
        country = user_data[5].strip() if len(user_data) > 5 and user_data[5] else "❓"
        channel = user_data[6].strip() if len(user_data) > 6 and user_data[6] else "❓"
    else:
        country = "❓"
        channel = "❓"

    description = db.get_user_description(user_id) or "Нет описания"
    checks_count = db.get_check_count(user_id)
    logging.info(f"Количество проверок для user_id {user_id} после увеличения: {checks_count}")

    # ✅ ИСПРАВЛЕНО: Получаем ПРАВИЛЬНОЕ количество для отображения
    # Используем новый метод get_display_scammers_count
    scammers_display_count = db.get_display_scammers_count(user_id)

    # Для обратной совместимости
    scammers_slept = scammers_display_count

    logging.info(f"Custom image URL retrieved for user {user_id}: {custom_image_url}")

    current_time = datetime.now().strftime("%d.%m.%Y")

    try:
        user_rating = db.get_user_rating(user_id)
        rating_stars = "⭐" * int(user_rating / 2)
        rating_display = f"{rating_stars} {user_rating:.1f}/10"
    except Exception as e:
        logging.error(f"Ошибка получения рейтинга для {user_id}: {e}")
        rating_display = "⭐ 5.0/10"

    # Получаем имя пользователя для отображения
    if hasattr(user, 'first_name') and user.first_name:
        user_name = user.first_name
    elif hasattr(user, 'title') and user.title:
        user_name = user.title
    elif hasattr(user, 'username') and user.username:
        user_name = f"@{user.username}"
    else:
        user_name = f"ID: {user.id}"

    if hasattr(user, 'username') and user.username:
        profile_url = f"https://t.me/{user.username}"
    else:
        profile_url = f"tg://user?id={user.id}"

    # Получаем все записи о скамере для отображения в профиле
    scammer_details = ""
    if role_id in [2, 3, 4, 5]:  # Все типы скамеров
        details = db.get_scammer_details(user_id)
        if details:
            scammer_details = "\n\n📋 **История заносов:**\n"
            for i, (reason, proof_link, reported_by, added_date) in enumerate(details, 1):
                # Форматируем дату
                if added_date:
                    try:
                        if isinstance(added_date, str):
                            date_obj = datetime.strptime(added_date, '%Y-%m-%d %H:%M:%S')
                            formatted_date = date_obj.strftime('%d.%m.%Y %H:%M')
                        else:
                            formatted_date = str(added_date)[:16]
                    except:
                        formatted_date = str(added_date)
                else:
                    formatted_date = "Неизвестно"

                # Формируем текст доказательств
                if proof_link and proof_link.startswith(('http://', 'https://')):
                    proof_text = f"[Доказательства]({proof_link})"
                elif proof_link:
                    proof_text = f"📎 {proof_link[:50]}..."
                else:
                    proof_text = "Нет доказательств"

                scammer_details += f"**{i}. 📝 Причина:** {reason}\n"
                scammer_details += f"   🔗 **{proof_text}**\n"
                scammer_details += f"   👮 **Занес:** {reported_by}\n"
                scammer_details += f"   📅 **Дата:** {formatted_date}\n\n"

    profile_url = f"tg://user?id={user.id}"
    buttons = [
        [
            Button.url("🎧 Профиль", profile_url),
            Button.inline("⚖️ Апелляция", f"appeal_{user_id}".encode())
        ],
        [
            Button.inline("🚫 Слить скаммера", f"sliv_scammers_{user_id}".encode()),
            Button.inline("⭐ Оценить", f"rate_user_{user_id}".encode())
        ]
    ]

    if role_id in [2, 3, 4, 5]:  # Возможно скамер, Скамер, Петух, Подозрение на скам
        buttons.append([Button.inline("🚫 Вынести из базы", f"remove_from_db_{user_id}".encode())])

    warnings_count = db.get_warnings_count(user_id)

    emojis = ["🧛‍♂️", "👩‍💻", "🎮", "🔥", "⛄", "☃", "🌟", "🐻", "🐳", "🐵", "🦢", "💸", "🌸", "💥", "🌈", "🐹", "🦉"]

    message_text = ""

    random_emoji = random.choice(emojis) if country == "❓" else ""

    country_display = f"[Не указана](https://telegra.ph/Kak-ustanovit-stranu-v-bote-05-29)" if country == "❓" else country

    granted_by_id = db.get_granted_by(user.id) if hasattr(user, 'id') else None
    logging.info(f"Получен ID гаранта: {granted_by_id}")

    granted_by_username = "Неизвестный гарант"
    if granted_by_id is not None:
        try:
            logging.info(f"Попытка получить информацию о гаранте с ID {granted_by_id}")
            granted_by_user = await bot.get_entity(granted_by_id)
            granted_by_username = granted_by_user.username if hasattr(granted_by_user,
                                                                      'username') and granted_by_user.username else (
                granted_by_user.first_name if hasattr(granted_by_user, 'first_name') else f"ID: {granted_by_id}")
            logging.info(f"Имя гаранта: {granted_by_username}")
        except Exception as e:
            logging.error(f"Ошибка при получении информации о гаранте с ID {granted_by_id}: {e}")
    else:
        logging.warning("granted_by_id равен None, гарант не найден.")

    theme_url = ROLES.get(role_id, {}).get('preview_url', '')

    if not theme_url:
        theme_url = "https://i.ibb.co/ycyPRXrb/photo-2025-04-17-17-44-20-2.jpg"  # Стандартная картинка

    theme_preview = f"{theme_url}\n\n"

    # ============ ФОРМИРОВАНИЕ ТЕКСТА ПО РОЛЯМ ============
    # ✅ ИСПРАВЛЕНО: Везде используется scammers_display_count вместо scammers_slept

    if role_id == 0:
        preview_url = ROLES[role_id]['preview_url']
        message_text = (
            f"[👤][ {user_name} ](tg://user?id={user_id}) #id{user_id} [⠀]({ROLES[role_id]['preview_url']})\n\n"
            f"[❌] Статус: не найден в базе. Риск скама: **44%**\n"
            f"[ℹ️] [Узнайте о гарантах](https://telegra.ph/Kto-takoj-GARANT-05-29)\n\n"
            f"[📍] Регион: {country_display}\n"
            f"[🚫] Разоблачено скаммеров: {scammers_display_count}\n\n"  # ✅ ИСПРАВЛЕНО
            f"[🔒] Используйте Гарантов infinity для безопасных сделок.\n\n"
            f"[📅] Дата: {current_time} | 🔎Проверок: {checks_count}\n"
        )

    elif role_id == 12:
        preview_url = ROLES[role_id]['preview_url']
        message_text = (
            f"[👤][ {user_name} ](tg://user?id={user_id}) #id{user_id} [⠀]({ROLES[role_id]['preview_url']})\n\n"
            f"[❌] Статус: Проверен(а) гарантом | [ {granted_by_username} ](tg://user?id={granted_by_id}) ✅\n"
            f"[ℹ️] [Узнайте о гарантам](https://telegra.ph/Kto-takoj-GARANT-05-29)\n\n"
            f"[📍] Регион: {country_display}\n"
            f"[🚫] Разоблачено скаммеров: {scammers_display_count}\n\n"  # ✅ ИСПРАВЛЕНО
            f"[🔒] Используйте Гарантов infinity для безопасных сделок.\n\n"
            f"[📅] Дата: {current_time} | 🔎Проверок: {checks_count}\n"
        )

    elif role_id == 1:
        preview_url = ROLES[role_id]['preview_url']
        message_text = (
            f"[👤][ {user_name} ](tg://user?id={user_id}) #id{user_id} [⠀]({ROLES[role_id]['preview_url']})\n\n"
            f"[✅] Статус: Гарант\n"
            f"[ℹ️] [Узнайте о гарантах](https://telegra.ph/Kto-takoj-GARANT-05-29)\n\n"
            f"[📍] Регион: {country_display}\n"
            f"[🚫] Разоблачено скаммеров: {scammers_display_count}\n\n"  # ✅ ИСПРАВЛЕНО
            f"[🔒] Данный пользователь является проверенным гарантом infinity\n\n"
            f"[📅] Дата: {current_time} | 🔎Проверок: {checks_count}\n"
        )

    elif role_id == 10:
        preview_url = ROLES[role_id]['preview_url']
        message_text = (
            f"[👤][ {user_name} ](tg://user?id={user_id}) #id{user_id} [⠀]({ROLES[role_id]['preview_url']})\n\n"
            f"[💢] Статус: Владелец\n"
            f"[💖] [Персонал infinity](https://t.me/infinityantiscam)\n\n"
            f"[📍] Регион: {country_display}\n"
            f"[🚫] Разоблачено скаммеров: {scammers_display_count}\n\n"  # ✅ ИСПРАВЛЕНО
            f"[🔒] Данный пользователь является Cоздателем базы infinity\n\n"
            f"[📅] Дата: {current_time} | 🔎Проверок: {checks_count}\n"
        )

    elif role_id == 9:
        preview_url = ROLES[role_id]['preview_url']
        message_text = (
            f"[👤][ {user_name} ](tg://user?id={user_id}) #id{user_id} [⠀]({ROLES[role_id]['preview_url']})\n\n"
            f"[🧿] Статус: Президент\n"
            f"[💖] [Персонал infinity](https://t.me/infinityantiscam)\n\n"
            f"[📍] Регион: {country_display}\n"
            f"[🚫] Разоблачено скаммеров: {scammers_display_count}\n\n"  # ✅ ИСПРАВЛЕНО
            f"[⚠] Выговоры: {warnings_count} "
            f"[🔒] Данный пользователь является надёжным президентом базы infinity\n\n"
            f"[📅] Дата: {current_time} | 🔎Проверок: {checks_count}\n"
        )

    elif role_id == 4:  # Петух
        preview_url = ROLES[role_id]['preview_url']
        message_text = (
            f"[👤][ {user_name} ](tg://user?id={user_id}) #id{user_id} [⠀]({ROLES[role_id]['preview_url']})\n\n"
            f"[🐓] Статус: Петух\n\n"
            f"[📍] Регион: {country_display}\n"
            f"[🚫] Разоблачено скаммеров: {scammers_display_count}\n\n"  # ✅ ИСПРАВЛЕНО
            f"📚 Описание: {description}\n"
            f"{scammer_details}"
            f"[❌] Данный пользователь является подозрительной личностью! \n\n"
            f"[📅] Дата: {current_time} | 🔎Проверок: {checks_count}\n"
        )

    elif role_id == 3:  # Скамер
        preview_url = ROLES[role_id]['preview_url']
        message_text = (
            f"[👤][ {user_name} ](tg://user?id={user_id}) #id{user_id} [⠀]({ROLES[role_id]['preview_url']})\n\n"
            f"[🛑] Статус: Скаммер\n\n"
            f"[📍] Регион: {country_display}\n"
            f"[🚫] Разоблачено скаммеров: {scammers_display_count}\n\n"  # ✅ ИСПРАВЛЕНО
            f"📚 Описание: {description}\n"
            f"{scammer_details}"
            f"[❌] Данный пользователь Является скаммером! Не идите первыми!\n\n"
            f"[📅] Дата: {current_time} | 🔎Проверок: {checks_count}\n"
        )

    elif role_id == 7:
        preview_url = ROLES[role_id]['preview_url']
        message_text = (
            f"[👤][ {user_name} ](tg://user?id={user_id}) #id{user_id} [⠀]({ROLES[role_id]['preview_url']})\n\n"
            f"[🔍] Статус: Админ\n"
            f"[💖] [Персонал infinity](https://t.me/infinityantiscam)\n\n"
            f"[📍] Регион: {country_display}\n"
            f"[🚫] Разоблачено скаммеров: {scammers_display_count}\n\n"  # ✅ ИСПРАВЛЕНО
            f"[⚠] Выговоры: {warnings_count} "
            f"[🔒] Данный пользователь является Администратором Базы infinity\n\n"
            f"[📅] Дата: {current_time} | 🔎Проверок: {checks_count}\n"
        )

    elif role_id == 5:  # Подозрения на скам
        preview_url = ROLES[role_id]['preview_url']
        message_text = (
            f"[👤][ {user_name} ](tg://user?id={user_id}) #id{user_id} [⠀]({ROLES[role_id]['preview_url']})\n\n"
            f"[🛑] Статус: Подозрения На Скам\n\n"
            f"[📍] Регион: {country_display}\n"
            f"[🚫] Разоблачено скаммеров: {scammers_display_count}\n\n"  # ✅ ИСПРАВЛЕНО
            f"📚 Описание: {description}\n"
            f"{scammer_details}"
            f"[❌] Данный пользователь Является подозрительной личностью, будьте осторожны!\n\n"
            f"[📅] Дата: {current_time} | 🔎Проверок: {checks_count}\n"
        )

    elif role_id == 2:  # Возможно скамер
        preview_url = ROLES[role_id]['preview_url']
        message_text = (
            f"[👤][ {user_name} ](tg://user?id={user_id}) #id{user_id} [⠀]({ROLES[role_id]['preview_url']})\n\n"
            f"[🛑] Статус: Возможно скаммер\n\n"
            f"[📍] Регион: {country_display}\n"
            f"[🚫] Разоблачено скаммеров: {scammers_display_count}\n\n"  # ✅ ИСПРАВЛЕНО
            f"📚 Описание: {description}\n"
            f"{scammer_details}"
            f"[❌] Данный пользователь Является Потонциальным скаммером, будьте осторожны!\n\n"
            f"[📅] Дата: {current_time} | 🔎Проверок: {checks_count}\n"
        )

    elif role_id == 6:
        preview_url = ROLES[role_id]['preview_url']
        message_text = (
            f"[👤][ {user_name} ](tg://user?id={user_id}) #id{user_id} [⠀]({ROLES[role_id]['preview_url']})\n\n"
            f"[👨‍🎓] Статус: Стажер\n"
            f"[💖] [Персонал infinity](https://t.me/infinityantiscam)\n\n"
            f"[📍] Регион: {country_display}\n"
            f"[🚫] Разоблачено скаммеров: {scammers_display_count}\n\n"  # ✅ ИСПРАВЛЕНО
            f"[⚠] Выговоры: {warnings_count}\n"
            f"[📣] Канал: {channel}\n\n"
            f"[🔒] Данный пользователь является Стажёром Базы infinity\n\n"
            f"[📅] Дата: {current_time} | 🔎Проверок: {checks_count}\n"
        )

    elif role_id == 8:
        preview_url = ROLES[role_id]['preview_url']
        message_text = (
            f"[👤][ {user_name} ](tg://user?id={user_id}) #id{user_id} [⠀]({ROLES[role_id]['preview_url']})\n\n"
            f"[‍🎩] Статус: Директор\n"
            f"[💖] [Персонал infinity](https://t.me/infinityantiscam)\n\n"
            f"[📍] Регион: {country_display}\n"
            f"[🚫] Разоблачено скаммеров: {scammers_display_count}\n\n"  # ✅ ИСПРАВЛЕНО
            f"[⚠] Выговоры: {warnings_count}\n"
            f"[📣] Канал: {channel}\n\n"
            f"[🔒] Данный пользователь является Директором Базы infinity\n\n"
            f"[📅] Дата: {current_time} | 🔎Проверок: {checks_count}\n"
        )

    elif role_id == 11:
        preview_url = ROLES[role_id]['preview_url']
        message_text = (
            f"[👤][ {user_name} ](tg://user?id={user_id}) #id{user_id} [⠀]({ROLES[role_id]['preview_url']})\n\n"
            f"[👨‍💻] Статус: Кодер\n"
            f"[💖] [Персонал infinity](https://t.me/infinityantiscam)\n\n"
            f"[📍] Регион: {country_display}\n"
            f"[🚫] Разоблачено скаммеров: {scammers_display_count}\n\n"  # ✅ ИСПРАВЛЕНО
            f"[⚠] Выговоры: {warnings_count}\n"
            f"[📣] Канал: {channel}\n\n"
            f"[🔒] Данный пользователь является Техническим Специалистом Базы infinity\n\n"
            f"[📅] Дата: {current_time} | 🔎Проверок: {checks_count}\n"
        )
    else:
        logging.warning(f"Неизвестная роль: {role_id}")
        message_text = "❌ Неизвестная роль"
        return message_text, buttons

    return message_text, buttons


@bot.on(events.NewMessage(pattern='/profile'))
async def handler(event):
    user = event.sender  # Получаем пользователя, который отправил сообщение
    user_data = db.get_user_data(user.id)  # Получаем данные пользователя из базы


async def send_response(event, response_text, buttons=None):
    if buttons:
        await event.respond(response_text, buttons=buttons, parse_mode='md')
    else:
        await event.respond(response_text, parse_mode='md')


@bot.on(events.CallbackQuery(data=re.compile(r"^profile_(\d+)$")))
async def callback_handler(event):
    user_id = int(event.pattern_match.group(1))

    # Перекидываем пользователя на профиль выбранного человека
    await event.client.send_message(event.chat_id, f"tg://user?id={user_id}", link_preview=False)


last_check_time = {}

# Глобальная переменная для хранения кэша
joined_users_cache = set()


# Функция для сброса кэша
def reset_cache():
    global joined_users_cache
    joined_users_cache.clear()  # Очищаем кэш
    logging.info('Кэш успешно сброшен.')


@bot.on(events.NewMessage(pattern=r'(?i)^(чек|чек ми|чек я|чек себя|check|/check).*'))
@Database.cooldown(3)
async def check_user(event):
    global checks_count
    user_id = event.sender_id
    user = await event.get_sender()

    # Проверка на спам
    current_time = time.time()

    # Проверяем, не выполняется ли уже проверка
    if user_id in active_checks and active_checks[user_id]:
        await event.respond("⏳ Ваш предыдущий запрос еще обрабатывается...")
        return

    # Проверяем время последнего вызова
    if user_id in last_check_time:
        elapsed_time = current_time - last_check_time[user_id]
        if elapsed_time < check_cooldown:
            remaining_time = check_cooldown - elapsed_time
            await event.respond(f"⏳ Подождите {remaining_time:.1f} секунд перед повторной проверкой!")
            return

    # Устанавливаем блокировку
    active_checks[user_id] = True
    last_check_time[user_id] = current_time

    loading_msg = None
    try:
        loading_msg = await event.respond("🔍 Ищу данные о пользователе в базе...")
        await asyncio.sleep(0.5)

        user_to_check = None
        user_data = None

        # Определяем пользователя для проверки
        if event.reply_to_msg_id:
            replied = await event.get_reply_message()
            try:
                user_to_check = await event.client.get_entity(replied.sender_id)
                user_data = db.get_user(user_to_check.id) if db else None
            except Exception as e:
                await loading_msg.delete()
                active_checks[user_id] = False
                return await send_response(event, "❌ | Не удалось получить информацию о пользователе.")
        else:
            text_lower = event.raw_text.lower()
            if any(keyword in text_lower for keyword in ["чек ми", "чек я", "чек себя", "check me"]):
                user_to_check = user
                user_data = db.get_user(user.id) if db else None
            else:
                # Парсим аргументы
                args = event.raw_text.split()
                if len(args) > 1:
                    target = args[1].strip()

                    try:
                        # Пытаемся получить пользователя из Telegram
                        if target.isdigit():
                            # Для ID сначала пытаемся получить через get_entity
                            try:
                                user_to_check = await event.client.get_entity(int(target))
                            except:
                                # Если не получается, проверяем в базе данных
                                user_id_to_check = int(target)
                                user_data = db.get_user(user_id_to_check)
                                if user_data:
                                    # Создаем минимальный объект с данными из базы
                                    user_to_check = type('UserObject', (), {
                                        'id': user_id_to_check,
                                        'first_name': f"ID: {user_id_to_check}",
                                        'username': user_data[1] if len(user_data) > 1 and user_data[1] else None
                                    })()
                                else:
                                    # ДОБАВЛЯЕМ ПОЛЬЗОВАТЕЛЯ С РОЛЬЮ 0, ЕСЛИ ЕГО НЕТ В БАЗЕ
                                    db.add_user(user_id_to_check, str(user_id_to_check), 0)
                                    user_data = db.get_user(user_id_to_check)
                                    user_to_check = type('UserObject', (), {
                                        'id': user_id_to_check,
                                        'first_name': f"ID: {user_id_to_check}",
                                        'username': None
                                    })()
                        else:
                            # Для юзернейма
                            if target.startswith('@'):
                                target = target[1:]
                            user_to_check = await event.client.get_entity(target)

                        # Получаем данные из базы
                        user_data = db.get_user(user_to_check.id) if db and user_to_check else None

                    except Exception as e:
                        logging.error(f"Ошибка при получении пользователя: {e}")
                        await loading_msg.delete()
                        active_checks[user_id] = False
                        return await send_response(event, "❌ | Не удалось найти пользователя.")
                else:
                    # Если нет аргументов, проверяем себя
                    user_to_check = user
                    user_data = db.get_user(user.id) if db else None

        if user_to_check is None:
            await loading_msg.delete()
            active_checks[user_id] = False
            return await send_response(event, "❌ | Не удалось определить пользователя.")

        # Получаем данные, если они еще не загружены
        if not user_data and db:
            # ДОБАВЛЯЕМ ПОЛЬЗОВАТЕЛЯ С РОЛЬЮ 0, ЕСЛИ ЕГО НЕТ В БАЗЕ
            username = getattr(user_to_check, 'username', None) or getattr(user_to_check, 'first_name', f"ID: {user_to_check.id}")
            db.add_user(user_to_check.id, username, 0)
            user_data = db.get_user(user_to_check.id)

        # Увеличиваем счетчики проверок
        if db and user_to_check:
            db.increment_check_count(user_to_check.id)
        checks_count += 1

        # Формируем ответ
        response = await get_user_profile_response(event, user_to_check, user_data)

        if isinstance(response, tuple):
            message_text, buttons = response
        else:
            message_text = response
            buttons = []

        # Отправка результата
        try:
            await send_response(event, message_text[:4096] if len(message_text) > 4096 else message_text, buttons)
        except Exception as e:
            logging.error(f"Ошибка при отправке сообщения: {e}")

        # Уведомление для премиум-пользователей
        if db and db.is_premium_user(user_id) and event.raw_text.lower() in ('чек', '/check'):
            await bot.send_message(
                user_id,
                f'🔍 Пользователь [{user.first_name}](tg://user?id={user_id}) проверял вас в боте!',
                buttons=Button.inline("↩Скрыть", b"hide_message")
            )

        # Удаляем сообщение о загрузке
        if loading_msg:
            await loading_msg.delete()

    except Exception as e:
        logging.error(f"Ошибка в check_user: {e}")
        try:
            if loading_msg:
                await loading_msg.delete()
        except:
            pass
    finally:
        # Снимаем блокировку
        if user_id in active_checks:
            active_checks[user_id] = False


@bot.on(events.NewMessage(pattern=r'(?i)^/on$'))
async def enable_chat(event):
    """Команда для разрешения участникам чата писать сообщения."""
    user_id = event.sender_id

    # Проверяем, является ли пользователь с ролью 10
    if db.get_user_role(user_id) != 10:
        await event.respond("❌ У вас нет прав для выполнения этой команды.")
        return

    # Разрешаем всем участникам писать сообщения
    await bot.edit_permissions(event.chat_id, send_messages=True)

    await event.respond(
        "🔓 Предложка открыта, вы снова можете писать сообщения в чат![⠀](https://i.ibb.co/JFq2r3Dg/image.jpg)")


@bot.on(events.NewMessage(pattern=r'(?i)^/off$'))
async def disable_chat(event):
    """Команда для запрета участникам чата писать сообщения."""
    user_id = event.sender_id

    # Проверяем, является ли пользователь с ролью 10
    if db.get_user_role(user_id) != 10:
        await event.respond("❌ У вас нет прав для выполнения этой команды.")
        return

    # Запрещаем всем участникам писать сообщения
    await bot.edit_permissions(event.chat_id, send_messages=False)

    await event.respond(
        "🔒 Предложка закрыта на время, скоро мы вернёмся в строй, следите за новостями![⠀](https://i.ibb.co/JFq2r3Dg/image.jpg)")


@bot.on(events.CallbackQuery(pattern=r'remove_from_db_(\d+)'))
async def remove_from_db_handler(event):
    user_id = int(event.pattern_match.group(1))

    # Проверяем права пользователя
    sender_role = db.get_user_role(event.sender_id)
    allowed_roles = [6, 7, 8, 9, 10, 11, 13]  # Стажер, Админ, Директор, Президент, Создатель, Кодер, Заместитель

    if sender_role not in allowed_roles:
        await event.answer("❌ У вас нет прав для выполнения этого действия!", alert=True)
        return

    try:
        # Получаем информацию о пользователе, которого нужно вынести из базы
        target_user = await bot.get_entity(user_id)
        target_role = db.get_user_role(user_id)

        # Проверяем, что пользователь действительно имеет одну из ролей скамера
        if target_role not in [2, 3, 4, 5]:
            await event.answer("❌ Этот пользователь не является скамером!", alert=True)
            return

        # Снимаем статус скамера - устанавливаем роль "Нет в базе" (0)
        db.update_role(user_id, 0)

        # Удаляем из таблицы scammers
        db.cursor.execute('DELETE FROM scammers WHERE user_id = ?', (user_id,))
        db.conn.commit()

        # Получаем информацию о пользователе, который выполнил действие
        admin_user = await bot.get_entity(event.sender_id)

        await event.answer("✅ Пользователь успешно вынесен из базы!", alert=True)

        # Редактируем сообщение, убирая кнопку
        await event.edit(
            f"👤 Пользователь [{target_user.first_name}](tg://user?id={user_id}) был вынесен из базы\n"
            f"👮 Вынес: [{admin_user.first_name}](tg://user?id={event.sender_id})\n"
            f"📅 Время: {datetime.now().strftime('%d.%m.%Y %H:%M')}",
            buttons=[
                [
                    Button.url("🎧 Профиль",
                               f"https://t.me/{target_user.username}" if target_user.username else f"tg://user?id={user_id}"),
                    Button.inline("⚖️ Аппеляция", f"appeal_{user_id}")
                ]
            ],
            parse_mode='md'
        )

        logging.info(f"Пользователь {user_id} вынесен из базы пользователем {event.sender_id}")

    except Exception as e:
        logging.error(f"Ошибка при выносе пользователя из базы: {e}")
        await event.answer("❌ Произошла ошибка при выносе из базы!", alert=True)


@bot.on(events.NewMessage(pattern=r'(?i)^/cur|/курировать|/кур'))
async def cur_command(event):
    try:
        sender = await event.get_sender()
        user_id = sender.id

        user_role = db.get_user_role(user_id)
        allowed_roles = [8, 9, 13, 10]  # Директор, Президент, Заместитель, Владелец

        if user_role not in allowed_roles:
            await event.respond("❌ У вас нет прав для назначения кураторов.")
            return

        if event.is_reply:
            replied = await event.get_reply_message()
            target_id = replied.sender_id
            try:
                target = await event.client.get_entity(target_id)
            except ValueError:
                await event.reply("❌ Не могу найти пользователя.")
                return
        else:
            args = event.raw_text.split()
            if len(args) < 2:
                await event.reply("❌ Используйте: /cur @username или ответьте на сообщение.")
                return

            username = args[1].lstrip('@')
            try:
                target = await event.client.get_entity(username)
            except Exception as e:
                await event.reply("❌ Не могу найти указанного пользователя.")
                return

        # Проверяем, что целевой пользователь - стажёр
        target_role = db.get_user_role(target.id)
        if target_role != 6:
            await event.reply("❌ Указанный пользователь не является стажёром.")
            return

        # Назначаем куратора
        db.cursor.execute('UPDATE users SET curator_id = ? WHERE user_id = ?', (user_id, target.id))
        db.conn.commit()

        # Получаем имена для отображения
        try:
            target_user = await event.client.get_entity(target.id)
            target_name = target_user.first_name
        except:
            target_name = f"ID:{target.id}"

        sender_name = sender.first_name

        await event.reply(
            f"✅ Стажёру {target_name} назначен куратор: {sender_name}.\n\n"
            f"Теперь стажёр может использовать команду /скам для отправки заявок на скаммеров, "
            f"которые будут приходить вам на проверку."
        )

    except Exception as e:
        print(f"Ошибка в команде /cur: {e}")
        await event.reply("❌ Произошла ошибка при выполнении команды.")


@bot.on(events.NewMessage(pattern=r'(?i)^/mycur|/мойкуратор'))
async def my_curator_command(event):
    user_id = event.sender_id

    curator_id = db.get_user_curator(user_id)

    if not curator_id:
        await event.respond("❌ У вас нет назначенного куратора.")
        return

    try:
        curator = await event.client.get_entity(curator_id)
        curator_name = curator.first_name

        await event.respond(
            f"👨‍🏫 **Ваш куратор:** {curator_name}\n"
            f"🆔 ID: {curator_id}\n\n"
            f"Для отправки заявок на скаммеров используйте команду:\n"
            f"`/скам @username причина`\n\n"
            f"Заявки будут приходить вашему куратору на проверку."
        )
    except:
        await event.respond(f"👨‍🏫 **Ваш куратор:** ID {curator_id}")


# Функция для периодической очистки старых данных о нажатиях
async def cleanup_old_button_data():
    """Очищает старые данные о нажатиях кнопок"""
    while True:
        await asyncio.sleep(3600)  # Каждый час
        current_time = time.time()
        to_remove = []

        for user_id, data in user_button_presses.items():
            if current_time - data['window_start'] > BUTTON_PRESS_WINDOW * 2:  # Двойное окно времени
                to_remove.append(user_id)

        for user_id in to_remove:
            del user_button_presses[user_id]

        logging.info(f"Очищены данные о нажатиях для {len(to_remove)} пользователей")


@bot.on(events.NewMessage(pattern=r'(?i)^(выговор|/выговор)'))
async def warning_handler(event):
    if event.is_reply:
        replied = await event.get_reply_message()
        target_user = await event.client.get_entity(replied.sender_id)
    else:
        await event.reply("❌ Пожалуйста, используйте команду в ответ на сообщение пользователя.")
        return

    # Проверяем права
    user_role = db.get_user_role(event.sender_id)
    logging.info(f"Пользователь {event.sender_id} с ролью {user_role} пытается выдать выговор.")

    # Проверяем, чтобы создающий пользователь (роль 10) не получал выговоры
    target_user_role = db.get_user_role(target_user.id)
    if target_user_role == 10:
        await event.reply("Ты шо ахуел?, нельзя владельцу выговоры выдавать!.")
        return

    # Проверяем, что у пользователя есть права на выдачу выговоров
    if user_role not in [13, 8, 9, 10]:  # Только админ, директор, президент
        await event.reply("❌ У вас нет прав для выдачи выговора.")
        return

    # Получаем количество выговоров
    result = db.cursor.execute('SELECT warnings FROM users WHERE user_id = ?', (target_user.id,)).fetchone()

    if result is None:
        # Если пользователь не найден, добавляем его в базу с 0 выговорами
        db.add_user(target_user.id, target_user.username, 0)  # Добавляем пользователя с нулевым количеством выговоров
        warnings_count = 0
    else:
        warnings_count = result[0]

    # Увеличиваем количество выговоров
    db.update_warnings(target_user.id)

    # Получаем обновленное количество выговоров
    new_warnings_count = \
        db.cursor.execute('SELECT warnings FROM users WHERE user_id = ?', (target_user.id,)).fetchone()[0]

    if new_warnings_count >= 3:
        # Снимаем статус пользователя
        db.update_role(target_user.id, 0)

        # Сбрасываем количество выговоров до 0
        db.reset_warnings(target_user.id)

        await event.reply(
            f"✅ Пользователь [{target_user.first_name}](tg://user/{target_user.id}) получил 3 выговора и теперь имеет статус 'Нет в базе'.")
    else:
        await event.reply(
            f"✅ Выговор выдан пользователю [{target_user.first_name}](tg://user/{target_user.id})")


@bot.on(events.NewMessage(pattern=r'(?i)^(/-выговор|снять выговор)'))
async def remove_warnings_handler(event):
    if event.is_reply:
        replied = await event.get_reply_message()
        target_user = await event.client.get_entity(replied.sender_id)
    else:
        await event.reply("❌ Пожалуйста, используйте команду в ответ на сообщение пользователя.")
        return

    # Проверяем права
    user_role = db.get_user_role(event.sender_id)
    logging.info(f"Пользователь {event.sender_id} с ролью {user_role} пытается снять выговоры.")

    # Проверяем, что у пользователя есть права на снятие выговоров
    if user_role not in [13, 8, 9, 10]:  # Только админ, директор, президент
        await event.reply("❌ У вас нет прав для снятия выговоров.")
        return

    # Получаем текущее количество выговоров
    result = db.cursor.execute('SELECT warnings FROM users WHERE user_id = ?', (target_user.id,)).fetchone()
    if result is None:
        await event.reply("❌ Пользователь не найден в базе.")
        return

    warnings_count = result[0]

    if warnings_count <= 0:
        await event.reply(f"❌ У пользователя [{target_user.first_name}](tg://user/{target_user.id}) нет выговоров.")
        return

    # Уменьшаем количество выговоров на 1
    db.cursor.execute('UPDATE users SET warnings = warnings - 1 WHERE user_id = ?', (target_user.id,))
    db.conn.commit()

    # Получаем обновленное количество выговоров
    new_warnings_count = \
        db.cursor.execute('SELECT warnings FROM users WHERE user_id = ?', (target_user.id,)).fetchone()[0]

    await event.reply(
        f"✅ выговор снят у пользователя [{target_user.first_name}](tg://user/{target_user.id})."
    )


# Глобальная переменная для хранения времени последнего использования команды
last_sell_command_time = {}


@bot.on(events.NewMessage(pattern='/профиль'))
async def profile_command(event):
    user = await event.get_sender()  # Получаем пользователя, который отправил команду
    user_id = user.id
    role = db.get_user_role(user_id)  # Получаем роль пользователя

    # Получаем данные пользователя из базы
    user_data = db.get_user(user_id)
    custom_photo = user_data[7] if user_data else None
    preview_url = custom_photo if custom_photo else ROLES[role]['preview_url']
    checks_count = db.get_check_count(user_id)

    # Инициализация scammers_count
    scammers_count = 0
    scammers_info = ""

    # Для персонала показываем количество слитых скамеров
    if role in [6, 7, 8, 9, 10]:  # Проверяем, если это персонал
        scammers_count = db.get_user_reported_scammers_count(user_id)  # Получаем количество слитых скаммеров
        scammers_info = f"🔥 **Скаммеров слито:** `{scammers_count}`\n"
    else:
        scammers_info = "🔥 **Скаммеров слито:** `0`\n"  # Если не персонал, показываем 0

    # Формируем текст профиля
    profile_text = f"""
**👤 Профиль пользователя в базе: {user.first_name}](tg://user/{user_id})**

🔍 **Вас проверяли:** `{checks_count}` раз
{scammers_info}
**📝 Роль в базе:** {ROLES[role]['name']}
**Infinity Премиум:** {'✅' if db.get_premium_expiry(user_id) else '❌'}
[⠀](https://i.ibb.co/ycyPRXrb/photo-2025-04-17-17-44-20-2.jpg)
"""

    await event.respond(profile_text, parse_mode='md')


@bot.on(events.NewMessage(pattern=r'(?i)^\+спасибо'))
async def thank_command(event):
    logging.info("Команда +спасибо была вызвана.")
    user_id = event.sender_id

    # Проверка роли пользователя, который вызывает команду
    user_role = db.get_user_role(user_id)
    allowed_roles = [6, 7, 8, 9, 10, 11, 13]  # Стажер, Админ, Директор, Президент, Создатель, Кодер, Заместитель

    if user_role not in allowed_roles:
        await event.respond("❌ Только персонал базы может выдавать +спасибо!")
        return

    if event.reply_to_msg_id:
        reply_message = await event.get_reply_message()
        target_user_id = reply_message.sender_id

        # Получаем текущее РЕАЛЬНОЕ количество слитых скамеров
        current_real_count = db.get_user_reported_scammers_count(target_user_id)

        # Увеличиваем счетчик в таблице users (scammers_count)
        try:
            # Получаем текущее значение
            db.cursor.execute('SELECT scammers_count FROM users WHERE user_id = ?', (target_user_id,))
            result = db.cursor.fetchone()

            if result:
                current_count = result[0] if result[0] is not None else 0
                new_count = current_count + 1
            else:
                new_count = 1

            # Обновляем значение
            db.cursor.execute('UPDATE users SET scammers_count = ? WHERE user_id = ?', (new_count, target_user_id))
            db.conn.commit()

            logging.info(f"Счетчик scammers_count для {target_user_id} увеличен: {new_count}")
        except Exception as e:
            logging.error(f"Ошибка обновления scammers_count: {e}")
            new_count = current_real_count + 1

        try:
            sender = await event.get_sender()
            target_user = await bot.get_entity(target_user_id)

            # Формируем сообщение с ПРАВИЛЬНЫМИ данными
            response_text = (
                f"📛 **Пользователю выдано +спасибо!**\n\n"
                f"👤 **Получил:** [{target_user.first_name}](tg://user?id={target_user_id})\n"
                f"👮 **Выдал:** [{sender.first_name}](tg://user?id={user_id})\n"
                f"🔥 **Общий счетчик:** {new_count}\n"
                f"🎯 **Реально слито скамеров:** {current_real_count}\n\n"
                f"📈 Спасибо, что боретесь со скамом вместе с Infinity!"
            )

            await event.respond(response_text, parse_mode='md')

            # Уведомляем получателя
            try:
                await bot.send_message(
                    target_user_id,
                    f"🎉 **Вам выдано +спасибо!**\n\n"
                    f"👮 **Выдал:** {sender.first_name}\n"
                    f"🔥 **Ваш счетчик:** {new_count}\n"
                    f"🎯 **Реально слито скамеров:** {current_real_count}\n\n"
                    f"Спасибо за ваш вклад в борьбу со скамом!",
                    buttons=Button.inline("↩Скрыть", b"hide_message")
                )
            except:
                pass  # Пользователь мог заблокировать бота

        except Exception as e:
            logging.error(f"Ошибка получения данных пользователя: {e}")
            await event.respond(f"✅ +спасибо выдано пользователю с ID: {target_user_id}\n🔥 Новый счетчик: {new_count}")
    else:
        await event.respond("❌ Ответьте на сообщение пользователя, которому хотите выдать +спасибо.")


@bot.on(events.NewMessage())
async def message_handler(event):
    user_id = event.sender_id

    # Игнорируем сообщения от ботов
    if event.sender.bot:
        return

    # ПРОВЕРКА: только для групп/супергрупп
    if not event.is_group and not event.is_channel:
        return  # Игнорируем личные сообщения

    # Получаем текущее время
    current_time = datetime.now()

    # Добавляем временную метку сообщения
    user_message_count[user_id].append(current_time)

    # Удаляем временные метки старше 30 секунд
    user_message_count[user_id] = [timestamp for timestamp in user_message_count[user_id]
                                   if current_time - timestamp < timedelta(seconds=30)]

    # Проверяем количество сообщений
    if len(user_message_count[user_id]) > 8:
        try:
            # Выдаём мут на 10 минут (ТОЛЬКО В ГРУППАХ)
            await bot.edit_permissions(
                event.chat_id,
                user_id,
                until_date=current_time + timedelta(minutes=10),
                send_messages=False,
                send_media=False,
                send_stickers=False,
                send_gifs=False,
                send_games=False,
                send_inline=False
            )

            await event.respond(f"🔇 Пользователь {event.sender.first_name} был замучен за спам на 10 минут!")
            logging.info(f"Пользователь {user_id} замучен за спам.")

            # Очищаем записи сообщений после мута
            del user_message_count[user_id]

        except Exception as e:
            logging.error(f"Ошибка при выдаче мута: {e}")
            # Игнорируем ошибки, не прерываем работу бота

# Глобальные переменные
games = {}
joined_users_cache = set()
guesses = {}
muted_users = {}  # {user_id: expiry_time}
last_scam_times = {}
START_USERS = set()  # Пользователи, использовавшие /start
BOT_CHATS = set()  # Чаты, где есть бот
LAST_CHECKED = {}  # Последний проверенный пользователь
TEMP_STORAGE = {}  # Временное хранилище данных
COUNTRIES = [
    "США 🇺🇸", "Канада 🇨🇦", "Мексика 🇲🇽", "Бразилия 🇧🇷",
    "Аргентина 🇦🇷", "Великобритания 🇬🇧", "Франция 🇫🇷",
    "Германия 🇩🇪", "Италия 🇮🇹", "Испания 🇪🇸", "Китай 🇨🇳",
    "Япония 🇯🇵", "Австралия 🇦🇺", "Индия 🇮🇳", "Россия 🇷🇺",
    "Южноафриканская Республика 🇿🇦", "Египет 🇪🇬", "ОАЭ 🇦🇪",
    "Турция 🇹🇷", "Греция 🇬🇷", "Швеция 🇸🇪", "Норвегия 🇳🇴",
    "Финляндия 🇫🇮", "Дания 🇩🇰", "Польша 🇵🇱", "Чехия 🇨🇿",
    "Австрия 🇦🇹", "Швейцария 🇨🇭", "Нидерланды 🇳🇱", "Бельгия 🇧🇪",
    "Ирландия 🇮🇪", "Португалия 🇵🇹", "Румыния 🇷🇴", "Словакия 🇸🇰",
    "Словения 🇸🇮", "Хорватия 🇭🇷", "Латвия 🇱🇻", "Литва 🇱🇹",
    "Эстония 🇪🇪", "Мальта 🇲🇹", "Кипр 🇨🇾", "Исландия 🇮🇸",
    "Албания 🇦🇱", "Сербия 🇷🇸", "Босния и Герцеговина 🇧🇦",
    "Черногория 🇲🇪", "Македония 🇲🇰", "Косово 🇽🇰", "Беларусь 🇧🇾",
    "Украина 🇺🇦", "Грузия 🇬🇪", "Армения 🇦🇲", "Азербайджан 🇦🇿",
    "Казахстан 🇰🇿", "Узбекистан 🇺🇿", "Таджикистан 🇹🇯",
    "Туркменистан 🇹🇲", "Кыргызстан 🇰🇬", "Монголия 🇲🇳",
    "Иран 🇮🇷", "Ирак 🇮🇶", "Сирия 🇸🇾", "Ливан 🇱🇧",
    "Иордания 🇯🇴", "Катар 🇶🇦", "Бахрейн 🇧🇭", "Кувейт 🇰🇼",
    "Саудовская Аравия 🇸🇦", "Йемен 🇾🇪", "Вьетнам 🇻🇳",
    "Таиланд 🇹🇭", "Малайзия 🇲🇾", "Индонезия 🇮🇩", "Филиппины 🇵🇭",
    "Сингапур 🇸🇬", "Непал 🇳🇵", "Шри-Ланка 🇱🇰", "Бангладеш 🇧🇩",
    "Пакистан 🇵🇰", "Мьянма 🇲🇲", "Лаос 🇱🇦", "Камбоджа 🇰🇭",
    "Тайвань 🇹🇼", "Гонконг 🇭🇰", "Южная Корея 🇰🇷", "Северная Корея 🇰🇵",
    "Австралия 🇦🇺", "Новая Зеландия 🇳🇿", "Папуа — Новая Гвинея 🇵🇬",
    "Фиджи 🇫🇯", "Самоа 🇼🇸", "Тонга 🇹🇴", "Вануату 🇻🇺",
    "Микронезия 🇫🇲", "Науру 🇳🇷", "Тувалу 🇹🇻", "Соломоновы Острова 🇸🇧",
    "Кирибати 🇰🇷", "Сент-Люсия 🇱🇨", "Сент-Винсент и Гренадины 🇻🇨",
    "Барбадос 🇧🇧", "Ямайка 🇯🇲", "Тринидад и Тобаго 🇹🇹",
    "Багамы 🇧🇸", "Гренада 🇬🇩", "Антигуа и Барбуда 🇦🇬",
    "Сент-Китс и Невис 🇰🇳"
]

# API для загрузки изображений
IMG_API_KEY = "cb21b904cc405cdfc05731896bc29c64"


# Функция для проверки премиум статуса
def is_premium(user_id):
    expiry_date = db.get_premium_expiry(user_id)
    if not expiry_date:
        return False
    return datetime.strptime(expiry_date, "%Y-%m-%d %H:%M:%S") > datetime.now()


def get_main_buttons(user_id):
    """Возвращает главное меню с учетом прав пользователя"""
    buttons = [
        [Button.text("🎭 Профиль", resize=True)],
        [Button.text("👥 Состав базы", resize=True), Button.text("🔰 Проверенные пользователи", resize=True)],
        [Button.text("📊 Статистика базы", resize=True), Button.text("🚫 Слить скаммера!", resize=True)],
        [Button.text("🎮 Игры", resize=True), Button.text("❓ Частые вопросы", resize=True)]
    ]

    if user_id in OWNER_ID:
        buttons.append([Button.text("👑 Админ панель", resize=True)])

    return buttons

@bot.on(events.NewMessage(pattern='/start'))
@subscription_required
async def start(event):
    sender = await event.get_sender()
    user_id = event.sender_id
    START_USERS.add(user_id)

    # Основное сообщение с reply-кнопками
    await event.respond(
        "👋 Добро пожаловать в Infinity!\n\n"
        "Мы — официальный бот антискам базы Infinity | Scam Base, созданный для обеспечения вашей безопасности в мире обменов и сделок.\n\n"
        "🔒 Ваше доверие — наша приоритетная задача!\n\n[⠀](https://i.ibb.co/q3qgMsQz/photo-2025-04-17-17-44-18.jpg)!\n"
        "Спасибо, что выбрали Infinity! Вместе мы сделаем все безопаснее!",
        buttons=get_main_buttons(user_id)
    )

    # Дополнительные inline-кнопки
    inline_buttons = [
        [Button.url("🌍 Предложка", "https://t.me/InfinityAntiscam")],
        [Button.url("🔐 Кодер Бота", "https://t.me/Rewylerss")],
        [Button.url("🔍 Трейдинг Чат", "https://t.me/steal_a_brainrotchat1]")]
    ]

    await event.respond(
        "📌 **Дополнительные функции:**",
        buttons=inline_buttons,
        parse_mode='md'
    )

    # Сообщение с кнопкой добавления в чат
    await event.respond(
        "Спасибо за выбор Infinity🤗\n\nВы можете поддержать нас, добавив нашего бота в чат",
        buttons=[
            [Button.url("💌 добавить в чат",
                        "http://t.me/InfinityASB_bot?startgroup=newgroup&admin=manage_chat+delete_messages+restrict_members+invite_users+restrict_members+change_info+pin_messages+manage_video_chats")]
        ]
    )

@bot.on(events.CallbackQuery(pattern='about_project'))
async def about_project(event):
    about_text = (
        "спасибо за выбор Infinity🤗\n"
       f"вы можете поддержать нас,добавив нашего бота в чат\n"
   )


    buttons = [
        [Button.inline("🤔 Как стать гарантом?", "how_to_become_guarantor")],
        [Button.inline("🤑 У кого и как купить траст?", "how_to_buy_trust")],
        [Button.inline("😈 Как слить вам скаммера?", "how_to_report_scammer")]
    ]

    await event.respond(
        about_text,
        buttons=buttons,
        parse_mode='md'
    )


# Обработчики для других кнопок
@bot.on(events.CallbackQuery(pattern='how_to_become_guarantor'))
async def how_to_become_guarantor(event):
    response_text = (
        "Чтобы стать гарантом, нужно пройти набор в нашу базу. "
        "Владельцы регулярно проводят наборы на многие роли, в том числе гарантов. "
        "Если ты хочешь стать гарантом, просто пройди набор в нашу базу🤗[⠀](https://i.ibb.co/ZR8qJ80N/1.jpg)"
    )
    await event.respond(response_text)


@bot.on(events.CallbackQuery(pattern='how_to_buy_trust'))
async def how_to_buy_trust(event):
    response_text = (
        "Хочешь купить траст? Да ты богач🤗. Чтобы купить траст, тебе нужно написать нашим гарантам "
        "Гарантов ты можешь найти в чате или же нажав на кнопку 'Гаранты' в главном меню🤑[⠀](https://i.ibb.co/rGBBGyng/photo-2025-04-17-17-44-20.jpg)"
    )
    await event.respond(response_text)


@bot.on(events.CallbackQuery(pattern='how_to_report_scammer'))
async def how_to_report_scammer(event):
    response_text = (
        "Ох, тебя тоже задрали скаммеры? Меня тоже😡 Если ты действительно хочешь слить скаммера, "
        "то тебе нужно зайти в нашу базу и написать в чат доказательства скама и юзернейм-айди. "
        "Наши волонтёры занесут этого скаммера😈[⠀](https://i.ibb.co/bj4g7h3y/photo-2025-04-17-17-44-19-3.jpg)"
    )
    await event.respond(response_text)


# Обработчик кнопки "Поддержать"
@bot.on(events.CallbackQuery(pattern='support_handler'))
async def support_handler(event):
    support_text = (
        "Если вы хотите помочь кодеру для продвижения ботов, "
        "то вы можете поддержать кодера этими вариантами:\n\n"
        "Если вы хотите поддержать кодера в ттд ник: **pisun11000**[⠀](https://i.ibb.co/0x7KTr0/image.jpg)"
    )

    buttons = [
        [Button.url("💌 Крипто ботом", "https://t.me/send?start=IVdGVHgwlEsa")],
        [Button.url("💞 Наш кодер", "https://t.me/Steach_Garant")]
    ]

    await event.respond(
        support_text,
        buttons=buttons,
        parse_mode='md'
    )


@bot.on(events.NewMessage(pattern="❓ Частые вопросы"))
@subscription_required
async def faq_handler(event):
    user_id = event.sender_id

    # Проверка защиты от спама
    can_press, message = check_button_spam_protection(user_id)
    if not can_press:
        await event.respond(message)
        return

    # Показываем загрузку
    await show_button_loading(event, "частые вопросы")

    query = event.raw_text
    faq_buttons = [
        [Button.inline("Кто такой гарант?", "who_is_guarantee")],
        [Button.inline("Как найти гаранта?", "find_guarantee")],
        [Button.inline("Как стать волонтёром?", "become_volunteer")],
        [Button.inline("Как стать гарантом?", "become_guarantee")],
        [Button.inline("Как слить скаммера?", "report_scammer")],
        [Button.inline("Когда набор на админов?", "admin_recruitment")],
        [Button.inline("Можно ли купить роль в базе?", "buy_role")],
        [Button.inline("Можно ли купить снятие из базы?", "buy_removal")],
        [Button.inline("Вернуться ↩", "back_to_main")]
    ]

    # Удаляем сообщение о загрузке если есть
    if user_id in button_loading_messages:
        try:
            await bot.delete_messages(event.chat_id, button_loading_messages[user_id])
            del button_loading_messages[user_id]
        except:
            pass

    await event.respond("Выберите нужный вам пункт:[⠀](https://i.ibb.co/q3bGLp9J/image.png)", buttons=faq_buttons)


@bot.on(events.CallbackQuery(data="who_is_guarantee"))
async def who_is_guarantee_handler(event):
    response_text = (
        "💁‍♂️ Кто такой гарант?\n\n"
        "[У нас есть мини-статья об этом (ТЫК)](https://telegra.ph/Kto-takoj-GARANT-05-29)"
    )
    back_button = [Button.inline("Вернуться ↩", "back_to_main")]
    await event.respond(response_text, buttons=back_button)


@bot.on(events.CallbackQuery(data="find_guarantee"))
async def find_guarantee_handler(event):
    response_text = (
        "💁‍♂️ Как найти гаранта?\n\n"
        "В лс с ботом жмём кнопку 'Гаранты' или вводим /mms.\n\n"
        "Бот отобразит вам проверенных людей, которые безопасно проведут сделку 😉"
    )
    back_button = [Button.inline("Вернуться ↩", "back_to_main")]
    await event.respond(response_text, buttons=back_button)


@bot.on(events.CallbackQuery(data="become_volunteer"))
async def become_volunteer_handler(event):
    response_text = (
        "💁‍♂️ Как стать волонтёром?\n\n"
        "Следите за информацией в новостнике базы и участвуйте в наборах."
    )
    back_button = [Button.inline("Вернуться ↩", "back_to_main")]
    await event.respond(response_text, buttons=back_button)


@bot.on(events.CallbackQuery(data="become_guarantee"))
async def become_guarantee_handler(event):
    response_text = (
        "💁‍♂️ Как стать гарантом?\n\n"
        "Следите за информацией в новостнике базы и участвуйте в наборах."
    )
    back_button = [Button.inline("Вернуться ↩", "back_to_main")]
    await event.respond(response_text, buttons=back_button)


@bot.on(events.CallbackQuery(data="report_scammer"))
async def report_scammer_handler(event):
    response_text = (
        "💁‍♂️ Как слить скаммера?\n\n"
        "Слить скаммера можно в нашей группе жалоб - новостнике базы.\n"
        "- Заходите в группу и кидаете пруфы скама, админы их рассматривают и принимают решение."
    )
    back_button = [Button.inline("Вернуться ↩", "back_to_main")]
    await event.respond(response_text, buttons=back_button)


@bot.on(events.CallbackQuery(data="admin_recruitment"))
async def admin_recruitment_handler(event):
    response_text = (
        "💁‍♂️ Когда набор на админов?\n\n"
        "В среднем наборы проходят 2 раза в месяц."
    )
    back_button = [Button.inline("Вернуться ↩", "back_to_main")]
    await event.respond(response_text, buttons=back_button)


@bot.on(events.CallbackQuery(data="buy_role"))
async def buy_role_handler(event):
    response_text = (
        "НЕТ. Мы НЕ продаём админки/ роли гарантов в нашей базе. "
        "Если вы хотите поддержать нашу базу - /premium."
    )
    back_button = [Button.inline("Вернуться ↩", "back_to_main")]
    await event.respond(response_text, buttons=back_button)


@bot.on(events.CallbackQuery(data="buy_removal"))
async def buy_removal_handler(event):
    response_text = (
        "НЕТ. Мы НЕ удаляем пользователей. Наша цель - быть надёжным и честным источником информации."
    )
    back_button = [Button.inline("Вернуться ↩", "back_to_main")]
    await event.respond(response_text, buttons=back_button)


@bot.on(events.CallbackQuery(data="back_to_main"))
async def back_to_main_handler(event):
    # Возвращаем пользователя к сообщению с кнопками выбора
    faq_buttons = [
        [Button.inline("Кто такой гарант?", "who_is_guarantee")],
        [Button.inline("Как найти гаранта?", "find_guarantee")],
        [Button.inline("Как стать волонтёром?", "become_volunteer")],
        [Button.inline("Как стать гарантом?", "become_guarantee")],
        [Button.inline("Как слить скаммера?", "report_scammer")],
        [Button.inline("Когда набор на админов?", "admin_recruitment")],
        [Button.inline("Можно ли купить роль в базе?", "buy_role")],
        [Button.inline("Можно ли купить снятие из базы?", "buy_removal")],
        [Button.inline("Вернуться ↩", "back_to_main")]
    ]

    await event.respond("Выберите нужный вам пункт:[⠀](https://i.ibb.co/q3bGLp9J/image.png)", buttons=faq_buttons)


# Обработчик команды /help
@bot.on(events.NewMessage(pattern='/help'))
async def help_cmd(event):
    help_text = """
🤖 **Команды бота:**

📋 **Проверка пользователей:**
• `Чек [юзернейм/ID]` - проверить пользователя
• `Чек` (ответом на сообщение) - проверить пользователя
• `Чек ми/я/себя` - проверить себя

👮‍♂️ **Выдача ролей:**
• `+стажер` (ответом) - выдать роль стажера  
• `+админ` (ответом) - выдать роль админа
• `+директор` (ответом) - выдать роль директора
• `+президент` (ответом) - выдать роль президента 
• `+создатель` (ответом) - выдать роль создателя
• `+кодер` (ответом) - выдать роль кодера
• `+гарант` (ответом) - выдать роль гаранта

🔄 **Снятие ролей:**
• `-стажер` (ответом) - снять роль стажера
• `-админ` (ответом) - снять роль админа  
• `-директор` (ответом) - снять роль директора
• `-президент` (ответом) - снять роль президента
• `-создатель` (ответом) - снять роль создателя  
• `-кодер` (ответом) - снять роль кодера
• `-гарант` (ответом) - снять роль гаранта

⚠️ **Примечание:**
Команды выдачи и снятия ролей доступны только создателю и кодеру!
"""
    await event.respond(help_text, parse_mode='md')


# Переменные для хранения статистики
guarantors_count = len(get_guarantors())  # Получаем количество гарантов
trainees_count = len(get_trainees())  # Получаем количество стажеров
total_messages = 0
verified_guarantors_count = 0
checks_count = 0
scammers_count = 0

# Словари для хранения времени последнего вызова команд
admin_cooldowns = {}
guarantor_cooldowns = {}


# Обработчик команды "админы!"
@bot.on(events.NewMessage(pattern=r'(?i)^админы!$'))
async def call_admins(event):
    user_id = event.sender_id
    current_time = datetime.now()

    # Проверка времени последнего вызова команды
    if user_id in admin_cooldowns:
        time_diff = current_time - admin_cooldowns[user_id]
        if time_diff < timedelta(hours=4):
            remaining = timedelta(hours=4) - time_diff
            hours = remaining.seconds // 3600
            minutes = (remaining.seconds % 3600) // 60
            await event.respond(
                f"**⏳ Подождите {hours} ч. {minutes} мин. прежде чем снова вызывать админов!**"
            )
            return

    admin_cooldowns[user_id] = current_time

    # Получение администраторов из базы данных
    conn =get_db_connection()
    cursor = conn.cursor()
    cursor.execute('''
        SELECT user_id FROM users 
        WHERE role IN ("Стажер", "Админ", "Директор", "Президент", "Создатель", "Заместитель")
    ''')
    admins = cursor.fetchall()
    conn.close()

    # Создаем текст с невидимыми упоминаниями
    mentions_text = "**✅ Админы вызваны!**"
    for admin in admins:
        mentions_text += f"[\u200b](tg://user?id={admin[0]})"  # Невидимое упоминание

        # Отправка личного сообщения админам
        caller_username = event.sender.username
        caller_mention = f"@{caller_username}" if caller_username else event.sender.mention

        admin_message = f"**🚨 В чате пользователь {caller_mention} вызывает админов!**"

        await bot.send_message(admin[0], admin_message)
        logging.info(f"Сообщение отправлено администратору: {admin[0]}")

    # Отправляем одно сообщение с текстом и скрытыми упоминаниями
    await event.respond(mentions_text)


# Обработчик команды "гаранты!"
@bot.on(events.NewMessage(pattern=r'(?i)^гаранты!$'))
async def call_guarantors(event):
    user_id = event.sender_id
    current_time = datetime.now()

    # Проверка времени последнего вызова команды
    if user_id in guarantor_cooldowns:
        time_diff = current_time - guarantor_cooldowns[user_id]
        if time_diff < timedelta(hours=1):
            remaining = timedelta(hours=1) - time_diff
            hours = remaining.seconds // 3600
            minutes = (remaining.seconds % 3600) // 60
            await event.respond(
                f"**⏳ Подождите {hours} ч. {minutes} мин. прежде чем снова вызывать гарантов!**"
            )
            return

    guarantor_cooldowns[user_id] = current_time

    # Получение гарантов из базы данных
    guarantors = get_guarantors()

    # Создаем текст с невидимыми упоминаниями
    mentions_text = "**🔰 Гаранты вызваны!**"
    for guarantor_id in guarantors:
        mentions_text += f"[\u200b](tg://user?id={guarantor_id})"  # Невидимое упоминание

        # Отправка личного сообщения гарантам
        caller_username = event.sender.username
        caller_mention = f"@{caller_username}" if caller_username else event.sender.mention

        guarantor_message = f"**🚨 В чате пользователь {caller_mention} вызывает гарантов!**"

        await bot.send_message(guarantor_id, guarantor_message)
        logging.info(f"Сообщение отправлено гаранту: {guarantor_id}")

    # Отправляем одно сообщение с текстом и скрытыми упоминаниями
    await event.respond(mentions_text)


# Обработчик команды "/stata"
@bot.on(events.NewMessage(pattern=r'(?i)^/stata$'))
async def show_statistics(event):
    global total_messages, verified_guarantors_count, checks_count, scammers_count, trainees_count

    # Получаем общее количество сообщений через экземпляр db
    total_messages = db.get_total_messages()  # Используйте экземпляр db

    # Получаем количество гарантов
    guarantors_count = len(get_guarantors())  # Получаем актуальное количество гарантов

    statistics = (
        f"📊 **Статистика чата:[⠀](https://i.ibb.co/dwfVKmMH/photo-2025-04-17-17-44-19-2.jpg)**\n"
        f"👥 Гаранты: {guarantors_count}\n"
        f"👨‍🎓 Стажеры: {trainees_count}\n"
        f"📩 Общее количество сообщений: {total_messages}\n"
        f"✅ Проверены гарантом: {verified_guarantors_count}\n"
        f"🔍 Число проверок: {checks_count}\n"
        f"🚫 Скаммеры в базе: {scammers_count}"
    )

    await event.respond(statistics)


@bot.on(events.NewMessage())
async def count_messages(event):
    global total_messages
    total_messages += 1
    db.update_total_messages(1)
    logging.info(f"Общее количество сообщений: {total_messages}")


@bot.on(events.NewMessage(pattern=r'(?i)^/del'))
async def delete_message(event):
    if event.is_reply:
        replied_message = await event.get_reply_message()
        await replied_message.delete()  # Удаляем сообщение, на которое ответили
    else:
        await event.reply("❌ Пожалуйста, ответьте на сообщение, которое хотите удалить.")

@bot.on(events.NewMessage)
async def debug_all_messages(event):
    if event.sender_id == 262511724:  # Только ваши сообщения
        logging.info(f"DEBUG: Получено сообщение от {event.sender_id}: {event.raw_text}")

@bot.on(events.NewMessage(pattern=r'[+-](?:[А-Яа-яёЁ]+)(?:\s+(?:@?\w+|\d+))?'))
async def handle_role_command(event):
    logging.info(f"=== ПОПЫТКА ОБРАБОТКИ КОМАНДЫ РОЛИ ===")
    logging.info(f"Сообщение: {event.raw_text}")
    logging.info(f"Отправитель ID: {event.sender_id}")

    user_role = db.get_user_role(event.sender_id)
    is_admin = event.sender_id in [262511724] or user_role == 10

    if not is_admin:
        msg = await event.reply("❌ У вас нет прав для выполнения этой команды",
                                buttons=Button.inline("↩Скрыть", b"hide_message"))
        bot.last_message_id = msg.id
        return

    command_parts = event.raw_text.split()
    action = command_parts[0][0]  # + или -
    role = command_parts[0][1:].lower()

    # Получаем целевого пользователя
    try:
        if len(command_parts) > 1:
            target = command_parts[1]
            if target.isdigit():
                user = await event.client.get_entity(int(target))
            else:
                if target.startswith('@'):
                    target = target[1:]
                user = await event.client.get_entity(target)
        else:
            if not event.is_reply:
                msg = await event.reply("❌ Укажите пользователя или ответьте на его сообщение",
                                        buttons=Button.inline("↩Скрыть", b"hide_message"))
                bot.last_message_id = msg.id
                return
            replied = await event.get_reply_message()
            user = await event.client.get_entity(replied.sender_id)
    except:
        msg = await event.reply("❌ Не удалось найти пользователя!",
                                buttons=Button.inline("↩Скрыть", b"hide_message"))
        bot.last_message_id = msg.id
        return

    role_mapping = {
        'стажер': 6,
        'админ': 7,
        'директор': 8,
        'президент': 9,
        'гарант': 1,
        'кодер': 11,
        'владелец': 10,
        'айдош': 13,
        'создатель': 10
    }

    current_role = db.get_user_role(user.id)

    # Проверяем, есть ли пользователь в базе
    user_exists = db.get_user(user.id) is not None

    if action == '+':
        # Проверка для президента
        if user_role == 9 and role in ['президент']:
            msg = await event.reply("❌ Президент не может выдавать роль президента.",
                                    buttons=Button.inline("↩Скрыть", b"hide_message"))
            bot.last_message_id = msg.id
            return

        # Специальные права для ID 262511724 (владение)
        if event.sender_id in [262511724] and role in ['кодер', 'создатель', 'владелец']:
            # Добавляем пользователя если его нет
            if not user_exists:
                db.add_user(user.id, user.username)
            # Обновляем роль
            db.update_role(user.id, role_mapping[role])
            msg = await event.reply(
                f"✅ Роль {role} выдана пользователю [{user.first_name}](tg://user?id={user.id})",
                buttons=Button.inline("↩Скрыть", b"hide_message"))
            bot.last_message_id = msg.id
            return

        # Обычная выдача ролей - проверяем что у пользователя роль 0 (нет в базе)
        if current_role == 0 or not user_exists:
            # Добавляем пользователя если его нет
            if not user_exists:
                db.add_user(user.id, user.username)
            # Обновляем роль
            db.update_role(user.id, role_mapping[role])
            msg = await event.reply(
                f"✅ Роль {role} выдана пользователю [{user.first_name}](tg://user?id={user.id})",
                buttons=Button.inline("↩Скрыть", b"hide_message"))
            bot.last_message_id = msg.id
        else:
            msg = await event.reply("❌ У пользователя уже есть роль.",
                                    buttons=Button.inline("↩Скрыть", b"hide_message"))
            bot.last_message_id = msg.id
    else:  # action == '-'
        # Снятие ролей
        # Снятие роли создателя/владельца
        if current_role == 10 and event.sender_id in [262511724]:
            db.update_role(user.id, 0)
            msg = await event.reply(
                f"✅ Роль снята с пользователя [{user.first_name}](tg://user?id={user.id})",
                buttons=Button.inline("↩Скрыть", b"hide_message"))
            bot.last_message_id = msg.id
            return

        if current_role == 10 and user_role == 10:
            msg = await event.reply("❌ Создатель не может снять роль создателя у другого создателя.",
                                    buttons=Button.inline("↩Скрыть", b"hide_message"))
            bot.last_message_id = msg.id
            return

        if current_role > 0:
            db.update_role(user.id, 0)
            msg = await event.reply(
                f"✅ Роль снята с пользователя [{user.first_name}](tg://user?id={user.id})",
                buttons=Button.inline("↩Скрыть", b"hide_message"))
            bot.last_message_id = msg.id
        else:
            msg = await event.reply("❌ У пользователя нет роли",
                                    buttons=Button.inline("↩Скрыть", b"hide_message"))
            bot.last_message_id = msg.id


@bot.on(events.CallbackQuery(data=b"hide_message"))
async def hide_message_handler(event):
    try:
        await event.delete()
    except Exception as e:
        print(f"Ошибка при удалении сообщения: {e}")


# Права для мута (ограничения)
MUTE_RIGHTS = ChatBannedRights(
    until_date=None,
    send_messages=True,
    send_media=True,
    send_stickers=True,
    send_gifs=True,
    send_games=True,
    send_inline=True,
)

# Права для размута (снятие ограничений)
UNMUTE_RIGHTS = ChatBannedRights(
    until_date=None,
    send_messages=False,
    send_media=False,
    send_stickers=False,
    send_gifs=False,
    send_games=False,
    send_inline=False,
)


async def check_admin(chat, user_id):
    """Проверяет, является ли пользователь администратором"""
    if user_id in ADMINS:
        return True

    try:
        participant = await bot.get_permissions(chat, user_id)
        return participant.is_admin
    except Exception as e:
        logger.error(f"Ошибка проверки прав администратора: {e}")
        return False


async def send_log(action, admin, user, duration, chat, reason=None, message_link=None):
    """Отправляет логи в канал"""
    text = (
        f"**{action.upper()}**\n\n"
        f"👤 **Пользователь:** {user.first_name} (`{user.id}`)\n"
        f"👮 **Администратор:** {admin.first_name} (`{admin.id}`)\n"
        f"💬 **Чат:** {chat.title} (`{chat.id}`)\n"
    )

    if duration:
        text += f"⏳ **Длительность:** {duration}\n"
    if reason:
        text += f"📝 **Причина:** {reason}\n"
    if message_link:
        text += f"🔗 **Сообщение:** [ссылка]({message_link})"

    try:
        await bot.send_message(LOG_CHANNEL, text, link_preview=False)
    except Exception as e:
        logger.error(f"Ошибка отправки лога: {e}")


@bot.on(events.NewMessage(pattern=r'(?i)^(/|\.)?(mute|мут)(@\w+)?\s*(\d+[дмчh])\s*(.*)'))
async def mute_handler(event):
    """Обработчик команды мута"""
    # Получаем имя пользователя, если указано
    username = event.pattern_match.group(3)
    replied = await event.get_reply_message() if not username else None

    if username:
        try:
            user = await bot.get_entity(username)
        except Exception:
            await event.reply("⚠️ **Не удалось найти пользователя**")
            return
    elif replied:
        user = await replied.get_sender()
    else:
        await event.reply("⚠️ **Команда должна быть ответом на сообщение пользователя или содержать @username**")
        return

    if not await check_admin(event.chat_id, event.sender_id):
        await event.reply("⛔️ **У вас недостаточно прав для выполнения данного действия**")
        return

    # Парсим аргументы
    args = event.pattern_match
    time_str = args.group(4).lower()
    reason = args.group(5) or "Не указана"

    # Парсим время
    duration_match = re.match(r"(\d+)([дмчh])", time_str)
    if not duration_match:
        await event.reply(
            "⚠️ **Неверный формат времени**\n\n"
            "**Примеры:**\n"
            "30м - 30 минут\n"
            "2ч - 2 часа\n"
            "1д - 1 день"
        )
        return

    amount = int(duration_match.group(1))
    unit = duration_match.group(2)

    # Определяем длительность
    if unit in ["м", "m"]:
        duration = timedelta(minutes=amount)
        duration_text = f"{amount} минут"
    elif unit in ["ч", "h"]:
        duration = timedelta(hours=amount)
        duration_text = f"{amount} часов"
    elif unit in ["д", "d"]:
        duration = timedelta(days=amount)
        duration_text = f"{amount} дней"
    else:
        await event.reply("⚠️ **Неверная единица времени**")
        return

    try:
        # Применяем мут
        until_date = datetime.now() + duration
        await bot.edit_permissions(
            event.chat_id,
            user.id,
            until_date=until_date,
            send_messages=False,
            send_media=False,
            send_stickers=False,
            send_gifs=False,
            send_games=False,
            send_inline=False
        )

        # Формируем сообщение
        mute_text = (
            f"📛 **Пользователю {user.first_name} (`{user.id}`) был выдан мут на {duration_text}!**"
        )

        # Создание кнопок для ответа
        keyboard = [
            [Button.inline("🕐 Снять мут", f"unmute_{user.id}")],
            [Button.url("👓 Чат для оффтопа", "https://t.me/steal_a_brainrotchat1")],
        ]

        mute_msg = await event.reply(mute_text, buttons=keyboard)

        # Логируем действие
        await send_log(
            "Мут",
            event.sender,
            user,
            duration_text,
            await event.get_chat(),
            reason,
            f"https://t.me/c/{event.chat_id}/{mute_msg.id}"
        )

        # Удаляем сообщение, если команда delmute
        if event.pattern_match.group(2).lower() in ["delmute", "делмут"]:
            await replied.delete()

    except Exception as e:
        await event.reply(f"❌ **Ошибка:** {str(e)}")


@bot.on(events.NewMessage(pattern=r'(?i)^(/|\.)?(unmute|анмут)(@\w+)?'))
async def unmute_handler(event):
    """Обработчик команды размута"""
    # Получаем имя пользователя, если указано
    username = event.pattern_match.group(3)
    replied = await event.get_reply_message() if not username else None

    if username:
        try:
            user = await bot.get_entity(username)
        except Exception:
            await event.reply("⚠️ **Не удалось найти пользователя**")
            return
    elif replied:
        user = await replied.get_sender()
    else:
        await event.reply("⚠️ **Команда должна быть ответом на сообщение пользователя или содержать @username**")
        return

    if not await check_admin(event.chat_id, event.sender_id):
        await event.reply("⛔️ **У вас недостаточно прав для выполнения данного действия**")
        return

    try:
        # Снимаем мут
        await bot.edit_permissions(
            event.chat_id,
            user.id,
            send_messages=True,
            send_media=True,
            send_stickers=True,
            send_gifs=True,
            send_games=True,
            send_inline=True,
        )

        # Формируем сообщение
        unmute_text = (
            f"🔊 **Пользователь размучен!**\n\n"
            f"👤 **Снял мут:** {event.sender.first_name}"
        )

        keyboard = [
            [Button.url("👑 Чат для оффтопа", "https://t.me/steal_a_brainrotchat1")],
        ]

        unmute_msg = await event.reply(unmute_text, buttons=keyboard)

        # руем действие
        await send_log(
            "Размут",
            event.sender,
            user,
            "Досрочно",
            await event.get_chat(),
            message_link=f"https://t.me/c/{event.chat_id}/{unmute_msg.id}"
        )

    except Exception as e:
        await event.reply(f"❌ **Ошибка:** {str(e)}")


@bot.on(events.NewMessage(pattern=r'(?i)^(/|\.)?(ban|бан)(@\w+)?\s*(\d+[дмчh])\s*(.*)'))
async def ban_handler(event):
    """Обработчик команды бана"""
    # Получаем имя пользователя, если указано
    username = event.pattern_match.group(3)
    replied = await event.get_reply_message() if not username else None

    if username:
        try:
            user = await bot.get_entity(username)
        except Exception:
            await event.reply("⚠️ **Не удалось найти пользователя**")
            return
    elif replied:
        user = await replied.get_sender()
    else:
        await event.reply("⚠️ **Команда должна быть ответом на сообщение пользователя или содержать @username**")
        return

    if not await check_admin(event.chat_id, event.sender_id):
        await event.reply("⛔️ **У вас недостаточно прав для выполнения данного действия**")
        return

    # Парсим аргументы
    args = event.pattern_match
    time_str = args.group(4).lower()
    reason = args.group(5) or "Не указана"

    # Парсим время
    duration_match = re.match(r"(\d+)([дмчh])", time_str)
    if not duration_match:
        await event.reply(
            "⚠️ **Неверный формат времени**\n\n"
            "**Примеры:**\n"
            "30м - 30 минут\n"
            "2ч - 2 часа\n"
            "1д - 1 день"
        )
        return

    amount = int(duration_match.group(1))
    unit = duration_match.group(2)

    # Определяем длительность
    if unit in ["м", "m"]:
        duration = timedelta(minutes=amount)
        duration_text = f"{amount} минут"
    elif unit in ["ч", "h"]:
        duration = timedelta(hours=amount)
        duration_text = f"{amount} часов"
    elif unit in ["д", "d"]:
        duration = timedelta(days=amount)
        duration_text = f"{amount} дней"

    try:
        # Баним пользователя
        until_date = datetime.now() + duration
        await bot.edit_permissions(
            event.chat_id,
            user.id,
            until_date=until_date,
            view_messages=False
        )

        # Формируем сообщение
        ban_text = (
            f"📛 **Пользователю был выдан бан!**\n\n"
            f"🧸 **Бан выдан:** {user.first_name} (`{user.id}`)\n"
            f"🔮 **Выдал бан:** {event.sender.first_name}\n"
            f"🕐 **Длительность бана:** {duration_text}\n"
            f"📝 **Причина бана:** {reason}"
        )

        keyboard = [
            [Button.inline("🔓 Снять бан пользователю", f"unban_{user.id}")],
            [Button.url("💭 Чат для оффтопа", "https://t.me/steal_a_brainrotchat1")],
        ]

        ban_msg = await event.reply(ban_text, buttons=keyboard)

        # Логируем действие
        await send_log(
            "Бан",
            event.sender,
            user,
            duration_text,
            await event.get_chat(),
            reason,
            f"https://t.me/c/{event.chat_id}/{ban_msg.id}"
        )

    except Exception as e:
        await event.reply(f"❌ **Ошибка:** {str(e)}")


@bot.on(events.NewMessage(pattern=r'(?i)^(/|\.)?(unban|разбан)(@\w+)?'))
async def unban_handler(event):
    """Обработчик команды разбана"""
    # Получаем имя пользователя, если указано
    username = event.pattern_match.group(3)
    replied = await event.get_reply_message() if not username else None

    if username:
        try:
            user = await bot.get_entity(username)
        except Exception:
            await event.reply("⚠️ **Не удалось найти пользователя**")
            return
    elif replied:
        user = await replied.get_sender()
    else:
        await event.reply("⚠️ **Команда должна быть ответом на сообщение пользователя или содержать @username**")
        return

    if not await check_admin(event.chat_id, event.sender_id):
        await event.reply("⛔️ **У вас недостаточно прав для выполнения данного действия**")
        return

    try:
        # Разбаниваем пользователя
        await bot.edit_permissions(
            event.chat_id,
            user.id,
            view_messages=True
        )

        # Формируем сообщение
        unban_text = (
            f"💫 **Пользователю снят бан!**\n\n"
            f"👨‍💻 **Пользователь:** {user.first_name} (`{user.id}`)\n"
            f"⛱ **Снял бан:** {event.sender.first_name}"
        )

        keyboard = [
            [Button.url("💭 Чат для оффтопа", "https://steal_a_brainrotchat1")],
        ]

        unban_msg = await event.reply(unban_text, buttons=keyboard)

        # Логируем действие
        await send_log(
            "Разбан",
            event.sender,
            user,
            "Досрочно",
            await event.get_chat(),
            message_link=f"https://t.me/c/{event.chat_id}/{unban_msg.id}"
        )

    except Exception as e:
        await event.reply(f"❌ **Ошибка:** {str(e)}")


@bot.on(events.CallbackQuery(pattern=rb'accept_scam_(.+)'))
async def accept_scam_handler(event):
    data = event.data.decode('utf-8')
    unique_id = data.split('_')[2]

    if unique_id not in TEMP_STORAGE:
        await event.answer("❌ Заявка не найдена или уже обработана", alert=True)
        return

    scam_data = TEMP_STORAGE[unique_id]

    # Проверяем, что отвечает куратор
    curator_id = event.sender_id
    trainee_curator = db.get_user_curator(scam_data['user_id'])

    if curator_id != trainee_curator:
        await event.answer("❌ Вы не являетесь куратором этого стажёра", alert=True)
        return

    try:
        # Получаем выбранную роль стажёром
        role_id = scam_data.get('selected_role', 3)  # По умолчанию скамер
        role_name = scam_data.get('selected_role_name', '❌ Скамер')

        # Добавляем пользователя если его нет
        if not db.get_user(scam_data['scammer_id']):
            db.add_user(scam_data['scammer_id'], scam_data['scammer_name'])

        # Обновляем роль
        db.update_role(scam_data['scammer_id'], role_id)

        # Добавляем запись в таблицу scammers
        scammer_added = db.add_scammer(
            scam_data['scammer_id'],
            scam_data['reason'],
            scam_data['user_name'],
            scam_data['reason'],
            unique_id,
            scam_data['proof_link'],
            scam_data['user_id']
        )

        if scammer_added:
            # Увеличиваем счётчик слитых скаммеров у стажёра
            current_count = db.get_user_reported_scammers_count(scam_data['user_id'])
            new_count = current_count + 1
            db.update_user_scammers_count(scam_data['user_id'], new_count)

            # Обновляем статус заявки
            TEMP_STORAGE[unique_id]['status'] = 'accepted'

            # Уведомляем стажёра
            await bot.send_message(
                scam_data['user_id'],
                f"✅ Ваша заявка на скаммера {scam_data['scammer_name']} принята куратором!\n"
                f"🎯 Роль: {role_name}\n"
                f"📎 Доказательства: {scam_data['proof_link']}\n"
                f"📝 Причина: {scam_data['reason']}\n"
                f"🎉 Теперь у вас {new_count} слитых скаммеров!"
            )

            # Редактируем сообщение у куратора
            await event.edit(
                f"✅ **Заявка принята!**\n\n"
                f"👤 Скаммер: {scam_data['scammer_name']}\n"
                f"🎯 Роль: {role_name}\n"
                f"📎 Доказательства: {scam_data['proof_link']}\n"
                f"📝 Причина: {scam_data['reason']}\n"
                f"👨‍🎓 Стажёр: {scam_data['user_name']}\n\n"
                f"📊 У стажёра теперь: {new_count} слитых скаммеров",
                buttons=None
            )

            # Удаляем из временного хранилища
            if unique_id in TEMP_STORAGE:
                del TEMP_STORAGE[unique_id]

            await event.answer("✅ Заявка принята!", alert=True)
        else:
            await event.answer("❌ Ошибка при добавлении скаммера", alert=True)

    except Exception as e:
        logging.error(f"Ошибка при принятии заявки: {e}")
        await event.answer("❌ Произошла ошибка", alert=True)


@bot.on(events.CallbackQuery(pattern=rb'reject_scam_(.+)'))
async def reject_scam_handler(event):
    data = event.data.decode('utf-8')
    unique_id = data.split('_')[2]

    if unique_id not in TEMP_STORAGE:
        await event.answer("❌ Заявка не найдена или уже обработана", alert=True)
        return

    scam_data = TEMP_STORAGE[unique_id]

    # Проверяем, что отвечает куратор
    curator_id = event.sender_id
    trainee_curator = db.get_user_curator(scam_data['user_id'])

    if curator_id != trainee_curator:
        await event.answer("❌ Вы не являетесь куратором этого стажёра", alert=True)
        return

    # Получаем выбранную роль для отображения
    role_name = scam_data.get('selected_role_name', 'Не выбрана')

    # Обновляем статус заявки
    TEMP_STORAGE[unique_id]['status'] = 'rejected'

    # Уведомляем стажёра
    await bot.send_message(
        scam_data['user_id'],
        f"❌ Ваша заявка на скаммера {scam_data['scammer_name']} отклонена куратором.\n"
        f"🎯 Выбранная роль: {role_name}\n"
        f"📎 Доказательства: {scam_data['proof_link']}\n"
        f"📝 Причина заявки: {scam_data['reason']}\n\n"
        f"❌ **Причина отклонения:** недостаточно доказательств или неверная информация."
    )

    # Редактируем сообщение у куратора
    await event.edit(
        f"❌ **Заявка отклонена!**\n\n"
        f"👤 Скаммер: {scam_data['scammer_name']}\n"
        f"🎯 Выбранная роль: {role_name}\n"
        f"📎 Доказательства: {scam_data['proof_link']}\n"
        f"📝 Причина заявки: {scam_data['reason']}\n"
        f"👨‍🎓 Стажёр: {scam_data['user_name']}",
        buttons=None
    )

    # Удаляем из временного хранилища
    if unique_id in TEMP_STORAGE:
        del TEMP_STORAGE[unique_id]

    await event.answer("❌ Заявка отклонена", alert=True)


@bot.on(events.NewMessage(pattern=r'(?i)^(?:/скам|/sc|/scam)'))
async def scam_command(event):
    logging.info("Команда /скам была вызвана.")
    user_id = event.sender_id

    user_role = db.get_user_role(user_id)

    # Проверяем, может ли пользователь заносить
    if user_role not in CAN_ADD_SCAMMER_ROLES:
        await event.respond("❌ Только персонал базы может заносить скамеров!")
        return

    args = event.raw_text.split(maxsplit=2)
    if len(args) < 3:
        await event.respond("❌ Используйте: /скам @username/ID *ссылка_на_доказательства причина*")
        return

    target = args[1]
    full_reason = args[2].strip('*')

    # Разделяем ссылку на доказательства и причину
    parts = full_reason.split(' ', 1)
    if len(parts) >= 2:
        proof_link = parts[0]
        reason = parts[1]
    else:
        proof_link = full_reason
        reason = "Без подробного описания"

    logging.info(f"Пользователь {user_id} (роль: {user_role}) пытается добавить скамера: {target}")

    try:
        scammer_user = None
        # Проверка по ID
        if target.isdigit():
            user_id_to_check = int(target)
            # Пытаемся получить пользователя из Telegram
            try:
                scammer_user = await event.client.get_entity(user_id_to_check)
            except:
                # Если не можем получить из Telegram, создаем фейковый объект
                scammer_user = type('UserObject', (), {
                    'id': user_id_to_check,
                    'first_name': f"ID: {user_id_to_check}"
                })()
        else:
            # Проверка по юзернейму
            if target.startswith('@'):
                target = target[1:]
            scammer_user = await event.client.get_entity(target)

    except Exception as e:
        await event.respond("❌ Не могу найти пользователя")
        logging.error(f"Ошибка при получении пользователя: {e}")
        return

    # Проверяем, является ли целевой пользователь персоналом?
    scammer_role = db.get_user_role(scammer_user.id)

    if scammer_role in STAFF_ROLES:
        scammer_role_name = ROLES.get(scammer_role, {}).get('name', 'Неизвестно')
        user_role_name = ROLES.get(user_role, {}).get('name', 'Неизвестно')

        await event.respond(
            f"🚫 **ЗАПРЕЩЕНО ЗАНОСИТЬ ПЕРСОНАЛ!**\n\n"
            f"👤 **Вы:** {user_role_name}\n"
            f"🎯 **Пытаетесь занести:** {scammer_role_name}\n\n"
            f"❌ Персонал базы Infinity не может быть занесен в базу скамеров!"
        )
        return

    # Получаем имя пользователя
    if hasattr(scammer_user, 'first_name'):
        scammer_name = scammer_user.first_name
    elif hasattr(scammer_user, 'title'):
        scammer_name = scammer_user.title
    else:
        scammer_name = f"ID: {scammer_user.id}"

    # Сохраняем временные данные о заносе
    unique_id = str(uuid.uuid4())
    TEMP_STORAGE[unique_id] = {
        'scammer_id': scammer_user.id,
        'scammer_name': scammer_name,
        'user_id': user_id,
        'user_name': (await event.get_sender()).first_name,
        'user_role': user_role,
        'reason': reason,
        'proof_link': proof_link,
        'status': 'pending',
        'selected_role': None,
        'selected_role_name': None
    }

    # Для стажёров показываем окно выбора роли
    if user_role == 6:  # Стажёр
        buttons = [
            [Button.inline("🐓 Петух", f"select_role_petuh_{unique_id}".encode()),
             Button.inline("❌ Скамер", f"select_role_scammer_{unique_id}".encode())],
            [Button.inline("⚠️ Подозрения на скам", f"select_role_suspect_{unique_id}".encode()),
             Button.inline("⚠️ Возможно скамер", f"select_role_possible_{unique_id}".encode())]
        ]

        await event.respond(
            f"Выберите роль для скамера {scammer_name}:\n\n"
            f"📋 **Проверка:** Целевой пользователь НЕ является персоналом базы ✅\n"
            f"🐓 **Петух** - пользователь с плохой репутацией\n"
            f"❌ **Скамер** - доказанный мошенник\n"
            f"⚠️ **Подозрения на скам** - есть подозрения\n"
            f"⚠️ **Возможно скамер** - потенциальный мошенник\n\n"
            f"📎 **Доказательства:** {proof_link}\n"
            f"📝 **Причина:** {reason}",
            buttons=buttons
        )
    else:
        # Для остального персонала сразу показываем окно выбора роли
        buttons = [
            [Button.inline("🐓 Петух", f"direct_role_petuh_{unique_id}".encode()),
             Button.inline("❌ Скамер", f"direct_role_scammer_{unique_id}".encode())],
            [Button.inline("⚠️ Подозрения на скам", f"direct_role_suspect_{unique_id}".encode()),
             Button.inline("⚠️ Возможно скамер", f"direct_role_possible_{unique_id}".encode())]
        ]

        await event.respond(
            f"Выберите роль для скамера {scammer_name}:\n\n"
            f"📋 **Проверка:** Целевой пользователь НЕ является персоналом базы ✅\n"
            f"🐓 **Петух** - пользователь с плохой репутацией\n"
            f"❌ **Скамер** - доказанный мошенник\n"
            f"⚠️ **Подозрения на скам** - есть подозрения\n"
            f"⚠️ **Возможно скамер** - потенциальный мошенник\n\n"
            f"📎 **Доказательства:** {proof_link}\n"
            f"📝 **Причина:** {reason}",
            buttons=buttons
        )


@bot.on(events.CallbackQuery(pattern=rb'select_role_petuh_(.+)'))
async def select_role_petuh_handler(event):
    await process_trainee_role_selection(event, 4, "🐓 Петух")


@bot.on(events.CallbackQuery(pattern=rb'select_role_scammer_(.+)'))
async def select_role_scammer_handler(event):
    await process_trainee_role_selection(event, 3, "❌ Скамер")


@bot.on(events.CallbackQuery(pattern=rb'select_role_suspect_(.+)'))
async def select_role_suspect_handler(event):
    await process_trainee_role_selection(event, 5, "⚠️ Подозрения на скам")


@bot.on(events.CallbackQuery(pattern=rb'select_role_possible_(.+)'))
async def select_role_possible_handler(event):
    await process_trainee_role_selection(event, 2, "⚠️ Возможно скамер")


async def process_trainee_role_selection(event, role_id, role_name):
    data = event.data.decode('utf-8')
    unique_id = data.split('_')[3]

    if unique_id not in TEMP_STORAGE:
        await event.answer("❌ Заявка не найдена или уже обработана", alert=True)
        return

    scam_data = TEMP_STORAGE[unique_id]

    # Двойная проверка: что нажимает тот же пользователь
    if event.sender_id != scam_data['user_id']:
        await event.answer("❌ Вы не можете нажимать на чужие кнопки", alert=True)
        return

    # ПОВТОРНАЯ ПРОВЕРКА: является ли скамер персоналом
    scammer_role = db.get_user_role(scam_data['scammer_id'])
    if scammer_role in STAFF_ROLES:
        scammer_role_name = ROLES.get(scammer_role, {}).get('name', 'Неизвестно')

        await event.edit(
            f"🚫 **ОШИБКА!**\n\n"
            f"❌ Нельзя заносить персонал базы!\n"
            f"👤 **Целевой пользователь:** {scam_data['scammer_name']}\n"
            f"🎯 **Роль:** {scammer_role_name}\n\n"
            f"ℹ️ Персонал Infinity не может быть занесен в базу скамеров.\n"
            f"Если у вас есть жалобы на персонал, обратитесь к создателю базы.",
            buttons=None
        )
        return

    # Сохраняем выбранную роль
    TEMP_STORAGE[unique_id]['selected_role'] = role_id
    TEMP_STORAGE[unique_id]['selected_role_name'] = role_name

    # Получаем куратора стажёра
    curator_id = db.get_user_curator(scam_data['user_id'])
    if not curator_id:
        await event.edit("❌ У вас нет назначенного куратора!")
        return

    # Формируем сообщение для куратора
    scammer_info = f"""
🚨 **Новая заявка на скаммера от стажёра**

👤 **Скаммер:** {scam_data['scammer_name']} (ID: {scam_data['scammer_id']})
🎯 **Выбранная роль:** {role_name}
📎 **Доказательства:** {scam_data['proof_link']}
📝 **Причина:** {scam_data['reason']}

👨‍🎓 **Стажёр:** {scam_data['user_name']} (ID: {scam_data['user_id']})
⏰ **Время:** {datetime.now().strftime('%d.%m.%Y %H:%M')}

🆔 **ID заявки:** {unique_id}
    """

    buttons = [
        [Button.inline("✅ Принять", f"accept_scam_{unique_id}".encode()),
         Button.inline("❌ Отклонить", f"reject_scam_{unique_id}".encode())]
    ]

    try:
        # Отправляем куратору в ЛС
        await bot.send_message(
            curator_id,
            scammer_info,
            buttons=buttons,
            parse_mode='md'
        )

        # Редактируем сообщение стажёру
        await event.edit(
            f"✅ Заявка на скамера {scam_data['scammer_name']} отправлена вашему куратору.\n"
            f"🎯 Выбранная роль: {role_name}\n"
            f"📎 Доказательства: {scam_data['proof_link']}\n"
            f"📝 Причина: {scam_data['reason']}\n\n"
            f"Ожидайте рассмотрения заявки.",
            buttons=None
        )

    except Exception as e:
        logging.error(f"Ошибка отправки заявки куратору: {e}")
        await event.edit("❌ Не удалось отправить заявку куратору")


@bot.on(events.CallbackQuery(pattern=rb'direct_role_petuh_(.+)'))
async def direct_role_petuh_handler(event):
    data = event.data.decode('utf-8')
    unique_id = data.split('_')[3]

    if unique_id not in TEMP_STORAGE:
        await event.answer("❌ Заявка не найдена или уже обработана", alert=True)
        return

    scam_data = TEMP_STORAGE[unique_id]

    # ЗАЩИТА: проверяем, что нажимает тот же пользователь
    if event.sender_id != scam_data['user_id']:
        await event.answer("❌ Вы не можете нажимать на чужие кнопки", alert=True)
        return

    await process_direct_role_selection(event, unique_id, 4, "🐓 Петух")


@bot.on(events.CallbackQuery(pattern=rb'direct_role_scammer_(.+)'))
async def direct_role_scammer_handler(event):
    data = event.data.decode('utf-8')
    unique_id = data.split('_')[3]

    if unique_id not in TEMP_STORAGE:
        await event.answer("❌ Заявка не найдена или уже обработана", alert=True)
        return

    scam_data = TEMP_STORAGE[unique_id]

    # ЗАЩИТА: проверяем, что нажимает тот же пользователь
    if event.sender_id != scam_data['user_id']:
        await event.answer("❌ Вы не можете нажимать на чужие кнопки", alert=True)
        return

    await process_direct_role_selection(event, unique_id, 3, "❌ Скамер")


@bot.on(events.CallbackQuery(pattern=rb'direct_role_suspect_(.+)'))
async def direct_role_suspect_handler(event):
    data = event.data.decode('utf-8')
    unique_id = data.split('_')[3]

    if unique_id not in TEMP_STORAGE:
        await event.answer("❌ Заявка не найдена или уже обработана", alert=True)
        return

    scam_data = TEMP_STORAGE[unique_id]

    # ЗАЩИТА: проверяем, что нажимает тот же пользователь
    if event.sender_id != scam_data['user_id']:
        await event.answer("❌ Вы не можете нажимать на чужие кнопки", alert=True)
        return

    await process_direct_role_selection(event, unique_id, 5, "⚠️ Подозрения на скам")


@bot.on(events.CallbackQuery(pattern=rb'direct_role_possible_(.+)'))
async def direct_role_possible_handler(event):
    data = event.data.decode('utf-8')
    unique_id = data.split('_')[3]

    if unique_id not in TEMP_STORAGE:
        await event.answer("❌ Заявка не найдена или уже обработана", alert=True)
        return

    scam_data = TEMP_STORAGE[unique_id]

    # ЗАЩИТА: проверяем, что нажимает тот же пользователь
    if event.sender_id != scam_data['user_id']:
        await event.answer("❌ Вы не можете нажимать на чужие кнопки", alert=True)
        return

    await process_direct_role_selection(event, unique_id, 2, "⚠️ Возможно скамер")


async def process_direct_role_selection(event, unique_id, role_id, role_name):
    scam_data = TEMP_STORAGE[unique_id]

    # Проверка, что нажимает тот же пользователь
    if event.sender_id != scam_data['user_id']:
        await event.answer("❌ Вы не можете нажимать на чужие кнопки", alert=True)
        return

    # ПРОВЕРКА: является ли скамер персоналом
    scammer_role = db.get_user_role(scam_data['scammer_id'])
    if scammer_role in STAFF_ROLES:
        scammer_role_name = ROLES.get(scammer_role, {}).get('name', 'Неизвестно')
        user_role_name = ROLES.get(scam_data['user_role'], {}).get('name', 'Неизвестно')

        await event.edit(
            f"🚫 **ЗАПРЕЩЕНО!**\n\n"
            f"👤 **Вы:** {user_role_name}\n"
            f"🎯 **Цель:** {scammer_role_name}\n\n"
            f"❌ Нельзя заносить персонал базы Infinity!\n"
            f"📋 Защищенные роли:\n"
            f"• Гаранты (1) 🛡️\n• Стажёры (6) 🎓\n• Админы (7) 👮\n"
            f"• Директора (8) 👔\n• Президенты (9) 👑\n• Владельцы (10) ⭐\n"
            f"• Кодеры (11) 💻\n• Проверенные (12) ✅\n• Айдош (13) ⭐\n\n"
            f"ℹ️ Если у вас есть жалобы на персонал, обратитесь к создателю базы.",
            buttons=None
        )
        return

    try:
        # Добавляем пользователя если его нет
        if not db.get_user(scam_data['scammer_id']):
            db.add_user(scam_data['scammer_id'], scam_data['scammer_name'])

        # Обновляем роль
        db.update_role(scam_data['scammer_id'], role_id)

        # Добавляем запись в таблицу scammers
        scammer_added = db.add_scammer(
            scam_data['scammer_id'],
            scam_data['reason'],
            scam_data['user_name'],
            scam_data['reason'],
            unique_id,
            scam_data['proof_link'],
            scam_data['user_id']
        )

        if scammer_added:
            # Увеличиваем счётчик слитых скамеров у пользователя
            if scam_data['user_role'] in STAFF_ROLES:
                current_count = db.get_user_reported_scammers_count(scam_data['user_id'])
                new_count = current_count + 1
                db.update_user_scammers_count(scam_data['user_id'], new_count)

                # Уведомляем пользователя
                await bot.send_message(
                    scam_data['user_id'],
                    f"✅ Скаммер {scam_data['scammer_name']} успешно занесён!\n"
                    f"🎯 Роль: {role_name}\n"
                    f"📎 Доказательства: {scam_data['proof_link']}\n"
                    f"📝 Причина: {scam_data['reason']}\n"
                    f"🎉 Теперь у вас {new_count} слитых скаммеров!"
                )
            else:
                await bot.send_message(
                    scam_data['user_id'],
                    f"✅ Скаммер {scam_data['scammer_name']} успешно занесён!\n"
                    f"🎯 Роль: {role_name}\n"
                    f"📎 Доказательства: {scam_data['proof_link']}\n"
                    f"📝 Причина: {scam_data['reason']}"
                )

            # Редактируем сообщение с выбором роли
            await event.edit(
                f"✅ **Скаммер занесён!**\n\n"
                f"👤 Скаммер: {scam_data['scammer_name']}\n"
                f"🎯 Роль: {role_name}\n"
                f"📎 Доказательства: {scam_data['proof_link']}\n"
                f"📝 Причина: {scam_data['reason']}",
                buttons=None
            )

            # Удаляем из временного хранилища
            if unique_id in TEMP_STORAGE:
                del TEMP_STORAGE[unique_id]

        else:
            await event.answer("❌ Ошибка при добавлении скаммера в базу", alert=True)

    except Exception as e:
        logging.error(f"Ошибка при добавлении скаммера: {e}")
        await event.answer("❌ Произошла ошибка при заносе", alert=True)


@bot.on(events.CallbackQuery)
async def callback_handler(event):
    """Глобальный обработчик callback с защитой от спама"""
    user_id = event.sender_id
    current_time = time.time()

    # Проверка на спам кнопками
    if user_id in last_button_click:
        elapsed_time = current_time - last_button_click[user_id]
        if elapsed_time < button_cooldown:
            # Показываем всплывающее уведомление
            await event.answer(f"⏳ Подождите {button_cooldown - elapsed_time:.1f} секунд", alert=False)
            return

    # Обновляем время последнего клика
    last_button_click[user_id] = current_time

    # Обработка конкретных callback
    try:
        # Проверяем, какой callback пришел
        data = event.data.decode('utf-8') if isinstance(event.data, bytes) else event.data

        if data.startswith('rate_user_'):
            await rate_user_handler(event)
        elif data.startswith('vote_'):
            await vote_handler(event)
        elif data.startswith('appeal_'):
            await appeal_handler(event)
        elif data.startswith('sliv_scammers_'):
            await sliv_scammers_handler(event)
        elif data.startswith('remove_from_db_'):
            await remove_from_db_handler(event)
        elif data == 'hide_message':
            await hide_message_handler(event)
        elif data == 'top_trainees':
            await top_trainees_handler(event)
        elif data == 'top_day':
            await top_day_handler(event)
        elif data == 'return_to_stats':
            await return_to_stats_handler(event)
        # Добавьте другие обработчики...

    except Exception as e:
        logging.error(f"Ошибка в callback_handler: {e}")

# Удаление сообщения с кнопками после нажатия
@bot.on(events.CallbackQuery)
async def callback_handler(callback_event):
    # Удаляем сообщение с кнопками
    await callback_event.delete()


@bot.on(events.NewMessage(pattern="👑 Админ панель"))
async def admin_panel(event):
    if not event.is_private:
        return

    user_id = event.sender_id

    # Проверка прав - ТОЛЬКО OWNER_ID
    if user_id not in OWNER_ID:
        await event.respond("❌ У вас нет доступа к админ-панели!")
        return

    # Показываем загрузку
    await show_button_loading(event, "админ панель")

    # Получаем статистику
    sponsors_count = db.cursor.execute('SELECT COUNT(*) FROM sponsors').fetchone()[0] or 0
    tasks_count = db.cursor.execute('SELECT COUNT(*) FROM daily_tasks WHERE is_active = 1').fetchone()[0] or 0

    text = f"""👑 **АДМИН-ПАНЕЛЬ INFINITY**
━━━━━━━━━━━━━━━━━━━━
[⠀](https://i.ibb.co/ycyPRXrb/photo-2025-04-17-17-44-20-2.jpg)

👤 **Владелец:** {user_id}

📊 **ТЕКУЩАЯ СТАТИСТИКА:**
• Обязательных спонсоров: {sponsors_count}
• Активных заданий: {tasks_count}

⚙️ **ДОСТУПНЫЕ ФУНКЦИИ:**

1️⃣ **УПРАВЛЕНИЕ СПОНСОРАМИ**
   • Добавить обязательную подписку
   • Удалить спонсора
   • Список всех спонсоров

2️⃣ **УПРАВЛЕНИЕ ЗАДАНИЯМИ**
   • Создать ежедневное задание
   • Удалить задание
   • Список активных заданий

━━━━━━━━━━━━━━━━━━━━
Выберите раздел:
"""

    admin_buttons = [
        [Button.inline("📢 Управление спонсорами", "admin_sponsors")],
        [Button.inline("📋 Управление заданиями", "admin_tasks")],
        [Button.inline("📊 Полная статистика", "admin_stats")],
        [Button.inline("↩ Назад", "back_to_main")]
    ]

    # Удаляем сообщение о загрузке
    if user_id in button_loading_messages:
        try:
            await bot.delete_messages(event.chat_id, button_loading_messages[user_id])
            del button_loading_messages[user_id]
        except:
            pass

    await event.respond(text, buttons=admin_buttons, parse_mode='md', link_preview=True)


@bot.on(events.CallbackQuery(pattern="admin_sponsors"))
async def admin_sponsors(event):
    user_id = event.sender_id

    # Проверка прав - ТОЛЬКО OWNER_ID
    if user_id not in OWNER_ID:
        await event.answer("❌ У вас нет доступа!", alert=True)
        return

    # Получаем список спонсоров
    sponsors = db.get_sponsors()

    text = """📢 **УПРАВЛЕНИЕ СПОНСОРАМИ**
━━━━━━━━━━━━━━━━━━━━

❓ **Что такое обязательная подписка?**
Пользователи должны быть подписаны на указанные каналы/чаты, чтобы пользоваться ботом.

📝 **Как добавить спонсора:**
Отправь сообщение в формате:
`ID_канала | Название | ссылка`

Пример:
`-100123456789 | МойКанал | https://t.me/mychannel`

📋 **Текущие спонсоры:**
"""

    if sponsors:
        for sponsor in sponsors:
            sponsor_id, chat_id, chat_title, chat_link, added_by, added_date, is_mandatory = sponsor
            text += f"• [{sponsor_id}] {chat_title} - {chat_link or 'Нет ссылки'}\n"
    else:
        text += "• Список пуст\n"

    text += "\n━━━━━━━━━━━━━━━━━━━━"

    buttons = [
        [Button.inline("➕ Добавить спонсора", "add_sponsor")],
        [Button.inline("❌ Удалить спонсора", "remove_sponsor")],
        [Button.inline("◀ Назад в админ-панель", "admin_panel_back")]
    ]

    # Отправляем новое сообщение
    await event.respond(text, buttons=buttons)
    await event.answer()

@bot.on(events.NewMessage)
async def handle_admin_input(event):
    if not event.is_private:
        return

    user_id = event.sender_id

    # Проверяем, что пользователь в админ-панели
    if user_id not in user_states:
        return

    state = user_states[user_id].get("state")

    if state == "waiting_sponsor_data":
        await handle_sponsor_input(event)
    elif state == "waiting_task_data":
        await handle_task_input_admin(event)


async def handle_sponsor_input(event):
    user_id = event.sender_id
    text = event.raw_text.strip()

    if text.lower() == "отмена":
        del user_states[user_id]
        await event.respond("❌ Добавление отменено")
        # Возвращаемся к спонсорам
        fake_event = type('FakeEvent', (), {'sender_id': user_id, 'edit': lambda x: None})()
        await admin_sponsors(fake_event)
        return

    # Парсим данные
    try:
        parts = text.split('|')
        if len(parts) != 3:
            await event.respond("❌ Неверный формат! Используй: ID | Название | ссылка")
            return

        chat_id = int(parts[0].strip())
        chat_title = parts[1].strip()
        chat_link = parts[2].strip()

        # Добавляем спонсора
        success = db.add_sponsor(chat_id, chat_title, chat_link, user_id)

        if success:
            await event.respond(f"✅ Спонсор '{chat_title}' успешно добавлен!")
            # Возвращаемся к спонсорам
            fake_event = type('FakeEvent', (), {'sender_id': user_id, 'edit': lambda x: None})()
            await admin_sponsors(fake_event)
        else:
            await event.respond("❌ Ошибка при добавлении спонсора")

    except ValueError:
        await event.respond("❌ ID канала должен быть числом!")
    except Exception as e:
        await event.respond(f"❌ Ошибка: {str(e)}")

    # Очищаем состояние
    if user_id in user_states:
        del user_states[user_id]


async def handle_task_input_admin(event):
    user_id = event.sender_id
    text = event.raw_text.strip()

    if text.lower() == "отмена":
        del user_states[user_id]
        await event.respond("❌ Добавление отменено")
        fake_event = type('FakeEvent', (), {'sender_id': user_id, 'edit': lambda x: None})()
        await admin_tasks(fake_event)
        return

    # Парсим данные
    try:
        parts = text.split('|')
        if len(parts) != 3:
            await event.respond("❌ Неверный формат! Используй: Название | Описание | Награда")
            return

        task_name = parts[0].strip()
        task_desc = parts[1].strip()
        reward = int(parts[2].strip())

        # Добавляем задание
        task_id = db.add_daily_task(task_name, task_desc, reward, user_id)

        if task_id:
            await event.respond(f"✅ Задание '{task_name}' успешно добавлено! ID: {task_id}")
            fake_event = type('FakeEvent', (), {'sender_id': user_id, 'edit': lambda x: None})()
            await admin_tasks(fake_event)
        else:
            await event.respond("❌ Ошибка при добавлении задания")

    except ValueError:
        await event.respond("❌ Награда должна быть числом!")
    except Exception as e:
        await event.respond(f"❌ Ошибка: {str(e)}")

    # Очищаем состояние
    if user_id in user_states:
        del user_states[user_id]



@bot.on(events.CallbackQuery(pattern=r'mark_(scammer|possible|suspect|rooster)_(\d+)_(.+)'))
async def mark_user_handler(event):
    logging.info(f"Обработчик вызван с данными: {event.pattern_match.groups()}")

    role_mapping = {
        'scammer': 3,
        'possible': 2,
        'suspect': 5,
        'rooster': 4
    }

    role_type = event.pattern_match.group(1)
    user_id = int(event.pattern_match.group(2))
    reason = event.pattern_match.group(3).strip()

    logging.info(f"Попытка изменить роль пользователя {user_id} на {role_type} с причиной: {reason}")

    # ПРОВЕРКА: является ли целевой пользователь персоналом?
    target_role = db.get_user_role(user_id)
    if target_role in STAFF_ROLES:
        target_role_name = ROLES.get(target_role, {}).get('name', 'Неизвестно')

        await event.answer(
            f"🚫 Нельзя заносить персонал базы!\n"
            f"👤 Роль: {target_role_name}\n"
            f"ℹ️ Персонал Infinity защищен от заноса.",
            alert=True
        )
        return

    # ДОПОЛНИТЕЛЬНАЯ ПРОВЕРКА: Уже ли пользователь имеет роль скаммера
    current_role = db.get_user_role(user_id)
    if current_role in [2, 3, 4, 5]:  # Если уже имеет роль скаммера/подозреваемого
        await event.answer("❌ Этот пользователь уже находится в базе!", alert=True)
        return

    # Получаем роль отправителя
    user_role = db.get_user_role(event.sender_id)
    logging.info(f"Роль пользователя {event.sender_id}: {user_role}")

    # Проверка прав для изменения роли
    if user_role not in [1, 6, 8, 10, 11, 9] and event.sender_id != OWNER_ID:
        await event.answer("⛔ У вас нет прав лол.", alert=True)
        return

    if not reason:
        await event.answer("❌ Причина не может быть пустой!", alert=True)
        return

    # Обновление роли в базе данных
    db.update_role(user_id, role_mapping[role_type])

    # Увеличиваем количество слитых скаммеров для пользователя, который инициировал команду
    current_count = db.get_user_reported_scammers_count(event.sender_id)
    logging.info(f"Текущее количество слитых скаммеров для пользователя {event.sender_id}: {current_count}")

    scammers_slept = current_count + 1
    logging.info(f"Пользователь {event.sender_id} теперь должен иметь {scammers_slept} слитых скаммеров.")

    if not db.update_user_scammers_slept(event.sender_id, scammers_slept):
        logging.error(f"Не удалось обновить количество слитых скаммеров для пользователя {event.sender_id}.")
        await event.answer("Ошибка при обновлении количества слитых скаммеров.", alert=True)
        return

    logging.info(
        f"Количество слитых скаммеров для пользователя {event.sender_id} успешно обновлено на {scammers_slept}.")

    chat_id = event.chat_id
    await event.client.send_message(
        chat_id,
        message=f"🔥 Вы успешно занесли скаммера! | Скаммеров слито: {scammers_slept}"
    )


@bot.on(events.CallbackQuery(pattern=r'remove_from_db_(\d+)_(.+)'))
async def remove_from_db_handler(event):
    user_id = int(event.pattern_match.group(1))
    role_type = event.pattern_match.group(2).strip()

    logging.info(f"Попытка удалить пользователя ID: {user_id} с ролью: {role_type}")

    user_role = db.get_user_role(event.sender_id)

    # Проверка прав для удаления
    if user_role in [0, 1, 2, 3, 4, 5, 6, 7, 12]:
        await event.answer("❌ Действие не допустимо, у вас нет прав!", alert=True)
        return

    # Проверка, был ли пользователь уже удален
    if not db.is_scammer(user_id):  # Предположим, что функция `is_scammer` возвращает False, если пользователь удален
        await event.answer("ℹ️ Вы уже вынесли пользователя из базы, действие отменено!", alert=True)
        return

    # Снимаем статус скаммера
    if db.remove_scammer_status(user_id):
        await event.answer("✅ Статус был успешно снят!", alert=True)
        await event.client.send_message(event.sender_id, "✅ Статус был успешно снят!")
    else:
        await event.answer("❌ Не удалось снять статус, попробуйте позже.", alert=True)


@bot.on(events.CallbackQuery(pattern=rb'sliv_scammers_(\d+)'))
async def sliv_scammers_handler(event):
    sender_id = event.sender_id

    try:
        # Сначала показываем окошко что инструкция отправлена в ЛС
        await event.answer("📨 Инструкции отправлены вам в личные сообщения", alert=True)

        # Ждём немного перед отправкой в ЛС
        await asyncio.sleep(0.5)

        # Отправляем ОДНО сообщение с инструкцией и кнопкой
        instruction_text = (
            f"чтобы слить скаммера, вам надо зайти в чат-слива скаммеров👇.\n\n"
            "просто скиньте в чат следующие данные:\n"
            "- все доказательства (фото, видео, сообщения)\n"
            "- айди или юзернейм скамера\n\n"
            "если вы сделали все это, и наши волонтеры занесут скаммера, то вам выдадут +спасибо"
        )

        keyboard = [[Button.url("предложка🔍", "https://t.me/Infinityantiscam")]]

        # Отправляем одно сообщение с инструкцией и кнопкой
        await bot.send_message(
            sender_id,
            instruction_text,
            buttons=keyboard
        )

    except Exception as e:
        await event.answer("❌ Не удалось отправить сообщение. Пожалуйста, запустите бота", alert=True)
        logging.error(f"Ошибка отправки сообщения в ЛС: {e}")


@bot.on(events.NewMessage(pattern=r'(?i)^(/|\.)?(add) (@\w+) (.+)'))
async def add_reason_handler(event):
    """Обработчик для обновления описания заноса"""
    # Получаем роль отправителя
    user_role = db.get_user_role(event.sender_id)

    # Проверяем права (только админы/владельцы)
    if event.sender_id not in OWNER_ID and user_role not in [6, 8, 10, 11]:
        await event.reply("❌ Только администраторы могут использовать эту команду")
        return

    # Получаем целевого пользователя и текст нового описания
    target_username = event.pattern_match.group(3)  # Юзернейм
    new_description = event.pattern_match.group(4).strip()  # Новое описание

    # Проверка формата юзернейма
    if not target_username.startswith('@'):
        await event.reply("❌ Юзернейм должен начинаться с '@'.")
        return

    # Проверка, что новое описание не пустое
    if not new_description:
        await event.reply("❌ Описание не может быть пустым.")
        return

    # Логируем информацию о попытке получения пользователя
    logging.info(f"Попытка получить пользователя по юзернейму: {target_username}")

    try:
        target = await event.client.get_entity(target_username)
        logging.info(f"Пользователь найден: {target.first_name} (ID: {target.id})")
    except Exception as e:
        logging.error(f"Ошибка при получении пользователя {target_username}: {str(e)}")
        await event.reply("❌ Не могу найти указанного пользователя. Проверьте правильность юзернейма.")
        return

    # Логируем информацию о целевом пользователе и новом описании
    logging.info(
        f"Попытка обновления описания для пользователя {target.first_name} ({target.id}) с новым описанием: {new_description}")

    # Обновляем описание в базе данных
    try:
        # Обновляем описание (предполагается, что эта функция обновляет поле description)
        db.update_description(target.id, new_description)  # Убедитесь, что эта функция обновляет поле description
        logging.info(
            f"Описание для пользователя {target.first_name} ({target.id}) успешно обновлено на: {new_description}")
        await event.reply(
            f"✅ Описание для пользователя [{target.first_name}](tg://user/{target.id}) обновлено: {new_description}",
            parse_mode='md')
    except Exception as e:
        logging.error(f"Ошибка при обновлении описания для пользователя {target.id}: {str(e)}")
        await event.reply(f"❌ Произошла ошибка при обновлении описания: {str(e)}")


@bot.on(events.NewMessage(pattern=r'(?i)^(/add1) (@\w+) (.+)'))
async def add_additional_reason_handler(event):
    """Обработчик для добавления дополнительного описания"""
    # Получаем роль отправителя
    user_role = db.get_user_role(event.sender_id)

    # Проверяем права (только админы/владельцы)
    if event.sender_id not in OWNER_ID and user_role not in [6, 8, 10, 11]:
        await event.reply("❌ Только администраторы могут использовать эту команду")
        return

    # Получаем целевого пользователя и текст дополнительного описания
    target_username = event.pattern_match.group(2)
    additional_reason_text = event.pattern_match.group(3).strip()

    try:
        target = await event.client.get_entity(target_username)
    except Exception as e:
        await event.reply("❌ Не могу найти указанного пользователя")
        return

    # Добавляем дополнительное описание в базе данных
    db.add_additional_reason(target.id, additional_reason_text)

    await event.reply(
        f"✅ Дополнительное описание для пользователя [{target.first_name}](tg://user/{target.id}) добавлено: {additional_reason_text}",
        parse_mode='md')


@bot.on(events.NewMessage(pattern=r'/траст|!trust'))
async def trust_command(event):
    sender = await event.get_sender()

    # Проверка роли: разрешено гарантам и создателям
    if db.get_user_role(sender.id) not in [1, 10]:
        await event.reply(
            "**⚠️ Отказано в доступе!**\n\n"
            f"**👤 Пользователь:** [{sender.first_name}](tg://user/{sender.id})\n"
            "**📛 Причина:** Недостаточно прав\n"
            "**ℹ️ Информация:** Выдавать траст могут только гаранты и создатель\n"
            "[⠀](https://i.ibb.co/rGBBGyng/photo-2025-04-17-17-44-20.jpg)",
            parse_mode='md',
            link_preview=True
        )
        return

    # Получаем целевого пользователя
    target = await get_target_user(event)
    if not target:
        return

    # Получаем ID гаранта (того, кто выдает роль)
    granted_by_username = sender.username if sender.username else f"ID: {sender.id}"

    # Проверяем, является ли целевой пользователь владельцем, кодером, стажером, гарантом, президентом, админом или директором
    target_role = db.get_user_role(target.id)
    if target_role in [6, 7, 8, 9, 10, 11, 12]:
        await event.reply(
            "**❌ Ошибка!**\n\n"
            "**📛 Причина:** Нельзя выдавать траст владельцу, кодеру, стажеру, гаранту, президенту, админу или директору.\n"
            f"**📝 Текущая роль:** {ROLES.get(target_role, {}).get('name', 'Неизвестно')}"
        )
        return

    try:
        # Проверяем, есть ли пользователь в базе
        if not db.get_user(target.id):
            # Добавляем пользователя если его нет
            db.add_user(target.id, target.username if hasattr(target, 'username') else target.first_name)

        # Устанавливаем роль "Проверен гарантом" (роль 12)
        success = db.update_role(target.id, 12, granted_by_id=sender.id)

        if not success:
            await event.reply("❌ Ошибка при обновлении роли в базе данных")
            return

        # Сохраняем информацию о трасте в таблицу trust
        db.add_grant(target.id, sender.id)

        # Отправляем сообщение об успешной выдаче траста
        await event.reply(
            f"**✅ Траст успешно выдан!**\n\n"
            f"**👤 Получатель:** [{target.first_name if hasattr(target, 'first_name') else target.title}](tg://user/{target.id})\n"
            f"**👮 Выдал:** [{sender.first_name}](tg://user/{sender.id})\n"
            f"💙 Репутация: Проверен(а) гарантом {granted_by_username} ✅",
            parse_mode='md'
        )

        # Логируем
        logging.info(f"Траст выдан: {target.id} -> {sender.id}")

    except Exception as e:
        await event.reply(f"**❌ Ошибка:** `{str(e)}`")
        logging.error(f"Ошибка выдачи траста: {e}")


async def get_target_user(event):
    if event.is_reply:
        replied = await event.get_reply_message()
        return await event.client.get_entity(replied.sender_id)
    else:
        args = event.raw_text.split()
        if len(args) < 2:
            await event.reply(
                "**❌ Ошибка использования команды!**\n\n"
                "**✏️ Правильное использование:**\n"
                "• `/trust` (ответом на сообщение)\n"
                "• `/trust @username`\n"
                "• `/trust ID`",
                parse_mode='md'
            )
            return None
        try:
            return await event.client.get_entity(args[1])
        except:
            await event.reply(
                "**❌ Ошибка!**\n\n"
                "**📛 Причина:** Не удалось найти пользователя\n"
                "**💡 Совет:** Проверьте правильность указанного юзернейма/ID",
                parse_mode='md'
            )
            return None


@bot.on(events.NewMessage(pattern=r'/untrust|/антраст|-антраст'))
async def untrust_command(event):
    sender = await event.get_sender()

    # Проверяем права (гарант, создатель, владельцы или кодер)
    sender_role = db.get_user_role(sender.id)
    if db.get_user_role(sender.id) not in [1, 10]:
        await event.reply(
            "**⚠️ Отказано в доступе!**\n\n"
            f"**👤 Пользователь:** [{sender.first_name}](tg://user/{sender.id})\n"
            "**📛 Причина:** Недостаточно прав\n"
            "**ℹ️ Информация:** Снимать траст могут только гаранты, создатель и владельцы\n"
            "[⠀](https://i.ibb.co/rGBBGyng/photo-2025-04-17-17-44-20.jpg)",
            parse_mode='md',
            link_preview=True
        )
        return

    # Получаем целевого пользователя
    if event.is_reply:
        replied = await event.get_reply_message()
        target = await event.client.get_entity(replied.sender_id)
    else:
        args = event.raw_text.split()
        if len(args) < 2:
            await event.reply(
                "**❌ Ошибка использования команды!**\n\n"
                "**✏️ Правильное использование:**\n"
                "• `/untrust` (ответом на сообщение)\n"
                "• `/untrust @username`\n"
                "• `/untrust ID`",
                parse_mode='md'
            )
            return

        try:
            target = await event.client.get_entity(args[1])
        except:
            await event.reply(
                "**❌ Ошибка!**\n\n"
                "**📛 Причина:** Не удалось найти пользователя\n"
                "**💡 Совет:** Проверьте правильность указанного юзернейма/ID",
                parse_mode='md'
            )
            return

    # Проверяем, есть ли у пользователя траст
    if db.get_user_role(target.id) != 12:
        await event.reply(
            "**❌ Ошибка!**\n\n"
            "**📛 Причина:** У пользователя нет траста",
            parse_mode='md'
        )
        return

    # Снимаем траст (устанавливаем стандартную роль)
    db.update_role(target.id, 0)

    await event.reply(
        "**✅ Траст успешно снят!**\n\n"
        f"**👤 Пользователь:** [{target.first_name}](tg://user/{target.id})\n"
        f"**👮 Снял:** [{sender.first_name}](tg://user/{sender.id})",
        parse_mode='md'
    )


# Обработчик команды /untrust
@bot.on(events.NewMessage(pattern=r'/untrust|/антраст|-антраст'))
async def untrust_command(event):
    sender = await event.get_sender()

    # Проверяем права (гарант, создатель, владельцы или кодер)
    sender_role = db.get_user_role(sender.id)
    if sender_role != 1 and sender.id not in OWNER_ID and sender_role not in [10, 11]:
        await event.reply(
            "**⚠️ Отказано!**\n\n"
            f"**👤 Пользователь:** [{sender.first_name}](tg://user/{sender.id})\n"
            "**📛 Причина:** У тя прав нету пон?\n"
            "**ℹ️ Информация:** Снимать траст могут только гаранты, создатель и владельцы\n"
            "[⠀](https://i.ibb.co/rGBBGyng/photo-2025-04-17-17-44-20.jpg)",
            parse_mode='md',
            link_preview=True
        )
        return

    # Получаем целевого пользователя
    if event.is_reply:
        replied = await event.get_reply_message()
        target = await event.client.get_entity(replied.sender_id)
    else:
        args = event.raw_text.split()
        if len(args) < 2:
            await event.reply(
                "**❌ Ошибка использования команды!**\n\n"
                "**✏️ Правильное использование:**\n"
                "• `/untrust` (ответом на сообщение)\n"
                "• `/untrust @username`\n"
                "• `/untrust ID`",
                parse_mode='md'
            )
            return

        try:
            target = await event.client.get_entity(args[1])
        except:
            await event.reply(
                "**❌ ну, ошибочка вышла):**\n\n"
                "**📛 Причина:** Не удалось найти пользователя\n"
                "**💡 Совет:** Дебик, правильно ник введи или айди, заебали уже честно.",
                parse_mode='md'
            )
            return

    # Проверяем, есть ли у пользователя траст
    if db.get_user_role(target.id) != 12:
        await event.reply(
            "**❌ Ну не плач только ошибочка получилась**\n\n"
            "**📛 Причина:** Его нет в базе даун..",
            parse_mode='md'
        )
        return

    # Снимаем траст (устанавливаем стандартную роль)
    await db.update_role(target.id, 0)

    await event.reply(
        "**✅ Траст успешно снят!, плаки плаки ):**\n\n"
        f"**👤 Пользователь:** [{target.first_name}](tg://user/{target.id})\n"
        f"**👮 Снял:** [{sender.first_name}](tg://user/{sender.id})",
        parse_mode='md'
    )


# Команда /гаранты
@bot.on(events.NewMessage(pattern='/гаранты'))
async def list_online_garants(event):
    await event.respond("Ищу онлайн гарантов...")

    # Получаем всех гарантов из базы (роль 1)
    try:
        garants = [row[0] for row in db.cursor.execute('SELECT user_id FROM users WHERE role_id = 1')]
        logging.info(f"Найдено {len(garants)} гарантов с ролью 1.")
    except Exception as e:
        logging.error(f"Ошибка при получении гарантов из базы: {e}")
        await event.respond("Не удалось получить список гарантов из базы данных.")
        return

    online_garants = []

    for uid in garants:
        try:
            user = await bot.get_entity(uid)
            logging.info(f"Проверяем пользователя: {user.id}, Имя: {user.first_name}, Статус: {user.status}")

            # Проверяем статус пользователя
            if user.status is None:
                online_garants.append(user)
                logging.info(f"Пользователь {user.first_name} ({user.id}) онлайн (нет статуса).")
            elif user.status == "online":
                online_garants.append(user)
                logging.info(f"Пользователь {user.first_name} ({user.id}) онлайн.")
            elif isinstance(user.status, UserStatusRecently):
                # Учитываем пользователей с состоянием Recently как онлайн
                online_garants.append(user)
                logging.info(f"Пользователь {user.first_name} ({user.id}) был в сети недавно.")
            else:
                logging.info(f"Пользователь {user.first_name} ({user.id}) не онлайн. Статус: {user.status}")
        except Exception as e:
            logging.error(f"Ошибка при получении пользователя {uid}: {e}")
            continue

    if not online_garants:
        await event.respond("На данный момент онлайн гарантов нет ⛔")
        logging.info("Нет онлайн гарантов.")
        return

    # Формируем текст ответа
    text = "📊 Вот наш список онлайн гарантов:\n"
    buttons = []

    for user in online_garants:
        buttons.append([Button.inline(f"🛡️ {user.first_name}", f"check_{user.id}")])

    await event.respond(text, buttons=buttons, parse_mode='markdown', link_preview=True)
    logging.info("Список онлайн гарантов успешно отправлен.")


@bot.on(events.CallbackQuery(data=b"top_admins"))
async def top_admins_handler(event):
    """Топ админов по слитым скамерам"""
    try:
        await bot.delete_messages(event.chat_id, event.message_id)
    except:
        pass

    # Получаем топ админов
    top_admins = db.cursor.execute('''
        SELECT u.user_id, u.username, u.scammers_count, u.role_id,
               (SELECT COUNT(*) FROM scammers WHERE reporter_id = u.user_id) as real_scammers
        FROM users u 
        WHERE u.role_id IN (7, 8, 9, 10, 11, 13) 
        AND COALESCE(scammers_count, 0) > 0
        ORDER BY COALESCE(scammers_count, 0) DESC 
        LIMIT 10
    ''').fetchall()

    if not top_admins:
        text = """👮 **ТОП АДМИНИСТРАЦИИ**
━━━━━━━━━━━━━━━━━━━━
[⠀](https://i.ibb.co/SKysgbvC/photo-2025-04-17-17-44-20-3.jpg)

📭 **Администраторы пока не занесли скамеров**

👮 Администрация базы Infinity
🔪 Заносите скамеров и подавайте пример!
🎯 Ваши заносы вдохновляют стажёров

━━━━━━━━━━━━━━━━━━━━
"""
        buttons = [
            [Button.inline("🔄 Обновить", b"top_admins")],
            [Button.inline("🏆 Топ Стажёров", b"top_trainees")],
            [Button.inline("📊 Назад к статистике", b"return_to_stats")]
        ]

        msg = await event.respond(text, parse_mode='md', link_preview=True, buttons=buttons)
        return

    text = f"""👮 **ТОП-{len(top_admins)} АДМИНИСТРАЦИИ**
━━━━━━━━━━━━━━━━━━━━
[⠀](https://i.ibb.co/SKysgbvC/photo-2025-04-17-17-44-20-3.jpg)

🏆 **Топ администраторов по заносам:**
━━━━━━━━━━━━━━━━━━━━
"""

    buttons = []

    for i, (user_id, username, scammers_count, role_id, real_scammers) in enumerate(top_admins, 1):
        # Медальки
        if i == 1:
            medal = "👑"
        elif i == 2:
            medal = "🥈"
        elif i == 3:
            medal = "🥉"
        else:
            medal = f"{i}."

        # Имя для отображения
        if username:
            display_name = f"@{username}" if not username.startswith('@') else username
        else:
            display_name = f"ID:{user_id}"

        if len(display_name) > 18:
            display_name = display_name[:15] + "..."

        # Роль
        role_name = ROLES.get(role_id, {}).get('name', 'Админ')
        role_emoji = get_role_emoji(role_id)

        # Количество
        final_count = max(scammers_count or 0, real_scammers or 0)

        text += f"{medal} **{display_name}** ({role_emoji}{role_name}) - `{final_count}`🔪\n"

        # Кнопка
        if i <= 5:
            button_text = f"{role_emoji} {display_name} - {final_count}🔪"
            if len(button_text) > 20:
                display_short = display_name[:8] + ".." if len(display_name) > 8 else display_name
                button_text = f"{role_emoji} {display_short} - {final_count}🔪"

            buttons.append([Button.inline(button_text, f"admin_stats_{user_id}")])

    # Статистика
    total_scammers = sum(max(row[2] or 0, row[4] or 0) for row in top_admins)
    avg_scammers = total_scammers / len(top_admins) if top_admins else 0

    text += f"""
━━━━━━━━━━━━━━━━━━━━
📊 **СТАТИСТИКА:**
• Всего заносов админов: `{total_scammers}`
• В среднем на админа: `{avg_scammers:.1f}`
• Админов в топе: `{len(top_admins)}`

👮 **Администрация ведёт за собой!**
• Подавайте пример стажёрам
• Активно работайте со скамерами
• Поддерживайте порядок в базе

━━━━━━━━━━━━━━━━━━━━
"""

    # Кнопки
    nav_buttons = [
        [Button.inline("🔄 Обновить", b"top_admins")],
        [Button.inline("🏆 Топ Стажёров", b"top_trainees")],
        [Button.inline("📊 Общая статистика", b"return_to_stats")]
    ]

    if buttons:
        paired_buttons = []
        for i in range(0, len(buttons), 2):
            if i + 1 < len(buttons):
                paired_buttons.append([buttons[i][0], buttons[i + 1][0]])
            else:
                paired_buttons.append([buttons[i][0]])

        for nav_row in nav_buttons:
            paired_buttons.append(nav_row)

        buttons = paired_buttons
    else:
        buttons = nav_buttons

    msg = await event.respond(text, parse_mode='md', link_preview=True, buttons=buttons)


@bot.on(events.CallbackQuery(pattern=rb'trainee_stats_(\d+)'))
async def trainee_stats_handler(event):
    """Статистика конкретного стажера"""
    trainee_id = int(event.pattern_match.group(1))

    try:
        # Получаем данные стажера
        db.cursor.execute('''
            SELECT u.user_id, u.username, u.scammers_count, u.check_count, 
                   u.warnings, u.curator_id,
                   (SELECT COUNT(*) FROM scammers WHERE reporter_id = u.user_id) as real_scammers,
                   (SELECT COUNT(*) FROM scammers WHERE reported_by LIKE '%' || u.username || '%') as by_name
            FROM users u 
            WHERE u.user_id = ? AND u.role_id = 6
        ''', (trainee_id,))

        trainee_data = db.cursor.fetchone()

        if not trainee_data:
            await event.answer("❌ Стажёр не найден", alert=True)
            return

        (user_id, username, scammers_count, check_count,
         warnings, curator_id, real_scammers, by_name) = trainee_data

        # Имя для отображения
        display_name = username or f"ID:{user_id}"

        # Куратор
        curator_name = "Не назначен"
        if curator_id:
            db.cursor.execute('SELECT username FROM users WHERE user_id = ?', (curator_id,))
            curator_data = db.cursor.fetchone()
            if curator_data and curator_data[0]:
                curator_name = curator_data[0]

        # Общее количество
        total_scammers = max(scammers_count or 0, real_scammers or 0, by_name or 0)

        # Получаем последние заносы
        db.cursor.execute('''
            SELECT scammer_id, reason, added_date 
            FROM scammers 
            WHERE reporter_id = ? OR reported_by LIKE '%' || ? || '%'
            ORDER BY added_date DESC 
            LIMIT 5
        ''', (user_id, username or str(user_id)))

        last_scams = db.cursor.fetchall()

        # Формируем текст
        text = f"""👨‍🎓 **СТАТИСТИКА СТАЖЁРА**
━━━━━━━━━━━━━━━━━━━━
[⠀](https://i.ibb.co/fjNVRVw/photo-2025-04-17-17-44-20-4.jpg)

**👤 Имя:** {display_name}
**🆔 ID:** `{user_id}`

📊 **ПОКАЗАТЕЛИ:**
• 🔪 Слито скамеров: `{total_scammers}`
• 📝 В поле scammers_count: `{scammers_count or 0}`
• 🎯 Реально занесено: `{real_scammers or 0}`
• 🔍 Проверок: `{check_count or 0}`
• ⚠️ Выговоры: `{warnings or 0}`
• 👨‍🏫 Куратор: {curator_name}

"""

        if last_scams:
            text += "📋 **ПОСЛЕДНИЕ ЗАНОСЫ:**\n"
            for scam_id, reason, date in last_scams:
                # Форматируем дату
                try:
                    date_str = date[:10] if date else "Неизвестно"
                except:
                    date_str = "Неизвестно"

                # Сокращаем причину
                short_reason = reason[:30] + "..." if len(reason) > 30 else reason
                text += f"• `{scam_id}` - {short_reason} ({date_str})\n"

        text += """
━━━━━━━━━━━━━━━━━━━━
🎯 **СОВЕТЫ ДЛЯ РОСТА:**
• Активно ищите скамеров в чатах
• Всегда предоставляйте пруфы
• Следуйте инструкциям куратора
• Избегайте выговоров

━━━━━━━━━━━━━━━━━━━━
"""

        # Кнопки
        buttons = [
            [Button.inline("🔄 Обновить", f"trainee_stats_{trainee_id}")],
            [Button.inline("🏆 Назад к топу", b"top_trainees")],
            [Button.inline("📊 Общая статистика", b"return_to_stats")],
            [Button.inline("👤 Профиль", f"check_{trainee_id}")]
        ]

        await event.edit(text, buttons=buttons, parse_mode='md', link_preview=True)

    except Exception as e:
        logging.error(f"Ошибка в trainee_stats_handler: {e}")
        await event.answer("❌ Ошибка загрузки", alert=True)


# Глобальная переменная для хранения ID последнего сообщения с топом
last_top_message = {}


@bot.on(events.CallbackQuery(data=b"top_trainees"))
async def top_trainees_handler(event):
    """Топ стажеров по слитым скамерам с защитой от дублирования"""
    user_id = event.sender_id
    chat_id = event.chat_id

    try:
        # Проверяем, не обрабатывается ли уже запрос
        if user_id in last_top_message and time.time() - last_top_message[user_id].get('timestamp', 0) < 2:
            await event.answer("⏳ Подождите, загрузка уже идет...", alert=False)
            return

        # Сохраняем время запроса
        last_top_message[user_id] = {
            'timestamp': time.time(),
            'chat_id': chat_id
        }

        # Удаляем предыдущее сообщение если оно есть в этом же чате
        if user_id in last_top_message and 'message_id' in last_top_message[user_id]:
            try:
                await bot.delete_messages(chat_id, last_top_message[user_id]['message_id'])
            except:
                pass

        # Удаляем сообщение с кнопкой (если возможно)
        try:
            await event.delete()
        except:
            pass

        # Получаем топ стажеров
        top_trainees = db.cursor.execute('''
            SELECT u.user_id, u.username, u.scammers_count, 
                   u.check_count, u.warnings,
                   (SELECT COUNT(*) FROM scammers WHERE reporter_id = u.user_id) as real_scammers
            FROM users u 
            WHERE u.role_id = 6 
            ORDER BY COALESCE(scammers_count, 0) DESC 
            LIMIT 15
        ''').fetchall()

        if not top_trainees:
            text = """🏆 **ТОП СТАЖЁРОВ**
━━━━━━━━━━━━━━━━━━━━
[⠀](https://i.ibb.co/fjNVRVw/photo-2025-04-17-17-44-20-4.jpg)

📭 **Список стажёров пока пуст!**

👨‍🎓 Станьте первым в этом топе!
🔪 Заносите скамеров через /скам
🎯 Получайте +спасибо от персонала

━━━━━━━━━━━━━━━━━━━━
"""
            buttons = [
                [Button.inline("🔄 Обновить", b"top_trainees")],
                [Button.inline("📊 Назад к статистике", b"return_to_stats")],
                [Button.url("📚 Как стать стажёром?", "https://t.me/Infinityantiscam")]
            ]

            msg = await event.respond(text, parse_mode='md', link_preview=True, buttons=buttons)
            last_top_message[user_id]['message_id'] = msg.id
            return

        # Формируем текст
        text = f"""🏆 **ТОП-{len(top_trainees)} СТАЖЁРОВ**
━━━━━━━━━━━━━━━━━━━━
[⠀](https://i.ibb.co/fjNVRVw/photo-2025-04-17-17-44-20-4.jpg)

👑 **Топ по количеству слитых скамеров:**
━━━━━━━━━━━━━━━━━━━━
"""

        buttons = []

        for i, (user_id_trainee, username, scammers_count, check_count, warnings, real_scammers) in enumerate(
                top_trainees, 1):
            if i == 1:
                medal = "🥇"
            elif i == 2:
                medal = "🥈"
            elif i == 3:
                medal = "🥉"
            else:
                medal = f"{i}."

            display_name = f"@{username}" if username else f"ID:{user_id_trainee}"
            if len(display_name) > 20:
                display_name = display_name[:17] + "..."

            final_count = max(scammers_count or 0, real_scammers or 0)
            text += f"{medal} **{display_name}** - `{final_count}`🔪\n"

            if i <= 5:
                if final_count >= 50:
                    knife_emoji = "🔪🔪🔪"
                elif final_count >= 20:
                    knife_emoji = "🔪🔪"
                elif final_count >= 10:
                    knife_emoji = "🔪"
                else:
                    knife_emoji = "🗡️"

                button_text = f"{medal} {display_name} {final_count}{knife_emoji}"
                if len(button_text) > 25:
                    display_short = display_name[:10] + ".." if len(display_name) > 10 else display_name
                    button_text = f"{medal} {display_short} {final_count}{knife_emoji}"

                buttons.append([Button.inline(button_text, f"trainee_stats_{user_id_trainee}")])

        # Статистика
        total_scammers = sum(max(row[2] or 0, row[5] or 0) for row in top_trainees)
        avg_scammers = total_scammers / len(top_trainees) if top_trainees else 0

        text += f"""
━━━━━━━━━━━━━━━━━━━━
📊 **СТАТИСТИКА ТОПА:**
• Всего скамеров в топе: `{total_scammers}`
• В среднем на стажёра: `{avg_scammers:.1f}`
• Стажёров в топе: `{len(top_trainees)}`

🎯 **Как попасть в топ?**
• Активно заносите скамеров через /скам
• Получайте +спасибо от персонала
• Не получайте выговоров

━━━━━━━━━━━━━━━━━━━━
"""

        # Кнопки
        nav_buttons = [
            [Button.inline("🔄 Обновить", b"top_trainees")],
            [Button.inline("📊 Общая статистика", b"return_to_stats")],
            [Button.inline("🏅 Топ Админов", b"top_admins")]
        ]

        if buttons:
            paired_buttons = []
            for i in range(0, len(buttons), 2):
                if i + 1 < len(buttons):
                    paired_buttons.append([buttons[i][0], buttons[i + 1][0]])
                else:
                    paired_buttons.append([buttons[i][0]])

            for nav_row in nav_buttons:
                paired_buttons.append(nav_row)

            buttons = paired_buttons
        else:
            buttons = nav_buttons

        buttons.append([Button.url("👨‍🎓 Чат стажёров", "https://t.me/Infinityantiscam")])

        # Отправляем ОДНО сообщение
        msg = await event.respond(text, parse_mode='md', link_preview=True, buttons=buttons)

        # Сохраняем ID сообщения
        last_top_message[user_id]['message_id'] = msg.id

        # Отвечаем на callback чтобы убрать "часики"
        await event.answer()

    except Exception as e:
        logging.error(f"Ошибка в top_trainees_handler: {e}")
        await event.answer("❌ Ошибка загрузки", alert=True)


@bot.on(events.CallbackQuery(data=b"return_to_stats"))
async def return_to_stats_handler(event):
    """Вернуться к статистике с защитой от дублирования"""
    user_id = event.sender_id
    chat_id = event.chat_id

    try:
        # Защита от быстрых кликов
        if user_id in last_top_message and time.time() - last_top_message[user_id].get('timestamp', 0) < 2:
            await event.answer("⏳ Подождите...", alert=False)
            return

        last_top_message[user_id] = {
            'timestamp': time.time(),
            'chat_id': chat_id
        }

        # Удаляем текущее сообщение
        try:
            await event.delete()
        except:
            pass

        # Удаляем предыдущее сообщение топа если есть
        if user_id in last_top_message and 'message_id' in last_top_message[user_id]:
            try:
                await bot.delete_messages(chat_id, last_top_message[user_id]['message_id'])
                del last_top_message[user_id]['message_id']
            except:
                pass

        # Вызываем статистику через имитацию события
        from telethon import events

        # Создаем фейковое событие
        class FakeEvent:
            def __init__(self):
                self.is_private = True
                self.sender_id = user_id
                self.chat_id = chat_id
                self._respond_called = False

            async def respond(self, *args, **kwargs):
                if not self._respond_called:
                    self._respond_called = True
                    msg = await event.respond(*args, **kwargs)
                    last_top_message[user_id]['message_id'] = msg.id
                    return msg

        fake_event = FakeEvent()
        await statistics(fake_event)

        await event.answer()

    except Exception as e:
        logging.error(f"Ошибка в return_to_stats_handler: {e}")
        await event.answer("❌ Ошибка", alert=True)


# Функция для периодической очистки
async def cleanup_old_messages():
    """Очищает старые записи о сообщениях"""
    while True:
        await asyncio.sleep(3600)  # Каждый час
        current_time = time.time()
        to_remove = []

        for user_id, data in last_top_message.items():
            if current_time - data.get('timestamp', 0) > 7200:  # 2 часа
                to_remove.append(user_id)

        for user_id in to_remove:
            del last_top_message[user_id]

        if to_remove:
            logging.info(f"Очищено {len(to_remove)} старых записей сообщений")


# Добавьте в main() перед запуском бота:
bot.loop.create_task(cleanup_old_messages())




@bot.on(events.CallbackQuery(data=b"top_day"))
async def top_day_handler(event):
    try:
        await bot.delete_messages(event.chat_id, bot.stat_message_id)
    except Exception as e:
        print(f"Ошибка при удалении сообщения: {e}")

    try:
        # Проверяем существование таблицы messages
        db.cursor.execute("SELECT name FROM sqlite_master WHERE type='table' AND name='messages'")
        if not db.cursor.fetchone():
            msg = await event.respond("⚠️ Таблица сообщений ещё не создана. Активность не отслеживается.",
                                      buttons=Button.inline("↩Скрыть", b"hide_message"))
            bot.last_message_id = msg.id
            return

        # ИСПРАВЛЕННЫЙ ЗАПРОС - используем message_id вместо id
        top_users = db.cursor.execute('''
            SELECT u.user_id, u.username, COUNT(m.message_id) as count
            FROM users u
            JOIN messages m ON u.user_id = m.user_id
            WHERE m.timestamp >= datetime('now', '-1 day')
            GROUP BY u.user_id
            ORDER BY count DESC
            LIMIT 10
        ''').fetchall()

        if not top_users:
            msg = await event.respond("📭 Пока нет активности за последние 24 часа!",
                                      buttons=Button.inline("↩Скрыть", b"hide_message"))
            bot.last_message_id = msg.id
            return

        response = "😎 Топ 10 активных пользователей за 24 часа:\n\n"
        for i, (user_id, username, count) in enumerate(top_users, 1):
            user_link = f"[{username or f'ID:{user_id}'}](tg://user?id={user_id})"
            response += f"{i}. {user_link} — ✉️ {count} сообщений\n"

        msg = await event.respond(response, buttons=Button.inline("↩Скрыть", b"hide_message"))
        bot.last_message_id = msg.id

    except sqlite3.Error as e:
        await event.respond(f"⚠️ Ошибка базы данных: {str(e)}", buttons=Button.inline("↩Скрыть", b"hide_message"))
    except Exception as e:
        await event.respond(f"⚠️ Произошла ошибка: {str(e)}", buttons=Button.inline("↩Скрыть", b"hide_message"))



@bot.on(events.CallbackQuery(data=b"hide_message"))
async def hide_message_handler(event):
    try:
        await event.delete()
    except Exception as e:
        print(f"Ошибка при удалении сообщения: {e}")


@bot.on(events.NewMessage(pattern="🎮 Игры"))
@subscription_required
async def games_menu(event):
    if not event.is_private:
        return

    user_id = event.sender_id

    # Убеждаемся, что у пользователя есть стартовый капитал
    db.ensure_start_balance(user_id)

    # Проверка защиты от спама
    can_press, message = check_button_spam_protection(user_id)
    if not can_press:
        await event.respond(message)
        return

    # Показываем загрузку
    await show_button_loading(event, "меню игр")

    # Получаем баланс
    balance = db.get_balance(user_id)

    # Проверяем доступность ежедневного бонуса
    can_claim, claim_info = db.can_claim_daily(user_id)
    daily_status = "✅ Доступен!" if can_claim else f"⏳ Через {claim_info}"

    text = f"""🎮 **ИГРОВОЙ ЦЕНТР INFINITY**
━━━━━━━━━━━━━━━━━━━━
[⠀](https://i.ibb.co/ycyPRXrb/photo-2025-04-17-17-44-20-2.jpg)

{format_balance(user_id)}

🎁 **Ежедневный бонус:** {daily_status}
💎 **Награда:** 100 💎 каждые 24 часа

📝 **Как играть:**
Просто напиши название игры и ставку:
• `мины 50` - игра Мины со ставкой 50 💎
• `кости 25` - игра Кости со ставкой 25 💎
• `кено 30` - игра Кено со ставкой 30 💎
• `колесо 40` - Колесо фортуны со ставкой 40 💎

🎲 **Доступные игры:**

💣 **МИНЫ** - рискни и умножь ставку!
   • 20 клеток, 7 мин
   • +0.5 к множителю за шаг
   • Забери выигрыш в любой момент

🎲 **КОСТИ** - классическая игра
   • Угадай больше/меньше/ровно 7
   • Множители x2 и x5

🎰 **КЕНО** - выбери числа
   • До x500 множителя
   • Выбери от 1 до 10 чисел

🎡 **КОЛЕСО ФОРТУНЫ** - крути и выигрывай!
   • Множители x0.5 до x10
   • Джекпот x25!

━━━━━━━━━━━━━━━━━━━━
💡 Пример: **мины 50** - начать игру Мины со ставкой 50
"""

    buttons = []
    if can_claim:
        buttons.append([Button.inline("🎁 ЗАБРАТЬ БОНУС 100 💎", "claim_daily")])
    buttons.append([Button.inline("💼 Ежедневные задания", "daily_tasks_menu")])
    buttons.append([Button.inline("↩ Назад", "back_to_main")])

    # Удаляем сообщение о загрузке
    if user_id in button_loading_messages:
        try:
            await bot.delete_messages(event.chat_id, button_loading_messages[user_id])
            del button_loading_messages[user_id]
        except:
            pass

    await event.respond(text, buttons=buttons, parse_mode='md', link_preview=True)


@bot.on(events.CallbackQuery(pattern="daily_tasks_menu"))
async def daily_tasks_menu(event):
    """Меню ежедневных заданий"""
    user_id = event.sender_id

    # Убеждаемся, что есть стартовый капитал
    db.ensure_start_balance(user_id)

    # Получаем активные задания
    tasks = db.get_active_tasks()
    completed_tasks = db.get_user_completed_tasks(user_id)

    if not tasks:
        text = f"""📋 **ЕЖЕДНЕВНЫЕ ЗАДАНИЯ**
━━━━━━━━━━━━━━━━━━━━
{format_balance(user_id)}

📭 **Пока нет активных заданий**

Задания появятся здесь, когда администратор их добавит.

━━━━━━━━━━━━━━━━━━━━
[⠀](https://i.ibb.co/ycyPRXrb/photo-2025-04-17-17-44-20-2.jpg)"""

        buttons = [[Button.inline("◀ Назад в игры", "games_menu")]]

        try:
            await event.edit(text, buttons=buttons, parse_mode='md', link_preview=True)
        except:
            await event.respond(text, buttons=buttons, parse_mode='md', link_preview=True)
        return

    text = f"""📋 **ЕЖЕДНЕВНЫЕ ЗАДАНИЯ**
━━━━━━━━━━━━━━━━━━━━
{format_balance(user_id)}

📌 **Доступные задания:**
"""

    buttons = []

    for task in tasks:
        task_id, task_name, task_description, reward, is_active, created_by, created_date = task

        # Проверяем, выполнено ли задание сегодня
        if task_id in completed_tasks:
            status = "✅ ВЫПОЛНЕНО"
            button_text = f"✅ {task_name}"
            callback = f"task_completed_{task_id}"
        else:
            status = "⏳ ДОСТУПНО"
            button_text = f"📌 {task_name} - {reward}💎"
            callback = f"complete_task_{task_id}"

        text += f"\n**{task_name}**\n📝 {task_description}\n💰 Награда: {reward} 💎 | {status}\n"
        buttons.append([Button.inline(button_text, callback)])

    text += """
━━━━━━━━━━━━━━━━━━━━
💡 Задания обновляются каждый день в 00:00
"""

    buttons.append([Button.inline("◀ Назад в игры", "games_menu")])

    try:
        await event.edit(text, buttons=buttons, parse_mode='md', link_preview=True)
    except:
        await event.respond(text, buttons=buttons, parse_mode='md', link_preview=True)

@bot.on(events.CallbackQuery(pattern="claim_daily"))
async def claim_daily_handler(event):
    user_id = event.sender_id

    # Убеждаемся, что у пользователя есть стартовый капитал
    db.ensure_start_balance(user_id)

    # Пытаемся получить бонус
    success, result = db.claim_daily(user_id)

    if success:
        amount, new_balance, next_time = result
        await event.answer(f"✅ Получено {amount} 💎! Новый баланс: {new_balance} 💎", alert=True)

        # Обновляем сообщение
        text = f"""🎁 **БОНУС ПОЛУЧЕН!**
━━━━━━━━━━━━━━━━━━━━

💰 **Получено:** +{amount} 💎
💎 **Новый баланс:** {new_balance} 💎
⏰ **Следующий бонус:** {next_time}

[⠀](https://i.ibb.co/ycyPRXrb/photo-2025-04-17-17-44-20-2.jpg)"""

        await event.edit(text, buttons=[Button.inline("🎮 В меню игр", "games_menu")])
    else:
        # Не прошло 24 часа
        await event.answer(f"⏳ Бонус будет доступен через {result}", alert=True)

@bot.on(events.NewMessage(pattern=r'^(мины|кости|кено|колесо)\s+(\d+)$'))
async def game_text_command(event):
    """Обработчик текстовых команд для игр: мины 50, кости 25 и т.д."""
    if not event.is_private:
        await event.delete()
        return

    user_id = event.sender_id
    game_type = event.pattern_match.group(1).lower()
    bet = int(event.pattern_match.group(2))

    # Проверка на существующую игру
    if user_id in mines_games or user_id in dice_games or user_id in keno_games or user_id in wheel_games:
        await event.respond("❌ У тебя уже есть активная игра! Заверши её сначала.")
        return

    # Проверка баланса
    balance = db.get_balance(user_id)
    if bet < 10:
        await event.respond("❌ Минимальная ставка: 10 💎")
        return

    if bet > balance:
        await event.respond(f"❌ Недостаточно средств! Твой баланс: {balance} 💎")
        return

    # Списываем ставку
    db.remove_balance(user_id, bet)

    # Запускаем соответствующую игру
    if game_type == "мины":
        await start_mines_game(event, user_id, bet)
    elif game_type == "кости":
        await start_dice_game(event, user_id, bet)
    elif game_type == "кено":
        await start_keno_game(event, user_id, bet)
    elif game_type == "колесо":
        await start_wheel_game(event, user_id, bet)


@bot.on(events.NewMessage(pattern=r'^(мины|кости|кено|колесо)'))
async def game_text_no_bet(event):
    """Если написали только название игры без ставки"""
    if not event.is_private:
        return

    await event.respond(
        "❌ Укажи ставку! Например:\n"
        "• `мины 50`\n"
        "• `кости 25`\n"
        "• `кено 30`\n"
        "• `колесо 40`"
    )


async def start_mines_game(event, user_id, bet):
    """Запуск игры Мины"""

    # Создаем игровое поле
    cells = ["🔲"] * 20
    mine_positions = random.sample(range(20), 7)

    # Сохраняем игру
    mines_games[user_id] = {
        "state": "playing",
        "bet": bet,
        "cells": cells,
        "mines": mine_positions,
        "opened": [],
        "multiplier": 1.0,
        "steps": 0,
        "message_id": None
    }

    await show_mines_field(event, user_id, is_new=True)


@bot.on(events.CallbackQuery(pattern=r"mine_(\d+)"))
async def mine_click_handler(event):
    user_id = event.sender_id
    cell_index = int(event.pattern_match.group(1))

    if user_id not in mines_games or mines_games[user_id].get("state") != "playing":
        await event.answer("❌ Игра не найдена", alert=True)
        return

    game = mines_games[user_id]

    # Проверяем, не открыта ли уже клетка
    if cell_index in game["opened"]:
        await event.answer("⚠️ Эта клетка уже открыта", alert=True)
        return

    # Открываем клетку
    game["opened"].append(cell_index)

    # Проверяем мину
    if cell_index in game["mines"]:
        # Проигрыш
        game["cells"][cell_index] = "💥"

        # Показываем все мины
        for mine in game["mines"]:
            game["cells"][mine] = "💣"

        # Удаляем игру
        del mines_games[user_id]

        # Сообщение о проигрыше
        text = f"""💥 **БАБАХ! Ты наступил на мину!**
━━━━━━━━━━━━━━━━━━━━

💰 **Потеряно:** {game['bet']} 💎
📉 **Новый баланс:** {db.get_balance(user_id)} 💎

[⠀](https://i.ibb.co/ycyPRXrb/photo-2025-04-17-17-44-20-2.jpg)

Попробуй ещё раз!"""

        buttons = [Button.inline("🎮 В меню игр", "games_menu")]

        try:
            await event.edit(text, buttons=buttons)
        except Exception as e:
            logging.error(f"Ошибка в mine_click_handler (проигрыш): {e}")
            await event.respond(text, buttons=buttons)
        return

    # Успех - открываем безопасную клетку
    game["cells"][cell_index] = "✅"
    game["steps"] += 1
    game["multiplier"] = 1.0 + (game["steps"] * 0.5)

    # Проверяем, не открыты ли все безопасные клетки
    if len(game["opened"]) == 13:  # 20 всего - 7 мин = 13 безопасных
        # Победа - открыты все безопасные
        win = int(game['bet'] * game['multiplier'])
        db.add_balance(user_id, win)

        text = f"""🎉 **ПОБЕДА! Ты открыл все клетки!**
━━━━━━━━━━━━━━━━━━━━

💰 **Выигрыш:** {win} 💎
📊 **Множитель:** x{game['multiplier']:.1f}
📈 **Чистый профит:** +{win - game['bet']} 💎
{format_balance(user_id)}

[⠀](https://i.ibb.co/ycyPRXrb/photo-2025-04-17-17-44-20-2.jpg)"""

        del mines_games[user_id]

        try:
            await event.edit(text, buttons=[Button.inline("🎮 В меню игр", "games_menu")])
        except Exception as e:
            logging.error(f"Ошибка в mine_click_handler (победа): {e}")
            await event.respond(text, buttons=[Button.inline("🎮 В меню игр", "games_menu")])
        return

    # Обновляем поле
    await show_mines_field(event, user_id)

@bot.on(events.CallbackQuery(pattern=r"mine_(\d+)"))
async def mine_click_handler(event):
    user_id = event.sender_id
    cell_index = int(event.pattern_match.group(1))

    if user_id not in mines_games or mines_games[user_id].get("state") != "playing":
        await event.answer("❌ Игра не найдена", alert=True)
        return

    game = mines_games[user_id]

    # Проверяем, не открыта ли уже клетка
    if cell_index in game["opened"]:
        await event.answer("⚠️ Эта клетка уже открыта", alert=True)
        return

    # Открываем клетку
    game["opened"].append(cell_index)

    # Проверяем мину
    if cell_index in game["mines"]:
        # Проигрыш
        game["cells"][cell_index] = "💥"

        # Показываем все мины
        for mine in game["mines"]:
            game["cells"][mine] = "💣"

        # Удаляем игру
        del mines_games[user_id]

        # Сообщение о проигрыше
        text = f"""💥 **БАБАХ! Ты наступил на мину!**
━━━━━━━━━━━━━━━━━━━━

💰 **Потеряно:** {game['bet']} 💎
📉 **Новый баланс:** {db.get_balance(user_id)} 💎

[⠀](https://i.ibb.co/ycyPRXrb/photo-2025-04-17-17-44-20-2.jpg)

Попробуй ещё раз!"""

        buttons = [Button.inline("🎮 В меню игр", "games_menu")]

        try:
            await event.edit(text, buttons=buttons)
        except Exception as e:
            logging.error(f"Ошибка в mine_click_handler (проигрыш): {e}")
            await event.respond(text, buttons=buttons)
        return

    # Успех - открываем безопасную клетку
    game["cells"][cell_index] = "✅"
    game["steps"] += 1
    game["multiplier"] = 1.0 + (game["steps"] * 0.5)

    # Проверяем, не открыты ли все безопасные клетки
    if len(game["opened"]) == 13:  # 20 всего - 7 мин = 13 безопасных
        # Победа - открыты все безопасные
        win = int(game['bet'] * game['multiplier'])
        db.add_balance(user_id, win)

        text = f"""🎉 **ПОБЕДА! Ты открыл все клетки!**
━━━━━━━━━━━━━━━━━━━━

💰 **Выигрыш:** {win} 💎
📊 **Множитель:** x{game['multiplier']:.1f}
📈 **Чистый профит:** +{win - game['bet']} 💎
{format_balance(user_id)}

[⠀](https://i.ibb.co/ycyPRXrb/photo-2025-04-17-17-44-20-2.jpg)"""

        del mines_games[user_id]

        try:
            await event.edit(text, buttons=[Button.inline("🎮 В меню игр", "games_menu")])
        except Exception as e:
            logging.error(f"Ошибка в mine_click_handler (победа): {e}")
            await event.respond(text, buttons=[Button.inline("🎮 В меню игр", "games_menu")])
        return

    # Обновляем поле
    await show_mines_field(event, user_id)


@bot.on(events.CallbackQuery(pattern="cashout_mines"))
async def mines_cashout(event):
    user_id = event.sender_id

    if user_id not in mines_games or mines_games[user_id].get("state") != "playing":
        await event.answer("❌ Игра не найдена", alert=True)
        return

    game = mines_games[user_id]
    win = int(game['bet'] * game['multiplier'])

    # Начисляем выигрыш
    db.add_balance(user_id, win)

    text = f"""💰 **ТЫ ЗАБРАЛ ВЫИГРЫШ!**
━━━━━━━━━━━━━━━━━━━━

💎 **Итог:** +{win - game['bet']} 💎
📊 **Множитель:** x{game['multiplier']:.1f}
🔓 **Открыто клеток:** {len(game['opened'])}/20
{format_balance(user_id)}

[⠀](https://i.ibb.co/ycyPRXrb/photo-2025-04-17-17-44-20-2.jpg)"""

    del mines_games[user_id]

    try:
        await event.edit(text, buttons=[Button.inline("🎮 В меню игр", "games_menu")])
    except Exception as e:
        logging.error(f"Ошибка в mines_cashout: {e}")
        await event.respond(text, buttons=[Button.inline("🎮 В меню игр", "games_menu")])


async def start_mines_game(event, user_id, bet):
    """Запуск игры Мины"""

    # Создаем игровое поле
    cells = ["🔲"] * 20
    mine_positions = random.sample(range(20), 7)

    # Сохраняем игру
    mines_games[user_id] = {
        "state": "playing",
        "bet": bet,
        "cells": cells,
        "mines": mine_positions,
        "opened": [],
        "multiplier": 1.0,
        "steps": 0,
        "message_id": None
    }

    await show_mines_field(event, user_id, is_new=True)


async def show_mines_field(event, user_id, is_new=False):
    """Отображает поле игры Мины"""
    game = mines_games[user_id]

    # Создаем клавиатуру с клетками
    buttons = []
    row = []

    for i in range(20):
        cell_display = game["cells"][i]

        # Если клетка уже открыта
        if i in game["opened"]:
            if i in game["mines"]:
                cell_display = "💥"
            else:
                cell_display = "✅"

        callback_data = f"mine_{i}"

        if i > 0 and i % 5 == 0:
            buttons.append(row)
            row = []

        row.append(Button.inline(cell_display, callback_data))

    if row:
        buttons.append(row)

    # Кнопки управления (ТОЛЬКО "ЗАБРАТЬ", без "ВЫЙТИ")
    current_win = int(game['bet'] * game['multiplier'])
    buttons.append([
        Button.inline(f"💰 Забрать {current_win} 💎", "cashout_mines")
    ])

    text = f"""💣 **ИГРА "МИНЫ"** | Ставка: {game['bet']} 💎
━━━━━━━━━━━━━━━━━━━━

📊 **Множитель:** x{game['multiplier']:.1f}
🔓 **Открыто клеток:** {len(game['opened'])}/20
💣 **Мин осталось:** {7 - sum(1 for i in game['opened'] if i in game['mines'])}

💰 **Текущий выигрыш:** {current_win} 💎

━━━━━━━━━━━━━━━━━━━━
Выбирай клетки кнопками ниже:
"""

    if is_new:
        msg = await event.respond(text, buttons=buttons)
        game['message_id'] = msg.id
    else:
        try:
            await event.edit(text, buttons=buttons)
        except Exception as e:
            logging.error(f"Ошибка в show_mines_field: {e}")
            msg = await event.respond(text, buttons=buttons)
            game['message_id'] = msg.id


@bot.on(events.CallbackQuery(pattern=r"mine_(\d+)"))
async def mine_click_handler(event):
    user_id = event.sender_id
    cell_index = int(event.pattern_match.group(1))

    if user_id not in mines_games or mines_games[user_id].get("state") != "playing":
        await event.answer("❌ Игра не найдена", alert=True)
        return

    game = mines_games[user_id]

    # Проверяем, не открыта ли уже клетка
    if cell_index in game["opened"]:
        await event.answer("⚠️ Эта клетка уже открыта", alert=True)
        return

    # Открываем клетку
    game["opened"].append(cell_index)

    # Проверяем мину
    if cell_index in game["mines"]:
        # Проигрыш
        game["cells"][cell_index] = "💥"

        # Показываем все мины
        for mine in game["mines"]:
            game["cells"][mine] = "💣"

        # Удаляем игру
        del mines_games[user_id]

        # Сообщение о проигрыше
        await event.edit(
            f"""💥 **БАБАХ! Ты наступил на мину!**
━━━━━━━━━━━━━━━━━━━━

💰 **Потеряно:** {game['bet']} 💎
📉 **Новый баланс:** {db.get_balance(user_id)} 💎

[⠀](https://i.ibb.co/ycyPRXrb/photo-2025-04-17-17-44-20-2.jpg)

Попробуй ещё раз!""",
            buttons=[Button.inline("🎮 В меню игр", "games_menu")]
        )
        return

    # Успех - открываем безопасную клетку
    game["cells"][cell_index] = "✅"
    game["steps"] += 1
    game["multiplier"] = 1.0 + (game["steps"] * 0.5)

    # Проверяем, не открыты ли все безопасные клетки
    if len(game["opened"]) == 13:  # 20 всего - 7 мин = 13 безопасных
        # Победа - открыты все безопасные
        win = int(game['bet'] * game['multiplier'])
        db.add_balance(user_id, win)

        text = f"""🎉 **ПОБЕДА! Ты открыл все клетки!**
━━━━━━━━━━━━━━━━━━━━

💰 **Выигрыш:** {win} 💎
📊 **Множитель:** x{game['multiplier']:.1f}
📈 **Чистый профит:** +{win - game['bet']} 💎
{format_balance(user_id)}

[⠀](https://i.ibb.co/ycyPRXrb/photo-2025-04-17-17-44-20-2.jpg)"""

        del mines_games[user_id]
        await event.edit(text, buttons=[Button.inline("🎮 В меню игр", "games_menu")])
        return

    # Обновляем поле
    await show_mines_field(event, user_id)


@bot.on(events.CallbackQuery(pattern="cashout_mines"))
async def mines_cashout(event):
    user_id = event.sender_id

    if user_id not in mines_games or mines_games[user_id].get("state") != "playing":
        await event.answer("❌ Игра не найдена", alert=True)
        return

    game = mines_games[user_id]
    win = int(game['bet'] * game['multiplier'])

    # Начисляем выигрыш
    db.add_balance(user_id, win)

    text = f"""💰 **ТЫ ЗАБРАЛ ВЫИГРЫШ!**
━━━━━━━━━━━━━━━━━━━━

💎 **Итог:** +{win - game['bet']} 💎
📊 **Множитель:** x{game['multiplier']:.1f}
🔓 **Открыто клеток:** {len(game['opened'])}/20
{format_balance(user_id)}

[⠀](https://i.ibb.co/ycyPRXrb/photo-2025-04-17-17-44-20-2.jpg)"""

    del mines_games[user_id]
    await event.edit(text, buttons=[Button.inline("🎮 В меню игр", "games_menu")])


@bot.on(events.CallbackQuery(pattern="cancel_mines"))
async def cancel_mines(event):
    user_id = event.sender_id

    if user_id in mines_games:
        # Возвращаем ставку
        db.add_balance(user_id, mines_games[user_id]["bet"])
        del mines_games[user_id]

    text = "❌ Игра отменена. Ставка возвращена."
    buttons = [Button.inline("🎮 В меню игр", "games_menu")]

    try:
        await event.edit(text, buttons=buttons)
    except Exception as e:
        logging.error(f"Ошибка в cancel_mines: {e}")
        await event.respond(text, buttons=buttons)


async def start_dice_game(event, user_id, bet):
    """Запуск игры Кости"""
    dice_games[user_id] = {
        "state": "playing",
        "bet": bet
    }

    text = f"""🎲 **ИГРА "КОСТИ"** | Ставка: {bet} 💎
━━━━━━━━━━━━━━━━━━━━

{format_balance(user_id)}

📋 **Выбери вариант:**

• **БОЛЬШЕ 7** - выигрыш x2 (сумма 8-12)
• **МЕНЬШЕ 7** - выигрыш x2 (сумма 2-6)
• **РОВНО 7** - выигрыш x5 (сумма 7)

━━━━━━━━━━━━━━━━━━━━"""

    await event.respond(
        text,
        buttons=[
            [Button.inline("🔴 БОЛЬШЕ 7 (x2)", f"dice_high_{bet}")],
            [Button.inline("🔵 МЕНЬШЕ 7 (x2)", f"dice_low_{bet}")],
            [Button.inline("🟢 РОВНО 7 (x5)", f"dice_seven_{bet}")]
        ]
    )


@bot.on(events.CallbackQuery(pattern=r"dice_(high|low|seven)_(\d+)"))
async def dice_choice_handler(event):
    user_id = event.sender_id
    choice = event.pattern_match.group(1).decode('utf-8')
    bet = int(event.pattern_match.group(2))

    if user_id not in dice_games:
        await event.answer("❌ Игра не найдена", alert=True)
        return

    # Бросаем кости
    dice1 = random.randint(1, 6)
    dice2 = random.randint(1, 6)
    total = dice1 + dice2

    # Определяем результат
    win = False
    multiplier = 2

    if choice == "seven":
        multiplier = 5
        win = (total == 7)
    elif choice == "high":
        win = (total > 7)
    elif choice == "low":
        win = (total < 7)

    # Формируем результат
    result_text = f"""🎲 **РЕЗУЛЬТАТ БРОСКА**
━━━━━━━━━━━━━━━━━━━━

🎯 **Кости:** {dice1} + {dice2} = **{total}**
📊 **Твой выбор:** {'БОЛЬШЕ 7' if choice == 'high' else 'МЕНЬШЕ 7' if choice == 'low' else 'РОВНО 7'}

"""

    if win:
        win_amount = bet * multiplier
        db.add_balance(user_id, win_amount)
        result_text += f"""✅ **ТЫ ВЫИГРАЛ!**
💰 **Выигрыш:** {win_amount} 💎
📈 **Чистый профит:** +{win_amount - bet} 💎
{format_balance(user_id)}"""
    else:
        result_text += f"""❌ **ТЫ ПРОИГРАЛ**
💰 **Потеряно:** {bet} 💎
{format_balance(user_id)}"""

    result_text += "\n\n[⠀](https://i.ibb.co/ycyPRXrb/photo-2025-04-17-17-44-20-2.jpg)"

    # Удаляем игру
    del dice_games[user_id]

    try:
        await event.edit(result_text, buttons=[Button.inline("🎮 В меню игр", "games_menu")])
    except Exception as e:
        logging.error(f"Ошибка в dice_choice_handler: {e}")
        await event.respond(result_text, buttons=[Button.inline("🎮 В меню игр", "games_menu")])


async def start_keno_game(event, user_id, bet):
    """Запуск игры Кено"""
    keno_games[user_id] = {
        "state": "selecting",
        "bet": bet,
        "selected": []
    }

    await show_keno_selection(event, user_id)

async def show_keno_selection(event, user_id):
    """Показывает интерфейс выбора чисел для Кено"""
    game = keno_games[user_id]
    selected = game["selected"]

    text = f"""🎰 **ИГРА "КЕНО"** | Ставка: {game['bet']} 💎
━━━━━━━━━━━━━━━━━━━━

{format_balance(user_id)}

📋 **Выбери числа (от 1 до 10):**
Выбрано: {len(selected)}/10 чисел

📊 **Множители:**
• 1 число - x2    • 6 чисел - x15
• 2 числа - x4    • 7 чисел - x25
• 3 числа - x6    • 8 чисел - x50
• 4 числа - x9    • 9 чисел - x100
• 5 чисел - x12   • 10 чисел - x500

━━━━━━━━━━━━━━━━━━━━
Нажимай на числа, чтобы выбрать:
"""

    # Создаем клавиатуру с числами 1-80
    buttons = []
    row = []

    for i in range(1, 81):
        # Определяем отображение клетки
        if i in selected:
            cell_display = f"✅{i}"
        else:
            cell_display = f"{i}"

        callback = f"keno_select_{i}"

        row.append(Button.inline(cell_display, callback))

        if i % 10 == 0:
            buttons.append(row)
            row = []

    if row:
        buttons.append(row)

    # Кнопки управления (ТОЛЬКО "ИГРАТЬ", без "ОТМЕНА")
    nav_buttons = []
    if len(selected) >= 1:
        nav_buttons.append(Button.inline("🎲 ИГРАТЬ!", f"keno_play"))

    if nav_buttons:
        buttons.append(nav_buttons)

    await event.respond(text, buttons=buttons)

@bot.on(events.CallbackQuery(pattern=r"keno_select_(\d+)"))
async def keno_select_handler(event):
    user_id = event.sender_id
    number = int(event.pattern_match.group(1))

    if user_id not in keno_games or keno_games[user_id].get("state") != "selecting":
        await event.answer("❌ Игра не найдена", alert=True)
        return

    game = keno_games[user_id]
    selected = game["selected"]

    if number in selected:
        # Убираем число
        selected.remove(number)
        await event.answer(f"❌ Число {number} убрано", alert=False)
    else:
        # Добавляем число
        if len(selected) >= 10:
            await event.answer("❌ Максимум 10 чисел!", alert=True)
            return
        selected.append(number)
        await event.answer(f"✅ Число {number} добавлено", alert=False)

    # Обновляем сообщение
    await event.delete()
    await show_keno_selection(event, user_id)


@bot.on(events.CallbackQuery(pattern="keno_play"))
async def keno_play_handler(event):
    user_id = event.sender_id

    if user_id not in keno_games or keno_games[user_id].get("state") != "selecting":
        await event.answer("❌ Игра не найдена", alert=True)
        return

    game = keno_games[user_id]
    selected = game["selected"]
    bet = game["bet"]

    if len(selected) == 0:
        await event.answer("❌ Выбери хотя бы одно число!", alert=True)
        return

    # Генерируем выигрышные числа (20 случайных из 80)
    winning_numbers = sorted(random.sample(range(1, 81), 20))

    # Считаем совпадения
    matches = len(set(selected) & set(winning_numbers))

    # Определяем множитель
    multipliers = {
        1: 2, 2: 4, 3: 6, 4: 9, 5: 12,
        6: 15, 7: 25, 8: 50, 9: 100, 10: 500
    }
    multiplier = multipliers.get(matches, 0)

    # Формируем результат
    win_amount = bet * multiplier if multiplier > 0 else 0

    result_text = f"""🎰 **РЕЗУЛЬТАТ КЕНО**
━━━━━━━━━━━━━━━━━━━━

🎯 **Твои числа:** {', '.join(map(str, sorted(selected)))}
✨ **Выигрышные:** {', '.join(map(str, winning_numbers[:10]))}...
📊 **Совпадений:** {matches}

"""

    if win_amount > 0:
        db.add_balance(user_id, win_amount)
        result_text += f"""✅ **ТЫ ВЫИГРАЛ!**
💰 **Множитель:** x{multiplier}
💎 **Выигрыш:** {win_amount} 💎
📈 **Чистый профит:** +{win_amount - bet} 💎
{format_balance(user_id)}"""
    else:
        result_text += f"""❌ **ТЫ ПРОИГРАЛ**
💰 **Потеряно:** {bet} 💎
{format_balance(user_id)}"""

    result_text += "\n\n[⠀](https://i.ibb.co/ycyPRXrb/photo-2025-04-17-17-44-20-2.jpg)"

    # Удаляем игру
    del keno_games[user_id]

    try:
        await event.edit(result_text, buttons=[Button.inline("🎮 В меню игр", "games_menu")])
    except Exception as e:
        logging.error(f"Ошибка в keno_play_handler: {e}")
        await event.respond(result_text, buttons=[Button.inline("🎮 В меню игр", "games_menu")])


@bot.on(events.CallbackQuery(pattern="cancel_keno"))
async def cancel_keno(event):
    user_id = event.sender_id

    if user_id in keno_games:
        # Возвращаем ставку
        db.add_balance(user_id, keno_games[user_id]["bet"])
        del keno_games[user_id]

    text = "❌ Игра отменена. Ставка возвращена."
    buttons = [Button.inline("🎮 В меню игр", "games_menu")]

    try:
        await event.edit(text, buttons=buttons)
    except Exception as e:
        logging.error(f"Ошибка в cancel_keno: {e}")
        await event.respond(text, buttons=buttons)


async def start_wheel_game(event, user_id, bet):
    """Запуск игры Колесо фортуны"""

    # Возможные сектора колеса
    sectors = [0.5, 1, 1, 2, 2, 3, 3, 5, 5, 10, 25]

    wheel_games[user_id] = {
        "bet": bet,
        "sectors": sectors
    }

    text = f"""🎡 **КОЛЕСО ФОРТУНЫ** | Ставка: {bet} 💎
━━━━━━━━━━━━━━━━━━━━

{format_balance(user_id)}

🎯 **Возможные множители:**
• x0.5 (проигрыш половины)
• x1 (возврат ставки)
• x2, x3, x5, x10, x25

━━━━━━━━━━━━━━━━━━━━
Готов крутить колесо?
"""

    await event.respond(
        text,
        buttons=[
            [Button.inline("🎡 КРУТИТЬ!", f"wheel_spin_{bet}")]
        ]
    )

@bot.on(events.CallbackQuery(pattern=r"wheel_spin_(\d+)"))
async def wheel_spin_handler(event):
    user_id = event.sender_id
    bet = int(event.pattern_match.group(1))

    if user_id not in wheel_games:
        await event.answer("❌ Игра не найдена", alert=True)
        return

    # Крутим колесо
    sectors = wheel_games[user_id]["sectors"]
    multiplier = random.choice(sectors)

    # Эмодзи для результата
    if multiplier >= 10:
        result_emoji = "🏆 ДЖЕКПОТ"
    elif multiplier >= 5:
        result_emoji = "🎉 ОГО"
    elif multiplier >= 2:
        result_emoji = "👍 НЕПЛОХО"
    elif multiplier == 1:
        result_emoji = "🤝 ВЕРНУЛ СТАВКУ"
    else:
        result_emoji = "😢 УВЫ"

    # Рассчитываем результат
    if multiplier >= 1:
        win_amount = int(bet * multiplier)
        db.add_balance(user_id, win_amount)
        profit = win_amount - bet
    else:
        # x0.5 - возвращаем половину
        win_amount = int(bet * 0.5)
        db.add_balance(user_id, win_amount)
        profit = win_amount - bet

    result_text = f"""🎡 **РЕЗУЛЬТАТ**
━━━━━━━━━━━━━━━━━━━━

{result_emoji}

📊 **Множитель:** x{multiplier}
💎 **Итог:** {win_amount} 💎
📈 **Профит:** {profit:+} 💎
{format_balance(user_id)}

[⠀](https://i.ibb.co/ycyPRXrb/photo-2025-04-17-17-44-20-2.jpg)"""

    # Удаляем игру
    del wheel_games[user_id]

    try:
        await event.edit(result_text, buttons=[Button.inline("🎮 В меню игр", "games_menu")])
    except Exception as e:
        logging.error(f"Ошибка в wheel_spin_handler: {e}")
        await event.respond(result_text, buttons=[Button.inline("🎮 В меню игр", "games_menu")])


@bot.on(events.CallbackQuery(pattern="cancel_wheel"))
async def cancel_wheel(event):
    user_id = event.sender_id

    if user_id in wheel_games:
        # Возвращаем ставку
        db.add_balance(user_id, wheel_games[user_id]["bet"])
        del wheel_games[user_id]

    text = "❌ Игра отменена. Ставка возвращена."
    buttons = [Button.inline("🎮 В меню игр", "games_menu")]

    try:
        await event.edit(text, buttons=buttons)
    except Exception as e:
        logging.error(f"Ошибка в cancel_wheel: {e}")
        await event.respond(text, buttons=buttons)


@bot.on(events.CallbackQuery(pattern="games_menu"))
async def back_to_games_menu(event):
    """Возврат в меню игр"""
    user_id = event.sender_id

    # Очищаем все активные игры пользователя
    if user_id in mines_games:
        del mines_games[user_id]
    if user_id in dice_games:
        del dice_games[user_id]
    if user_id in keno_games:
        del keno_games[user_id]
    if user_id in wheel_games:
        del wheel_games[user_id]

    # Вызываем меню игр
    await games_menu(event)

@bot.on(events.NewMessage(pattern="🚫 Слить скаммера!"))
async def report_scammer(event):
    if not event.is_private:
        return  # Игнорируем, если не в ЛС

    user_id = event.sender_id

    # Проверка защиты от спама
    can_press, message = check_button_spam_protection(user_id)
    if not can_press:
        await event.respond(message)
        return

    # Показываем загрузку
    await show_button_loading(event, "меню слива скаммера")

    keyboard = types.KeyboardButtonUrl(text="🚨 Отправить жалобу", url="https://t.me/Infinityantiscam")

    # Удаляем сообщение о загрузке если есть
    if user_id in button_loading_messages:
        try:
            await bot.delete_messages(event.chat_id, button_loading_messages[user_id])
            del button_loading_messages[user_id]
        except:
            pass

    await event.respond(
        """🔥 Вы хотите слить скаммера? 🔥

⚡️ Лучшее решение:
• Нажмите кнопку "🚨 Отправить жалобу"
• Наш персонал примет меры в течение некоторого времени, просто скиньте пруфы!

🔒 Как избежать скама?:
1. ✅ Всегда проверяйте пользователя через /check
2. ✅ Используйте только официальных гарантов
3. ✅ Требуйте подтверждающие скриншоты
4. ✅ При малейших сомнениях - отменяйте сделку

📛 Помните: 95% скама можно избежать, следуя этим правилам!
[⠀](https://i.ibb.co/bj4g7h3y/photo-2025-04-17-17-44-19-3.jpg)""",
        parse_mode='md',
        link_preview=True,
        buttons=keyboard
    )


@bot.on(events.NewMessage(pattern="✅ Гаранты базы"))
async def list_garants(event):
    if not event.is_private:
        return

    user_id = event.sender_id

    # Проверка защиты от спама
    can_press, message =  check_button_spam_protection(user_id)
    if not can_press:
        await event.respond(message)
        return

    # Показываем загрузку
    await show_button_loading(event, "список гарантов")

    # Получаем всех гарантов из базы
    try:
        garants = [row[0] for row in db.cursor.execute('SELECT user_id FROM users WHERE role_id = 1')]
    except Exception as e:
        # Удаляем сообщение о загрузке если есть
        if user_id in button_loading_messages:
            try:
                await bot.delete_messages(event.chat_id, button_loading_messages[user_id])
                del button_loading_messages[user_id]
            except:
                pass
        return

    if not garants:
        # Удаляем сообщение о загрузке если есть
        if user_id in button_loading_messages:
            try:
                await bot.delete_messages(event.chat_id, button_loading_messages[user_id])
                del button_loading_messages[user_id]
            except:
                pass
        await event.respond("На данный момент Гарантов нету ⛔")
        return

    # Формируем текст с актуальным списком
    text = f"""💢 Актуальный список гарантов Infinity
━━━━━━━━━━━━━━
• Всего: {len(garants)}
━━━━━━━━━━━━━━
💡 Если хотите стать гарантом, пройдите набор!
[⠀](https://i.ibb.co/rGBBGyng/photo-2025-04-17-17-44-20.jpg)"""

    buttons = []

    for uid in garants:
        try:
            user = await bot.get_entity(uid)
            buttons.append([Button.inline(f"🛡️ {user.first_name}", f"check_{uid}")])
        except Exception as e:
            print(f"Ошибка при получении данных пользователя {uid}: {e}")
            continue

    # Удаляем сообщение о загрузке если есть
    if user_id in button_loading_messages:
        try:
            await bot.delete_messages(event.chat_id, button_loading_messages[user_id])
            del button_loading_messages[user_id]
        except:
            pass

    await event.respond(text, buttons=buttons, parse_mode='md', link_preview=True)


@bot.on(events.NewMessage(pattern="👨‍🎓 Волонтёры базы"))
async def list_volunteers(event):
    if not event.is_private:
        return

    user_id = event.sender_id

    # Проверка защиты от спама
    can_press, message =  check_button_spam_protection(user_id)
    if not can_press:
        await event.respond(message)
        return

    # Показываем загрузку
    await show_button_loading(event, "список волонтёров")

    # Получаем всех волонтеров (роли 6-10)
    volunteers = []
    for role_id in [6, 7, 8, 9, 10, 13]:  # Добавлена роль 13 (Айдош)
        volunteers.extend(
            [row[0] for row in db.cursor.execute('SELECT user_id FROM users WHERE role_id = ?', (role_id,))])

    if not volunteers:
        # Удаляем сообщение о загрузке если есть
        if user_id in button_loading_messages:
            try:
                await bot.delete_messages(event.chat_id, button_loading_messages[user_id])
                del button_loading_messages[user_id]
            except:
                pass
        await event.respond("На данный момент Волонтёров нету ⛔")
        return

    # Формируем текст с актуальным списком
    text = f"""🤝 Актуальный список волонтёров Infinity
━━━━━━━━━━━━━━
• Всего: {len(volunteers)}
━━━━━━━━━━━━━━
💡 Если вы хотите стать волонтёром базы, просто пройдите набор!
[⠀](https://i.ibb.co/rGKnW46r/photo-2025-04-17-17-44-19.jpg)"""

    buttons = []

    for uid in volunteers:
        try:
            user = await bot.get_entity(uid)
            role_id = db.get_user_role(uid)
            role_name = ROLES[role_id]["name"]
            buttons.append([Button.inline(f"{role_name} {user.first_name}", f"check_{uid}")])
        except:
            continue

    # Удаляем сообщение о загрузке если есть
    if user_id in button_loading_messages:
        try:
            await bot.delete_messages(event.chat_id, button_loading_messages[user_id])
            del button_loading_messages[user_id]
        except:
            pass

    await event.respond(text, buttons=buttons, parse_mode='md', link_preview=True)


@bot.on(events.NewMessage(pattern="🔰 Проверенные пользователи"))
@subscription_required
async def list_verified_users(event):
    if not event.is_private:
        return

    user_id = event.sender_id

    # Проверка защиты от спама
    can_press, message =  check_button_spam_protection(user_id)
    if not can_press:
        await event.respond(message)
        return

    # Показываем загрузку
    await show_button_loading(event, "проверенных пользователей")

    # Получаем всех проверенных пользователей (роль 12)
    verified_users = [row[0] for row in db.cursor.execute('SELECT user_id FROM users WHERE role_id = 12')]

    if not verified_users:
        # Удаляем сообщение о загрузке если есть
        if user_id in button_loading_messages:
            try:
                await bot.delete_messages(event.chat_id, button_loading_messages[user_id])
                del button_loading_messages[user_id]
            except:
                pass
        await event.respond("На данный момент проверенных пользователей нет ⛔")
        return

    text = "📊 Вот наш список проверенных пользователей:\n"
    buttons = []

    for uid in verified_users:
        try:
            user = await bot.get_entity(uid)
            buttons.append([Button.inline(f"✅ {user.first_name}", f"check_{uid}")])
        except:
            continue

    # Удаляем сообщение о загрузке если есть
    if user_id in button_loading_messages:
        try:
            await bot.delete_messages(event.chat_id, button_loading_messages[user_id])
            del button_loading_messages[user_id]
        except:
            pass

    await event.respond(text, buttons=buttons, parse_mode='md', link_preview=True)


# Обработчик команды /bt (показать кнопки)
@bot.on(events.NewMessage(pattern='/bt'))
async def show_buttons(event):
    # Проверяем права (только создатель, кодер или владельцы)
    user_id = event.sender_id
    user_role = db.get_user_role(user_id)

    if user_id not in OWNER_ID and user_role not in [10, 11]:
        await event.respond("❌ У вас нет прав для использования этой команды")
        return

    await event.respond(
        "Кнопки активированы ✅",
        buttons=get_main_buttons(user_id)
    )


# Обработчик команды /unbt (убрать кнопки)
@bot.on(events.NewMessage(pattern='/unbt'))
async def remove_buttons(event):
    # Проверяем права (только создатель, кодер или владельцы)
    user_id = event.sender_id
    user_role = db.get_user_role(user_id)

    if user_id not in OWNER_ID and user_role not in [10, 11]:
        await event.respond("❌ У вас нет прав для использования этой команды")
        return

    await event.respond(
        "Кнопки деактивированы ✅",
        buttons=[]  # Пустой список кнопок
    )


@bot.on(events.NewMessage(pattern='/оффтоп'))
async def handle_offtopic_command(event):
    # Проверяем права пользователя
    allowed_roles = [1, 6, 7, 8, 9, 10]  # 1 - владелец, 6 - стажер, 7 - админ, 8 - директор, 9 - президент
    if event.sender_id not in OWNER_ID and db.get_user_role(event.sender_id) not in allowed_roles:
        await event.respond("❌ У вас нет прав для использования этой команды.")
        return

    if event.is_reply:
        replied = await event.get_reply_message()
        target_user = await event.client.get_entity(replied.sender_id)
        try:
            # Выдаем мут на 30 минут
            await bot.edit_permissions(
                event.chat_id,
                target_user.id,
                until_date=time.time() + 1800,
                send_messages=False
            )
            # Формируем текст ответа
            mute_message = (
                f"{target_user.first_name} выдан мут на 30 минут\n\n"
                f"Причина: Оффтоп\n\n"
                f"общайтесь в нашем чате для оффтопа☕"
            )

            # Создаем кнопку
            keyboard = [
                [Button.url("Перейти", "https://t.me/steal_a_brainrotchat1")]
            ]

            # Отправляем сообщение с кнопкой
            await event.respond(mute_message, buttons=keyboard)

            # Удаляем сообщение, на которое был дан ответ
            await replied.delete()
        except Exception as e:
            await event.respond(f"❌ Не могу выдать мут: {e}")
    else:
        await event.respond("❌ Ответьте на сообщение пользователя, которому нужно выдать мут.")


# Обработчик сообщений для проверки мута
@bot.on(events.NewMessage())
async def check_message(event):
    user_id = event.sender_id

    # Проверка, замучен ли пользователь
    if user_id in muted_users:
        expiry_time = muted_users[user_id]
        if time.time() < expiry_time:
            await event.delete()  # Удаляем сообщение
            await event.respond("❌ Вы замучены и не можете отправлять сообщения.")
            return


joined_users_cache = set()


@bot.on(events.ChatAction)
async def handle_chat_join(event):
    if not (event.user_joined or event.user_added):
        return  # Не обрабатываем другие действия

    user = await event.get_user()
    user_id = user.id

    # Исключаем ботов
    if user.bot:
        return

    # Кэш от повторов
    if user_id in joined_users_cache:
        return
    joined_users_cache.add(user_id)
    asyncio.create_task(remove_from_cache_later(user_id))

    # УПРОЩЕННОЕ ПРИВЕТСТВИЕ ДЛЯ ВСЕХ РОЛЕЙ
    text = f"""
👋 Добро пожаловать! [{user.first_name}](tg://user?id={user.id})

[🤗](https://i.ibb.co/q3qgMsQz/photo-2025-04-17-17-44-18.jpg)
"""
    await event.respond(text, parse_mode='md', link_preview=True)

    # Проверка мута
    if user_id in muted_users:
        expiry_time = muted_users[user_id]
        if time.time() < expiry_time:
            await bot.edit_permissions(event.chat_id, user_id, view_messages=False)
        else:
            del muted_users[user_id]


# Очистка кэша
async def remove_from_cache_later(user_id, delay=600):
    await asyncio.sleep(delay)
    joined_users_cache.discard(user_id)


# Обработчик кнопки "Добавить в группу"
@bot.on(events.CallbackQuery(pattern='add_group'))
async def add_group_handler(event):
    url = "https://t.me/ROBLOXpvsb_bot?startgroup=newgroup&admin=manage_chat+delete_messages+restrict_members+invite_users+restrict_members+change_info+pin_messages+manage_video_chats"
    keyboard = types.KeyboardButtonUrl(text="добавить в группу", url=url)
    await event.edit(
        "Нажмите кнопку ниже, чтобы добавить бота в группу:",
        buttons=keyboard
    )


# Обработчик кнопки "Пожаловаться на скамера"
@bot.on(events.CallbackQuery(pattern='report_scammer'))
async def report_handler(event):
    await event.respond(
        "Для того что бы пожаловатся на скамера вы должны перейти в наш [специальный чат](https://t.me/InfinityAntiScam)\nВам нужны пруфы и скрины переписок!")


@bot.on(events.CallbackQuery(pattern="infinix_balance"))
async def show_balance(event):
    """Показывает баланс пользователя"""
    user_id = event.sender_id
    balance = db.get_balance(user_id)

    # Проверяем ежедневный бонус
    can_claim, claim_info = db.can_claim_daily(user_id)
    daily_status = "✅ Доступен!" if can_claim else f"⏳ Через {claim_info}"

    text = f"""💰 **ТВОЙ БАЛАНС**
💎 **Баланс:** `{balance}` Инфиниксов
🎁 **Следующий бонус:** {daily_status}"""

    buttons = []
    if can_claim:
        buttons.append([Button.inline("🎁 Забрать бонус 100 💎", "claim_daily")])
    buttons.append([Button.inline("🎮 В меню игр", "games_menu")])
    buttons.append([Button.inline("◀ Назад в профиль", "back_to_profile")])

    await event.respond(text, buttons=buttons, parse_mode='md')
    await event.answer()


@bot.on(events.CallbackQuery(pattern="claim_daily"))
async def claim_daily_handler(event):
    user_id = event.sender_id

    # Убеждаемся, что у пользователя есть стартовый капитал
    db.ensure_start_balance(user_id)

    # Пытаемся получить бонус
    success, result = db.claim_daily(user_id)

    if success:
        amount, new_balance, next_time = result
        await event.answer(f"✅ Получено {amount} 💎! Новый баланс: {new_balance} 💎", alert=True)

        # Отправляем новое сообщение вместо редактирования
        text = f"""🎁 **БОНУС ПОЛУЧЕН!**
━━━━━━━━━━━━━━━━━━━━

💰 **Получено:** +{amount} 💎
💎 **Новый баланс:** {new_balance} 💎
⏰ **Следующий бонус:** {next_time}

━━━━━━━━━━━━━━━━━━━━"""

        await event.respond(text, buttons=[Button.inline("🎮 В меню игр", "games_menu")])
    else:
        # Не прошло 24 часа
        await event.answer(f"⏳ Бонус будет доступен через {result}", alert=True)

# Обработчик кнопки "Создатель"
@bot.on(events.CallbackQuery(pattern='creator'))
async def creator_handler(event):
    user_id = event.original_update.user_id
    try:
        user = await bot.get_entity(user_id)
        username = user.username
        if username:
            user_info = f"@{username}"
        else:
            user_info = f"ID: {user_id}"
    except Exception as e:
        user_info = f"ID: {user_id} (не удалось получить имя)"

    await event.edit(f"{user_info}, вот информация:\nСоздатель - @half50k\nКодер - @MyNameIsLiner")


# Обработчик кнопки "Тема проверки"
@bot.on(events.CallbackQuery(pattern='themes_soon'))
async def themes_handler(event):
    # Списки фотографий для различных статусов
    status_photos = {
        6: [  # Стажер
            "https://cdn.streamable.com/video/mp4/z1j4w6.mp4",
            "https://i.ibb.co/jPQpWgg3/temp-5173733679-1248.jpg",
            "https://i.ibb.co/JRFhpf2d/temp-5173733679-1294.jpg",
            "https://i.ibb.co/dwXYzYvV/temp-5173733679-1312.jpg"
        ],
        8: [  # Директор
            "https://i.ibb.co/Z6qKqwvY/temp-5173733679.jpg",
            "https://i.ibb.co/XfYFmf8n/temp-5173733679-1178.jpg",
            "https://i.ibb.co/ynNp17dG/1.jpg"
        ],
        7: [  # Админ
            "https://i.ibb.co/VWYdQrwK/temp-5173733679-1310.jpg",
            "https://i.ibb.co/hRNMk3Pg/temp-5173733679-1295.jpg",
            "https://i.ibb.co/Y7fZWqkY/temp-5173733679-1183.jpg",
            "https://i.ibb.co/PbN53Mj/image.jpg",
            "https://i.ibb.co/7NXdHPd5/image.jpg"
        ],
        9: [  # Президент
            "https://i.ibb.co/d4jHKRZC/temp-5173733679-1311.jpg",
            "https://i.ibb.co/pjYcnsHk/temp-5173733679-1182.jpg",
            "https://i.ibb.co/Z1XrK4sB/image.jpg",
            "https://i.ibb.co/fYjWwYwH/1.jpg"
        ],
        0: [  # Нет в базе
            "https://i.ibb.co/qYfWnnvY/temp-5173733679-1176.jpg",
            "https://i.ibb.co/23G4pXk6/temp-5173733679.jpg",
            "https://i.ibb.co/RpfWS3Q0/image.jpg",
            "https://i.ibb.co/YB8849FG/temp-5173733679-1309.jpg"
        ]
    }

    user_id = event.sender_id
    role_id = db.get_user_role(user_id)

    # Получаем фотографии для текущего статуса пользователя
    photos = status_photos.get(role_id, [])
    current_index = 0

    if not photos:
        await event.respond("📸 У вас нет доступных фотографий для выбора.")
        return

    # Функция для отправки фото
    async def send_photo(index):
        if index < 0 or index >= len(photos):
            return  # Проверка выхода за пределы списка
        await event.respond(
            f"📸 Выберите фото для статуса:\n\n"
            f"[❤]({photos[index]})",
            buttons=[
                [
                    Button.inline("◀", f"photo_prev_{index}"),
                    Button.inline("Выбрать!", f"select_photo_{index}"),
                    Button.inline("▶", f"photo_next_{index}")
                ]
            ],
            link_preview=True
        )

    # Отправляем первое фото
    await send_photo(current_index)

    # Обработчик для кнопки "Выбрать!"
    @bot.on(events.CallbackQuery(pattern=r'select_photo_(\d+)'))
    async def select_photo_handler(event):
        index = int(event.pattern_match.group(1))
        user_id = event.sender_id

        # Сохраняем фото как кастомное для пользователя
        db.cursor.execute('UPDATE users SET custom_photo_url = ? WHERE user_id = ?', (photos[index], user_id))
        db.conn.commit()

        await event.respond(f"✅ Новое фото успешно установлено в статус!")

    # Обработчик для кнопки "◀"
    @bot.on(events.CallbackQuery(pattern=r'photo_prev_(\d+)'))
    async def photo_prev_handler(event):
        index = int(event.pattern_match.group(1)) - 1
        await send_photo(index)

    # Обработчик для кнопки "▶"
    @bot.on(events.CallbackQuery(pattern=r'photo_next_(\d+)'))
    async def photo_next_handler(event):
        index = int(event.pattern_match.group(1)) + 1
        await send_photo(index)


@bot.on(events.CallbackQuery(pattern='check_soon'))
async def check_soon_handler(event):
    try:
        user = await event.client.get_entity(event.sender_id)
        user_id = user.id

        # Получаем текущую роль пользователя
        current_role_id = db.get_user_role(user_id)

        # Добавляем запись о проверке
        db.add_check(user_id, user_id)

        current_time = datetime.now()
        role_info = ROLES[current_role_id]

        # Получаем данные пользователя из базы
        user_data = db.get_user(user_id)
        country = user_data[5] if user_data and user_data[5] else "Не указана"
        channel = user_data[6] if user_data and user_data[6] else None
        custom_photo = user_data[7] if user_data and user_data[7] else None

        # Проверяем, изменилась ли роль пользователя
        new_role_id = db.get_user_role(user_id)  # Получаем новую роль
        if new_role_id != current_role_id:  # Если роль изменилась
            custom_photo = None  # Сбрасываем фото
            db.cursor.execute('UPDATE users SET custom_photo_url = ? WHERE user_id = ?', (custom_photo, user_id))
            db.conn.commit()

        response = (
            f"👤 | Пользователь: [{user.first_name}](tg://user/{user.id})\n\n"
            f"🔍 | ID: `{user.id}`\n\n"
            f"🤗 | Роль в базе: {role_info['name']}\n\n"
            f"🌍 | Страна: {country}\n\n"
            f"📢 | Канал: {channel}\n\n"
            f"⚖ | Шанс скама: {role_info['scam_chance']}%\n\n"
            f"📅 {current_time.strftime('%d.%m.%Y')} | 🔍 {db.get_check_count(user_id)}\n\n"
            f"[Просмотреть медиа]({custom_photo if custom_photo else role_info['preview_url']})"
        )

        buttons = [
            [
                Button.url("👤 Профиль",
                           f"https://t.me/{user.username}" if user.username else f"tg://user?id={user.id}"),
                Button.url("🔗 Ссылка", f"https://t.me/{user.username}" if user.username else f"tg://user?id={user.id}")
            ],
            [
                Button.url("⚠️ Слить скаммера", "https://t.me/Infinityantiscam"),
                Button.url("⚖️ Аппеляция", "https://t.me/appelICE")
            ]
        ]

        # ВМЕСТО event.edit ИСПОЛЬЗУЕМ event.respond для нового сообщения
        await event.respond(response, buttons=buttons, parse_mode='md')

        # Подтверждаем callback чтобы убрать "часики" на кнопке
        await event.answer()

    except Exception as e:
        print(f"Ошибка в check_soon_handler: {e}")
        await event.answer("❌ Произошла ошибка при обработке запроса", alert=True)


@bot.on(events.NewMessage(pattern="🎭 Профиль"))
@subscription_required
async def my_profile(event):
    if not event.is_private:
        await event.delete()
        return

    user_id = event.sender_id

    # Проверка защиты от спама
    can_press, message = check_button_spam_protection(user_id)
    if not can_press:
        await event.respond(message)
        return

    # Показываем загрузку
    await show_button_loading(event, "профиль")

    # Получаем данные пользователя из базы
    user_data = db.get_user(user_id)
    if user_data is None:
        # Если пользователя нет в базе, добавляем
        db.add_user(user_id, None, 0)
        user_data = db.get_user(user_id)

    role_id = user_data[2]
    role_info = ROLES[role_id]

    # Получаем объект пользователя
    user = await event.get_sender()
    checks_count = db.get_check_count(user_id)

    # Получаем баланс (с выдачей стартового капитала при необходимости)
    balance = db.ensure_start_balance(user_id)

    # ПОЛНОСТЬЮ ИЗМЕНЯЕМ ФОРМИРОВАНИЕ ТЕКСТА ПРОФИЛЯ
    profile_message = f"""
👤 **Профиль пользователя** [{user.first_name}](tg://user/{user_id})

📛 **Роль:** {role_info['name']}
🆔 **ID:** {user_id}
💰 **Баланс:** {balance} 💎 Инфиниксов
🔍 **Проверок:** {checks_count}
[⠀](https://i.ibb.co/ycyPRXrb/photo-2025-04-17-17-44-20-2.jpg)
"""

    # СОЗДАЕМ КНОПКИ ДЛЯ ПРОФИЛЯ
    profile_buttons = [
        [Button.inline("🔎 Проверить себя", "check_soon"),
         Button.inline("🎨 Тема проверки", "themes_soon")],
        [Button.inline("📢 Канал", "channel_soon"),
         Button.inline("🌍 Страна", "country_soon")],
        [Button.inline("💎 Мой баланс", "infinix_balance")]
    ]

    # Удаляем сообщение о загрузке если есть
    if user_id in button_loading_messages:
        try:
            await bot.delete_messages(event.chat_id, button_loading_messages[user_id])
            del button_loading_messages[user_id]
        except:
            pass

    # Отправляем сообщение с ИМЕНОВАННЫМИ переменными
    await event.respond(
        profile_message,
        buttons=profile_buttons,
        parse_mode='md',
        link_preview=True
    )

# ============ ЭТА ФУНКЦИЯ ДОЛЖНА БЫТЬ ЗДЕСЬ, ПОСЛЕ my_profile ============
@bot.on(events.CallbackQuery(pattern="back_to_profile"))
async def back_to_profile_handler(event):
    """Возврат в профиль"""
    user_id = event.sender_id

    # Удаляем текущее сообщение
    try:
        await event.delete()
    except:
        pass

    # Получаем данные пользователя из базы
    user_data = db.get_user(user_id)
    if user_data is None:
        await event.respond("❌ Не удалось найти ваши данные в базе.")
        return

    # Получаем объект пользователя
    try:
        user = await event.client.get_entity(user_id)
    except:
        # Если не получается получить пользователя, создаем минимальный объект
        user = type('UserObject', (), {
            'id': user_id,
            'first_name': f"Пользователь",
            'username': None
        })()

    role_id = user_data[2] if len(user_data) > 2 else 0
    role_info = ROLES.get(role_id, ROLES[0])

    checks_count = db.get_check_count(user_id)
    balance = db.get_balance(user_id)

    # Формируем текст профиля
    profile_text = f"""
👤 **Профиль пользователя** [{user.first_name}](tg://user/{user_id})

📛 **Роль:** {role_info['name']}
🆔 **ID:** {user_id}
💰 **Баланс:** {balance} 💎 Инфиниксов
🔍 **Проверок:** {checks_count}
[⠀](https://i.ibb.co/ycyPRXrb/photo-2025-04-17-17-44-20-2.jpg)
"""

    # Создаем кнопки профиля
    profile_buttons = [
        [Button.inline("🔎 Проверить себя", "check_soon"),
         Button.inline("🎨 Тема проверки", "themes_soon")],
        [Button.inline("📢 Канал", "channel_soon"),
         Button.inline("🌍 Страна", "country_soon")],
        [Button.inline("💎 Мой баланс", "infinix_balance")]
    ]

    # Отправляем сообщение
    await event.respond(
        profile_text,
        buttons=profile_buttons,
        parse_mode='md',
        link_preview=True
    )

@bot.on(events.CallbackQuery(pattern='channel_soon'))
async def channel_soon_handler(event):
    user_id = event.sender_id

    # Проверка премиум-статуса
    if not db.is_premium_user(user_id):
        # Всплывающее уведомление вместо сообщения в чат
        await event.answer(
            "❌ У вас нет премиум статуса! Для установки канала приобретите премиум.",
            alert=True  # Ключевой параметр для всплывающего окна
        )
        return

    # Остальной код без изменений
    await event.respond("Отправьте username канала (например, @channelname)")

    @bot.on(events.NewMessage(from_users=user_id))
    async def channel_handler(channel_event):
        channel_name = channel_event.text.strip()
        if not channel_name.startswith('@'):
            await channel_event.reply("❌ Имя канала должно начинаться с @")
        elif len(channel_name) > 32:
            await channel_event.reply("❌ Имя канала слишком длинное (макс. 32 символа)")
        else:
            db.update_user(channel_event.sender_id, channel=channel_name)
            await channel_event.reply(f"✅ Канал {channel_name} успешно сохранен!")
        bot.remove_event_handler(channel_handler)


# Обработчик кнопки "Страна"
@bot.on(events.CallbackQuery(pattern='country_soon'))
async def country_soon_handler(event):
    countries = [
        "США 🇺🇸", "Канада 🇨🇦", "Мексика 🇲🇽", "Бразилия 🇧🇷",
        "Аргентина 🇦🇷", "Великобритания 🇬🇧", "Франция 🇫🇷",
        "Германия 🇩🇪", "Италия 🇮🇹", "Испания 🇪🇸", "Китай 🇨🇳",
        "Япония 🇯🇵", "Австралия 🇦🇺", "Индия 🇮🇳", "Россия 🇷🇺",
        "Южноафриканская Республика 🇿🇦", "Египет 🇪🇬", "ОАЭ 🇦🇪",
        "Турция 🇹🇷", "Греция 🇬🇷", "Швеция 🇸🇪", "Норвегия 🇳🇴",
        "Финляндия 🇫🇮", "Дания 🇩🇰", "Польша 🇵🇱", "Чехия 🇨🇿",
        "Австрия 🇦🇹", "Швейцария 🇨🇭", "Нидерланды 🇳🇱", "Бельгия 🇧🇪",
        "Ирландия 🇮🇪", "Португалия 🇵🇹", "Румыния 🇷🇴", "Словакия 🇸🇰",
        "Словения 🇸🇮", "Хорватия 🇭🇷", "Латвия 🇱🇻", "Литва 🇱🇹",
        "Эстония 🇪🇪", "Мальта 🇲🇹", "Кипр 🇨🇾", "Исландия 🇮🇸",
        "Албания 🇦🇱", "Сербия 🇷🇸", "Босния и Герцеговина 🇧🇦",
        "Черногория 🇲🇪", "Македония 🇲🇰", "Косово 🇽🇰", "Беларусь 🇧🇾",
        "Украина 🇺🇦", "Грузия 🇬🇪", "Армения 🇦🇲", "Азербайджан 🇦🇿",
        "Казахстан 🇰🇿", "Узбекистан 🇺🇿", "Таджикистан 🇹🇯",
        "Туркменистан 🇹🇲", "Кыргызстан 🇰🇬", "Монголия 🇲🇳",
        "Иран 🇮🇷", "Ирак 🇮🇶", "Сирия 🇸🇾", "Ливан 🇱🇧",
        "Иордания 🇯🇴", "Катар 🇶🇦", "Бахрейн 🇧🇭", "Кувейт 🇰🇼",
        "Саудовская Аравия 🇸🇦", "Йемен 🇾🇪", "Вьетнам 🇻🇳"
    ]

    buttons = [Button.inline(country, f"set_country_{i}")
               for i, country in enumerate(countries)]

    # Создаём новое сообщение с кнопками выбора страны
    await event.respond(
        "🌍 Выберите страну, выбраная вами страна будет стоять у вас в профиле!",
        buttons=[buttons[i:i + 3] for i in range(0, len(buttons), 3)]
    )


# Обработчик выбора страны
@bot.on(events.CallbackQuery(pattern=r'set_country_(\d+)'))
async def set_country_handler(event):
    country_idx = int(event.data.decode().split('_')[2])
    country = [
        "США 🇺🇸", "Канада 🇨🇦", "Мексика 🇲🇽", "Бразилия 🇧🇷",
        "Аргентина 🇦🇷", "Великобритания 🇬🇧", "Франция 🇫🇷",
        "Германия 🇩🇪", "Италия 🇮🇹", "Испания 🇪🇸", "Китай 🇨🇳",
        "Япония 🇯🇵", "Австралия 🇦🇺", "Индия 🇮🇳", "Россия 🇷🇺",
        "Южноафриканская Республика 🇿🇦", "Египет 🇪🇬", "ОАЭ 🇦🇪",
        "Турция 🇹🇷", "Греция 🇬🇷", "Швеция 🇸🇪", "Норвегия 🇳🇴",
        "Финляндия 🇫🇮", "Дания 🇩🇰", "Польша 🇵🇱", "Чехия 🇨🇿",
        "Австрия 🇦🇹", "Швейцария 🇨🇭", "Нидерланды 🇳🇱", "Бельгия 🇧🇪",
        "Ирландия 🇮🇪", "Португалия 🇵🇹", "Румыния 🇷🇴", "Словакия 🇸🇰",
        "Словения 🇸🇮", "Хорватия 🇭🇷", "Латвия 🇱🇻", "Литва 🇱🇹",
        "Эстония 🇪🇪", "Мальта 🇲🇹", "Кипр 🇨🇾", "Исландия 🇮🇸",
        "Албания 🇦🇱", "Сербия 🇷🇸", "Босния и Герцеговина 🇧🇦",
        "Черногория 🇲🇪", "Македония 🇲🇰", "Косово 🇽🇰", "Беларусь 🇧🇾",
        "Украина 🇺🇦", "Грузия 🇬🇪", "Армения 🇦🇲", "Азербайджан 🇦🇿",
        "Казахстан 🇰🇿", "Узбекистан 🇺🇿", "Таджикистан 🇹🇯",
        "Туркменистан 🇹🇲", "Кыргызстан 🇰🇬", "Монголия 🇲🇳",
        "Иран 🇮🇷", "Ирак 🇮🇶", "Сирия 🇸🇾", "Ливан 🇱🇧",
        "Иордания 🇯🇴", "Катар 🇶🇦", "Бахрейн 🇧🇭", "Кувейт 🇰🇼",
        "Саудовская Аравия 🇸🇦", "Йемен 🇾🇪", "Вьетнам 🇻🇳"
    ][country_idx]

    db.update_user(event.sender_id, country=country)

    # Создаём новое сообщение с подтверждением
    await event.respond(f"✅ Страна установлена: {country}")


# Обработчик кнопки "Помощь"
@bot.on(events.CallbackQuery(pattern='help_soon'))
async def help_soon_handler(event):
    help_text = """
🤖 **Команды бота:**

📋 **Проверка пользователей:**
• `Чек [юзернейм/ID]` - проверить пользователя
• `Чек` (ответом на сообщение) - проверить пользователя
• `Чек ми/я/себя` - проверить себя

👮‍♂️ **Выдача ролей (только для админов):**
• `+роль` (ответом на сообщение)
• `-роль` (снять роль)

📊 **Другие команды:**
• `/profile` - ваш профиль
• `/stats` - статистика бота
• `/report` - пожаловаться на скамера
"""
    await event.edit(help_text, buttons=[Button.inline("« Назад", "back_to_profile")])


# ============ ФУНКЦИИ РАССЫЛКИ ============

async def send_broadcast_message():
    """Функция для отправки сообщения рассылки в указанный чат"""
    global broadcast_active, BROADCAST_INTERVAL

    while broadcast_active:
        try:
            # Текст сообщения
            text = """➕ Infinity - надёжная анти-скам база для проверки пользователей!

Проект нацелен обезопасить вас и ваши сделки от недобросовестных скамеров, сливая скамеров — вы помогаете развивать проект, обезопасив себя и других пользователей!

💬 Как проверять на скам, показано на фото ниже:

[⠀](https://files.catbox.moe/zs50do.jpg)"""

            # Кнопки
            buttons = [
                [Button.url("💬 Предложка", "https://t.me/InfinityAntiScam")],
                [Button.url("📢 Новостник", "https://t.me/InfinityReportNews")]
            ]

            # Отправляем сообщение только в указанный чат
            try:
                logging.info(f"🔄 Пытаюсь отправить рассылку в чат ID: {BROADCAST_CHAT_ID}")
                message = await bot.send_message(
                    BROADCAST_CHAT_ID,
                    text,
                    buttons=buttons,
                    parse_mode='md',
                    link_preview=True
                )
                logging.info(f"✅ Рассылка успешно отправлена! ID сообщения: {message.id}")
                logging.info(f"⏱ Следующая рассылка через {BROADCAST_INTERVAL // 60} минут")
            except Exception as e:
                error_msg = str(e)
                logging.error(f"❌ ОШИБКА отправки рассылки:")
                logging.error(f"   Чат ID: {BROADCAST_CHAT_ID}")
                logging.error(f"   Тип ошибки: {type(e).__name__}")
                logging.error(f"   Подробности: {error_msg}")

                # Проверяем специфические ошибки
                if "CHAT_WRITE_FORBIDDEN" in error_msg or "no rights" in error_msg.lower():
                    logging.error("   🔒 У бота нет прав для отправки сообщений в этот чат!")
                elif "PEER_ID_INVALID" in error_msg:
                    logging.error("   ❌ Неверный ID чата!")
                elif "CHANNEL_PRIVATE" in error_msg:
                    logging.error("   🔒 Бот не является участником этого канала/чата!")

            # Ждем указанный интервал перед следующей рассылкой
            logging.info(f"⏳ Ожидание {BROADCAST_INTERVAL} секунд до следующей рассылки...")
            await asyncio.sleep(BROADCAST_INTERVAL)

        except Exception as e:
            logging.error(f"Критическая ошибка в функции рассылки: {e}")
            await asyncio.sleep(60)  # Подождать минуту при ошибке


@bot.on(events.NewMessage(pattern='/testbroadcast'))
async def test_broadcast(event):
    """Команда для тестовой отправки сообщения рассылки"""
    user_id = event.sender_id
    user_role = db.get_user_role(user_id)

    # Проверяем права (только создатель, кодер или администраторы)
    if user_id not in OWNER_ID and user_role not in [10]:
        await event.respond("❌ У вас нет прав для тестирования рассылки!")
        return

    try:
        # Текст сообщения
        text = """➕ Infinity - надёжная анти-скам база для проверки пользователей!

Проект нацелен обезопасить вас и ваши сделки от недобросовестных скамеров, сливая скамеров — вы помогаете развивать проект, обезопасив себя и других пользователей!

💬 Как проверять на скам, показано на фото ниже:

[⠀](https://files.catbox.moe/zs50do.jpg)"""

        # Кнопки
        buttons = [
            [Button.url("💬 Предложка", "https://t.me/InfinityAntiScam")],
            [Button.url("📢 Новостник", "https://t.me/InfinityReportNews")]
        ]

        # Пытаемся отправить тестовое сообщение
        await event.respond("🔄 Пытаюсь отправить тестовое сообщение...")

        try:
            message = await bot.send_message(
                BROADCAST_CHAT_ID,
                text,
                buttons=buttons,
                parse_mode='md',
                link_preview=True
            )
            await event.respond(f"✅ Тестовое сообщение успешно отправлено!\nID сообщения: {message.id}")
        except Exception as e:
            error_msg = str(e)
            await event.respond(f"❌ ОШИБКА отправки:\n\nТип: {type(e).__name__}\n\nОшибка: {error_msg}")

    except Exception as e:
        await event.respond(f"❌ Произошла ошибка: {str(e)}")

@bot.on(events.NewMessage(pattern=r'/setbroadcastinterval (\d+)'))
async def set_broadcast_interval(event):
    """Команда для изменения интервала рассылки (только для администраторов)"""
    user_id = event.sender_id
    user_role = db.get_user_role(user_id)

    # Проверяем права (только создатель, кодер или администраторы)
    if user_id not in OWNER_ID and user_role not in [10]:
        await event.respond("❌ У вас нет прав для изменения интервала рассылки!")
        return

    try:
        # Получаем новый интервал из аргументов команды
        args = event.raw_text.split()
        if len(args) < 2:
            await event.respond("❌ Использование: `/setbroadcastinterval <минуты>`")
            return

        new_interval_minutes = int(args[1])

        # Проверяем минимальный интервал (не менее 1 минуты)
        if new_interval_minutes < 1:
            await event.respond("❌ Интервал не может быть меньше 1 минуты!")
            return

        # Проверяем максимальный интервал (не более 24 часов = 1440 минут)
        if new_interval_minutes > 1440:
            await event.respond("❌ Интервал не может быть больше 24 часов!")
            return

        global BROADCAST_INTERVAL
        old_interval = BROADCAST_INTERVAL
        BROADCAST_INTERVAL = new_interval_minutes * 60  # Конвертируем минуты в секунды

        await event.respond(
            f"✅ Интервал рассылки изменен!\n\n"
            f"📊 Было: {old_interval // 60} минут\n"
            f"📊 Стало: {new_interval_minutes} минут\n\n"
            f"⏱ Следующее сообщение будет отправлено через {new_interval_minutes} минут"
        )

        logging.info(f"Интервал рассылки изменен: {old_interval // 60} минут → {new_interval_minutes} минут")

    except ValueError:
        await event.respond("❌ Неверный формат! Используйте число минут (например: 10)")
    except Exception as e:
        logging.error(f"Ошибка при изменении интервала рассылки: {e}")
        await event.respond("❌ Произошла ошибка при изменении интервала")


# Функция для автоматического запуска рассылки при старте бота
async def auto_start_broadcast():
    """Автоматический запуск рассылки при старте бота"""
    await asyncio.sleep(10)  # Ждем 10 секунд для инициализации бота
    global broadcast_active
    broadcast_active = True
    logging.info("🔄 Автозапуск рассылки в чат InfinityAntiScam")
    asyncio.create_task(send_broadcast_message())


@bot.on(events.NewMessage(pattern='/рассылкаON'))
async def start_broadcast(event):
    """Команда для запуска рассылки (только для администраторов)"""
    user_id = event.sender_id
    user_role = db.get_user_role(user_id)

    # Проверяем права (только создатель, кодер или администраторы)
    if user_id not in OWNER_ID and user_role not in [10]:
        await event.respond("❌ У вас нет прав для управления рассылкой!")
        return

    global broadcast_active
    if broadcast_active:
        await event.respond("✅ Рассылка уже активна!")
    else:
        broadcast_active = True
        await event.respond("✅ Рассылка запущена! Сообщения будут отправляться в чат InfinityAntiScam каждые 6 минут.")
        # Запускаем рассылку в фоне
        asyncio.create_task(send_broadcast_message())


@bot.on(events.NewMessage(pattern='/рассылкаOFF'))
async def stop_broadcast(event):
    """Команда для остановки рассылки (только для администраторов)"""
    user_id = event.sender_id
    user_role = db.get_user_role(user_id)

    # Проверяем права (только создатель, кодер или администраторы)
    if user_id not in OWNER_ID and user_role not in [10]:
        await event.respond("❌ У вас нет прав для управления рассылкой!")
        return

    global broadcast_active
    if not broadcast_active:
        await event.respond("❌ Рассылка уже остановлена!")
    else:
        broadcast_active = False
        await event.respond("⏸ Рассылка остановлена!")


@bot.on(events.NewMessage(pattern='/broadcaststatus'))
async def broadcast_status(event):
    """Команда для проверки статуса рассылки (только для администраторов)"""
    user_id = event.sender_id
    user_role = db.get_user_role(user_id)

    # Проверяем права (только создатель, кодер или администраторы)
    if user_id not in OWNER_ID and user_role not in [10]:
        await event.respond("❌ У вас нет прав для просмотра статуса рассылки!")
        return

    global broadcast_active, BROADCAST_INTERVAL, BROADCAST_CHAT_ID
    status = "🟢 Активна" if broadcast_active else "🔴 Остановлена"
    interval_minutes = BROADCAST_INTERVAL // 60

    await event.respond(
        f"📊 **Статус рассылки:** {status}\n"
        f"📌 **Чат:** InfinityAntiScam (ID: {BROADCAST_CHAT_ID})\n"
        f"⏱ **Интервал:** {interval_minutes} минут ({BROADCAST_INTERVAL} секунд)\n\n"
        f"⚙️ **Команды управления:**\n"
        f"• `/startbroadcast` - запустить рассылку\n"
        f"• `/stopbroadcast` - остановить рассылку\n"
        f"• `/setbroadcastinterval <минуты>` - изменить интервал\n"
        f"• `/broadcaststatus` - текущий статус"
    )


@bot.on(events.NewMessage(pattern=r'^\.дать\s+(\d+)(?:\s+(@\w+|\d+))?$'))
async def give_command(event):
    """Команда .дать <количество> [@username/ID]"""
    if not event.is_private:
        await event.delete()
        return

    user_id = event.sender_id
    match = event.pattern_match
    amount = int(match.group(1))

    # Проверяем минимальную сумму
    if amount < 1:
        await event.respond("❌ Минимальная сумма перевода: 1 💎")
        return

    # Определяем получателя
    target_id = None
    target_name = None

    # Проверяем, есть ли указанный пользователь в команде
    target_str = match.group(2)

    if target_str:
        # Если указан получатель в команде
        try:
            if target_str.isdigit():
                target_id = int(target_str)
                target = await bot.get_entity(target_id)
            else:
                target = await bot.get_entity(target_str)
            target_id = target.id
            target_name = target.first_name if hasattr(target, 'first_name') else f"ID:{target_id}"
        except Exception as e:
            await event.respond(f"❌ Не удалось найти пользователя: {e}")
            return
    else:
        # Если не указан получатель, проверяем, есть ли ответ на сообщение
        if event.is_reply:
            replied = await event.get_reply_message()
            target = await bot.get_entity(replied.sender_id)
            target_id = target.id
            target_name = target.first_name
        else:
            # Если нет получателя и нет ответа - выдаем себе (только для владельца)
            if user_id in OWNER_ID:
                target_id = user_id
                target_name = "себе"
            else:
                await event.respond(
                    "❌ Укажите получателя!\n\n"
                    "**Варианты:**\n"
                    "1. `.дать 100 @username` - перевести пользователю\n"
                    "2. `.дать 100` (ответом на сообщение) - перевести автору сообщения"
                )
                return

    # Получаем информацию об отправителе
    sender = await event.get_sender()
    sender_name = sender.first_name

    # Проверяем, является ли отправитель владельцем
    if user_id in OWNER_ID:
        # Владелец выдает монеты (без списания)
        success, result = db.give_balance(user_id, target_id, amount)

        if success:
            if target_id == user_id:
                await event.respond(
                    f"✅ **Владелец выдал монеты себе!**\n\n"
                    f"💰 **Сумма:** +{amount} 💎\n"
                    f"💎 **Новый баланс:** {result} 💎"
                )
            else:
                await event.respond(
                    f"✅ **Владелец выдал монеты!**\n\n"
                    f"👤 **Получатель:** {target_name}\n"
                    f"💰 **Сумма:** +{amount} 💎\n"
                    f"💎 **Новый баланс:** {result} 💎"
                )

            # Уведомляем получателя (если это не сам владелец)
            if target_id != user_id:
                try:
                    await bot.send_message(
                        target_id,
                        f"🎉 **Вам выданы монеты!**\n\n"
                        f"👑 **От:** Владелец\n"
                        f"💰 **Сумма:** +{amount} 💎\n"
                        f"💎 **Текущий баланс:** {result} 💎"
                    )
                except:
                    pass
        else:
            await event.respond(f"❌ Ошибка: {result}")

    else:
        # Обычный пользователь переводит со своего баланса
        # Запрещаем перевод самому себе
        if user_id == target_id:
            await event.respond("❌ Нельзя переводить монеты самому себе!")
            return

        # Проверяем баланс
        balance = db.get_balance(user_id)
        if balance < amount:
            await event.respond(f"❌ Недостаточно средств! Твой баланс: {balance} 💎")
            return

        # Выполняем перевод
        success, result = db.transfer_balance(user_id, target_id, amount)

        if success:
            new_from, new_to = result

            await event.respond(
                f"✅ **Перевод выполнен!**\n\n"
                f"👤 **Кому:** {target_name}\n"
                f"💰 **Сумма:** {amount} 💎\n"
                f"📉 **Твой новый баланс:** {new_from} 💎"
            )

            # Уведомляем получателя
            try:
                await bot.send_message(
                    target_id,
                    f"💰 **Вам перевели монеты!**\n\n"
                    f"👤 **От:** {sender_name}\n"
                    f"💰 **Сумма:** +{amount} 💎\n"
                    f"💎 **Текущий баланс:** {new_to} 💎"
                )
            except:
                pass
        else:
            await event.respond(f"❌ Ошибка: {result}")

def shutdown_handler(signal, frame):
    """Обработчик завершения работы бота"""
    logging.info("Завершение работы бота...")
    db.close()  # ТОЛЬКО ЗДЕСЬ закрываем БД
    sys.exit(0)


def main():
    print("Bot started...")
    bot.loop.create_task(cleanup_old_button_data())
    bot.run_until_disconnected()


bot.loop.create_task(auto_start_broadcast())
if __name__ == "__main__":
    import logging

    logging.basicConfig(level=logging.INFO)
    logger = logging.getLogger(__name__)

    signal.signal(signal.SIGINT, shutdown_handler)
    signal.signal(signal.SIGTERM, shutdown_handler)

    logger.info("Бот запущен и готов к работе.")
    bot.run_until_disconnected()
