## code
# Импорт необходимых модулей
import sqlite3  # Для работы с SQLite базой данных
import csv     # Для работы с CSV файлами
import json    # Для работы с JSON файлами
import random  # Для генерации случайных данных
from datetime import datetime  # Для работы с датой и временем
import os      # Для работы с файловой системой

class DatabaseApp:
    def __init__(self, db_name='database.db'):
        # Конструктор класса, инициализирует объект приложения
        self.db_name = db_name  # Сохраняет имя файла базы данных
        self._create_tables()   # Вызывает метод создания таблиц при инициализации
        
        # Данные для генерации случайных пользователей (списки русских имен, фамилий и доменов)
        self.first_names = ['Алексей', 'Мария', 'Дмитрий', 'Екатерина', 'Иван', 'Ольга', 'Сергей', 'Анна', 'Андрей', 'Наталья']
        self.last_names = ['Иванов', 'Петров', 'Сидоров', 'Смирнов', 'Кузнецов', 'Попов', 'Васильев', 'Соколов', 'Михайлов', 'Новиков']
        self.domains = ['gmail.com', 'mail.ru', 'yandex.ru', 'hotmail.com', 'outlook.com']

    def _get_connection(self):
        """Создает и возвращает соединение с базой данных"""
        conn = sqlite3.connect(self.db_name)  # Создает соединение с SQLite базой данных
        conn.execute("PRAGMA foreign_keys = ON")  # Включает поддержку внешних ключей (хотя в этой таблице их нет)
        return conn  # Возвращает объект соединения

    def _create_tables(self):
        """Создает необходимые таблицы в базе данных"""
        conn = self._get_connection()  # Получает соединение с БД
        cursor = conn.cursor()  # Создает курсор для выполнения SQL команд
        
        # SQL запрос для создания таблицы users, если она не существует
        cursor.execute('''
            CREATE TABLE IF NOT EXISTS users (
                id INTEGER PRIMARY KEY AUTOINCREMENT,  -- Автоинкрементируемый первичный ключ
                name TEXT NOT NULL,                    -- Поле для имени, не может быть NULL
                email TEXT UNIQUE NOT NULL,            -- Уникальный email, не может быть NULL
                age INTEGER,                           -- Поле для возраста, может быть NULL
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP  -- Дата создания с значением по умолчанию
            )
        ''')
        
        conn.commit()  # Фиксирует изменения в базе данных
        conn.close()   # Закрывает соединение с базой данных
        print("✅ Таблицы успешно созданы/проверены")  # Выводит сообщение об успехе

    def _generate_random_user(self):
        """Генерирует случайные данные пользователя"""
        first_name = random.choice(self.first_names)  # Выбирает случайное имя из списка
        last_name = random.choice(self.last_names)    # Выбирает случайную фамилию из списка
        name = f"{first_name} {last_name}"  # Создает полное имя из имени и фамилии
        
        # Создает email на основе имени и фамилии со случайным доменом
        email = f"{first_name.lower()}.{last_name.lower()}@{random.choice(self.domains)}"
        age = random.randint(18, 65)  # Генерирует случайный возраст от 18 до 65 лет
        
        return name, email, age  # Возвращает кортеж с данными пользователя

    def reorder_user_ids(self):
        """Переупорядочивает ID пользователей после удаления"""
        conn = self._get_connection()  # Получает соединение с БД
        cursor = conn.cursor()  # Создает курсор для выполнения SQL команд
        
        try:
            # Создает временную таблицу с такой же структурой как users
            cursor.execute('''
                CREATE TABLE temp_users (
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    name TEXT NOT NULL,
                    email TEXT UNIQUE NOT NULL,
                    age INTEGER,
                    created_at TIMESTAMP
                )
            ''')
            
            # Копирует данные из старой таблицы в новую, сортируя по ID
            # Это автоматически создаст новые последовательные ID
            cursor.execute('''
                INSERT INTO temp_users (name, email, age, created_at)
                SELECT name, email, age, created_at 
                FROM users 
                ORDER BY id
            ''')
            
            cursor.execute('DROP TABLE users')  # Удаляет старую таблицу users
            
            cursor.execute('ALTER TABLE temp_users RENAME TO users')  # Переименовывает временную таблицу в users
            
            conn.commit()  # Фиксирует изменения в базе данных
            print("✅ ID пользователей успешно переупорядочены")  # Сообщение об успехе
            
            self.read_users()  # Показывает обновленную таблицу пользователей
            return True  # Возвращает True при успешном выполнении
            
        except Exception as e:
            print(f"❌ Ошибка при переупорядочивании ID: {str(e)}")  # Сообщение об ошибке
            return False  # Возвращает False при ошибке
        finally:
            conn.close()  # Всегда закрывает соединение с БД

    def generate_random_users(self, count=5):
        """Генерирует указанное количество случайных пользователей"""
        conn = self._get_connection()  # Получает соединение с БД
        cursor = conn.cursor()  # Создает курсор для выполнения SQL команд
        
        added_count = 0  # Счетчик успешно добавленных пользователей
        for _ in range(count):  # Цикл для генерации указанного количества пользователей
            name, email, age = self._generate_random_user()  # Генерирует случайного пользователя
            try:
                # Пытается добавить пользователя в базу данных
                cursor.execute(
                    "INSERT INTO users (name, email, age) VALUES (?, ?, ?)",  # SQL запрос с параметрами
                    (name, email, age)  # Параметры для запроса
                )
                added_count += 1  # Увеличивает счетчик при успешном добавлении
            except sqlite3.IntegrityError:
                # Если email уже существует, пропускает этого пользователя
                continue
        
        conn.commit()  # Фиксирует изменения в базе данных
        conn.close()   # Закрывает соединение с БД
        print(f"✅ Успешно добавлено {added_count} случайных пользователей")  # Сообщение о результате
        return added_count  # Возвращает количество добавленных пользователей

    def create_user(self, name, email, age):
        """Создание нового пользователя"""
        conn = self._get_connection()  # Получает соединение с БД
        cursor = conn.cursor()  # Создает курсор для выполнения SQL команд
        
        try:
            # Вставляет нового пользователя в таблицу
            cursor.execute(
                "INSERT INTO users (name, email, age) VALUES (?, ?, ?)",  # SQL запрос
                (name, email, age)  # Параметры запроса
            )
            user_id = cursor.lastrowid  # Получает ID нового пользователя
            conn.commit()  # Фиксирует изменения в БД
            print(f"✅ Пользователь '{name}' успешно создан с ID: {user_id}")  # Сообщение об успехе
            return user_id  # Возвращает ID нового пользователя
        except sqlite3.IntegrityError:
            # Обрабатывает ошибку дублирования email
            print(f"❌ Ошибка: пользователь с email {email} уже существует.")
            return None  # Возвращает None при ошибке
        except Exception as e:
            # Обрабатывает любые другие ошибки
            print(f"❌ Неожиданная ошибка: {str(e)}")
            return None  # Возвращает None при ошибке
        finally:
            conn.close()  # Всегда закрывает соединение с БД

    def display_table(self, data=None, headers=None):
        """Красивое табличное отображение данных"""
        if data is None:  # Если данные не переданы
            conn = self._get_connection()  # Получает соединение с БД
            cursor = conn.cursor()  # Создает курсор
            cursor.execute("SELECT * FROM users ORDER BY id")  # Выбирает всех пользователей
            data = cursor.fetchall()  # Получает все строки результата
            conn.close()  # Закрывает соединение
            
        if not data:  # Если данных нет
            print("📭 Таблица пуста")  # Сообщение о пустой таблице
            return  # Выходит из функции
            
        if headers is None:  # Если заголовки не указаны
            headers = ['ID', 'Имя', 'Email', 'Возраст', 'Дата создания']  # Стандартные заголовки
        
        # Вычисляем максимальные ширины столбцов
        col_widths = []  # Список для хранения ширины каждого столбца
        for i in range(len(headers)):  # Для каждого столбца
            max_width = len(str(headers[i]))  # Начинает с ширины заголовка
            for row in data:  # Для каждой строки данных
                if i < len(row):  # Если столбец существует в строке
                    max_width = max(max_width, len(str(row[i])))  # Находит максимальную длину
            col_widths.append(max_width + 2)  # Добавляет ширину столбца с отступом
        
        # Создаем разделительную строку для таблицы
        separator = "+"  # Начинает с +
        for width in col_widths:  # Для каждой ширины столбца
            separator += "-" * width + "+"  # Добавляет горизонтальную линию
        
        # Выводим заголовок таблицы
        print(separator)  # Верхняя граница таблицы
        header_line = "|"  # Начинает строку заголовка с |
        for i, header in enumerate(headers):  # Для каждого заголовка
            header_line += f" {header:<{col_widths[i]-1}}|"  # Добавляет заголовок с выравниванием
        print(header_line)  # Выводит строку заголовков
        print(separator)  # Разделитель между заголовком и данными
        
        # Выводим данные таблицы
        for row in data:  # Для каждой строки данных
            data_line = "|"  # Начинает строку данных с |
            for i, cell in enumerate(row):  # Для каждой ячейки в строке
                data_line += f" {str(cell):<{col_widths[i]-1}}|"  # Добавляет данные с выравниванием
            print(data_line)  # Выводит строку данных
        
        print(separator)  # Нижняя граница таблицы
        print(f"📊 Всего записей: {len(data)}")  # Выводит количество записей

    def read_users(self, user_id=None):
        """Чтение пользователей с табличным отображением"""
        conn = self._get_connection()  # Получает соединение с БД
        cursor = conn.cursor()  # Создает курсор
        
        try:
            if user_id:  # Если указан конкретный ID пользователя
                cursor.execute("SELECT * FROM users WHERE id = ?", (user_id,))  # Ищет пользователя по ID
                user = cursor.fetchone()  # Получает одну строку результата
                if user:  # Если пользователь найден
                    print(f"\n🔍 НАЙДЕН ПОЛЬЗОВАТЕЛЬ:")  # Сообщение о найденном пользователе
                    self.display_table([user])  # Показывает пользователя в таблице
                else:  # Если пользователь не найден
                    print("❌ Пользователь не найден")  # Сообщение об ошибке
                return user  # Возвращает найденного пользователя или None
            else:  # Если ID не указан
                cursor.execute("SELECT * FROM users ORDER BY id")  # Выбирает всех пользователей
                users = cursor.fetchall()  # Получает все строки результата
                if users:  # Если есть пользователи
                    print(f"\n👥 ТАБЛИЦА ПОЛЬЗОВАТЕЛЕЙ:")  # Заголовок таблицы
                    self.display_table(users)  # Показывает всех пользователей
                else:  # Если нет пользователей
                    print("📭 В базе данных нет пользователей")  # Сообщение о пустой БД
                return users  # Возвращает список пользователей
        except Exception as e:  # Обрабатывает любые исключения
            print(f"❌ Ошибка при чтении данных: {str(e)}")  # Сообщение об ошибке
            return None  # Возвращает None при ошибке
        finally:
            conn.close()  # Всегда закрывает соединение с БД

    def update_user(self, user_id, **kwargs):
        """Обновление данных пользователя"""
        if not kwargs:  # Если не переданы параметры для обновления
            print("❌ Не указаны поля для обновления")  # Сообщение об ошибке
            return False  # Возвращает False
        
        conn = self._get_connection()  # Получает соединение с БД
        cursor = conn.cursor()  # Создает курсор
        
        try:
            set_clause = []  # Список для SET частей SQL запроса
            params = []      # Список для параметров запроса
            
            # Формирует SET части и параметры на основе переданных аргументов
            if 'name' in kwargs and kwargs['name']:  # Если передано новое имя
                set_clause.append("name = ?")  # Добавляет SET часть для имени
                params.append(kwargs['name'])  # Добавляет значение имени в параметры
            if 'email' in kwargs and kwargs['email']:  # Если передан новый email
                set_clause.append("email = ?")  # Добавляет SET часть для email
                params.append(kwargs['email'])  # Добавляет значение email в параметры
            if 'age' in kwargs and kwargs['age'] is not None:  # Если передан новый возраст
                set_clause.append("age = ?")  # Добавляет SET часть для возраста
                params.append(kwargs['age'])  # Добавляет значение возраста в параметры
            
            if not set_clause:  # Если нет полей для обновления
                print("❌ Нет данных для обновления")  # Сообщение об ошибке
                return False  # Возвращает False
            
            params.append(user_id)  # Добавляет ID пользователя в параметры для WHERE условия
            update_query = f"UPDATE users SET {', '.join(set_clause)} WHERE id = ?"  # Формирует полный SQL запрос
            
            cursor.execute(update_query, params)  # Выполняет запрос обновления
            conn.commit()  # Фиксирует изменения в БД
            
            if cursor.rowcount > 0:  # Если была обновлена хотя бы одна строка
                print(f"✅ Данные пользователя с ID {user_id} успешно обновлены")  # Сообщение об успехе
                cursor.execute("SELECT * FROM users WHERE id = ?", (user_id,))  # Выбирает обновленного пользователя
                updated_user = cursor.fetchone()  # Получает обновленные данные
                if updated_user:  # Если пользователь найден
                    print("\n🔄 ОБНОВЛЕННЫЕ ДАННЫЕ:")  # Заголовок для обновленных данных
                    self.display_table([updated_user])  # Показывает обновленные данные
                return True  # Возвращает True при успехе
            else:  # Если не было обновлено ни одной строки
                print("❌ Пользователь не найден или данные не изменились")  # Сообщение об ошибке
                return False  # Возвращает False
                
        except sqlite3.IntegrityError as e:  # Обрабатывает ошибки целостности данных
            print(f"❌ Ошибка целостности данных: {str(e)}")  # Сообщение об ошибке
            return False  # Возвращает False
        except Exception as e:  # Обрабатывает любые другие ошибки
            print(f"❌ Ошибка при обновлении: {str(e)}")  # Сообщение об ошибке
            return False  # Возвращает False
        finally:
            conn.close()  # Всегда закрывает соединение с БД

    def delete_user(self, user_id, reorder_ids=False):
        """Удаление пользователя"""
        conn = self._get_connection()  # Получает соединение с БД
        cursor = conn.cursor()  # Создает курсор
        
        try:
            cursor.execute("SELECT name FROM users WHERE id = ?", (user_id,))  # Ищет пользователя по ID
            user = cursor.fetchone()  # Получает данные пользователя
            
            if not user:  # Если пользователь не найден
                print("❌ Пользователь не найден")  # Сообщение об ошибке
                return False  # Возвращает False
            
            # Запрашивает подтверждение удаления
            confirm = input(f"⚠️  Вы уверены, что хотите удалить пользователя '{user[0]}' (ID: {user_id})? (y/N): ")
            if confirm.lower() != 'y':  # Если пользователь не подтвердил удаление
                print("❌ Удаление отменено")  # Сообщение об отмене
                return False  # Возвращает False
            
            cursor.execute("DELETE FROM users WHERE id = ?", (user_id,))  # Удаляет пользователя
            conn.commit()  # Фиксирует изменения в БД
            
            if cursor.rowcount > 0:  # Если была удалена хотя бы одна строка
                print(f"✅ Пользователь с ID {user_id} успешно удален")  # Сообщение об успехе
                
                if reorder_ids:  # Если установлен флаг переупорядочивания ID
                    reorder_confirm = input("🔄 Хотите переупорядочить ID пользователей? (y/N): ")  # Запрос подтверждения
                    if reorder_confirm.lower() == 'y':  # Если пользователь подтвердил
                        self.reorder_user_ids()  # Вызывает функцию переупорядочивания ID
                else:  # Если флаг не установлен
                    cursor.execute("SELECT COUNT(*) FROM users")  # Считает оставшихся пользователей
                    remaining_count = cursor.fetchone()[0]  # Получает количество
                    if remaining_count > 0:  # Если есть оставшиеся пользователи
                        print(f"\n📊 Осталось пользователей: {remaining_count}")  # Показывает количество
                        cursor.execute("SELECT * FROM users ORDER BY id LIMIT 10")  # Выбирает первых 10 пользователей
                        remaining_users = cursor.fetchall()  # Получает данные
                        self.display_table(remaining_users)  # Показывает таблицу
                return True  # Возвращает True при успехе
            else:  # Если не было удалено ни одной строки
                print("❌ Ошибка при удалении пользователя")  # Сообщение об ошибке
                return False  # Возвращает False
                
        except Exception as e:  # Обрабатывает любые исключения
            print(f"❌ Ошибка при удалении: {str(e)}")  # Сообщение об ошибке
            return False  # Возвращает False
        finally:
            conn.close()  # Всегда закрывает соединение с БД

    def export_to_csv(self, filename='data/users_export.csv'):
        """Экспорт данных пользователей в CSV"""
        conn = self._get_connection()  # Получает соединение с БД
        cursor = conn.cursor()  # Создает курсор
        
        try:
            cursor.execute("SELECT * FROM users")  # Выбирает всех пользователей
            users = cursor.fetchall()  # Получает все данные
            
            os.makedirs(os.path.dirname(filename), exist_ok=True)  # Создает папку если её нет
            
            with open(filename, 'w', newline='', encoding='utf-8') as csvfile:  # Открывает файл для записи
                writer = csv.writer(csvfile)  # Создает writer для CSV
                writer.writerow(['ID', 'Name', 'Email', 'Age', 'Created_At'])  # Записывает заголовки
                writer.writerows(users)  # Записывает данные
            
            print(f"✅ Данные успешно экспортированы в {filename}")  # Сообщение об успехе
            print(f"📊 Экспортировано записей: {len(users)}")  # Показывает количество экспортированных записей
            
            if users:  # Если есть данные
                print("\n📋 ПРЕВЬЮ ЭКСПОРТИРОВАННЫХ ДАННЫХ:")  # Заголовок превью
                preview_data = users[:3]  # Берет первые 3 записи для превью
                self.display_table(preview_data)  # Показывает превью
                if len(users) > 3:  # Если записей больше 3
                    print("... и еще {} записей".format(len(users) - 3))  # Показывает сколько еще записей
            
            return True  # Возвращает True при успехе
            
        except Exception as e:  # Обрабатывает исключения
            print(f"❌ Ошибка при экспорте в CSV: {str(e)}")  # Сообщение об ошибке
            return False  # Возвращает False
        finally:
            conn.close()  # Всегда закрывает соединение с БД

    def import_from_csv(self, filename='data/users.csv'):
        """Импорт данных пользователей из CSV"""
        conn = self._get_connection()  # Получает соединение с БД
        cursor = conn.cursor()  # Создает курсор
        
        try:
            with open(filename, 'r', newline='', encoding='utf-8') as csvfile:  # Открывает CSV файл для чтения
                reader = csv.DictReader(csvfile)  # Создает DictReader для чтения CSV как словаря
                imported_count = 0  # Счетчик импортированных записей
                skipped_count = 0   # Счетчик пропущенных записей
                imported_users = []  # Список для превью импортированных данных
                
                for row in reader:  # Для каждой строки в CSV
                    try:
                        # Пытается добавить пользователя (IGNORE пропускает дубликаты)
                        cursor.execute(
                            "INSERT OR IGNORE INTO users (name, email, age) VALUES (?, ?, ?)",
                            (row['Name'], row['Email'], int(row['Age']))  # Параметры из CSV
                        )
                        if cursor.rowcount > 0:  # Если запись была добавлена
                            imported_count += 1  # Увеличивает счетчик
                            if imported_count <= 3:  # Если это одна из первых 3 записей
                                cursor.execute("SELECT * FROM users WHERE email = ?", (row['Email'],))  # Ищет добавленного пользователя
                                user = cursor.fetchone()  # Получает данные
                                if user:  # Если пользователь найден
                                    imported_users.append(user)  # Добавляет в список для превью
                        else:  # Если запись не была добавлена (дубликат)
                            skipped_count += 1  # Увеличивает счетчик пропущенных
                    except (ValueError, KeyError) as e:  # Обрабатывает ошибки формата данных
                        print(f"⚠️  Ошибка в строке: {row} - {str(e)}")  # Сообщение об ошибке в строке
                        skipped_count += 1  # Увеличивает счетчик пропущенных
                
                conn.commit()  # Фиксирует изменения в БД
                print(f"✅ Импорт из CSV завершен:")  # Сообщение о завершении импорта
                print(f"   📥 Успешно импортировано: {imported_count}")  # Показывает количество импортированных
                print(f"   ⏭️  Пропущено: {skipped_count}")  # Показывает количество пропущенных
                
                if imported_users:  # Если есть импортированные данные для превью
                    print("\n📋 ПРЕВЬЮ ИМПОРТИРОВАННЫХ ДАННЫХ:")  # Заголовок превью
                    self.display_table(imported_users)  # Показывает превью
                    if imported_count > 3:  # Если импортировано больше 3 записей
                        print("... и еще {} записей".format(imported_count - 3))  # Показывает сколько еще
                
                return True  # Возвращает True при успехе
                
        except FileNotFoundError:  # Обрабатывает отсутствие файла
            print(f"❌ Файл {filename} не найден")  # Сообщение об ошибке
            return False  # Возвращает False
        except Exception as e:  # Обрабатывает другие исключения
            print(f"❌ Ошибка при импорте из CSV: {str(e)}")  # Сообщение об ошибке
            return False  # Возвращает False
        finally:
            conn.close()  # Всегда закрывает соединение с БД

    def export_to_json(self, filename='data/users_export.json'):
        """Экспорт данных пользователей в JSON"""
        conn = self._get_connection()  # Получает соединение с БД
        cursor = conn.cursor()  # Создает курсор
        
        try:
            cursor.execute("SELECT * FROM users")  # Выбирает всех пользователей
            users = cursor.fetchall()  # Получает все данные
            
            users_list = []  # Список для преобразованных данных
            for user in users:  # Для каждого пользователя
                users_list.append({  # Создает словарь с данными пользователя
                    'id': user[0],
                    'name': user[1],
                    'email': user[2],
                    'age': user[3],
                    'created_at': user[4]
                })
            
            os.makedirs(os.path.dirname(filename), exist_ok=True)  # Создает папку если её нет
            
            with open(filename, 'w', encoding='utf-8') as jsonfile:  # Открывает файл для записи JSON
                json.dump({'users': users_list}, jsonfile, indent=2, ensure_ascii=False)  # Записывает JSON с форматированием
            
            print(f"✅ Данные успешно экспортированы в {filename}")  # Сообщение об успехе
            print(f"📊 Экспортировано записей: {len(users_list)}")  # Показывает количество
            
            if users_list:  # Если есть данные
                print("\n📋 ПРЕВЬЮ ЭКСПОРТИРОВАННЫХ ДАННЫХ (первые 3 записи):")  # Заголовок превью
                preview_data = [(u['id'], u['name'], u['email'], u['age'], u['created_at']) for u in users_list[:3]]  # Создает данные для превью
                self.display_table(preview_data)  # Показывает превью
                if len(users_list) > 3:  # Если записей больше 3
                    print("... и еще {} записей".format(len(users_list) - 3))  # Показывает сколько еще
            
            return True  # Возвращает True при успехе
            
        except Exception as e:  # Обрабатывает исключения
            print(f"❌ Ошибка при экспорте в JSON: {str(e)}")  # Сообщение об ошибке
            return False  # Возвращает False
        finally:
            conn.close()  # Всегда закрывает соединение с БД

    def import_from_json(self, filename='data/users.json'):
        """Импорт данных пользователей из JSON"""
        conn = self._get_connection()  # Получает соединение с БД
        cursor = conn.cursor()  # Создает курсор
        
        try:
            with open(filename, 'r', encoding='utf-8') as jsonfile:  # Открывает JSON файл для чтения
                data = json.load(jsonfile)  # Загружает данные из JSON
                imported_count = 0  # Счетчик импортированных записей
                skipped_count = 0   # Счетчик пропущенных записей
                imported_users = []  # Список для превью импортированных данных
                
                for user in data.get('users', []):  # Для каждого пользователя в JSON
                    try:
                        # Пытается добавить пользователя (IGNORE пропускает дубликаты)
                        cursor.execute(
                            "INSERT OR IGNORE INTO users (name, email, age) VALUES (?, ?, ?)",
                            (user['name'], user['email'], user['age'])  # Параметры из JSON
                        )
                        if cursor.rowcount > 0:  # Если запись была добавлена
                            imported_count += 1  # Увеличивает счетчик
                            if imported_count <= 3:  # Если это одна из первых 3 записей
                                cursor.execute("SELECT * FROM users WHERE email = ?", (user['email'],))  # Ищет добавленного пользователя
                                imported_user = cursor.fetchone()  # Получает данные
                                if imported_user:  # Если пользователь найден
                                    imported_users.append(imported_user)  # Добавляет в список для превью
                        else:  # Если запись не была добавлена (дубликат)
                            skipped_count += 1  # Увеличивает счетчик пропущенных
                    except (KeyError, TypeError) as e:  # Обрабатывает ошибки в структуре данных
                        print(f"⚠️  Ошибка в данных пользователя: {user} - {str(e)}")  # Сообщение об ошибке
                        skipped_count += 1  # Увеличивает счетчик пропущенных
                
                conn.commit()  # Фиксирует изменения в БД
                print(f"✅ Импорт из JSON завершен:")  # Сообщение о завершении импорта
                print(f"   📥 Успешно импортировано: {imported_count}")  # Показывает количество импортированных
                print(f"   ⏭️  Пропущено: {skipped_count}")  # Показывает количество пропущенных
                
                if imported_users:  # Если есть импортированные данные для превью
                    print("\n📋 ПРЕВЬЮ ИМПОРТИРОВАННЫХ ДАННЫХ:")  # Заголовок превью
                    self.display_table(imported_users)  # Показывает превью
                    if imported_count > 3:  # Если импортировано больше 3 записей
                        print("... и еще {} записей".format(imported_count - 3))  # Показывает сколько еще
                
                return True  # Возвращает True при успехе
                
        except FileNotFoundError:  # Обрабатывает отсутствие файла
            print(f"❌ Файл {filename} не найден")  # Сообщение об ошибке
            return False  # Возвращает False
        except json.JSONDecodeError:  # Обрабатывает ошибки формата JSON
            print(f"❌ Ошибка формата JSON в файле {filename}")  # Сообщение об ошибке
            return False  # Возвращает False
        except Exception as e:  # Обрабатывает другие исключения
            print(f"❌ Ошибка при импорте из JSON: {str(e)}")  # Сообщение об ошибке
            return False  # Возвращает False
        finally:
            conn.close()  # Всегда закрывает соединение с БД

    def get_database_info(self):
        """Получает информацию о структуре базы данных"""
        conn = self._get_connection()  # Получает соединение с БД
        cursor = conn.cursor()  # Создает курсор
        
        cursor.execute("SELECT name FROM sqlite_master WHERE type='table'")  # Получает список таблиц
        tables = cursor.fetchall()  # Получает все таблицы
        
        print("\n📊 СТРУКТУРА БАЗЫ ДАННЫХ:")  # Заголовок
        print("=" * 40)  # Разделитель
        
        for table in tables:  # Для каждой таблицы
            table_name = table[0]  # Получает имя таблицы
            print(f"\n📋 Таблица: {table_name}")  # Выводит имя таблицы
            print("-" * 30)  # Разделитель
            
            cursor.execute(f"PRAGMA table_info({table_name})")  # Получает информацию о колонках таблицы
            columns = cursor.fetchall()  # Получает данные о колонках
            
            table_data = []  # Список для данных о структуре таблицы
            for col in columns:  # Для каждой колонки
                col_id, col_name, col_type, not_null, default_val, pk = col  # Распаковывает данные колонки
                constraints = []  # Список для ограничений
                if not_null:  # Если поле NOT NULL
                    constraints.append("NOT NULL")  # Добавляет ограничение
                if pk:  # Если первичный ключ
                    constraints.append("PRIMARY KEY")  # Добавляет ограничение
                
                constraints_str = " ".join(constraints)  # Объединяет ограничения в строку
                table_data.append((col_name, col_type, constraints_str))  # Добавляет данные колонки
            
            self.display_table(table_data, headers=['Имя', 'Тип', 'Ограничения'])  # Показывает структуру таблицы
        
        cursor.execute("SELECT COUNT(*) FROM users")  # Считает количество пользователей
        user_count = cursor.fetchone()[0]  # Получает количество
        print(f"\n📈 СТАТИСТИКА:")  # Заголовок статистики
        stats_data = [("👥 Пользователей в базе", user_count)]  # Данные для статистики
        self.display_table(stats_data, headers=['Параметр', 'Значение'])  # Показывает статистику
        
        conn.close()  # Закрывает соединение с БД

    def display_menu(self):
        """Отображение главного меню"""
        print("\n" + "=" * 50)  # Верхний разделитель
        print("🗃️  СИСТЕМА УПРАВЛЕНИЯ БАЗОЙ ДАННЫХ")  # Заголовок приложения
        print("=" * 50)  # Разделитель
        print("1. 👥 Управление пользователями")  # Пункт меню 1
        print("2. 🎲 Сгенерировать случайных пользователей")  # Пункт меню 2
        print("3. 📁 Работа с файлами")  # Пункт меню 3
        print("4. 📊 Информация о базе данных")  # Пункт меню 4
        print("5. 👀 Показать таблицу пользователей")  # Пункт меню 5
        print("6. 🔄 Переупорядочить ID пользователей")  # Пункт меню 6 (новая функция)
        print("7. 🚪 Выход")  # Пункт меню 7
        print("-" * 50)  # Нижний разделитель

    def users_menu(self):
        """Меню управления пользователями"""
        while True:  # Бесконечный цикл для подменю
            print("\n" + "=" * 40)  # Верхний разделитель
            print("👥 УПРАВЛЕНИЕ ПОЛЬЗОВАТЕЛЯМИ")  # Заголовок подменю
            print("=" * 40)  # Разделитель
            print("1. 📝 Создать пользователя")  # Пункт подменю 1
            print("2. 👀 Показать всех пользователей")  # Пункт подменю 2
            print("3. 🔍 Найти пользователя по ID")  # Пункт подменю 3
            print("4. ✏️  Обновить данные пользователя")  # Пункт подменю 4
            print("5. 🗑️  Удалить пользователя")  # Пункт подменю 5
            print("6. ↩️  Назад в главное меню")  # Пункт подменю 6 (выход)
            print("-" * 40)  # Нижний разделитель
            
            choice = input("Выберите действие: ").strip()  # Запрос выбора пользователя
            
            if choice == '1':  # Если выбран пункт 1
                self._create_user_interactive()  # Вызывает интерактивное создание пользователя
            elif choice == '2':  # Если выбран пункт 2
                self.read_users()  # Показывает всех пользователей
            elif choice == '3':  # Если выбран пункт 3
                self._find_user_interactive()  # Вызывает поиск пользователя по ID
            elif choice == '4':  # Если выбран пункт 4
                self._update_user_interactive()  # Вызывает обновление пользователя
            elif choice == '5':  # Если выбран пункт 5
                self._delete_user_interactive()  # Вызывает удаление пользователя
            elif choice == '6':  # Если выбран пункт 6
                break  # Выход из подменю
            else:  # Если выбран неверный пункт
                print("❌ Неверный выбор. Попробуйте снова.")  # Сообщение об ошибке

    def files_menu(self):
        """Меню работы с файлами"""
        while True:  # Бесконечный цикл для подменю
            print("\n" + "=" * 40)  # Верхний разделитель
            print("📁 РАБОТА С ФАЙЛАМИ")  # Заголовок подменю
            print("=" * 40)  # Разделитель
            print("1. 📥 Импорт из CSV")  # Пункт подменю 1
            print("2. 📥 Импорт из JSON")  # Пункт подменю 2
            print("3. 📤 Экспорт в CSV")  # Пункт подменю 3
            print("4. 📤 Экспорт в JSON")  # Пункт подменю 4
            print("5. ↩️  Назад в главное меню")  # Пункт подменю 5 (выход)
            print("-" * 40)  # Нижний разделитель
            
            choice = input("Выберите действие: ").strip()  # Запрос выбора пользователя
            
            if choice == '1':  # Если выбран пункт 1
                filename = input("Введите имя файла CSV (по умолчанию data/users.csv): ").strip()  # Запрос имени файла
                self.import_from_csv(filename or 'data/users.csv')  # Импорт из CSV
            elif choice == '2':  # Если выбран пункт 2
                filename = input("Введите имя файла JSON (по умолчанию data/users.json): ").strip()  # Запрос имени файла
                self.import_from_json(filename or 'data/users.json')  # Импорт из JSON
            elif choice == '3':  # Если выбран пункт 3
                filename = input("Введите имя файла для экспорта (по умолчанию data/users_export.csv): ").strip()  # Запрос имени файла
                self.export_to_csv(filename or 'data/users_export.csv')  # Экспорт в CSV
            elif choice == '4':  # Если выбран пункт 4
                filename = input("Введите имя файла для экспорта (по умолчанию data/users_export.json): ").strip()  # Запрос имени файла
                self.export_to_json(filename or 'data/users_export.json')  # Экспорт в JSON
            elif choice == '5':  # Если выбран пункт 5
                break  # Выход из подменю
            else:  # Если выбран неверный пункт
                print("❌ Неверный выбор. Попробуйте снова.")  # Сообщение об ошибке

    def _create_user_interactive(self):
        """Интерактивное создание пользователя"""
        print("\n📝 СОЗДАНИЕ НОВОГО ПОЛЬЗОВАТЕЛЯ")  # Заголовок
        print("-" * 30)  # Разделитель
        
        name = input("Введите имя: ").strip()  # Запрос имени
        email = input("Введите email: ").strip()  # Запрос email
        age = input("Введите возраст: ").strip()  # Запрос возраста
        
        if not name or not email:  # Проверка обязательных полей
            print("❌ Имя и email обязательны для заполнения")  # Сообщение об ошибке
            return  # Выход из функции
        
        try:  # Попытка преобразовать возраст в число
            age_int = int(age) if age else None  # Преобразование возраста
        except ValueError:  # Если возраст не число
            print("❌ Возраст должен быть числом")  # Сообщение об ошибке
            return  # Выход из функции
        
        user_id = self.create_user(name, email, age_int)  # Создание пользователя
        if user_id:  # Если пользователь создан успешно
            print("\n✅ СОЗДАН НОВЫЙ ПОЛЬЗОВАТЕЛЬ:")  # Сообщение об успехе
            self.read_users(user_id)  # Показывает созданного пользователя

    def _find_user_interactive(self):
        """Интерактивный поиск пользователя"""
        user_id = input("\nВведите ID пользователя: ").strip()  # Запрос ID пользователя
        
        try:  # Попытка преобразовать ID в число
            user_id_int = int(user_id)  # Преобразование ID
            self.read_users(user_id=user_id_int)  # Поиск пользователя по ID
        except ValueError:  # Если ID не число
            print("❌ ID должен быть числом")  # Сообщение об ошибке

    def _update_user_interactive(self):
        """Интерактивное обновление пользователя"""
        user_id = input("\nВведите ID пользователя для обновления: ").strip()  # Запрос ID пользователя
        
        try:  # Попытка преобразовать ID в число
            user_id_int = int(user_id)  # Преобразование ID
        except ValueError:  # Если ID не число
            print("❌ ID должен быть числом")  # Сообщение об ошибке
            return  # Выход из функции
        
        print("\n📋 ТЕКУЩИЕ ДАННЫЕ:")  # Заголовок
        current_user = self.read_users(user_id=user_id_int)  # Показывает текущие данные пользователя
        if not current_user:  # Если пользователь не найден
            return  # Выход из функции
        
        print("\n✏️  ОБНОВЛЕНИЕ ДАННЫХ ПОЛЬЗОВАТЕЛЯ")  # Заголовок
        print("(оставьте поле пустым, чтобы не изменять)")  # Подсказка
        print("-" * 40)  # Разделитель
        
        name = input("Новое имя: ").strip()  # Запрос нового имени
        email = input("Новый email: ").strip()  # Запрос нового email
        age = input("Новый возраст: ").strip()  # Запрос нового возраста
        
        update_data = {}  # Словарь для данных обновления
        if name:  # Если введено новое имя
            update_data['name'] = name  # Добавляет имя в данные обновления
        if email:  # Если введен новый email
            update_data['email'] = email  # Добавляет email в данные обновления
        if age:  # Если введен новый возраст
            try:  # Попытка преобразовать возраст в число
                update_data['age'] = int(age)  # Добавляет возраст в данные обновления
            except ValueError:  # Если возраст не число
                print("❌ Возраст должен быть числом")  # Сообщение об ошибке
                return  # Выход из функции
        
        if not update_data:  # Если нет данных для обновления
            print("⚠️  Не указано ни одного поля для обновления")  # Сообщение об ошибке
            return  # Выход из функции
        
        self.update_user(user_id_int, **update_data)  # Вызов обновления пользователя

    def _delete_user_interactive(self):
        """Интерактивное удаление пользователя"""
        user_id = input("\nВведите ID пользователя для удаления: ").strip()  # Запрос ID пользователя
        
        try:  # Попытка преобразовать ID в число
            user_id_int = int(user_id)  # Преобразование ID
            self.delete_user(user_id_int, reorder_ids=True)  # Удаление пользователя с предложением переупорядочить ID
        except ValueError:  # Если ID не число
            print("❌ ID должен быть числом")  # Сообщение об ошибке

    def run(self):
        """Запуск главного цикла приложения"""
        print("🚀 Запуск системы управления базой данных...")  # Приветственное сообщение
        print("🎲 Доступна генерация случайных пользователей!")  # Информация о возможностях
        print("📊 Данные отображаются в виде красивых таблиц!")  # Информация о возможностях
        print("🔄 Теперь можно переупорядочивать ID после удаления!")  # Информация о новой функции
        
        os.makedirs('data', exist_ok=True)  # Создает папку data если её нет
        
        while True:  # Бесконечный цикл главного меню
            self.display_menu()  # Показывает главное меню
            choice = input("Выберите действие: ").strip()  # Запрос выбора пользователя
            
            if choice == '1':  # Если выбран пункт 1
                self.users_menu()  # Переход в меню пользователей
            elif choice == '2':  # Если выбран пункт 2
                count = input("Сколько пользователей сгенерировать? (по умолчанию 5): ").strip()  # Запрос количества
                try:  # Попытка преобразовать в число
                    count_int = int(count) if count else 5  # Преобразование количества
                    self.generate_random_users(count_int)  # Генерация случайных пользователей
                    self.read_users()  # Показывает таблицу после генерации
                except ValueError:  # Если количество не число
                    print("❌ Введите число")  # Сообщение об ошибке
            elif choice == '3':  # Если выбран пункт 3
                self.files_menu()  # Переход в меню файлов
            elif choice == '4':  # Если выбран пункт 4
                self.get_database_info()  # Показывает информацию о БД
            elif choice == '5':  # Если выбран пункт 5
                self.read_users()  # Показывает таблицу пользователей
            elif choice == '6':  # Если выбран пункт 6
                self.reorder_user_ids()  # Переупорядочивает ID пользователей
            elif choice == '7':  # Если выбран пункт 7
                print("\n👋 До свидания!")  # Прощальное сообщение
                break  # Выход из приложения
            else:  # Если выбран неверный пункт
                print("❌ Неверный выбор. Попробуйте снова.")  # Сообщение об ошибке

# Функция для создания демонстрационных данных при первом запуске
def create_demo_data():
    """Создает демонстрационные JSON и CSV файлы"""
    os.makedirs('data', exist_ok=True)  # Создает папку data если её нет
    
    # Демо данные для JSON
    demo_users = {  # Создает словарь с демо пользователями
        "users": [  # Список пользователей
            {"name": "Иван Иванов", "email": "ivanov@example.com", "age": 28},  # Пользователь 1
            {"name": "Мария Петрова", "email": "petrova@example.com", "age": 32},  # Пользователь 2
            {"name": "Алексей Сидоров", "email": "sidorov@example.com", "age": 25}  # Пользователь 3
        ]
    }
    
    with open('data/users.json', 'w', encoding='utf-8') as f:  # Открывает файл для записи JSON
        json.dump(demo_users, f, indent=2, ensure_ascii=False)  # Записывает JSON с форматированием
    
    # Демо данные для CSV
    with open('data/users.csv', 'w', newline='', encoding='utf-8') as f:  # Открывает файл для записи CSV
        writer = csv.writer(f)  # Создает writer для CSV
        writer.writerow(['Name', 'Email', 'Age'])  # Записывает заголовки
        writer.writerow(['Сергей Козлов', 'kozlov@example.com', 35])  # Записывает строку 1
        writer.writerow(['Ольга Новикова', 'novikova@example.com', 29])  # Записывает строку 2
        writer.writerow(['Дмитрий Морозов', 'morozov@example.com', 41])  # Записывает строку 3
    
    print("📁 Созданы демонстрационные файлы в папке 'data/'")  # Сообщение о создании файлов

# Запуск приложения
if __name__ == "__main__":  # Проверка что скрипт запущен напрямую
    if not os.path.exists('data/users.json') or not os.path.exists('data/users.csv'):  # Проверка существования демо файлов
        create_demo_data()  # Создает демо файлы если их нет
    
    app = DatabaseApp()  # Создает экземпляр приложения
    app.run()  # Запускает приложение
