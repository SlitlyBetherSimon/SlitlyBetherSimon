import time
import random
import curses
import math
import pickle

class Character:
    def __init__(self, name, appearance):
        self.name = name
        self.appearance = appearance
        self.health = 100
        self.lives = 3
        self.coins = 0
        self.meat = 0
        self.armor = {"type": "Basic", "protection": 10, "price": 50}
        self.gun = {"type": "Basic", "damage": 10, "price": 50}
        self.position = [0, 0]
        self.aim_angle = 0
        self.running = False
        self.reloading = False

    def print_character(self, window, state):
        character_lines = self.appearance.splitlines()

        if state == "jump":
            character_lines.append("    ^")
        elif state == "kick":
            character_lines[-1] = character_lines[-1][:3] + "| |"
        elif state == "shoot":
            character_lines[-1] = character_lines[-1][:3] + "| |"
            character_lines.extend(["   / \\", "   | |", "   | |", "   | |"])
        elif state == "reload":
            character_lines[-1] = character_lines[-1][:3] + "| |"
            character_lines.extend(["   / \\", "   |-|", "   | |", "   |-|"])
        elif state == "hurt_walk":
            character_lines[-1] = character_lines[-1][:3] + "| |"
            character_lines.append("   /_\\")

        for i, line in enumerate(character_lines):
            window.addstr(self.position[0] + i, self.position[1], line)

        window.addstr(self.position[0] + len(character_lines), self.position[1], f"Health: {self.health}% Lives: {self.lives} Coins: {self.coins} Meat: {self.meat}")
        window.addstr(self.position[0] + len(character_lines) + 1, self.position[1], f"Armor: {self.armor['type']} Protection: {self.armor['protection']} Price: {self.armor['price']} coins")
        window.addstr(self.position[0] + len(character_lines) + 2, self.position[1], f"Gun: {self.gun['type']} Damage: {self.gun['damage']} Price: {self.gun['price']} coins")

    def move(self, direction):
        step = 2 if self.running else 1
        if direction == "w":
            self.position[0] -= step
        elif direction == "s":
            self.position[0] += step
        elif direction == "a":
            self.position[1] -= step
        elif direction == "d":
            self.position[1] += step

    def jump(self):
        if not self.running:
            self.position[0] -= 2
        else:
            self.position[0] -= 3

    def start_running(self):
        self.running = True

    def stop_running(self):
        self.running = False

    def aim(self, target_position):
        delta_x = target_position[1] - self.position[1]
        delta_y = target_position[0] - self.position[0]
        self.aim_angle = math.atan2(delta_y, delta_x)

    def shoot(self):
        # Perform shooting logic here
        pass

    def reload(self):
        self.reloading = True
        # Perform reloading logic here
        self.reloading = False

    def take_damage(self, damage):
        effective_protection = self.armor['protection']
        self.health -= max(0, damage - effective_protection)
        if self.health <= 0:
            self.lives -= 1
            self.health = 100
            return True
        return False

    def collect_coins(self, coins):
        self.coins += coins

    def collect_meat(self, meat):
        self.meat += meat
        if self.meat > 100:
            self.meat = 100

    def eat_meat(self):
        if self.meat >= 10:
            self.health += 10
            self.meat -= 10
        elif self.meat > 0:
            self.health += self.meat
            self.meat = 0

    def buy_armor(self, new_armor):
        if self.coins >= new_armor['price']:
            self.coins -= new_armor['price']
            self.armor = new_armor
            return True
        return False

    def buy_gun(self, new_gun):
        if self.coins >= new_gun['price']:
            self.coins -= new_gun['price']
            self.gun = new_gun
            return True
        return False

def print_monsters(window, position, level, dead_monsters):
    monsters = []

    for i in range(level):
        monster = ["   /-\\",
                   "  |o o|",
                   "   \\-/"]
        monsters.append((position[0] + i * 3, position[1], monster))

    for dead_monster in dead_monsters:
        window.addstr(dead_monster[0], dead_monster[1], " X ")
        window.addstr(dead_monster[0] + 1, dead_monster[1], "XXX")
        window.addstr(dead_monster[0] + 2, dead_monster[1], " X ")

    return monsters

def spawn_food(window, symbol, count):
    food_positions = []
    for _ in range(count):
        x_position = random.randint(0, 9)
        y_position = random.randint(0, 2)
        food_positions.append((x_position, y_position))
        window.addch(x_position, y_position, symbol)
    return food_positions

def spawn_poisonous_berries(window, symbol, count):
    berries_positions = []
    for _ in range(count):
        x_position = random.randint(0, 9)
        y_position = random.randint(0, 2)
        berries_positions.append((x_position, y_position))
        window.addch(x_position, y_position, symbol)
    return berries_positions

def display_controls(window):
    controls = [
        "Controls:",
        "  W: Walk Forward",
        "  S: Walk Backward",
        "  A: Walk Left",
        "  D: Walk Right",
        "  Space: Jump",
        "  LShift: Run",
        "  R: Reload",
        "  B: Open Shop",
        "  LMB: Aim",
        "  RMB: Shoot",
        "  P: Save Game",
        "  Tab: Show Controls",
        "  Q: Quit"
    ]
    for i, line in enumerate(controls):
        window.addstr(2 + i, 25, line)

def save_game(player, level, food_positions, berries_positions, dead_monsters):
    game_state = {
        'player': player,
        'level': level,
        'food_positions': food_positions,
        'berries_positions': berries_positions,
        'dead_monsters': dead_monsters
    }
    with open('saved_game.pkl', 'wb') as file:
        pickle.dump(game_state, file)

def load_game():
    try:
        with open('saved_game.pkl', 'rb') as file:
            game_state = pickle.load(file)
        return game_state
    except FileNotFoundError:
        return None

def main(stdscr):
    curses.curs_set(0)
    stdscr.clear()
    stdscr.refresh()

    saved_game = load_game()

    if saved_game:
        player = saved_game['player']
        level = saved_game['level']
        food_positions = saved_game['food_positions']
        berries_positions = saved_game['berries_positions']
        dead_monsters = saved_game['dead_monsters']
    else:
        player = Character("Player", """   O
      /|\\
      / \\""")
        level = 1
        food_positions = []
        berries_positions = []
        dead_monsters = []

    while True:
        monsters_position = (0, 20)
        dead_monsters = []

        for i in range(10):
            stdscr.clear()
            if random.random() < 0.2:
                player.print_character(stdscr, "hurt_walk")
                if player.take_damage(10):
                    stdscr.addstr(0, 0, "You died! Press 'q' to quit.")
                    stdscr.refresh()
                    stdscr.getch()
                    return

            display_controls(stdscr)
            player.print_character(stdscr, "walk")
            stdscr.addstr(0, 0, f"Coins: {player.coins}    ")  # Display coins in the top-left corner
            stdscr.refresh()

            key = stdscr.getch()

            if key == ord('q'):
                save_game(player, level, food_positions, berries_positions, dead_monsters)
                return
            elif key == ord('w'):
                player.move("w")
            elif key == ord('s'):
                player.move("s")
            elif key == ord('a'):
                player.move("a")
            elif key == ord('d'):
                player.move("d")
            elif key == ord('j'):
                player.jump()
            elif key == ord('k'):
                player.move("kick")
            elif key == curses.KEY_MOUSE:
                _, mouse_x, mouse_y, _, _ = curses.getmouse()

                if (
                    12 <= mouse_x <= 30
                    and 3 <= mouse_y <= 6
                    and player.coins >= upgraded_gun['price']
                ):
                    player.buy_gun(upgraded_gun)

            time.sleep(0.1)

        player.stop_running()

        for food_position in food_positions:
            # ... (previous code for food collision)

        for berries_position in berries_positions:
            # ... (previous code for berries collision)

        for monster in print_monsters(stdscr, monsters_position, level, dead_monsters):
            # ... (previous code for monster collision)

        stdscr.refresh()

curses.wrapper(main)

