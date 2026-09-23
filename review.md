import datetime as dt


class Record:
    def __init__(self, amount, comment, date=''):
        self.amount = amount

        # если передать некорректную дату, заданную не в том формате или если это будет 
        # не число, метод strptime не сможет её разобрать и программа аварийно завершится.
        # Можно обернуть разбор даты в конструкцию try/except.
        # Если формат неверный, программа просто подставит сегодняшнюю дату
        # и продолжит работать, а не упадёт.
        #   try:
        #       parsed = dt.datetime.strptime(date, '%d.%m.%Y').date()
        #   except (ValueError, TypeError):
        #       parsed = dt.datetime.now().date()
        #   self.date = parsed if date else dt.datetime.now().date()
        self.date = (
            dt.datetime.now().date() if
            not
            date else dt.datetime.strptime(date, '%d.%m.%Y').date())

        # (читаемость): условие разбито на три строки с «not» посередине — читается тяжело. 
        # Лучше записать в одну строку или через обычный if/else.
        # Как исправить:
        #   self.date = (dt.datetime.now().date()
        #                if not date
        #                else dt.datetime.strptime(date, '%d.%m.%Y').date())
        self.comment = comment


class Calculator:
    def __init__(self, limit):
        self.limit = limit
        self.records = []

    def add_record(self, record):
        self.records.append(record)

    def get_today_stats(self):
        # dt.datetime.now().date() вызывается для каждой записи в списке. 
        # Если цикл выполняется в момент смены даты (около полуночи), первые записи
        # сравнятся с одной датой, последние — с другой. Сумма будет посчитана неверно.
        # Можно вычислить дату один раз перед циклом и использовать переменную:
        #   today = dt.datetime.now().date()
        #   return sum(rec.amount for rec in self.records if rec.date == today)
        #
        # (скорость/память): квадратные скобки внутри sum()
        # создают промежуточный список в памяти. Лучше использовать
        # генераторное выражение без скобок, так числа будут складываться, не занимая память.
        return sum([rec.amount for rec in self.records if rec.date == dt.datetime.now().date()])

    def get_week_stats(self):
        today = dt.datetime.now().date()

        # (читаемость): условие 7 > X >= 0 читается непривычно. 
        # Математически естественнее записать как 
        # 0 <= X < 7 — «разница от 0 до 6 дней включительно».
        #   return sum(rec.amount for rec in self.records
        #              if 0 <= (today - rec.date).days < 7)
        #
        # (скорость/память): аналогично get_today_stats —
        # заменить список на генератор внутри sum().
        return sum([rec.amount for rec in self.records if 7 > (today - rec.date).days >= 0])


class CaloriesCalculator(Calculator):
    def get_calories_remained(self):
        # (читаемость): имя переменной x не говорит ни о чём. 
        # Лучше использовать осмысленное имя, например remaining.
        # maining = self.limit - self.get_today_stats()
        x = self.limit - self.get_today_stats()
        if x > 0:
            # f-строка искусственно разорвана на две части без необходимости. Можно записать одной строкой.
            return f'Сегодня можно съесть что-нибудь' \
                   f' ещё, но с общей калорийностью не более {x} кКал'
        else:
            return ('Хватит есть!')


class CashCalculator(Calculator):
    # (читаемость): float(60) избыточно — 60 и так число, а при делении Python 
    # автоматически получит float. Можно написать 60.0 или просто 60.
    USD_RATE = float(60)
    EURO_RATE = float(70)

    def get_today_cash_remained(self, currency,
                                USD_RATE=USD_RATE, EURO_RATE=EURO_RATE):
        # (устойчивость к изменениям): параметры по умолчанию 
        # (USD_RATE=USD_RATE) запоминаются один раз при запуске программы. 
        # Ошибка проявится, если в коде попытаться изменить курс валюты 
        # Функция продолжит считать по старому значению, так как запомнила 
        # его при создании. Улучшает устойчивость к изменениям.
        # Можно убрать параметры из скобок и брать курс 
        # напрямую через self.USD_RATE и self.EURO_RATE.
        def get_today_cash_remained(self, currency,
                                    USD_RATE=USD_RATE, EURO_RATE=EURO_RATE):

        # (скорость/устойчивость к изменениям): словарь d создаётся заново при каждом 
        # вызове метода, хотя его содержимое не меняется.
        # Лучше вынести его в атрибут класса (константу).
        #   CURRENCIES = {'usd': ('USD',), 'eur': ('Euro',), 'rub': ('руб',)}
        #   — и брать курс из self.USD_RATE / self.EURO_RATE отдельно.
        d = {'usd': (USD_RATE, 'USD'), 'eur': (EURO_RATE, 'Euro'), 'rub': (1.0, 'руб')}

        rate = d.get(currency)
        if rate is None:
            return f'Неизвестная валюта {currency}'

        cash_remained /= rate[0]

        # (скорость/читаемость): мы уже нашли нужную запись в словаре и 
        # сохранили её в переменную rate. Нет смысла искать её
        # второй раз через d.get(currency) — можно просто взять rate[1].
        # Работать будет одинаково, но так короче и быстрее.
        # currency_type = rate[1]
        currency_type = d.get(currency)[1]

        if cash_remained > 0:
            return (
                f'На сегодня осталось {round(cash_remained, 2)} '
                f'{currency_type}'
            )
        elif cash_remained == 0:
            return 'Денег нет, держись'
        elif cash_remained < 0:
            # (читаемость): в методе смешаны два стиля
            # форматирования строк f-strings выше и .format() здесь.
            # Лучше привести к единому виду (f-strings).
            #   return f'Денег нет, держись: твой долг - {-cash_remained:.2f} {currency_type}'
            return 'Денег нет, держись:' \
                   ' твой долг - {0:.2f} {1}'.format(-cash_remained,
                                                     currency_type)

    def get_week_stats(self):
        # метод переопределён, но не возвращает результат.
        # super().get_week_stats() вычисляет сумму за неделю, но она
        # никуда не сохраняется и не возвращается. При вызове
        # cash_calc.get_week_stats() вернётся None вместо числа и программа будет работать неверно.
        # Нужно добавить return: return super().get_week_stats()
        super().get_week_stats()