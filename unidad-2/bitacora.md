# Unidad 2

## Bitácora de proceso de aprendizaje

### Actividad 01

¿Cuáles son los estados en el programa?

Los estados del programa aparecen determinados como en WaitInOn y el WaitInOff, en los cuales el programa le dice al pixel que este prendido o apagado hasta que pase el evento Timer (hasta que el timer se acabe)

¿Cuáles son los eventos en el programa?

Los eventos del programa serian Entry, TimeOut y Exit en los cuales entry es cuando se entra al evento TimeOut, en el estado timeout se hace la cuenta atras y se deja que el Led este prendido o apagado por cierto tiempo hasta que llegue al evento exit donde sale de TimeOut y se genera un nuevo Evento, Accion o Estado. 

¿Cuáles son las acciones en el programa?

Las acciones del programa son encender y apagar el LED dependiendo del evento en el que se encuentre, cuando se cumple un evento se genera una accion.

### Actividad 02

Codigo Modificado:

    from microbit import *
    import utime

    class Timer:
    def __init__(self, owner, event_to_post, duration):
        self.owner = owner
        self.event = event_to_post
        self.duration = duration
        self.start_time = 0
        self.active = False

    def start(self, new_duration=None):
        if new_duration is not None:
            self.duration = new_duration
        self.start_time = utime.ticks_ms()
        self.active = True

    def stop(self):
        self.active = False

    def update(self):
        if self.active:
            if utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
                self.active = False
                self.owner.post_event(self.event)
 
    class Semaforo:
    def __init__(self,_x,_y,_timeInRed,_timeInGreen,_timeInYellow):
        self.event_queue = []
        self.timers = []
        self.x = _x
        self.y = _y
        self.timeInRed = _timeInRed
        self.timeInGreen = _timeInGreen
        self.timeInYellow = _timeInYellow
        self.myTimer = self.createTimer("Timeout",self.timeInRed)
        self.estado_actual = None
        self.transicion_a(self.estado_waitInRed)

    def createTimer(self,event,duration):
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        self.event_queue.append(ev)

    def update(self):
        for t in self.timers:
            t.update()
        while len(self.event_queue) > 0:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        if self.estado_actual: self.estado_actual("EXIT")
        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")

    def clear(self):
        display.set_pixel(self.x,self.y,0)
        display.set_pixel(self.x,self.y+1,0)
        display.set_pixel(self.x,self.y+2,0)

    def estado_waitInRed(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y,9)
            self.myTimer.start(self.timeInRed)
        if ev == "Timeout":
            display.set_pixel(self.x,self.y,0)
            self.transicion_a(self.estado_waitInGreen)

    def estado_waitInGreen(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+2,9)
            self.myTimer.start(self.timeInGreen)
        if ev == "Timeout":
            display.set_pixel(self.x,self.y+2,0)
            self.transicion_a(self.estado_waitInYellow)
        if ev == "A":
            display.set_pixel(self.x,self.y+2,0)
            self.transicion_a(self.estado_waitInYellow)

    def estado_waitInYellow(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+1,9)
            self.myTimer.start(self.timeInYellow)
        if ev == "Timeout":
            display.set_pixel(self.x,self.y+1,0)
            self.transicion_a(self.estado_waitInRed)

    semaforo1 = Semaforo(0,0,2000,1000,500)

      while True:
        if button_a.was_pressed():
        semaforo1.post_event("A")
        semaforo1.update()
        utime.sleep_ms(20)


PlantUML:

[![](https://img.plantuml.biz/plantuml/svg/VP6n3e8m48Rt9ds76z64k3iO78nnSI0in3WqS0EcjCbPT32-krUWY0fskkltfz-bbroT8jVKMWefJiipnF46KYnzoyYXR-0X1V3nvHfsnopDOGM5HaNHOWgXAT2KI43sOgS2hJLoLOq7muGVURwDUq8qmTarCzPlE7XlI2LEPyRgbdtASzJQteEzxsRqFtJmgrfHnN0cDvwXWs48_qr-1M7gZ3EsSL9q5nuDh_rZModwkb_eE2z5mcEsIty1)](https://editor.plantuml.com/uml/VP6n3e8m48Rt9ds76z64k3iO78nnSI0in3WqS0EcjCbPT32-krUWY0fskkltfz-bbroT8jVKMWefJiipnF46KYnzoyYXR-0X1V3nvHfsnopDOGM5HaNHOWgXAT2KI43sOgS2hJLoLOq7muGVURwDUq8qmTarCzPlE7XlI2LEPyRgbdtASzJQteEzxsRqFtJmgrfHnN0cDvwXWs48_qr-1M7gZ3EsSL9q5nuDh_rZModwkb_eE2z5mcEsIty1)


### Actividad 03:

Algo que aveces se me dificulta es diferenciar algunas cosas de la primera actividad, diferenciar aveces las acciones, eventos y estados, sin embargo aparte de esto, creo que estoy bien

## Bitácora de aplicación 

### Actividad 04:

Maquina de estados:

[![](https://img.plantuml.biz/plantuml/svg/TP9TIyCm68Nl0_aFBxrIOLDlbSasTJ2O8EhTA2DiSGQQfEJ33yJ_RfEiQLEQNeNtFCqvESbS6wACvHL6suiI9fCxk9VYdHsj8dic1KOODOYXwHX-COBcQ_zvajeeefxPDwse3bewyzXQx1NbL3IUS0F5eiYI3qnp9Yppt3BFBHGdwernwi7NmkLrQCWy5YielIJzSRJj3piFZFwSGLg5xyowhLxeKL7DFacNvTJTG1wqzwXhgeA5j2PewMm4SnwdPG6N87PrMyiChc_Xm7HTaP-f_f1VPwlrPhEDH7cUjKXFrhkY4NR5H2m-1kbSO0PNPNeTO-SCIhjNM_2Whsoqc25BO3Iduk0V-3y0)](https://editor.plantuml.com/uml/TP9TIyCm68Nl0_aFBxrIOLDlbSasTJ2O8EhTA2DiSGQQfEJ33yJ_RfEiQLEQNeNtFCqvESbS6wACvHL6suiI9fCxk9VYdHsj8dic1KOODOYXwHX-COBcQ_zvajeeefxPDwse3bewyzXQx1NbL3IUS0F5eiYI3qnp9Yppt3BFBHGdwernwi7NmkLrQCWy5YielIJzSRJj3piFZFwSGLg5xyowhLxeKL7DFacNvTJTG1wqzwXhgeA5j2PewMm4SnwdPG6N87PrMyiChc_Xm7HTaP-f_f1VPwlrPhEDH7cUjKXFrhkY4NR5H2m-1kbSO0PNPNeTO-SCIhjNM_2Whsoqc25BO3Iduk0V-3y0)

Codigo del Programa Para el ScapeRoom:

    from microbit import *
    import utime
    import music

    def make_fill_images(on='9', off='0'):
    imgs = []
    for n in range(26):
        rows = []
        k = 0
        for y in range(5):
            row = []
            for x in range(5):
                row.append(on if k < n else off)
                k += 1
            rows.append(''.join(row))
        imgs.append(Image(':'.join(rows)))
    return imgs

    FILL = make_fill_images()

    class Timer:
    def __init__(self, owner, event_to_post, duration):
        self.owner = owner
        self.event = event_to_post
        self.duration = duration
        self.start_time = 0
        self.active = False

    def start(self, new_duration=None):
        if new_duration is not None:
            self.duration = new_duration
        self.start_time = utime.ticks_ms()
        self.active = True

    def stop(self):
        self.active = False

    def update(self):
        if self.active:
            if utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
                self.active = False
                self.owner.post_event(self.event)

    class EscapeRoomTimer:
    def __init__(self):
        self.event_queue = []
        self.timers = []
        self.n_pixeles = 20
        self.myTimer = self.createTimer("Timeout",1000)
        self.estado_actual = None
        self.transicion_a(self.estado_configuracion)

    def createTimer(self, event, duration):
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        self.event_queue.append(ev)

    def update(self):
        for t in self.timers:
            t.update()
        while len(self.event_queue) > 0:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        if self.estado_actual: self.estado_actual("EXIT")
        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")

    def estado_configuracion(self, ev):
        if ev == "ENTRY":
            display.show(FILL[self.n_pixeles])
        if ev == "A":
            if self.n_pixeles < 25:
                self.n_pixeles += 1
                display.show(FILL[self.n_pixeles])
        if ev == "B":
            if self.n_pixeles > 15:
                self.n_pixeles -= 1
                display.show(FILL[self.n_pixeles])
        if ev == "S":
            self.transicion_a(self.estado_cuenta_regresiva)

    def estado_cuenta_regresiva(self, ev):
        if ev == "ENTRY":
            self.myTimer.start(1000)
        if ev == "Timeout":
            if self.n_pixeles > 0:
                self.n_pixeles -= 1
                display.show(FILL[self.n_pixeles])
                self.myTimer.start(1000)
            else:
                display.show(Image.SKULL)
                for _ in range(3):
                    music.pitch(440, 200)
                    utime.sleep_ms(100)
        if ev == "A":
            self.n_pixeles = 20
            self.transicion_a(self.estado_configuracion)

    escape_timer = EscapeRoomTimer()

    while True:
    if button_a.was_pressed():
        escape_timer.post_event("A")
    if button_b.was_pressed():
        escape_timer.post_event("B")
    if accelerometer.was_gesture("shake"):
        escape_timer.post_event("S")
    escape_timer.update()
    utime.sleep_ms(20)



## Bitácora de reflexión
## Actividad 5
Explica cómo resolviste el reto.

Coloca el código final en tu bitácora tanto para el micro:bit como para p5.js.
