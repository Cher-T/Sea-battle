from random import randint   # импорт функции
class Dot:   # класс точек, которые появляются на доске
    def __init__(self, x, y):
        self.x = x
        self.y = y
    def __eq__(self, other):    # метод, которые сравнивае 2 объекта
        return self.x == other.x and self.y == other.y
    def __repr__(self):    # метод, который отчечает за вывод точек
        return f"Dot({self.x}, {self.y})"  # на консоль


class BoardException(Exception):   # общий класс исключений
    pass

class BoardOutException(BoardException):  #исключение для игрока
    def __str__(self):
        return "Вы пытаетесь выстрелить за доску!"

class BoardUsedException(BoardException):   #исключение для игрока
    def __str__(self):
        return "Вы уже стреляли в эту клетку"

class BoardWrongShipException(BoardException):  # исключение для
    pass                                        # размещения кораблей


class Ship:   # класс корабля
    def __init__(self, bow, l, o):   # параметры корабля
        self.bow = bow
        self.l = l
        self.o = o
        self.lives = l

    @property
    def dots(self):   # метод,
        ship_dots = []   # созаётся список точек
        for i in range(self.l):   # в цикле "рисуется корабль"
            cur_x = self.bow.x
            cur_y = self.bow.y

            if self.o == 0:
                cur_x += i

            elif self.o == 1:
                cur_y += i

            ship_dots.append(Dot(cur_x, cur_y))

        return ship_dots   # получается список точек

    def shooten(self, shot):   # проверка попадания
        return shot in self.dots


class Board:   # класс поля
    def __init__(self, hid=False, size=6):
        self.size = size
        self.hid = hid

        self.count = 0

        self.field = [["O"] * size for _ in range(size)]

        self.busy = []  # занятые точки
        self.ships = []

    def __str__(self):   # создаёт доску
        res = ""
        res += "  | 1 | 2 | 3 | 4 | 5 | 6 |"
        for i, row in enumerate(self.field):
            res += f"\n{i + 1} | " + " | ".join(row) + " |"

        if self.hid:
            res = res.replace("■", "O")
        return res

    def out(self, d):  # определяет, находится ли точка за доской
        return not ((0 <= d.x < self.size) and (0 <= d.y < self.size))

    def contour(self, ship, verb=False):
        near = [                          # список, в котором находятся сдвиги
            (-1, -1), (-1, 0), (-1, 1),   # от точки
            (0, -1), (0, 0), (0, 1),
            (1, -1), (1, 0), (1, 1)
        ]
        for d in ship.dots:   # проходит в списке по всем точкам, рядом с кораблём
            for dx, dy in near:   # чтобы знать, в какие клетки нельзя
                cur = Dot(d.x + dx, d.y + dy)   # расположить корабли
                if not (self.out(cur)) and cur not in self.busy:  # если точка подходит
                    if verb:
                        self.field[cur.x][cur.y] = "."   # на её место добавляется "."
                    self.busy.append(cur)   # и она становится занятой

    def add_ship(self, ship):  # добавляет корабль
        for d in ship.dots:
            if self.out(d) or d in self.busy:  # проверяет точку на занятость
                raise BoardWrongShipException()   # и границы
        for d in ship.dots:   # ставит квадратик в нужные точки
            self.field[d.x][d.y] = "■"
            self.busy.append(d)

        self.ships.append(ship)  # добавляется список собственных кораблей
        self.contour(ship)   # он обводится по контуру

    def shot(self, d):   # выстрел
        if self.out(d):
            raise BoardOutException()  # если выходит за границы - исключение

        if d in self.busy:
            raise BoardUsedException()   # если занято - исключение

        self.busy.append(d)   # делаем точку занятой

        for ship in self.ships:   # проверяет принадлежность к кораблям
            if ship.shooten(d):   # если подстрерили
                ship.lives -= 1   # минус жизнь у корабля
                self.field[d.x][d.y] = "X"   # ставится "X" на месте точки
                if ship.lives == 0:   # если корабль добили
                    self.count += 1
                    self.contour(ship, verb=True)
                    print("Корабль убит!")
                    return False
                else:                 # если ранили
                    print("Корабль ранен!")
                    return True

        self.field[d.x][d.y] = "."    # если не попали
        print("Мимо!")
        return False

    def begin(self):   # обнуляет список занятых точек
        self.busy = []
    def defeat(self):
        return self.count == len(self.ships)


class Player:   # класс игрока
    def __init__(self, board, enemy):
        self.board = board
        self.enemy = enemy

    def ask(self):   # метод для потомков
        raise NotImplementedError()

    def move(self):  # выстрел
        while True:
            try:
                target = self.ask()   # просится выстрел
                repeat = self.enemy.shot(target)  # продолжается пока нет
                return repeat     # исключений
            except BoardException as e:
                print(e)


class AI(Player):  # игрок компьютер, который рандомно выбирает корабли
    def ask(self):
        d = Dot(randint(0, 5), randint(0, 5))
        print(f"Ход компьютера: {d.x + 1} {d.y + 1}")
        return d


class User(Player):
    def ask(self):
        while True:
            cords = input("Ваш ход: ").split()

            if len(cords) != 2:   # проверяет на кол-во координат
                print(" Введите 2 координаты! ")
                continue

            x, y = cords

            if not (x.isdigit()) or not (y.isdigit()):   # проверяет на тип данных
                print(" Введите числа! ")
                continue

            x, y = int(x), int(y)

            return Dot(x - 1, y - 1)  # возвращает точку -1


class Game:
    def __init__(self, size=6):
        self.size = size
        pl = self.random_board()
        co = self.random_board()
        co.hid = True

        self.ai = AI(co, pl)
        self.us = User(pl, co)
    def try_board(self):  # метод раставляющий корабли на доску
        lens = [3, 2, 2, 1, 1, 1, 1]
        board = Board(size=self.size)   # создаётся доска
        attempts = 0
        for l in lens:   # цикл ставит корабли пока может
            while True:
                attempts += 1
                if attempts > 2000:  # tckb yt gjkexbkjcm
                    return None
                ship = Ship(Dot(randint(0, self.size), randint(0, self.size)), l, randint(0, 1))
                try:
                    board.add_ship(ship)
                    break
                except BoardWrongShipException:
                    pass
        board.begin()
        return board

    def random_board(self):   # метод, кторорый точно сделает доску
        board = None
        while board is None:
            board = self.try_board()
        return board

    def greet(self):   # интерфейс
        print("-------------------")
        print("  Приветсвуем вас  ")
        print("      в игре       ")
        print("    морской бой    ")
        print("         🚢")
        print("-------------------")
        print(" формат ввода: x y ")
        print(" x - номер строки  ")
        print(" y - номер столбца ")

    def loop(self):  # игровой цикл
        num = 0   # номер хода
        while True:  # вывод досок
            print("-" * 20)
            print("Доска пользователя:")
            print(self.us.board)
            print("-" * 20)
            print("Доска компьютера:")
            print(self.ai.board)
            print("-" * 20)
            if num % 2 == 0:   # ход игрока
                print("Ходит пользователь!")
                repeat = self.us.move()
            else:
                print("Ходит компьютер!")   # ход компьютера
                repeat = self.ai.move()
            if repeat:
                num -= 1  # повторение хода

            if self.ai.board.defeat():
                print("-" * 20)
                print("Пользователь выиграл!")
                break

            if self.us.board.defeat() == 7:
                print("-" * 20)
                print("Компьютер выиграл!")
                break
            num += 1

    def start(self):
        self.greet()
        self.loop()

g = Game()
g.start()
