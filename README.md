"""
Pantalla de celular manipulable para pruebas en Python.

Uso:
    python phone_screen.py        -> abre la GUI (tkinter), se maneja con mouse y teclado
    python phone_screen.py test   -> corre las pruebas automáticas (sin GUI)

Diseño:
    - Phone / Element: modelo puro en Python (sin GUI), ideal para tests.
    - PhoneGUI: dibuja el modelo en un Canvas y reenvía clics/teclas.
"""
import sys
import unittest


# ----------------------------- MODELO -----------------------------
class Element:
    def __init__(self, id, kind, x, y, w, h, text="", on_tap=None):
        self.id, self.kind = id, kind          # kind: "label" | "button" | "input"
        self.x, self.y, self.w, self.h = x, y, w, h
        self.text, self.on_tap = text, on_tap

    def contains(self, px, py):
        return self.x <= px <= self.x + self.w and self.y <= py <= self.y + self.h


class Phone:
    def __init__(self, width=360, height=640):
        self.width, self.height = width, height
        self.screens = {}          # nombre -> función que construye la pantalla
        self.elements = []         # elementos de la pantalla actual
        self.current = None
        self.history = []
        self.focus = None          # id del input con foco
        self.state = {}            # estado libre de la app (contadores, etc.)
        self.log = []              # registro de eventos para aserciones

    # --- pantallas ---
    def register(self, name, builder):
        self.screens[name] = builder

    def navigate(self, name, push=True):
        if push and self.current:
            self.history.append(self.current)
        self.current, self.focus = name, None
        self.elements = []
        self.screens[name](self)
        self.log.append(("navigate", name))

    def back(self):
        if self.history:
            self.navigate(self.history.pop(), push=False)
            self.log.append(("back", self.current))

    def home(self):
        self.history.clear()
        self.navigate("home", push=False)

    # --- construir elementos ---
    def add(self, element):
        self.elements.append(element)
        return element

    def label(self, id, x, y, w, h, text=""):
        return self.add(Element(id, "label", x, y, w, h, text))

    def button(self, id, x, y, w, h, text, on_tap=None):
        return self.add(Element(id, "button", x, y, w, h, text, on_tap))

    def input(self, id, x, y, w, h, text=""):
        return self.add(Element(id, "input", x, y, w, h, text))

    def find(self, id):
        return next((e for e in self.elements if e.id == id), None)

    def text_of(self, id):
        e = self.find(id)
        return e.text if e else None

    # --- interacciones ---
    def tap(self, x, y):
        for e in reversed(self.elements):          # el de arriba primero
            if e.contains(x, y):
                self.log.append(("tap", e.id))
                if e.kind == "input":
                    self.focus = e.id
                else:
                    self.focus = None
                    if e.on_tap:
                        e.on_tap(self, e)
                return e
        self.focus = None
        self.log.append(("tap", None))
        return None

    def tap_id(self, id):
        e = self.find(id)
        if not e:
            raise KeyError(f"No existe el elemento '{id}'")
        return self.tap(e.x + e.w / 2, e.y + e.h / 2)

    def type_text(self, text):
        e = self.find(self.focus) if self.focus else None
        if e:
            e.text += text
            self.log.append(("type", e.id, text))

    def backspace(self):
        e = self.find(self.focus) if self.focus else None
        if e and e.text:
            e.text = e.text[:-1]

    def swipe(self, dx, dy):
        direction = ("right" if dx > 0 else "left") if abs(dx) > abs(dy) else ("down" if dy > 0 else "up")
        self.log.append(("swipe", direction))
        if direction == "right":
            self.back()
        return direction


# ----------------------- APP DE EJEMPLO ---------------------------
def build_home(p):
    p.label("title", 20, 40, 320, 40, "Mi App de Prueba")
    p.input("name", 20, 100, 320, 44, "")
    p.button("greet", 20, 160, 320, 44, "Saludar", on_greet)
    p.label("greeting", 20, 220, 320, 40, "")
    p.button("plus", 20, 280, 150, 44, "Contador +1", on_plus)
    p.label("counter", 190, 280, 150, 44, f"Cuenta: {p.state.get('count', 0)}")
    p.button("settings", 20, 560, 320, 44, "Ajustes", lambda ph, e: ph.navigate("settings"))


def build_settings(p):
    p.label("title", 20, 40, 320, 40, "Ajustes")
    p.button("dark", 20, 100, 320, 44, f"Modo oscuro: {'ON' if p.state.get('dark') else 'OFF'}", on_dark)
    p.button("back", 20, 560, 320, 44, "Volver", lambda ph, e: ph.back())


def on_greet(p, e):
    name = p.text_of("name").strip() or "mundo"
    p.find("greeting").text = f"¡Hola, {name}!"


def on_plus(p, e):
    p.state["count"] = p.state.get("count", 0) + 1
    p.find("counter").text = f"Cuenta: {p.state['count']}"


def on_dark(p, e):
    p.state["dark"] = not p.state.get("dark")
    e.text = f"Modo oscuro: {'ON' if p.state['dark'] else 'OFF'}"


def create_demo_phone():
    p = Phone()
    p.register("home", build_home)
    p.register("settings", build_settings)
    p.navigate("home", push=False)
    return p


# ----------------------------- GUI --------------------------------
class PhoneGUI:
    def __init__(self, phone):
        import tkinter as tk
        self.tk, self.phone = tk, phone
        self.root = tk.Tk()
        self.root.title("Celular de prueba")
        self.canvas = tk.Canvas(self.root, width=phone.width, height=phone.height, bg="white",
                                highlightthickness=4, highlightbackground="black")
        self.canvas.pack(padx=10, pady=10)
        self.canvas.bind("<Button-1>", self.on_press)
        self.canvas.bind("<ButtonRelease-1>", self.on_release)
        self.root.bind("<Key>", self.on_key)
        bar = tk.Frame(self.root)
        bar.pack(pady=(0, 10))
        tk.Button(bar, text="◀ Atrás", command=self.cmd(phone.back)).pack(side="left", padx=5)
        tk.Button(bar, text="⌂ Inicio", command=self.cmd(phone.home)).pack(side="left", padx=5)
        self.start = None
        self.draw()

    def cmd(self, fn):
        return lambda: (fn(), self.draw())

    def on_press(self, ev):
        self.start = (ev.x, ev.y)

    def on_release(self, ev):
        sx, sy = self.start or (ev.x, ev.y)
        dx, dy = ev.x - sx, ev.y - sy
        if abs(dx) > 40 or abs(dy) > 40:
            self.phone.swipe(dx, dy)
        else:
            self.phone.tap(ev.x, ev.y)
        self.draw()

    def on_key(self, ev):
        if ev.keysym == "BackSpace":
            self.phone.backspace()
        elif ev.char and ev.char.isprintable():
            self.phone.type_text(ev.char)
        self.draw()

    def draw(self):
        c, p = self.canvas, self.phone
        dark = p.state.get("dark")
        bg, fg = ("#1e1e1e", "white") if dark else ("white", "black")
        c.delete("all")
        c.configure(bg=bg)
        for e in p.elements:
            if e.kind == "button":
                c.create_rectangle(e.x, e.y, e.x + e.w, e.y + e.h, fill="#3b82f6", outline="")
                c.create_text(e.x + e.w / 2, e.y + e.h / 2, text=e.text, fill="white")
            elif e.kind == "input":
                border = "#3b82f6" if p.focus == e.id else "#999"
                c.create_rectangle(e.x, e.y, e.x + e.w, e.y + e.h, fill="white", outline=border, width=2)
                c.create_text(e.x + 8, e.y + e.h / 2, text=e.text or "Escribe tu nombre…",
                              anchor="w", fill="black" if e.text else "#aaa")
            else:
                c.create_text(e.x, e.y + e.h / 2, text=e.text, anchor="w", fill=fg, font=("Arial", 14))


# ---------------------------- PRUEBAS -----------------------------
class TestPhone(unittest.TestCase):
    def setUp(self):
        self.p = create_demo_phone()

    def test_contador(self):
        self.p.tap_id("plus")
        self.p.tap_id("plus")
        self.assertEqual(self.p.text_of("counter"), "Cuenta: 2")

    def test_saludo(self):
        self.p.tap_id("name")
        self.p.type_text("Ana")
        self.p.tap_id("greet")
        self.assertEqual(self.p.text_of("greeting"), "¡Hola, Ana!")

    def test_navegacion_y_swipe(self):
        self.p.tap_id("settings")
        self.assertEqual(self.p.current, "settings")
        self.p.swipe(120, 0)                     # deslizar a la derecha = atrás
        self.assertEqual(self.p.current, "home")

    def test_modo_oscuro(self):
        self.p.tap_id("settings")
        self.p.tap_id("dark")
        self.assertTrue(self.p.state["dark"])


if __name__ == "__main__":
    if len(sys.argv) > 1 and sys.argv[1] == "test":
        unittest.main(argv=[sys.argv[0]])
    else:
        PhoneGUI(create_demo_phone()).root.mainloop()
