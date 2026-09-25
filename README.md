[solar_system.py](https://github.com/user-attachments/files/32660359/solar_system.py)
# -
солнечная система на питоне

import tkinter as tk
import math, random

W, H = 1100, 760
FOCAL = 900.0
DIST0 = 1200.0

# name, a(orbit radius, px), radius(px), color, angular speed, phase
PLANETS = [
    ("Меркурий", 70,  4.5, "#b7a99a", 4.15, 0.3),
    ("Венера",   100, 7.0, "#e8c27a", 3.05, 2.1),
    ("Земля",    135, 7.5, "#4f8fe0", 2.45, 4.0),
    ("Марс",     175, 6.0, "#c1440e", 1.95, 1.2),
    ("Юпитер",   260, 16.0,"#d8a56a", 1.05, 5.2),
    ("Сатурн",   330, 13.5,"#e0c088", 0.78, 2.7),
    ("Уран",     395, 9.0, "#8fd8e0", 0.55, 3.9),
    ("Нептун",   450, 8.5, "#4f6fd8", 0.44, 0.8),
]
SUN_R = 26
SUN_COLOR = "#ffcf33"

class App:
    def __init__(self, root):
        self.root = root
        self.canvas = tk.Canvas(root, width=W, height=H, bg="#05060a", highlightthickness=0)
        self.canvas.pack(fill="both", expand=True)
        self.az = 0.6
        self.el = 0.45
        self.dist = DIST0
        self.t = 0.0
        self.paused = False
        self.auto = True
        self.drag = None
        self.stars = self._make_stars()
        self.canvas.bind("<Button-1>", self._down)
        self.canvas.bind("<B1-Motion>", self._move)
        self.canvas.bind("<ButtonRelease-1>", lambda e: setattr(self, "drag", None))
        self.canvas.bind("<MouseWheel>", self._wheel)
        self.canvas.bind("<Button-4>", lambda e: self._zoom(1.1))
        self.canvas.bind("<Button-5>", lambda e: self._zoom(0.9))
        root.bind("<space>", lambda e: self._pause())
        root.bind("r", lambda e: self._reset())
        root.bind("a", lambda e: self._toggle_auto())
        self.loop()

    def _make_stars(self):
        random.seed(3)
        return [(random.randint(0, W), random.randint(0, H), random.choice([1, 1, 1, 2])) for _ in range(420)]

    def _basis(self):
        ca, sa = math.cos(self.az), math.sin(self.az)
        ce, se = math.cos(self.el), math.sin(self.el)
        fx, fy, fz = ce*sa, se, ce*ca
        rx, ry, rz = ca, 0.0, -sa
        ux = ry*fz - rz*fy
        uy = rz*fx - rx*fz
        uz = rx*fy - ry*fx
        return (rx, ry, rz), (ux, uy, uz), (fx, fy, fz)

    def project(self, x, y, z, right, up, fwd):
        cx = x*right[0] + y*right[1] + z*right[2]
        cy = x*up[0] + y*up[1] + z*up[2]
        cz = x*fwd[0] + y*fwd[1] + z*fwd[2]
        d = self.dist - cz
        if d < 1:
            d = 1
        f = FOCAL / d
        return W/2 + cx*f, H/2 - cy*f, cz, f

    def _draw_static(self):
        self.canvas.delete("all")
        for sx, sy, r in self.stars:
            self.canvas.create_oval(sx-r, sy-r, sx+r, sy+r, fill="#9fb0d0", outline="")

    def _down(self, e):
        self.drag = (e.x, e.y)

    def _move(self, e):
        if not self.drag:
            return
        dx = e.x - self.drag[0]
        dy = e.y - self.drag[1]
        self.drag = (e.x, e.y)
        self.az -= dx * 0.008
        self.el += dy * 0.006
        self.el = max(-1.4, min(1.4, self.el))

    def _wheel(self, e):
        self._zoom(1.1 if e.delta > 0 else 0.9)

    def _zoom(self, k):
        self.dist = max(300, min(4000, self.dist / k))

    def _pause(self):
        self.paused = not self.paused

    def _reset(self):
        self.az, self.el, self.dist = 0.6, 0.45, DIST0

    def _toggle_auto(self):
        self.auto = not self.auto

    def _orbit(self, a, color, right, up, fwd):
        pts = []
        for i in range(0, 73):
            ang = i/72*2*math.pi
            x, y, z = a*math.cos(ang), 0.0, a*math.sin(ang)
            sx, sy, cz, f = self.project(x, y, z, right, up, fwd)
            pts += [sx, sy]
        self.canvas.create_line(*pts, fill=color, width=1, tags="dyn")

    def loop(self):
        if not self.paused:
            self.t += 0.05
            if self.auto and not self.drag:
                self.az += 0.003
        self._draw_static()
        right, up, fwd = self._basis()

        for name, a, r, color, w, ph in PLANETS:
            self._orbit(a, "#2a3350", right, up, fwd)

        bodies = []
        # sun
        sx, sy, cz, f = self.project(0, 0, 0, right, up, fwd)
        bodies.append((cz, "sun", sx, sy, f, None))
        for name, a, r, color, w, ph in PLANETS:
            ang = ph + w * self.t
            x, y, z = a*math.cos(ang), 0.0, a*math.sin(ang)
            sx, sy, cz, f = self.project(x, y, z, right, up, fwd)
            bodies.append((cz, name, sx, sy, f, (a, r, color, name)))

        bodies.sort(key=lambda b: b[0])
        for cz, tag, sx, sy, f, info in bodies:
            if tag == "sun":
                R = SUN_R * f
                self.canvas.create_oval(sx-R*1.8, sy-R*1.8, sx+R*1.8, sy+R*1.8,
                                        fill="#3a2a00", outline="", tags="dyn")
                self.canvas.create_oval(sx-R, sy-R, sx+R, sy+R, fill=SUN_COLOR, outline="", tags="dyn")
                self.canvas.create_text(sx, sy-R-10, text="Солнце", fill="#ffcf33",
                                        font=("TkDefaultFont", 9), tags="dyn")
            else:
                a, r, color, name = info
                R = max(1.5, r * f)
                self.canvas.create_oval(sx-R, sy-R, sx+R, sy+R, fill=color, outline="", tags="dyn")
                # highlight
                hr = R*0.45
                self.canvas.create_oval(sx-R*0.5-hr, sy-R*0.5-hr, sx-R*0.5+hr, sy-R*0.5+hr,
                                        fill="#ffffff", outline="", tags="dyn")
                self.canvas.create_text(sx, sy-R-8, text=name, fill="#c8d0e0",
                                        font=("TkDefaultFont", 8), tags="dyn")
        self.canvas.tag_raise("dyn")
        self.root.after(33, self.loop)

if __name__ == "__main__":
    root = tk.Tk()
    root.title("Солнечная система")
    App(root)
    root.mainloop()
