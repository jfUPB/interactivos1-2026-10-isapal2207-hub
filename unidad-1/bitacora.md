# Unidad 1

## Bitácora de proceso de aprendizaje
ACTIVIDAD 1
¿Qué es un sistema físico interactivo?
¿Cómo podrías aplicar lo que has visto en tu perfil profesional?

ACTIVIDAD 2
¿Qué es el diseño/arte generativo?
¿Cómo podrías aplicar lo que has visto en tu perfil profesional?

AVTIVIDAD 3
1.Sacude el micro:bit. ¿Qué pasa?
El circulo en la parte de vista previa cambia a color verde y muestra un "c" en el centro
2.Presiona el botón Send Love. ¿Qué pasa?
Aparece una carita feliz en el microbit y un corazon, despues, vuelve a aprecer la carita feliz hasta que se vuelve a presionar el boton

ACTIVIDAD 4
1.¿Por qué no funcionaba el programa con was_pressed() y por qué funciona con is_pressed()? Explica detalladamente.
 
## Bitácora de aplicación 
PROGRAMA DE P5.JS

let port;
let connectBtn;

function setup() {
    createCanvas(400, 400);
    background(220);
    port = createSerial();
    connectBtn = createButton('Connect to micro:bit');
    connectBtn.position(80, 300);
    connectBtn.mousePressed(connectBtnClick);
    
    fill('blue');
    MOVIMIENTOx = width / 2;
    ellipse(MOVIMIENTOx, height / 2, 100,100);
}

function draw() {

    if(port.availableBytes() > 0){
        let dataRx = port.read(1);
        if(dataRx == 'A'){
            fill('pink');
   
          MOVIMIENTOx = MOVIMIENTOx - 20;
         
        }
        else if(dataRx == 'B'){
            fill('yellow');
            MOVIMIENTOx = MOVIMIENTOx + 20;
        }

        background(220);
        ellipse(MOVIMIENTOx, height / 2, 100,100);
        fill('purple');
        
    }


    if (!port.opened()) {
        connectBtn.html('Connect to micro:bit');
    }
    else {
        connectBtn.html('Disconnect');
    }
}

function connectBtnClick() {
    if (!port.opened()) {
        port.open('MicroPython', 115200);
    } else {
        port.close();
    }
}

function sendBtnClick() {
    port.write('h');
}


PROGRAMA DE MICROBIT

from microbit import *

uart.init(baudrate=115200)


while True:
    if button_a.is_pressed():
        uart.write('A')
        sleep(500)
    if button_b.is_pressed():
        uart.write('B')
        sleep(500)



## Bitácora de reflexión
