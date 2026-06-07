# Flet Python App Design Skill

## Flet Architecture
Flet uses Flutter rendering via Python. Every control is a widget. State lives in Python variables. Use `page.update()` or `control.update()` to sync UI.

## App Bootstrap (0.21+)
```python
import flet as ft

def main(page: ft.Page):
    page.title = "MyApp"
    page.theme_mode = ft.ThemeMode.DARK
    page.bgcolor = "#0A0E1A"
    page.padding = 0
    page.window_width = 400; page.window_height = 820
    page.rtl = True  # Arabic RTL
    page.fonts = {"Cairo": "https://fonts.googleapis.com/css2?family=Cairo:wght@400;700;900"}
    page.theme = ft.Theme(font_family="Cairo",
                           color_scheme=ft.ColorScheme(primary="#00D4FF"))
    # Build UI
    page.add(build_app(page))
    page.update()

ft.app(target=main, assets_dir="assets")
```

## Layout Patterns
```python
# Responsive column
def page_col(*controls, scroll=True) -> ft.Column:
    return ft.Column(
        list(controls),
        scroll=ft.ScrollMode.AUTO if scroll else None,
        spacing=12, expand=True)

# Card container
def card(content, pad=16, radius=16, bg="#141B2D", border="#1E2E48"):
    return ft.Container(content=content, bgcolor=bg,
                         border_radius=radius, padding=pad,
                         border=ft.Border.all(1, border),
                         animate=ft.Animation(200, ft.AnimationCurve.EASE_OUT))

# Stat row (label left, value right)
def stat_row(label, value, vc="#E8F4FF"):
    return ft.Container(
        ft.Row([ft.Text(label, size=13, color="#8899BB", font_family="Cairo", rtl=True),
                ft.Text(value, size=13, weight=ft.FontWeight.W_700, color=vc)],
               alignment=ft.MainAxisAlignment.SPACE_BETWEEN),
        padding=ft.Padding.symmetric(vertical=10),
        border=ft.Border.only(bottom=ft.BorderSide(1, "#FFFFFF07")))
```

## Navigation (Bottom Nav)
```python
class AppNav:
    def __init__(self, page, routes: dict):
        self.page = page; self.routes = routes; self.current = None
        self.content = ft.Container(expand=True)
        self.bar = ft.Container(
            ft.Row([self._btn(ico,lbl,k) for ico,lbl,k in [
                ("🏠","الرئيسية","home"),("📱","الأجهزة","devices"),
                ("⚙️","إعدادات","settings")]],
                   alignment=ft.MainAxisAlignment.SPACE_AROUND),
            height=70, bgcolor="#0F1525",
            border=ft.Border.only(top=ft.BorderSide(1,"#1E2E48")))

    def _btn(self, ico, lbl, key):
        return ft.Container(
            ft.Column([ft.Text(ico,size=20,text_align=ft.TextAlign.CENTER),
                        ft.Text(lbl,size=10,color="#445577",font_family="Cairo")],
                       horizontal_alignment=ft.CrossAxisAlignment.CENTER,spacing=3),
            expand=True, on_click=lambda e,k=key: self.navigate(k), ink=True)

    def navigate(self, key):
        self.current = key
        fn = self.routes.get(key)
        if fn: self.content.content = fn()
        try: self.page.update()
        except: pass

    def build(self):
        return ft.Column([self.content, self.bar], spacing=0, expand=True)
```

## State Management
```python
# Reactive state — call update callbacks on change
class Store:
    def __init__(self):
        self._data = {}
        self._listeners: dict[str, list] = {}

    def set(self, key, value):
        self._data[key] = value
        for cb in self._listeners.get(key, []): cb(value)

    def get(self, key, default=None): return self._data.get(key, default)

    def watch(self, key, callback): self._listeners.setdefault(key,[]).append(callback)

store = Store()
```

## Threading (background tasks)
```python
import threading, time

def run_bg(fn, *args, on_done=None, on_error=None):
    """Run fn in background thread, call on_done/on_error on UI thread."""
    def task():
        try:
            result = fn(*args)
            if on_done: on_done(result)
        except Exception as e:
            if on_error: on_error(e)
    threading.Thread(target=task, daemon=True).start()

# Usage
def load_data():
    time.sleep(2)  # simulate network
    return {"users": 5, "online": True}

run_bg(load_data,
       on_done=lambda d: (setattr(status_txt,"value",f"Users: {d['users']}"), page.update()),
       on_error=lambda e: (setattr(err_txt,"value",str(e)), page.update()))
```

## Forms & Validation
```python
class FormField:
    def __init__(self, label, hint="", password=False, validator=None):
        self.validator = validator
        self.error_txt = ft.Text("", size=11, color="#FF4444", visible=False)
        self.field = ft.TextField(
            label=label, hint_text=hint, password=password,
            can_reveal_password=password,
            text_style=ft.TextStyle(color="#E8F4FF", size=15),
            label_style=ft.TextStyle(color="#8899BB"),
            bgcolor="#1A2340", border_radius=12,
            border_color="#1E2E48", focused_border_color="#00D4FF",
            content_padding=ft.Padding.all(14))

    @property
    def value(self): return self.field.value or ""

    def validate(self) -> bool:
        if not self.validator: return True
        err = self.validator(self.value)
        self.error_txt.value = err or ""
        self.error_txt.visible = bool(err)
        self.field.border_color = "#FF4444" if err else "#1E2E48"
        try: self.field.update(); self.error_txt.update()
        except: pass
        return not err

    def build(self):
        return ft.Column([self.field, self.error_txt], spacing=2)
```

## Dialogs & Snackbars
```python
def show_snack(page, msg, kind="info"):
    colors = {"success":"#00FF88","error":"#FF4444","warning":"#FF8800","info":"#00D4FF"}
    icons  = {"success":"✅","error":"❌","warning":"⚠️","info":"ℹ️"}
    page.overlay.append(ft.SnackBar(
        content=ft.Row([ft.Text(icons[kind],size=18),
                         ft.Text(msg,color="#E8F4FF",font_family="Cairo",rtl=True,expand=True)],spacing=8),
        bgcolor=f"{colors[kind]}20", duration=3000, open=True))
    page.update()

def confirm_dialog(page, title, msg, on_yes):
    def close(result):
        dlg.open = False; page.update()
        if result: on_yes()
    dlg = ft.AlertDialog(
        modal=True, title=ft.Text(title, font_family="Cairo"),
        content=ft.Text(msg, font_family="Cairo", rtl=True),
        actions=[
            ft.TextButton("إلغاء", on_click=lambda e: close(False)),
            ft.FilledButton("تأكيد", on_click=lambda e: close(True),
                             style=ft.ButtonStyle(bgcolor="#FF4444",color="#fff")),
        ])
    page.overlay.append(dlg); dlg.open = True; page.update()
```

## APK Build Commands
```bash
# Install
pip install flet

# Run desktop
python main.py

# Build Android APK
flet build apk \
  --project "MyApp" \
  --org "com.mycompany" \
  --splash-color "#0A0E1A" \
  --android-permissions "android.permission.INTERNET"

# Build Web
flet build web

# Build Windows EXE
flet build windows
```

## Design System Constants
```python
# colors
BG="#0A0E1A"; CARD="#141B2D"; CYAN="#00D4FF"
GREEN="#00FF88"; RED="#FF4444"; ORANGE="#FF8800"
TEXT1="#E8F4FF"; TEXT2="#8899BB"; TEXT3="#445577"
BORDER="#1E2E48"

# spacing helpers
P = lambda v: ft.Padding.all(v)
PS = lambda h,v: ft.Padding.symmetric(horizontal=h,vertical=v)
BA = lambda w,c: ft.Border.all(w,c)
```
