# Подробный гайд по написанию модулей для pwncat-vl

## Введение

pwncat-vl — это мощный фреймворк для управления обратными шеллами и автоматизации энумерации. Архитектура фреймворка построена на модульной системе, которая позволяет расширять функциональность путем написания собственных модулей. В этом руководстве мы подробно рассмотрим, как писать модули для pwncat-vl.

## 1. Архитектура модульной системы

Модульная система pwncat-vl основана на трех основных типах модулей:

1. **Base Modules** - базовые модули для выполнения различных действий на цели
2. **Enumerate Modules** - модули для сбора информации о цели
3. **Implant Modules** - модули для установки персистента и эскалации привилегий

## 2. Типы модулей

### 2.1. Enumerate (модули энумерации)
Модули энумерации используются для сбора информации о целевой системе. Они автоматически кешируют результаты и обеспечивают повторное использование информации другим кодом.

### 2.2. Implant (модули импланта)
Модули импланта устанавливают персистентные механизмы на целевую систему, такие как SSH ключи, cron задачи или SUID бинари.

### 2.3. Base Modules
Общие модули, которые выполняют произвольные действия на цели и возвращают результаты.

## 3. Основные компоненты модуля

### 3.1. Структура базового модуля

```python
#!/usr/bin/env python3

from pwncat.modules import Status, Argument, BaseModule, ModuleFailed
from pwncat.platform.linux import Linux

class Module(BaseModule):
    """ Описание модуля """
    
    PLATFORM = [Linux]  # Платформа, для которой работает модуль (или None если не зависит от платформы)
    ARGUMENTS = {       # Аргументы, которые принимает модуль
        "arg_name": Argument(str, default="default_value", help="help text")
    }
    ALLOW_KWARGS = False  # Разрешить дополнительные аргументы (не указанные в ARGUMENTS)
    COLLAPSE_RESULT = False  # Если True, одиночный результат возвращается как значение, а не как список
    
    def run(self, session, progress=None, **kwargs):
        """ Основная логика работы модуля """
        # session - текущая сессия
        # progress - флаг отображения прогресса
        # **kwargs - аргументы, указанные в ARGUMENTS
        
        yield Status("Сообщение о статусе")
        # Ваш код здесь
        return result
```

### 3.2. Структура модуля энумерации

```python
#!/usr/bin/env python3

from pwncat.db import Fact
from pwncat.platform.linux import Linux
from pwncat.modules.enumerate import Schedule, EnumerateModule, Scope

class MyFact(Fact):
    """ Описание найденной информации """
    
    def __init__(self, source, data):
        super().__init__(source=source, types=["my.category"])
        self.data = data
    
    def title(self, session):
        """Краткое название для отображения"""
        return f"[cyan]Заголовок[/cyan]: {self.data}"
    
    def description(self, session):
        """Детальное описание (опционально)"""
        return f"Описание: {self.data}"
    
    def category(self, session):
        """Категория факта (для группировки результатов)"""
        return "My Category"

class Module(EnumerateModule):
    """ Модуль энумерации """
    
    PROVIDES = ["my.category"]  # Типы информации, которые предоставляет модуль
    PLATFORM = [Linux]          # Поддерживаемые платформы
    SCHEDULE = Schedule.ONCE    # Расписание выполнения
    SCOPE = Scope.HOST          # Область действия
    
    def enumerate(self, session):
        """ Основная логика энумерации """
        # Ваш код для сбора информации
        yield MyFact(self.name, "some_data")
```

### 3.3. Структура модуля импланта

```python
#!/usr/bin/env python3

from pwncat.facts import Implant
from pwncat.modules import Status, Argument, ModuleFailed
from pwncat.platform.linux import Linux
from pwncat.modules.implant import ImplantModule

class MyImplant(Implant):
    """ Описание установленного импланта """
    
    def __init__(self, source, data):
        super().__init__(source=source, types=["implant.my_type"], uid=0)
        self.data = data
    
    def title(self, session):
        """Заголовок импланта"""
        return f"[red]Мой имплант[/red]: {self.data}"

    def remove(self, session):
        """Метод удаления импланта"""
        # Логика удаления импланта
        pass

class Module(ImplantModule):
    """ Модуль установки импланта """
    
    PLATFORM = [Linux]
    
    def install(self, session, **kwargs):
        """ Установка импланта """
        yield Status("Установка импланта")
        # Логика установки импланта
        return MyImplant(self.name, "installed_data")
```

## 4. Типы расписаний (Schedule)

Для модулей энумерации доступны следующие типы расписаний:

- `Schedule.ONCE` - выполнить только один раз за все время использования pwncat
- `Schedule.PER_USER` - выполнить один раз для каждого уникального пользователя
- `Schedule.ALWAYS` - выполнять каждый раз при обращении к модулю

## 5. Области действия (Scope)

- `Scope.HOST` - результаты сохраняются в базе данных и доступны между различными сессиями
- `Scope.SESSION` - результаты доступны только в текущей сессии
- `Scope.NONE` - результаты не сохраняются (используется с `Schedule.ALWAYS`)

## 6. Определение аргументов

Аргументы модуля определяются через словарь `ARGUMENTS`:

```python
from pwncat.modules import Argument, List, Bool

ARGUMENTS = {
    "string_arg": Argument(str, default="default", help="String argument"),
    "int_arg": Argument(int, default=42, help="Integer argument"),
    "bool_arg": Argument(Bool, default=False, help="Boolean argument"),
    "list_arg": Argument(List(str), default=["a", "b"], help="List argument"),
    "required_arg": Argument(str, help="Required argument (no default = required)")
}
```

### Вспомогательные типы аргументов:

- `List(_type=str)` - принимает список значений указанного типа
- `Bool` - принимает булевые значения (true/false, 1/0)
- `NoValue` - специальное значение для обязательных аргументов (без значения по умолчанию)

## 7. Работа с платформой

Всегда используйте абстракцию платформы `session.platform` вместо прямых системных вызовов:

```python
# Правильно
result = session.platform.run(["command", "arg"], capture_output=True, text=True)
with session.platform.open("/path/to/file", "r") as f:
    content = f.read()

# Неправильно
result = subprocess.run(["command", "arg"], capture_output=True, text=True)
with open("/path/to/file", "r") as f:
    content = f.read()
```

### Все методы платформы:

#### Методы выполнения команд:
- `platform.run(args, **kwargs)` - выполнение команды с ожиданием завершения (аналог subprocess.run)
- `platform.Popen(args, **kwargs)` - выполнение команды асинхронно (аналог subprocess.Popen)
- `platform.sudo(command, user=None, group=None, **kwargs)` - выполнение команды с sudo
- `platform.su(user, password=None)` - выполнение команды от другого пользователя

#### Методы работы с файлами:
- `platform.open(path, mode)` - открытие файла
- `platform.Path(path)` - создание объекта Path для работы с файлами
- `platform.stat(path)` - получение статистики о файле
- `platform.lstat(path)` - получение статистики о символической ссылке
- `platform.chmod(path, mode, link=False)` - изменение прав доступа
- `platform.touch(path)` - создание/обновление файла
- `platform.unlink(path)` - удаление файла
- `platform.mkdir(path, mode=0o777, parents=False)` - создание директории
- `platform.rmdir(path)` - удаление директории
- `platform.rename(source, target)` - переименование файла
- `platform.symlink_to(source, target)` - создание символической ссылки
- `platform.link_to(source, target)` - создание жёсткой ссылки

#### Методы работы с директориями:
- `platform.listdir(path=None)` - список файлов в директории
- `platform.chdir(path)` - изменение текущей директории
- `platform.abspath(path)` - получение абсолютного пути
- `platform.readlink(path)` - чтение символической ссылки

#### Методы идентификации:
- `platform.getuid()` - получение текущего UID
- `platform.refresh_uid()` - обновление кэшированного UID
- `platform.whoami()` - получение имени текущего пользователя
- `platform.getenv(name)` - получение значения переменной окружения
- `platform.get_host_hash()` - получение уникального идентификатора хоста

#### Другие методы:
- `platform.which(name)` - поиск бинарного файла
- `platform.compile(sources, output=None, suffix=None, cflags=None, ldflags=None)` - компиляция исходного кода
- `platform.umask(mask=None)` - установка/получение umask

### Работа с Path объектом:

```python
path = session.platform.Path("/path/to/file")

# Методы Path:
- `path.exists()` - проверка существования
- `path.is_file()` - проверка на файл
- `path.is_dir()` - проверка на директорию
- `path.is_symlink()` - проверка на символическую ссылку
- `path.stat()` - информация о файле
- `path.chmod(mode)` - изменение прав доступа
- `path.mkdir(mode=0o777, parents=False, exist_ok=False)` - создание директории
- `path.open(mode="r")` - открытие файла
- `path.read_text(encoding=None, errors=None)` - чтение текста
- `path.read_bytes()` - чтение байтов
- `path.write_text(data, encoding=None, errors=None)` - запись текста
- `path.write_bytes(data)` - запись байтов
- `path.unlink(missing_ok=False)` - удаление файла
- `path.rename(target)` - переименование
- `path.resolve(strict=False)` - разрешение пути
- `path.glob(pattern)` - поиск файлов по паттерну
- `path.iterdir()` - итерация по содержимому директории
```

## 8. Класс Session

Объект сессии предоставляет доступ ко всем компонентам pwncat и к целевой системе.

### Основные атрибуты и методы:

#### Атрибуты:
- `session.platform` - объект платформы для взаимодействия с целевой системой
- `session.config` - объект конфигурации
- `session.db` - объект базы данных
- `session.target` - объект цели
- `session.id` - идентификатор сессии

#### Методы:
- `session.current_user()` - получение информации о текущем пользователе
- `session.find_user(uid=None, name=None)` - поиск пользователя по UID или имени
- `session.find_group(gid=None, name=None)` - поиск группы по GID или имени
- `session.run(module_name, **kwargs)` - выполнение модуля
- `session.register_fact(fact, scope=Scope.HOST, commit=False)` - регистрация факта
- `session.log(*args, **kwargs)` - логирование сообщения
- `session.print(*args, **kwargs)` - вывод сообщения
- `session.task(*args, **kwargs)` - создание задачи с прогресс-баром
- `session.update_task(task, *args, **kwargs)` - обновление задачи

## 9. Класс Fact

Класс Fact представляет собой объект, содержащий информацию, найденную модулем энумерации.

### Структура:
```python
class MyFact(Fact):
    def __init__(self, source, data):
        super().__init__(source=source, types=["category.type"])
        self.data = data
    
    def title(self, session):
        """Краткое описание для отображения"""
        return f"[cyan]Title[/cyan]: {self.data}"
    
    def description(self, session):
        """Детальное описание (опционально)"""
        return f"Description: {self.data}"
    
    def category(self, session):
        """Категория для группировки"""
        return "Category"
```

### Атрибуты:
- `fact.types` - список типов факта (используется для поиска)
- `fact.source` - имя модуля, который создал факт
- `fact.hidden` - флаг скрытия факта

## 10. Классы исключений

- `ModuleFailed` - базовое исключение для ошибок модуля
- `ModuleNotFound` - модуль не найден
- `IncorrectPlatformError` - неверная платформа
- `ArgumentFormatError` - неверный формат аргумента
- `MissingArgument` - отсутствует обязательный аргумент
- `InvalidArgument` - неверный аргумент

## 11. Класс Status

Класс Status используется для обновления прогресса выполнения модуля:

```python
def run(self, session):
    yield Status("Выполняется этап 1")
    # Выполнение кода
    yield Status("Выполняется этап 2")
    # Выполнение кода
    return result
```

## 12. Утилиты и вспомогательные функции

### Классы доступа:
- `Access.NONE = 0` - нет доступа
- `Access.EXISTS = auto()` - файл существует
- `Access.READ = auto()` - доступ на чтение
- `Access.WRITE = auto()` - доступ на запись
- `Access.EXECUTE = auto()` - доступ на выполнение
- `Access.SUID = auto()` - установлен SUID бит
- `Access.SGID = auto()` - установлен SGID бит
- `Access.REGULAR = auto()` - обычный файл
- `Access.DIRECTORY = auto()` - директория
- `Access.PARENT_EXIST = auto()` - родительская директория существует
- `Access.PARENT_WRITE = auto()` - родительская директория доступна на запись

### Вспомогательные функции:
- `random_string(length=8)` - генерация случайной строки
- `human_readable_size(size, decimal_places=2)` - преобразование байтов в человекочитаемый формат
- `human_readable_delta(seconds)` - преобразование секунд в человекочитаемый формат
- `isprintable(data)` - проверка, являются ли данные печатаемыми
- `strip_markup(styled_text)` - удаление разметки Rich
- `strip_ansi_escape(s)` - удаление ANSI escape последовательностей
- `escape_markdown(s)` - экранирование markdown символов

## 13. Практические примеры

### 13.1. Модуль энумерации - поиск SUID бинарей

```python
#!/usr/bin/env python3

import subprocess
from pwncat.db import Fact
from pwncat.platform.linux import Linux
from pwncat.modules.enumerate import Schedule, EnumerateModule

class SuidBinary(Fact):
    """Описание SUID бинаря"""
    
    def __init__(self, source, path, permissions):
        super().__init__(source=source, types=["file.suid"])
        self.path = path
        self.permissions = permissions
    
    def title(self, session):
        return f"[yellow]SUID[/yellow] {self.path} ({self.permissions})"

class Module(EnumerateModule):
    """Поиск SUID бинарей"""
    
    PROVIDES = ["file.suid"]
    PLATFORM = [Linux]
    SCHEDULE = Schedule.ONCE
    
    def enumerate(self, session):
        try:
            # Выполняем команду на удаленной системе с помощью абстракции платформы
            proc = session.platform.Popen(
                ["find", "/", "-perm", "-4000", "-type", "f", "-exec", "ls", "-la", "{}", ";"],
                stdout=subprocess.PIPE,
                stderr=subprocess.DEVNULL,
                text=True
            )
            
            with proc.stdout as stream:
                for line in stream:
                    line = line.strip()
                    if line:
                        parts = line.split(None, 8)  # Разбиваем по первым 8 пробелам
                        if len(parts) >= 9:
                            perms = parts[0]
                            path = parts[8]
                            yield SuidBinary(self.name, path, perms)
                            
        except Exception as e:
            session.print(f"[red]Ошибка при поиске SUID бинарей: {e}[/red]")
```

### 13.2. Модуль энумерации - поиск файлов с интересными разрешениями

```python
#!/usr/bin/env python3

from pwncat.db import Fact
from pwncat.platform.linux import Linux
from pwncat.modules.enumerate import Schedule, EnumerateModule

class InterestingFile(Fact):
    """Описание файла с интересными разрешениями"""
    
    def __init__(self, source, path, description):
        super().__init__(source=source, types=["file.interesting"])
        self.path = path
        self.description = description
    
    def title(self, session):
        return f"[green]{self.description}[/green]: {self.path}"

class Module(EnumerateModule):
    """Поиск интересных файлов"""
    
    PROVIDES = ["file.interesting"]
    PLATFORM = [Linux]
    SCHEDULE = Schedule.ONCE
    
    def enumerate(self, session):
        # Поиск файлов с интересными разрешениями
        search_commands = [
            (["find", "/", "-perm", "-002", "-type", "f", "-print0"], "World writable file"),
            (["find", "/", "-perm", "-4000", "-type", "f", "-print0"], "SUID file"),
            (["find", "/", "-perm", "-2000", "-type", "f", "-print0"], "SGID file"),
            (["find", "/", "-perm", "-1002", "-type", "f", "-print0"], "World writable and sticky bit"),
        ]
        
        for cmd, description in search_commands:
            try:
                result = session.platform.run(cmd, capture_output=True, text=True)
                if result.returncode == 0:
                    for path in result.stdout.split('\x00'):
                        if path.strip():
                            yield InterestingFile(self.name, path.strip(), description)
            except Exception:
                # Игнорируем ошибки доступа к некоторым каталогам
                pass
```

### 13.3. Модуль импланта - установка SSH ключа

```python
#!/usr/bin/env python3

import os
from pathlib import Path
from pwncat.facts import PrivateKey
from pwncat.modules import Status, Argument, ModuleFailed
from pwncat.platform.linux import Linux
from pwncat.modules.implant import ImplantModule

class SSHKeyImplant(PrivateKey):
    """Установленный SSH ключ"""
    
    def __init__(self, source, user, key_path, public_key):
        super().__init__(
            source=source,
            path=key_path,
            uid=user.id,
            content=Path(key_path).read_text(),
            encrypted=False,
            authorized=True,
        )
        self.public_key = public_key

    def title(self, session):
        user = session.find_user(uid=self.uid)
        return f"[blue]SSH ключ[/blue] установлен для [green]{user.name}[/green]"

    def remove(self, session):
        """Удаление установленного ключа"""
        user = session.find_user(uid=self.uid)
        ssh_dir = session.platform.Path(user.home) / ".ssh"
        if not ssh_dir.exists():
            return

        auth_keys_path = ssh_dir / "authorized_keys"
        if not auth_keys_path.exists():
            return

        with session.platform.open(auth_keys_path, "r") as f:
            lines = f.readlines()

        # Удаляем наш ключ
        with session.platform.open(auth_keys_path, "w") as f:
            for line in lines:
                if line.strip() != self.public_key.strip():
                    f.write(line)

class Module(ImplantModule):
    """Установка SSH ключа"""
    
    PLATFORM = [Linux]
    ARGUMENTS = {
        **ImplantModule.ARGUMENTS,
        "ssh_key_path": Argument(str, help="Путь к SSH приватному ключу"),
        "user": Argument(str, default="__pwncat_current__", help="Пользователь для установки ключа"),
    }
    
    def install(self, session, ssh_key_path, user):
        yield Status("Проверка прав доступа")
        
        current_user = session.current_user()
        if user != "__pwncat_current__" and current_user.id != 0:
            raise ModuleFailed("Только root может установить ключ для другого пользователя")
        
        if not os.path.exists(ssh_key_path):
            raise ModuleFailed(f"Файл ключа {ssh_key_path} не существует")
        
        # Читаем публичный ключ
        try:
            with open(ssh_key_path + ".pub", "r") as f:
                public_key = f.read().strip() + "\n"
        except FileNotFoundError:
            raise ModuleFailed(f"Публичный ключ {ssh_key_path}.pub не найден")
        
        # Находим пользователя
        if user == "__pwncat_current__":
            user_info = current_user
        else:
            user_info = session.find_user(name=user)
            if not user_info:
                raise ModuleFailed(f"Пользователь {user} не найден")
        
        # Создаем .ssh директорию
        ssh_dir = session.platform.Path(user_info.home) / ".ssh"
        if not ssh_dir.exists():
            ssh_dir.mkdir(parents=True)
            session.platform.chmod(str(ssh_dir), 0o700)
        
        # Добавляем ключ в authorized_keys
        auth_keys_path = ssh_dir / "authorized_keys"
        with session.platform.open(auth_keys_path, "a") as f:
            f.write(public_key)
        
        # Устанавливаем правильные права
        session.platform.chown(str(auth_keys_path), user_info.id, user_info.gid)
        session.platform.chmod(str(auth_keys_path), 0o600)
        
        yield Status("SSH ключ установлен")
        return SSHKeyImplant(self.name, user_info, ssh_key_path, public_key)
```

## 14. Рекомендации по разработке

### 14.1. Обработка ошибок
Всегда оборачивайте код в блоки try/except и используйте исключения типа `ModuleFailed`:

```python
try:
    # Ваш код
    result = session.platform.run(["command"], capture_output=True, text=True)
except subprocess.CalledProcessError as e:
    session.print(f"[yellow]Команда завершилась с ошибкой: {e}[/yellow]")
    return  # Не возвращаем факты, если команда не удалась
except Exception as e:
    session.print(f"[red]Неожиданная ошибка: {e}[/red]")
    return
```

### 14.2. Использование Rich для оформления
Используйте Rich разметку для красивого отображения результатов:

```python
def title(self, session):
    return f"[green]SUID файл[/green] [yellow]{self.path}[/yellow]"
```

### 14.3. Эффективное использование ресурсов
- Используйте `Popen` с потоковой передачей для больших выходных данных
- Устанавливайте таймауты для команд
- Очищайте ресурсы в блоках finally

### 14.4. Интеграция с pwncat
- Используйте `session.print()` для вывода вместо `print()`
- Используйте `yield Status()` для обновления прогресса
- Следуйте шаблонам фактов и модулей

## 15. Лучшие практики

### 15.1. Именование
- Используйте понятные имена для типов фактов: `file.suid`, `system.process`, `network.connection`
- Следуйте шаблону `category.subcategory` для типов

### 15.2. Структура файлов
Создайте модуль в соответствующем каталоге:
- `pwncat/modules/linux/enumerate/my_module.py` для Linux модулей энумерации
- `pwncat/modules/windows/enumerate/my_module.py` для Windows модулей энумерации
- `pwncat/modules/linux/implant/my_implant.py` для Linux модулей импланта

### 15.3. Запуск модуля
После создания модуля его можно запустить с помощью:
```
run linux.enumerate.my_module  # для модуля энумерации
run linux.implant.my_implant   # для модуля импланта
```

## 16. Документация по основным API модулям

### 16.1. pwncat.commands
Этот модуль реализует парсер команд, лексер, подсветку и т.д. для pwncat. Каждая команда определяется как отдельный модуль под pwncat/commands, который определяет класс Command, наследующийся от pwncat.commands.CommandDefinition.

```python
from pwncat.commands import CommandDefinition, Parameter, Complete

class Command(CommandDefinition):
    """ Комментарии к документации команды помещаются в docstring """
    
    PROG = "custom"
    ARGS = {
        "--option,-o": Parameter(Complete.NONE, help="information", action="store_true"),
        "positional": Parameter(
            Complete.CHOICES,
            metavar="POSITIONAL",
            choices=["hello", "world"],
            help="help information",
        ),
    }

    def run(self, manager: "pwncat.manager.Manager", args: "argparse.Namespace"):
        manager.log("we ran a custom command!")
```

### 16.2. pwncat.config
Модуль для работы с конфигурацией pwncat. Конфигурация pwncat не является свободной. Есть определенный набор известных имен конфигурации, которые можно установить, и каждая из них имеет определенный тип. Проверка типов осуществляется при присваивании путем выполнения вызываемого объекта, используемого в качестве типа для данной конфигурации. Типы конфигурации, как правило, хорошо справляются с преобразованием строк из интерактивной строки ввода в допустимые значения для изменяемой конфигурации. Если преобразование невозможно, возникает ValueError.

```python
from pwncat.config import Config

# Получение значения
value = session.config.get('name', default_value)

# Установка значения
session.config.set('name', value)
```

### 16.3. pwncat.db
Пакет определяет все объекты базы данных. Внутри pwncat использует базу данных ZODB, которая хранит данные в виде постоянных объектов Python. Каждый класс, определенный в этом пакете, является постоянным Python-классом, который хранится как есть в базе данных. Дополнительную информацию о создании постоянных классов см. в документации ZODB.

```python
from pwncat.db import Fact

class MyFact(Fact):
    def __init__(self, source, types):
        super().__init__(types=types, source=source)
        # Дополнительные атрибуты
```

### 16.4. pwncat.facts
Модуль содержит общие факты, используемые для стандартных перечислений. Некоторые типы фактов используются для нескольких платформ, поэтому они были выделены сюда. В общем, вам не нужно использовать эти типы, за исключением случаев, когда вы взаимодействуете с данными, возвращенными модулем перечисления.

```python
from pwncat.facts import ArchData, DistroVersionData, EscalationReplace, EscalationSpawn

# Примеры специфичных для платформы фактов
from pwncat.facts.linux import LinuxUser, LinuxGroup
from pwncat.facts.windows import WindowsUser, WindowsGroup

# Факты способностей
from pwncat.facts.ability import ExecuteAbility, FileReadAbility, FileWriteAbility
```

### 16.5. pwncat.gtfobins
Модуль предоставляет абстрактный интерфейс к базе данных GTFOBins. База данных GTFOBins сопоставляет бинарные файлы со специальными разрешениями, которые могут быть использованы для эскалации привилегий (и не только). Внутри pwncat использует эту базу данных для идентификации способов чтения/записи файлов, а также при эскалации с помощью таких бинарных файлов, как SETUID и правила sudo.

```python
from pwncat.gtfobins import Capability, Stream

# Использование GTFOBins для поиска способов выполнения действий
binary = session.platform.gtfo.find_binary('/bin/sh', Capability.SHELL)
```

### 16.6. pwncat.manager
Менеджер - это ядро объекта в pwncat. Менеджер отвечает за поддержание конфигурации, состояние терминала и поддержание всех активных сессий pwncat. Менеджер может иметь ноль или более активных сессий в любое время.

```python
from pwncat.manager import Manager, InteractiveExit

# Использование менеджера
with Manager() as manager:
    session = manager.create_session(platform="linux", host="192.168.1.1", port=4444)
```

### 16.7. pwncat.modules
Модули pwncat - это расширяемая функция pwncat. Они предоставляют способ для пользователей эффективно выполнять сценарии взаимодействия с целями из локального терминала. Самая обширная функция - это модули перечисления, позволяющие пользователям быстро собирать полезную информацию с цели и сохранять данные в базе данных для последующего доступа.

```python
from pwncat.modules import BaseModule, Argument, Status, ModuleFailed
from pwncat.modules.enumerate import EnumerateModule, Schedule
from pwncat.modules.implant import ImplantModule
```

### 16.8. pwncat.platform
Платформа - это абстракция pwncat для ОС или конкретного дистрибутива. В общем, это позволяет pwncat в общем смысле взаимодействовать с целями на уровне ОС. Например, платформа предоставляет реализацию pathlib.Path, обеспечивающую беспрепятственный доступ к файлам. Платформа также определяет способы запроса переменных окружения, получения ID текущего пользователя и имени, а также общего запуска процессов.

```python
from pwncat.platform import Platform
from pwncat.platform.linux import Linux
from pwncat.platform.windows import Windows

# Платформо-специфичные классы
path = session.platform.Path("/etc/passwd")
```

### 16.9. pwncat.subprocess
Модуль предоставляет абстракцию для выполнения подпроцессов на удаленной системе.

```python
from pwncat.subprocess import Popen, PIPE, CompletedProcess

# Выполнение команды асинхронно
proc = session.platform.Popen(["command"], stdout=subprocess.PIPE)
stdout, stderr = proc.communicate()
```

### 16.10. pwncat.target
Цель - это структура данных, хранящаяся в ZODB. Она содержит все перечисленные факты, установленные импланты, уникальный ID, последний идентифицированный удаленный адрес и другую информацию, необходимую для идентификации или взаимодействия с целью через сессии pwncat.

```python
from pwncat.target import Target

# Доступ к цели из сессии
target = session.target
```

### 16.11. pwncat.util
Различные вспомогательные методы и классы, которые не подходят ни в один другой модуль или пакет.

```python
from pwncat.util import Access, Init, State, random_string, human_readable_size

# Классы доступа
access = Access.READ | Access.WRITE

# Вспомогательные функции
random_str = random_string(10)
size_str = human_readable_size(1024*1024*5)
```

### 16.12. pwncat.facts.tamper
Tampers (изменения) - это модификации, которые мы сознательно внесли в целевой хост. pwncat отслеживает изменения, где это возможно, чтобы предупреждать пользователя о внесенных изменениях и в некоторых случаях обеспечивать возможность отката этих изменений.

```python
from pwncat.facts.tamper import CreatedFile, ReplacedFile, CreatedDirectory

# Примеры tampers
tamper = CreatedFile(source="my_module", uid=0, path="/tmp/test.txt")
```

## 17. Заключение

Разработка модулей для pwncat-vl позволяет значительно расширить функциональность фреймворка. Следуя шаблонам, описанным в этом руководстве, вы можете создавать собственные модули для выполнения практически любых задач на целевой системе, от простой энумерации до сложных манипуляций с системой и установки персистента.

Важно помнить о безопасности и этике при использовании pwncat-vl, всегда работайте только с системами, на которые у вас есть явное разрешение проводить тестирование.
