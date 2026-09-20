# «Анализ заказов интернет-магазина»

---

## 1. Описание задачи

Необходимо реализовать консольную программу для анализа заказов интернет-магазина.

Каждый заказ представлен одной строкой фиксированного формата:

```
Товар;ИмяФамилия;АдресПокупателя;КоличествоТовара;ЦенаТовара
```

Пример входных данных:

```
Телефон;ИванИванов;Нальчик,Ленина,1,11;2;20000
```

Где:

- **Товар** — строка
- **ИмяФамилия** — строка без пробелов
- **Адрес** — строка (город, улица, дом, квартира)
- **Количество** — целое число
- **Цена** — цена за одну единицу товара

Один покупатель может делать **несколько заказов**.
Один и тот же товар может встречаться **в разных заказах**.

---

## 2. Требования к программе

Программа должна:

1. Распарсить входные данные
2. Сохранить их в удобной структуре данных
3. Уметь добавлять новые заказы
4. Выполнять аналитические операции

---

## 3. Что должна уметь программа

### Обязательный функционал

1. **Вывод списка всех товаров (без дубликатов)**
2. **Подсчет общего чека одного клиента (по имени и фамилии)**
3. **Анализ продаж одного товара**

   - сколько всего штук продано
   - на какую общую сумму

---

## 4. Ограничения

Разрешено использовать:

- `list`, `dict`, `set`, `tuple`
- функции
- циклы `for`, `while`
- `input()`, `print()`

Запрещено:

- классы
- файлы
- сторонние библиотеки

---

## 5. Структура данных

После парсинга каждый заказ должен быть представлен словарем:

```python
{
    'product': 'Телефон',
    'customer': 'ИванИванов',
    'address': 'Нальчик,Ленина,1,11',
    'quantity': 2,
    'price': 20000
}
```

Все заказы хранятся в списке:

```python
orders = [order1, order2, order3, ...]
```

---

## 6. Функции, которые нужно реализовать

### 6.1 Парсинг входных данных

```python
def parse_orders(data: str) -> list:
    """
    Преобразует входные строки в список заказов.
    """
```

---

### 6.2 Вывод всех заказов

```python
def print_orders(orders: list) -> None:
    """
    Красивый вывод всех заказов.
    """
```

---

### 6.3 Добавление нового заказа

```python
def add_order(orders: list, product: str, customer: str, address: str, quantity: int, price: int) -> None:
    """
    Добавляет новый заказ.
    """
```

---

### 6.4 Список всех товаров

```python
def get_all_products(orders: list) -> set:
    """
    Возвращает множество всех товаров.
    """
```

---

### 6.5 Общий чек клиента

```python
def total_customer_bill(orders: list, customer: str) -> int:
    """
    Возвращает общую сумму покупок клиента.
    """
```

---

### 6.6 Анализ продаж товара

```python
def product_sales_info(orders: list, product: str) -> tuple:
    """
    Возвращает (общее количество, общая сумма).
    """
```

---

## 7. Исходные данные (10 товаров)

```python
RAW_DATA = """
Телефон;ИванИванов;Нальчик,Ленина,1,11;2;20000
Телевизор;ПетрПетров;Нальчик,Пушкина,2,22;1;50000
Микроволновка;АсланАсланов;Нальчик,Шогенцукова,3,33;3;10000
Ноутбук;ИванИванов;Нальчик,Ленина,1,11;1;60000
Планшет;МарияСидорова;Нальчик,Кешокова,4,44;2;30000
Наушники;ПетрПетров;Нальчик,Пушкина,2,22;4;5000
Монитор;АсланАсланов;Нальчик,Шогенцукова,3,33;1;25000
Клавиатура;ИванИванов;Нальчик,Ленина,1,11;2;4000
Мышь;МарияСидорова;Нальчик,Кешокова,4,44;3;3000
Принтер;ПетрПетров;Нальчик,Пушкина,2,22;1;20000
"""
```

---

## 8. Полное решение

<details>
<summary>Решение</summary>

```python
RAW_DATA = """
Телефон;ИванИванов;Нальчик,Ленина,1,11;2;20000
Телевизор;ПетрПетров;Нальчик,Пушкина,2,22;1;50000
Микроволновка;АсланАсланов;Нальчик,Шогенцукова,3,33;3;10000
Ноутбук;ИванИванов;Нальчик,Ленина,1,11;1;60000
Планшет;МарияСидорова;Нальчик,Кешокова,4,44;2;30000
Наушники;ПетрПетров;Нальчик,Пушкина,2,22;4;5000
Монитор;АсланАсланов;Нальчик,Шогенцукова,3,33;1;25000
Клавиатура;ИванИванов;Нальчик,Ленина,1,11;2;4000
Мышь;МарияСидорова;Нальчик,Кешокова,4,44;3;3000
Принтер;ПетрПетров;Нальчик,Пушкина,2,22;1;20000
"""


def parse_orders(data):
    orders = []
    lines = data.strip().split('\n')

    for line in lines:
        parts = line.split(';')

        order = {
            'product': parts[0],
            'customer': parts[1],
            'address': parts[2],
            'quantity': int(parts[3]),
            'price': int(parts[4])
        }

        orders.append(order)

    return orders


def print_orders(orders):
    for order in orders:
        total = order['quantity'] * order['price']
        print(f"""
Товар: {order['product']}
Покупатель: {order['customer']}
Адрес: {order['address']}
Количество: {order['quantity']}
Цена за штуку: {order['price']}
Сумма: {total}
""")


def add_order(orders, product, customer, address, quantity, price):
    orders.append({
        'product': product,
        'customer': customer,
        'address': address,
        'quantity': quantity,
        'price': price
    })


def get_all_products(orders):
    return {order['product'] for order in orders}


def total_customer_bill(orders, customer):
    total = 0
    for order in orders:
        if order['customer'] == customer:
            total += order['quantity'] * order['price']
    return total


def product_sales_info(orders, product):
    total_quantity = 0
    total_sum = 0

    for order in orders:
        if order['product'] == product:
            total_quantity += order['quantity']
            total_sum += order['quantity'] * order['price']

    return total_quantity, total_sum


def main():
    orders = parse_orders(RAW_DATA)

    while True:
        print("""
1. Показать все заказы
2. Добавить заказ
3. Список всех товаров
4. Общий чек клиента
5. Информация о продажах товара
0. Выход
""")

        choice = input('Выберите пункт: ')

        if choice == '1':
            print_orders(orders)

        elif choice == '2':
            product = input('Товар: ')
            customer = input('Покупатель: ')
            address = input('Адрес: ')
            quantity = int(input('Количество: '))
            price = int(input('Цена: '))
            add_order(orders, product, customer, address, quantity, price)
            print('Заказ добавлен')

        elif choice == '3':
            products = get_all_products(orders)
            print('Товары:')
            for product in products:
                print('-', product)

        elif choice == '4':
            customer = input('Имя покупателя: ')
            print('Общий чек:', total_customer_bill(orders, customer))

        elif choice == '5':
            product = input('Название товара: ')
            qty, total = product_sales_info(orders, product)
            print(f'Продано штук: {qty}')
            print(f'Общая сумма: {total}')

        elif choice == '0':
            break

        else:
            print('Неверный пункт меню')


main()
```

</details>

---

# Меню через словарь

Вместо большого `if / elif` мы можем:

- связать пункт меню с функцией;
- вызывать функцию по ключу словаря.

Пример общей идеи:

```python
menu = {
    '1': some_function,
    '2': another_function
}
```

Пользователь вводит `'1'`, программа делает:

```python
menu['1']()
```

---

## 1. Подготовка функций-оберток для меню

Так как многие функции принимают аргументы, для меню удобно сделать **функции-обертки**, которые:

- спрашивают данные у пользователя;
- вызывают основную логику.

---

## 2. Полное решение с меню через словарь

Ниже — **полный рабочий код**, который можно **целиком скопировать и вставить**.
Отлично подходит для демонстрации на последнем этапе проекта.

---

### Полный код

<details>
<summary>Решение</summary>

```python
RAW_DATA = """
Телефон;ИванИванов;Нальчик,Ленина,1,11;2;20000
Телевизор;ПетрПетров;Нальчик,Пушкина,2,22;1;50000
Микроволновка;АсланАсланов;Нальчик,Шогенцукова,3,33;3;10000
Ноутбук;ИванИванов;Нальчик,Ленина,1,11;1;60000
Планшет;МарияСидорова;Нальчик,Кешокова,4,44;2;30000
Наушники;ПетрПетров;Нальчик,Пушкина,2,22;4;5000
Монитор;АсланАсланов;Нальчик,Шогенцукова,3,33;1;25000
Клавиатура;ИванИванов;Нальчик,Ленина,1,11;2;4000
Мышь;МарияСидорова;Нальчик,Кешокова,4,44;3;3000
Принтер;ПетрПетров;Нальчик,Пушкина,2,22;1;20000
"""


#---------- Бизнес-логика ----------

def parse_orders(data):
    orders = []
    for line in data.strip().split('\n'):
        product, customer, address, quantity, price = line.split(';')
        orders.append({
            'product': product,
            'customer': customer,
            'address': address,
            'quantity': int(quantity),
            'price': int(price)
        })
    return orders


def print_orders(orders):
    for order in orders:
        total = order['quantity'] * order['price']
        print(f"""
Товар: {order['product']}
Покупатель: {order['customer']}
Адрес: {order['address']}
Количество: {order['quantity']}
Цена: {order['price']}
Сумма: {total}
""")


def add_order(orders, product, customer, address, quantity, price):
    orders.append({
        'product': product,
        'customer': customer,
        'address': address,
        'quantity': quantity,
        'price': price
    })


def get_all_products(orders):
    return {order['product'] for order in orders}


def total_customer_bill(orders, customer):
    total = 0
    for order in orders:
        if order['customer'] == customer:
            total += order['quantity'] * order['price']
    return total


def product_sales_info(orders, product):
    qty = 0
    total = 0
    for order in orders:
        if order['product'] == product:
            qty += order['quantity']
            total += order['quantity'] * order['price']
    return qty, total


#---------- Функции-обертки для меню ----------

def menu_show_orders(orders):
    print_orders(orders)


def menu_add_order(orders):
    product = input('Товар: ')
    customer = input('Покупатель: ')
    address = input('Адрес: ')
    quantity = int(input('Количество: '))
    price = int(input('Цена: '))
    add_order(orders, product, customer, address, quantity, price)
    print('Заказ добавлен')


def menu_show_products(orders):
    products = get_all_products(orders)
    print('Список товаров:')
    for product in products:
        print('-', product)


def menu_customer_bill(orders):
    customer = input('Имя покупателя: ')
    total = total_customer_bill(orders, customer)
    print('Общий чек:', total)


def menu_product_info(orders):
    product = input('Название товара: ')
    qty, total = product_sales_info(orders, product)
    print('Продано штук:', qty)
    print('Общая сумма:', total)


def menu_exit(_):
    print('Выход из программы')
    return False


#--------- main с меню через словарь ----------

def main():
    orders = parse_orders(RAW_DATA)

    menu = {
        '1': ('Показать все заказы', menu_show_orders),
        '2': ('Добавить заказ', menu_add_order),
        '3': ('Список всех товаров', menu_show_products),
        '4': ('Общий чек клиента', menu_customer_bill),
        '5': ('Информация о продажах товара', menu_product_info),
        '0': ('Выход', menu_exit)
    }

    running = True

    while running:
        print('\nМеню:')
        for key, value in menu.items():
            print(f'{key}. {value[0]}')

        choice = input('Выберите пункт: ')

        if choice in menu:
            result = menu[choice][1](orders)
            if result is False:
                running = False
        else:
            print('Неверный пункт меню')


main()
```

</details>

---
