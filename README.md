from kivy.app import App
from kivy.uix.gridlayout import GridLayout
from kivy.uix.button import Button
from kivy.uix.label import Label
from kivy.uix.boxlayout import BoxLayout
import random

# Константи
BOARD_SIZE = 10
SHIPS_COUNT = 5

class SeaBattleGame(App):
    def build(self):
        # Ініціалізація гри
        self.player_board = [["~"] * BOARD_SIZE for _ in range(BOARD_SIZE)]
        self.bot_board = [["~"] * BOARD_SIZE for _ in range(BOARD_SIZE)]
        self.player_ships = []
        self.bot_ships = self.generate_ships()
        self.is_placing_ships = True
        self.is_player_turn = True
        self.special_attack = None  # Для зберігання типу спецатаки ("plane", "plane_3x3", "nuclear")
        self.game_over = False  # Стан гри

        # Головне вікно
        root_layout = BoxLayout(orientation='vertical', spacing=10, padding=10)

        # Інформаційний текст
        self.info_label = Label(text="Розставте свої кораблі: 0/5", font_size=20, size_hint=(1, 0.1))
        root_layout.add_widget(self.info_label)

        # Поля гравця та противника
        self.board_layout = GridLayout(cols=2, spacing=10, size_hint=(1, 0.8))

        # Поле гравця
        self.grid_player = GridLayout(cols=BOARD_SIZE)
        self.buttons_player = [[None for _ in range(BOARD_SIZE)] for _ in range(BOARD_SIZE)]
        for i in range(BOARD_SIZE):
            for j in range(BOARD_SIZE):
                btn = Button(text="~", on_press=lambda instance, x=i, y=j: self.place_ship(x, y))
                self.buttons_player[i][j] = btn
                self.grid_player.add_widget(btn)
        self.board_layout.add_widget(self.grid_player)

        # Поле противника
        self.grid_bot = GridLayout(cols=BOARD_SIZE)
        self.buttons_bot = [[None for _ in range(BOARD_SIZE)] for _ in range(BOARD_SIZE)]
        for i in range(BOARD_SIZE):
            for j in range(BOARD_SIZE):
                btn = Button(text="~", on_press=lambda instance, x=i, y=j: self.handle_attack(x, y))
                self.buttons_bot[i][j] = btn
                self.grid_bot.add_widget(btn)
        self.board_layout.add_widget(self.grid_bot)

        root_layout.add_widget(self.board_layout)

        # Панель дій
        action_layout = BoxLayout(orientation='horizontal', spacing=10, size_hint=(1, 0.1))

        self.plane_button = Button(text="Літак 1xN", on_press=lambda instance: self.activate_special("plane"), disabled=True)
        self.plane_3x3_button = Button(text="Літак 3x3", on_press=lambda instance: self.activate_special("plane_3x3"), disabled=True)
        self.nuclear_button = Button(text="Ядерка 5x5", on_press=lambda instance: self.activate_special("nuclear"), disabled=True)

        action_layout.add_widget(self.plane_button)
        action_layout.add_widget(self.plane_3x3_button)
        action_layout.add_widget(self.nuclear_button)
        root_layout.add_widget(action_layout)

        return root_layout

    def place_ship(self, x, y):
        """Розставлення кораблів гравця."""
        if self.is_placing_ships and len(self.player_ships) < SHIPS_COUNT:
            if self.player_board[x][y] == "~":
                self.player_board[x][y] = "S"
                self.player_ships.append((x, y))
                self.buttons_player[x][y].text = "S"
                self.buttons_player[x][y].background_color = (0, 1, 0, 1)  # Зелений колір
                self.info_label.text = f"Розставте свої кораблі: {len(self.player_ships)}/{SHIPS_COUNT}"

            if len(self.player_ships) == SHIPS_COUNT:
                self.is_placing_ships = False
                self.info_label.text = "Усі кораблі розставлено! Починайте стріляти!"
                self.enable_special_buttons()

    def enable_special_buttons(self):
        """Активує кнопки спеціальних атак."""
        self.plane_button.disabled = False
        self.plane_3x3_button.disabled = False
        self.nuclear_button.disabled = False

    def handle_attack(self, x, y):
        """Обробка атаки на поле противника."""
        if self.game_over:  # Перевірка стану гри
            return
        if self.is_player_turn:
            if self.special_attack == "plane":
                self.plane_attack(x)
            elif self.special_attack == "plane_3x3":
                self.plane_3x3_attack(x, y)
            elif self.special_attack == "nuclear":
                self.nuclear_attack(x, y)
            else:
                self.player_shoot(x, y)

    def player_shoot(self, x, y):
        """Звичайний постріл гравця."""
        if self.bot_board[x][y] == "~":
            if (x, y) in self.bot_ships:
                self.bot_board[x][y] = "X"
                self.buttons_bot[x][y].text = "X"
                self.buttons_bot[x][y].background_color = (1, 0, 0, 1)  # Червоний колір
                self.bot_ships.remove((x, y))
                if not self.bot_ships:
                    self.info_label.text = "Ви виграли! Усі кораблі противника знищено!"
                    self.game_over = True
                    return
            else:
                self.bot_board[x][y] = "O"
                self.buttons_bot[x][y].text = "O"
                self.buttons_bot[x][y].background_color = (0, 0, 1, 1)  # Синій колір
            self.is_player_turn = False
            self.info_label.text = "Хід противника!"
            self.bot_turn()

    def bot_turn(self):
        """Хід противника."""
        if self.game_over:  # Якщо гра завершена, хід не виконується
            return
        x, y = random.randint(0, BOARD_SIZE - 1), random.randint(0, BOARD_SIZE - 1)
        while self.player_board[x][y] in ["X", "O"]:
            x, y = random.randint(0, BOARD_SIZE - 1), random.randint(0, BOARD_SIZE - 1)

        if self.player_board[x][y] == "S":
            self.player_board[x][y] = "X"
            self.buttons_player[x][y].text = "X"
            self.buttons_player[x][y].background_color = (1, 0, 0, 1)  # Червоний колір
            self.player_ships.remove((x, y))
            if not self.player_ships:
                self.info_label.text = "Противник виграв! Усі ваші кораблі знищено!"
                self.game_over = True
                return
        else:
            self.player_board[x][y] = "O"
            self.buttons_player[x][y].text = "O"
            self.buttons_player[x][y].background_color = (0, 0, 1, 1)  # Синій колір

        self.is_player_turn = True
        self.info_label.text = "Ваш хід!"

    def activate_special(self, special_type):
        """Активація спеціальної атаки."""
        self.special_attack = special_type
        self.info_label.text = f"Атакуйте клітинку ({special_type})"
        # Вимикаємо кнопку після використання
        if special_type == "plane":
            self.plane_button.disabled = True
        elif special_type == "plane_3x3":
            self.plane_3x3_button.disabled = True
        elif special_type == "nuclear":
            self.nuclear_button.disabled = True

    def plane_attack(self, x):
        """Атака літака 1xN."""
        if self.game_over:
            return
        for j in range(BOARD_SIZE):
            if (x, j) in self.bot_ships:
                self.bot_ships.remove((x, j))
                self.buttons_bot[x][j].text = "X"
                self.buttons_bot[x][j].background_color = (1, 0, 0, 1)
            else:
                self.buttons_bot[x][j].text = "O"
                self.buttons_bot[x][j].background_color = (0, 0, 1, 1)
        self.special_attack = None
        if not self.bot_ships:
            self.info_label.text = "Ви виграли! Усі кораблі противника знищено!"
            self.game_over = True

    def plane_3x3_attack(self, x, y):
        """Атака літака 3x3."""
        if self.game_over:
            return
        for dx in range(-1, 2):
            for dy in range(-1, 2):
                nx, ny = x + dx, y + dy
                if 0 <= nx < BOARD_SIZE and 0 <= ny < BOARD_SIZE:
                    if (nx, ny) in self.bot_ships:
                        self.bot_ships.remove((nx, ny))
                        self.buttons_bot[nx][ny].text = "X"
                        self.buttons_bot[nx][ny].background_color = (1, 0, 0, 1)
                    else:
                        self.buttons_bot[nx][ny].text = "O"
                        self.buttons_bot[nx][ny].background_color = (0, 0, 1, 1)
        self.special_attack = None
        if not self.bot_ships:
            self.info_label.text = "Ви виграли! Усі кораблі противника знищено!"
            self.game_over = True

    def nuclear_attack(self, x, y):
        """Атака ядеркою 5x5."""
        if self.game_over:
            return
        for dx in range(-2, 3):
            for dy in range(-2, 3):
                nx, ny = x + dx, y + dy
                if 0 <= nx < BOARD_SIZE and 0 <= ny < BOARD_SIZE:
                    if (nx, ny) in self.bot_ships:
                        self.bot_ships.remove((nx, ny))
                        self.buttons_bot[nx][ny].text = "X"
                        self.buttons_bot[nx][ny].background_color = (1, 0, 0, 1)
                    else:
                        self.buttons_bot[nx][ny].text = "O"
                        self.buttons_bot[nx][ny].background_color = (0, 0, 1, 1)
        self.special_attack = None
        if not self.bot_ships:
            self.info_label.text = "Ви виграли! Усі кораблі противника знищено!"
            self.game_over = True

    def generate_ships(self):
        """Генерація кораблів для бота."""
        ships = []
        while len(ships) < SHIPS_COUNT:
            x = random.randint(0, BOARD_SIZE - 1)
            y = random.randint(0, BOARD_SIZE - 1)
            if (x, y) not in ships:
                ships.append((x, y))
        return ships


if __name__ == "__main__":
    SeaBattleGame().run()
