# Unidad 1

## Bitácora de proceso de aprendizaje
ACTIVIDAD 1
¿Qué es un sistema físico interactivo?
 Es un sistema que mezcla componentes físicos con tecnología digital y reacciona en tiempo real a las acciones del usuario, utilizando sensores para identificar lo que sucede en el entorno. Después de procesar la información obtenida, produce una respuesta perceptible o tangible, como luces, sonidos, movimientos o variaciones en una pantalla. De esta manera, se establece una interacción directa entre el individuo y el sistema.
¿Cómo podrías aplicar lo que has visto en tu perfil profesional?
Los sistemas físicos interactivos tienen la capacidad de ser utilizados en el diseño de experiencias inmersivas en las que el usuario no solamente observa, sino que se involucra activamente con su cuerpo y su entorno. Ejemplos de esto son los videojuegos con sensores de movimiento, las instalaciones interactivas que fusionan animación, sonido y respuesta física, así como las experiencias de realidad aumentada o virtual.

ACTIVIDAD 2
¿Qué es el diseño/arte generativo?
El diseño o arte generativo es un tipo de creación en el que la persona que diseña o crea establece reglas, algoritmos o sistemas (usualmente a través de programación) y es el sistema mismo el que produce las formas, imágenes, sonidos o animaciones, fusionando la creatividad humana con procesos computacionales para dar origen a obras dinámicas y siempre cambiantes.
¿Cómo podrías aplicar lo que has visto en tu perfil profesional?
Puedo aplicar el diseño y arte generativo para crear experiencias interactivas y videojuegos dinámicos, donde los escenarios, animaciones, efectos visuales o narrativas se generen y transformen en tiempo real según las acciones del usuario, permitiéndome diseñar mundos más inmersivos.

AVTIVIDAD 3
1.Sacude el micro:bit. ¿Qué pasa?
El circulo en la parte de vista previa cambia a color verde y muestra un "c" en el centro
2.Presiona el botón Send Love. ¿Qué pasa?
Aparece una carita feliz en el microbit y un corazon, despues, vuelve a aprecer la carita feliz hasta que se vuelve a presionar el boton

ACTIVIDAD 4
1.¿Por qué no funcionaba el programa con was_pressed() y por qué funciona con is_pressed()? Explica detalladamente.
  Cuando se usa wasPressed(), el pulso se detecta momentáneamente y solo se vuelve verdadero durante un breve período de tiempo, por lo que no representa una acción continua en la pantalla. Por el contrario, el método isPressed() devuelve continuamente verdadero mientras se presiona el botón, proporcionando una respuesta más fluida y continua del microbit porque no se limita a un solo pulso, sino que continúa enviando datos durante la duración que se presiona el botón.

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
ACTIVIDAD 6
Vas a repasar lo aprendido en esta unidad. Regresa a la actividad 4 y trata de explicar en tus propias palabras de la manera más detallada que puedas cómo funciona el sistema físico interactivo. Analiza cada parte del código y su función dentro del sistema. Si aún tienes dudas sobre alguna parte, aprovecha para aclararlas.

El codigo muestra en vista previa un cuadrado verde, que cuando se presiona la tecla A en el microbit, cambia a color rojo 



