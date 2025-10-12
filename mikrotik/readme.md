## Файлы для роутеров Mikrotik
Здесь собраны файлы с наиболее популярными маршрутами, объединёнными по группам.
И оптимизированный файл, включающий все эти маршруты сразу.

#### Настройка
- используется маркировка трафика с маршрутизацией в конкретный VPN-интерфейс
- замени имя списка на свой, если надо
- добавь новую таблицу маршрутов и включи FIB
- в списке маршрутов укажи эту таблицу для VPN интерфейса
- в mangle/PREROUTING добавь маркировку пакетов по этой таблице для адресов назачения по спику (отключи passthr)
- в nat/POSTROUTNG (srcnat) добавь трансляцию для адресов по спику с маркой таблицы, на IP VPN интерфейса
- в nat можно добавть трансляцию в сеть VPN через интерфейс VPN, чтобы была связь с сервером и узлами, если нужно

**Внимание! Не забудь указать правилное имя списка!**

Если у тебя есть файл .BAT для Кинетика и нужно переделать его в Микротик, то вот скрипт:
- запустить в консоли `python scriptname.py`
- скрипт спросит ИМЯ списка
- скрипт создасn файлs .rsc для Микротика из всех найденных файлов .bat в текущем каталоге
- проверь файлы

#### Скрипт
<details>
<summary>Раскрой скрипт</summary>

```python
import os
import glob

def mask_to_cidr(mask):
    """Конвертирует маску (например, '255.0.0.0') в CIDR (например, 8)."""
    return sum(bin(int(x)).count('1') for x in mask.split('.'))

def convert_route_line(line, list_name):
    """Конвертирует строку 'ROUTE ADD ...' в формат MikroTik (регистронезависимо)."""
    parts = line.strip().split()
    if len(parts) >= 6:
        # Приводим к верхнему регистру для проверки
        part1_upper = parts[0].upper()
        part2_upper = parts[1].upper()
        
        if part1_upper == "ROUTE" and part2_upper == "ADD":
            ip = parts[2]
            mask = parts[4]
            cidr = mask_to_cidr(mask)
            return f"add address={ip}/{cidr} list={list_name}"
    return None

def process_bat_file(bat_file_path, list_name):
    """Обрабатывает один BAT файл и создает RSC файл."""
    converted_lines = []
    
    try:
        with open(bat_file_path, 'r', encoding='utf-8') as f:
            for line in f:
                converted = convert_route_line(line, list_name)
                if converted:
                    converted_lines.append(converted)
    except UnicodeDecodeError:
        # Если UTF-8 не работает, пробуем другие кодировки
        try:
            with open(bat_file_path, 'r', encoding='cp1251') as f:
                for line in f:
                    converted = convert_route_line(line, list_name)
                    if converted:
                        converted_lines.append(converted)
        except:
            print(f"Ошибка чтения файла: {bat_file_path}")
            return False
    
    if not converted_lines:
        print(f"В файле {bat_file_path} не найдено подходящих строк ROUTE ADD")
        return False
    
    # Создаем имя для RSC файла
    base_name = os.path.splitext(bat_file_path)[0]
    rsc_file_path = base_name + '.rsc'
    
    # Сохраняем в RSC формате
    with open(rsc_file_path, 'w', encoding='utf-8') as f:
        f.write("/ip firewall address-list\n")
        f.write('\n'.join(converted_lines))
    
    return True

def find_and_process_bat_files(list_name):
    """Находит все BAT файлы в текущей директории и обрабатывает их."""
    # Ищем все .bat файлы в текущей директории
    bat_files = glob.glob("*.bat")
    
    if not bat_files:
        print("Не найдено BAT файлов в текущей директории!")
        return
    
    print(f"Найдено BAT файлов: {len(bat_files)}")
    print(f"Имя списка для MikroTik: {list_name}")
    print("-" * 50)
    
    successful_conversions = 0
    
    for bat_file in bat_files:
        print(f"Обрабатываю файл: {bat_file}")
        if process_bat_file(bat_file, list_name):
            successful_conversions += 1
            print(f"  ✓ Создан RSC файл: {os.path.splitext(bat_file)[0]}.rsc")
        else:
            print(f"  ✗ Ошибка обработки файла: {bat_file}")
    
    print(f"\nГотово! Успешно обработано файлов: {successful_conversions}/{len(bat_files)}")

def get_list_name():
    """Запрашивает у пользователя имя списка для MikroTik."""
    while True:
        list_name = input("Введите имя списка для MikroTik (например: IP_LIST, BLOCK_LIST, ALLOW_LIST): ").strip()
        
        if not list_name:
            print("Имя списка не может быть пустым! Попробуйте снова.")
            continue
            
        # Проверяем допустимые символы (только буквы, цифры, подчеркивания)
        if all(c.isalnum() or c == '_' for c in list_name):
            return list_name
        else:
            print("Имя списка может содержать только буквы, цифры и символ подчеркивания (_). Попробуйте снова.")

if __name__ == "__main__":
    print("=== Конвертер BAT в RSC для MikroTik ===")
    print("Поиск BAT файлов в текущей директории...")
    
    # Показываем текущую директорию
    current_dir = os.getcwd()
    print(f"Текущая директория: {current_dir}")
    
    # Запрашиваем имя списка
    list_name = get_list_name()
    print()
    
    # Обрабатываем файлы
    find_and_process_bat_files(list_name)
    
    input("\nНажмите Enter для выхода...")

```

</details>
