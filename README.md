import tkinter as tk
from tkinter import filedialog, messagebox
import ezdxf

class CADApp:
    def __init__(self, root):
        self.root = root
        self.root.title("Basit Python CAD - DXF Destekli")
        self.root.geometry("1000 x 700")

        # Çizim modları: 'select', 'line', 'rectangle', 'circle'
        self.mode = "line"
        
        # Veri yapıları
        self.shapes = []        # Ekrandaki tüm şekiller
        self.undo_stack = []    # Undo için geçmiş durumlar
        self.redo_stack = []    # Redo için gelecek durumlar

        # Seçim ve Taşıma değişkenleri
        self.selected_shape = None
        self.drag_start = None

        # Geçici çizim değişkenleri
        self.start_point = None
        self.temp_shape_id = None

        self._setup_ui()
        self._bind_events()

    def _setup_ui(self):
        # Araç Çubuğu (Toolbar)
        toolbar = tk.Frame(self.root, bd=1, relief=tk.RAISED)
        toolbar.pack(side=tk.TOP, fill=tk.X)

        tk.Button(toolbar, text="Çizgi", command=lambda: self.set_mode("line")).pack(side=tk.LEFT, padx=2, pady=2)
        tk.Button(toolbar, text="Dikdörtgen", command=lambda: self.set_mode("rectangle")).pack(side=tk.LEFT, padx=2, pady=2)
        tk.Button(toolbar, text="Çember", command=lambda: self.set_mode("circle")).pack(side=tk.LEFT, padx=2, pady=2)
        tk.Button(toolbar, text="Seç / Taşı", command=lambda: self.set_mode("select")).pack(side=tk.LEFT, padx=2, pady=2)
        
        tk.Frame(toolbar, width=20).pack(side=tk.LEFT) # Ayraç
        
        tk.Button(toolbar, text="Geri Al (Undo)", command=self.undo).pack(side=tk.LEFT, padx=2, pady=2)
        tk.Button(toolbar, text="Yeniden Yap (Redo)", command=self.redo).pack(side=tk.LEFT, padx=2, pady=2)

        tk.Frame(toolbar, width=20).pack(side=tk.LEFT) # Ayraç

        tk.Button(toolbar, text="DXF Dışa Aktar", command=self.export_dxf).pack(side=tk.LEFT, padx=2, pady=2)

        # Çizim Alanı (Canvas)
        self.canvas = tk.Canvas(self.root, bg="white")
        self.canvas.pack(fill=tk.BOTH, expand=True)

    def _bind_events(self):
        self.canvas.bind("<ButtonPress-1>", self.on_press)
        self.canvas.bind("<B1-Motion>", self.on_drag)
        self.canvas.bind("<ButtonRelease-1>", self.on_release)

    def set_mode(self, mode):
        self.mode = mode
        self.selected_shape = None
        self.redraw()

    def save_state(self):
        """Mevcut durumun kopyasını undo yığınına ekler."""
        self.undo_stack.append([dict(shape) for shape in self.shapes])
        self.redo_stack.clear()

    # --- OLAY YÖNETİCİLERİ ---

    def on_press(self, event):
        x, y = event.x, event.y

        if self.mode == "select":
            items = self.canvas.find_withtag("current")
            if items:
                shape_id = items[0]
                for shape in self.shapes:
                    if shape["canvas_id"] == shape_id:
                        self.selected_shape = shape
                        self.drag_start = (x, y)
                        break
            else:
                self.selected_shape = None
            self.redraw()

        elif self.mode in ["line", "rectangle", "circle"]:
            self.start_point = (x, y)

    def on_drag(self, event):
        x, y = event.x, event.y

        if self.mode == "select" and self.selected_shape and self.drag_start:
            dx = x - self.drag_start[0]
            dy = y - self.drag_start[1]
            self.move_shape(self.selected_shape, dx, dy)
            self.drag_start = (x, y)
            self.redraw()

        elif self.mode in ["line", "rectangle", "circle"] and self.start_point:
            if self.temp_shape_id:
                self.canvas.delete(self.temp_shape_id)

            sx, sy = self.start_point
            if self.mode == "line":
                self.temp_shape_id = self.canvas.create_line(sx, sy, x, y, dash=(2, 2))
            elif self.mode == "rectangle":
                self.temp_shape_id = self.canvas.create_rectangle(sx, sy, x, y, dash=(2, 2))
            elif self.mode == "circle":
                r = ((x - sx)**2 + (y - sy)**2)**0.5
                self.temp_shape_id = self.canvas.create_oval(sx - r, sy - r, sx + r, sy + r, dash=(2, 2))

    def on_release(self, event):
        x, y = event.x, event.y

        if self.mode in ["line", "rectangle", "circle"] and self.start_point:
            if self.temp_shape_id:
                self.canvas.delete(self.temp_shape_id)
                self.temp_shape_id = None

            sx, sy = self.start_point
            if sx == x and sy == y:
                self.start_point = None
                return

            self.save_state()

            shape_data = {"type": self.mode}
            if self.mode == "line":
                shape_data["coords"] = [sx, sy, x, y]
            elif self.mode == "rectangle":
                shape_data["coords"] = [min(sx, x), min(sy, y), max(sx, x), max(sy, y)]
            elif self.mode == "circle":
                r = ((x - sx)**2 + (y - sy)**2)**0.5
                shape_data["coords"] = [sx, sy, r]

            self.shapes.append(shape_data)
            self.start_point = None
            self.redraw()

        elif self.mode == "select" and self.selected_shape and self.drag_start:
            self.save_state()
            self.drag_start = None

    # --- ŞEKİL KONTROLÜ VE ÇİZİM ---

    def move_shape(self, shape, dx, dy):
        if shape["type"] == "line":
            shape["coords"][0] += dx
            shape["coords"][1] += dy
            shape["coords"][2] += dx
            shape["coords"][3] += dy
        elif shape["type"] == "rectangle":
            shape["coords"][0] += dx
            shape["coords"][1] += dy
            shape["coords"][2] += dx
            shape["coords"][3] += dy
        elif shape["type"] == "circle":
            shape["coords"][0] += dx
            shape["coords"][1] += dy

    def redraw(self):
        self.canvas.delete("all")

        for shape in self.shapes:
            stype = shape["type"]
            coords = shape["coords"]
            color = "red" if shape == self.selected_shape else "black"
            width = 2 if shape == self.selected_shape else 1

            if stype == "line":
                cid = self.canvas.create_line(coords[0], coords[1], coords[2], coords[3], fill=color, width=width)
            elif stype == "rectangle":
                cid = self.canvas.create_rectangle(coords[0], coords[1], coords[2], coords[3], outline=color, width=width)
            elif stype == "circle":
                cx, cy, r = coords[0], coords[1], coords[2]
                cid = self.canvas.create_oval(cx - r, cy - r, cx + r, cy + r, outline=color, width=width)

            shape["canvas_id"] = cid

    # --- UNDO / REDO ---

    def undo(self):
        if self.undo_stack:
            self.redo_stack.append([dict(shape) for shape in self.shapes])
            self.shapes = self.undo_stack.pop()
            self.selected_shape = None
            self.redraw()

    def redo(self):
        if self.redo_stack:
            self.undo_stack.append([dict(shape) for shape in self.shapes])
            self.shapes = self.redo_stack.pop()
            self.selected_shape = None
            self.redraw()

    # --- DXF DISA AKTARMA ---

    def export_dxf(self):
        file_path = filedialog.asksaveasfilename(defaultextension=".dxf", filetypes=[("DXF Files", "*.dxf")])
        if not file_path:
            return

        doc = ezdxf.new(dxfversion="R2010")
        msp = doc.modelspace()

        # Ekran Y koordinatını CAD standartlarına (yukarı doğru pozitif Y) dönüştürmek için yükseklik
        canvas_height = self.canvas.winfo_height()

        for shape in self.shapes:
            stype = shape["type"]
            coords = shape["coords"]

            if stype == "line":
                p1 = (coords[0], canvas_height - coords[1])
                p2 = (coords[2], canvas_height - coords[3])
                msp.add_line(p1, p2)

            elif stype == "rectangle":
                x1, y1, x2, y2 = coords
                points = [
                    (x1, canvas_height - y1),
                    (x2, canvas_height - y1),
                    (x2, canvas_height - y2),
                    (x1, canvas_height - y2)
                ]
                msp.add_lwpolyline(points, close=True)

            elif stype == "circle":
                cx, cy, r = coords
                center = (cx, canvas_height - cy)
                msp.add_circle(center, r)

        doc.saveas(file_path)
        messagebox.showinfo("Başarılı", f"DXF dosyası başarıyla kaydedildi:\n{file_path}")

if __name__ == "__main__":
    root = tk.Tk()
    app = CADApp(root)
    root.mainloop()
