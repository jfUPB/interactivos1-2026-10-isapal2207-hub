# Unidad 8

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 

Sketch: 

```
const EVENTS = {
    CONNECT: "CONNECT",
    DISCONNECT: "DISCONNECT",
    DATA: "DATA",
    KEY_PRESSED: "KEY_PRESSED",
    KEY_RELEASED: "KEY_RELEASED",
};

class PainterTask extends FSMTask {
    constructor() {
        super();

        this.eventQueue = [];
        this.activeAnimations = [];
        this.mode = null;

        this.oscColor         = [220, 60, 50];
        this.oscColorOverride = false;
        this.oscSize          = 0.5;
        this.oscBgLayer       = false;
        this.oscSpeed         = 1.0;

        this.transitionTo(this.estado_esperando);
        this.rxData = { x: 0, y: 0, btnA: false, btnB: false };
        this.modoBombo = 0;
        this.modoLinea = 0;
    }

    update() {
        super.update();
        this._flushQueue();
    }

    _flushQueue() {
        if (this.mode !== "strudel") return;

        const now = Date.now();
        this.eventQueue.sort((a, b) => a.timestamp - b.timestamp);

        while (this.eventQueue.length > 0) {
            const ev = this.eventQueue[0];

            if (ev.timestamp <= now) {
                const d = ev.payload;

                const col = this.oscColorOverride
                    ? this.oscColor
                    : getColorForSound(d.s);

                const scaledDelta = (d.delta * map(this.oscSize, 0, 1, 0.3, 2.0)) / this.oscSpeed;
                this.activeAnimations.push({
                    startTime: ev.timestamp,
                    duration:  scaledDelta * 1000,
                    type:      d.s,
                    x:         random(width * 0.2, width * 0.8),
                    y:         random(height * 0.2, height * 0.8),
                    color:     col,
                    size:      this.oscSize,
                    seed:      Math.floor(random(10000)), // semilla para ruido consistente por evento
                });

                this.eventQueue.shift();
            } else break;
        }
    }

    estado_esperando = (ev) => {
        if (ev.type === "ENTRY") {
            cursor();
            console.log("Waiting for connection...");
        } else if (ev.type === EVENTS.CONNECT) {
            this.transitionTo(this.estado_corriendo);
        }
    };

    estado_corriendo = (ev) => {
        if (ev.type === "ENTRY") {
            noCursor();
            background(0);
            console.log("Connected. Waiting for data...");
        } else if (ev.type === EVENTS.DISCONNECT) {
            this.mode = null;
            this.transitionTo(this.estado_esperando);
        } else if (ev.type === EVENTS.DATA) {
            this.updateLogic(ev.payload);
        } else if (ev.type === EVENTS.KEY_PRESSED) {
            this.handleKeys(ev.keyCode, ev.key);
        } else if (ev.type === EVENTS.KEY_RELEASED) {
            this.handleKeyRelease(ev.keyCode, ev.key);
        } else if (ev.type === "EXIT") {
            cursor();
        }
    };

    updateLogic(data) {
        if (data.type === "strudel") {
            this.mode = "strudel";
            this.eventQueue.push(data);
            return;
        }

        if (data.type === "osc") {
            this._applyOscControl(data.payload);
            return;
        }
        if (data.type === "microbit") {
            if (data.btnA && !this.prevBtnA) this.modoBombo = (this.modoBombo + 1) % 3;
            if (data.btnB && !this.prevBtnB) this.modoLinea = (this.modoLinea + 1) % 3;
            this.prevBtnA = data.btnA;
            this.prevBtnB = data.btnB;
            this.rxData   = data;
            return;
        }
    }

    _applyOscControl({ address, args }) {
        if (address === "/color") {
            this.oscColor = [
                Math.round(args[0]),
                Math.round(args[1]),
                Math.round(args[2])
            ];
            this.oscColorOverride = true;
            return;
        }

        if (address === "/size") {
            this.oscSize = constrain(args[0] ?? 0.5, 0, 1);
            return;
        }

        if (address === "/bg_layer") {
            this.oscBgLayer = args[0] === 1;
            return;
        }

        if (address === "/speed") {
            this.oscSpeed = constrain(args[0] ?? 1.0, 0.1, 2.0);
            return;
        }
    }
}

let painter;
let bridge;
let connectBtn;
const renderer = new Map();

function setup() {
    createCanvas(windowWidth, windowHeight);
    background(0);

    painter = new PainterTask();
    bridge = new BridgeClient("ws://127.0.0.1:8081");

    bridge.onConnect(() => {
        connectBtn.html("Disconnect");
        painter.postEvent({ type: EVENTS.CONNECT });
    });

    bridge.onDisconnect(() => {
        connectBtn.html("Connect");
        painter.postEvent({ type: EVENTS.DISCONNECT });
    });

    bridge.onStatus((s) => console.log("BRIDGE STATUS:", s.state, s.detail ?? ""));

    bridge.onData((data) => {
        if (data.type === "strudel") {
            painter.postEvent({ type: EVENTS.DATA, payload: data });
            return;
        }
        if (data.type === "osc") {
            painter.postEvent({ type: EVENTS.DATA, payload: data });
            return;
        }
        if (data.type === "microbit") {
            painter.postEvent({ type: EVENTS.DATA, payload: data });
            return;
        }
    });

    connectBtn = createButton("Connect");
    connectBtn.position(10, 10);
    connectBtn.mousePressed(() => {
        if (bridge.isOpen) bridge.close();
        else bridge.open();
    });

    renderer.set(painter.estado_corriendo, drawRunning);
}

function draw() {
    painter.update();
    renderer.get(painter.state)?.();
}

function drawRunning() {
    if (painter.mode === "strudel") {

        if (painter.oscBgLayer) {
            const [r, g, b] = painter.oscColor;
            background(r * 0.1, g * 0.1, b * 0.1, 30);
        } else {
            background(0, 30);
        }

        const now = Date.now();

        for (let i = painter.activeAnimations.length - 1; i >= 0; i--) {
            const anim = painter.activeAnimations[i];
            const elapsed = now - anim.startTime;
            const p = elapsed / anim.duration;

            if (p <= 1) {
                dibujarElemento(anim, p);
            } else {
                painter.activeAnimations.splice(i, 1);
            }
        }

        return;
    }
}

// ── FUNCIONES DE DIBUJO — ESTÉTICA GLITCH/CAÓTICO ─────────────────────────────

function dibujarElemento(anim, p) {
    push();
    switch (anim.type) {
        case 'tr909bd': dibujarBombo(anim, p, anim.color); break;
        case 'tr909sd':
        case 'tr909cp': dibujarCaja(anim, p, anim.color);  break;
        case 'tr909hh':
        case 'tr909oh': dibujarHat(anim, p, anim.color);   break;
        default:        dibujarDefault(anim, p, anim.color); break;
    }
    pop();
}

// ── BOMBO: bloques de glitch desplazados ──────────────────────────────────────
// Modo 0: bloques RGB desplazados (aberración cromática)
// Modo 1: explosión de segmentos rotos hacia afuera
// Modo 2: campo de píxeles de ruido que se disuelve
function dibujarBombo(anim, p, c) {
    const maxSz = map(anim.size, 0, 1, 80, 400);
    const alpha  = lerp(255, 0, p);

    const px = map(painter.rxData.x, -2048, 2047, width  * 0.2, width  * 0.8);
    const py = map(painter.rxData.y, -2048, 2047, height * 0.2, height * 0.8);

    randomSeed(anim.seed);

    if (painter.modoBombo === 0) {
        // Aberración cromática: 3 copias del bloque desplazadas en R, G, B
        const sz    = lerp(maxSz, maxSz * 0.1, p);
        const drift = lerp(maxSz * 0.3, 0, p); // desplazamiento máximo al inicio

        noStroke();
        rectMode(CENTER);

        // Canal R — desplazado izquierda
        fill(c[0], 0, 0, alpha * 0.85);
        rect(px - drift, py, sz * lerp(2.5, 0.5, p), sz * 0.4);

        // Canal G — centrado
        fill(0, c[1], 0, alpha * 0.85);
        rect(px, py + drift * 0.5, sz * lerp(2.0, 0.4, p), sz * 0.4);

        // Canal B — desplazado derecha
        fill(0, 0, c[2], alpha * 0.85);
        rect(px + drift, py - drift * 0.3, sz * lerp(2.5, 0.5, p), sz * 0.4);

        // Bloque blanco central parpadeante
        if (random() > 0.4) {
            fill(255, 255, 255, alpha * 0.6);
            rect(px, py, sz * 0.3, sz * 0.15);
        }

    } else if (painter.modoBombo === 1) {
        // Segmentos rotos que explotan hacia afuera
        const n   = 10;
        const rad = lerp(0, maxSz, p);
        strokeWeight(lerp(4, 1, p));
        noFill();

        for (let i = 0; i < n; i++) {
            randomSeed(anim.seed + i);
            const angle  = (TWO_PI / n) * i + random(-0.4, 0.4);
            const len    = lerp(maxSz * 0.5, 5, p) * random(0.5, 1.5);
            const jitter = random(-8, 8) * (1 - p);

            const x1 = px + cos(angle) * rad;
            const y1 = py + sin(angle) * rad;
            const x2 = px + cos(angle) * (rad + len);
            const y2 = py + sin(angle) * (rad + len);

            // Línea con color glitch
            const glitchR = random() > 0.7 ? 255 : c[0];
            const glitchG = random() > 0.7 ? 255 : c[1];
            const glitchB = random() > 0.7 ? 255 : c[2];
            stroke(glitchR, glitchG, glitchB, alpha);
            line(x1 + jitter, y1 + jitter, x2, y2);

            // Tick perpendicular al final
            if (random() > 0.5) {
                const perpX = cos(angle + HALF_PI) * 10 * (1 - p);
                const perpY = sin(angle + HALF_PI) * 10 * (1 - p);
                line(x2 - perpX, y2 - perpY, x2 + perpX, y2 + perpY);
            }
        }

    } else {
        // Campo de píxeles de ruido que aparece y se disuelve
        const cols   = 12;
        const cellSz = map(anim.size, 0, 1, 8, 30);
        const spread = lerp(maxSz, maxSz * 0.3, p);
        noStroke();

        for (let i = 0; i < cols; i++) {
            for (let j = 0; j < cols; j++) {
                randomSeed(anim.seed + i * 100 + j);
                if (random() > lerp(0.3, 0.85, p)) continue; // se van apagando

                const ox = (i - cols / 2) * (spread / cols);
                const oy = (j - cols / 2) * (spread / cols);

                // Color de cada pixel puede corromperse
                const corrupt = random() > 0.75;
                fill(
                    corrupt ? 255 : c[0],
                    corrupt ? 0   : c[1],
                    corrupt ? 80  : c[2],
                    alpha
                );
                rect(px + ox, py + oy, cellSz * random(0.5, 1.5), cellSz * random(0.2, 1.0));
            }
        }
    }
}

// ── CAJA/SNARE: scanlines y rectángulos fragmentados ─────────────────────────
// Modo 0: scanlines horizontales que se cortan y desplazan
// Modo 1: rectángulos fragmentados que caen
// Modo 2: barras de error tipo VHS
function dibujarCaja(anim, p, c) {
    const maxW  = map(anim.size, 0, 1, width * 0.3, width);
    const alpha = lerp(255, 0, p);

    randomSeed(anim.seed);

    if (painter.modoLinea === 0) {
        // Scanlines: líneas horizontales con cortes de glitch
        const nLines    = Math.floor(map(anim.size, 0, 1, 6, 20));
        const totalH    = map(anim.size, 0, 1, 60, 200);
        const lineH     = totalH / nLines;
        noStroke();

        for (let i = 0; i < nLines; i++) {
            randomSeed(anim.seed + i);
            const w       = lerp(maxW * random(0.5, 1.0), 0, p);
            const yOff    = (i - nLines / 2) * lineH;
            const xGlitch = random(-40, 40) * (1 - p); // desplazamiento horizontal

            // Probabilidad de que la línea sea blanca (corrupción)
            const isCorrupt = random() > 0.8;
            fill(
                isCorrupt ? 255 : c[0],
                isCorrupt ? 255 : c[1],
                isCorrupt ? 255 : c[2],
                alpha
            );
            rect(width / 2 - w / 2 + xGlitch, height / 2 + yOff, w, lineH * 0.7);
        }

    } else if (painter.modoLinea === 1) {
        // Rectángulos que se fragmentan y caen
        const n    = Math.floor(map(anim.size, 0, 1, 4, 12));
        noStroke();
        rectMode(CORNER);

        for (let i = 0; i < n; i++) {
            randomSeed(anim.seed + i * 77);
            const frac  = random(0.1, 0.4);
            const w     = maxW * frac;
            const h     = random(10, 50);
            const startX = random(width * 0.1, width * 0.9 - w);
            const fallY  = lerp(0, height * 0.3, p) * random(0.5, 1.5);

            fill(c[0], c[1], c[2], alpha * random(0.5, 1.0));
            rect(startX, height / 2 - h / 2 + fallY, w, h);

            // Línea de "corte" encima del bloque
            if (random() > 0.5) {
                stroke(255, 255, 255, alpha * 0.5);
                strokeWeight(1);
                line(startX, height / 2 + fallY, startX + w, height / 2 + fallY);
                noStroke();
            }
        }

    } else {
        // Barras de error VHS: bloques anchos de color corrupto
        const nBars = Math.floor(map(anim.size, 0, 1, 3, 8));
        noStroke();
        rectMode(CENTER);

        for (let i = 0; i < nBars; i++) {
            randomSeed(anim.seed + i * 33);
            const barW   = lerp(maxW * random(0.6, 1.0), 0, p);
            const barH   = random(8, 40);
            const yOff   = random(-height * 0.25, height * 0.25);
            const xShift = random(-30, 30) * (1 - p);

            // Colores alternados: color OSC vs inversión vs blanco
            const t = i % 3;
            if      (t === 0) fill(c[0], c[1], c[2], alpha);
            else if (t === 1) fill(255 - c[0], 255 - c[1], 255 - c[2], alpha * 0.8);
            else              fill(255, 255, 255, alpha * 0.4);

            rect(width / 2 + xShift, height / 2 + yOff, barW, barH);
        }
    }
}

// ── HAT: estático, ruido y señal corrupta ─────────────────────────────────────
// Modo 0: puntos de ruido estático dispersos
// Modo 1: líneas erráticas tipo osciloscopio roto
// Modo 2: cruces distorsionadas con aberración
function dibujarHat(anim, p, c) {
    const maxSz = map(anim.size, 0, 1, 30, 120);
    const alpha  = lerp(220, 0, p);

    const px = map(painter.rxData.x, -2048, 2047, width  * 0.1, width  * 0.9);
    const py = map(painter.rxData.y, -2048, 2047, height * 0.1, height * 0.9);

    randomSeed(anim.seed);

    if (painter.modoLinea === 0) {
        // Puntos de ruido estático (como televisor sin señal)
        const n      = Math.floor(map(anim.size, 0, 1, 20, 80));
        const spread = maxSz * 3;
        noStroke();

        for (let i = 0; i < n; i++) {
            randomSeed(anim.seed + i);
            if (random() > lerp(0.9, 0.3, p)) continue;

            const ox = random(-spread, spread);
            const oy = random(-spread, spread);
            const sz = random(1, 6);

            // Mayoría en color base, algunos blancos o invertidos
            const r = random();
            if      (r > 0.85) fill(255, 255, 255, alpha);
            else if (r > 0.70) fill(255, 0, 80, alpha);
            else               fill(c[0], c[1], c[2], alpha);

            rect(px + ox, py + oy, sz, sz * random(0.5, 3.0));
        }

    } else if (painter.modoLinea === 1) {
        // Osciloscopio roto: línea con jitter vertical extremo
        const n    = Math.floor(map(anim.size, 0, 1, 8, 24));
        const spanX = maxSz * 3;
        strokeWeight(1.5);
        noFill();

        // Línea principal
        stroke(c[0], c[1], c[2], alpha);
        beginShape();
        for (let i = 0; i <= n; i++) {
            randomSeed(anim.seed + i);
            const x = px - spanX + (spanX * 2 / n) * i;
            // El jitter es alto al inicio y decae
            const jitter = random(-maxSz, maxSz) * (1 - p);
            vertex(x, py + jitter);
        }
        endShape();

        // Segunda línea corrupta encima (offset de color)
        stroke(255, 255 - c[1], 0, alpha * 0.5);
        beginShape();
        for (let i = 0; i <= n; i++) {
            randomSeed(anim.seed + i + 1000);
            const x = px - spanX + (spanX * 2 / n) * i;
            const jitter = random(-maxSz * 0.5, maxSz * 0.5) * (1 - p);
            vertex(x + 3, py + jitter);
        }
        endShape();

    } else {
        // Cruces distorsionadas con aberración cromática
        const n = Math.floor(map(anim.size, 0, 1, 2, 6));

        for (let i = 0; i < n; i++) {
            randomSeed(anim.seed + i * 55);
            const ox   = random(-maxSz * 2, maxSz * 2) * (1 - p * 0.5);
            const oy   = random(-maxSz * 1, maxSz * 1) * (1 - p * 0.5);
            const sz   = maxSz * random(0.3, 1.0) * (1 - p * 0.7);
            const drift = lerp(8, 0, p);

            // Cruz R
            stroke(c[0], 0, 0, alpha);
            strokeWeight(2);
            line(px + ox - sz - drift, py + oy, px + ox + sz - drift, py + oy);
            line(px + ox - drift, py + oy - sz, px + ox - drift, py + oy + sz);

            // Cruz G (ligeramente desplazada)
            stroke(0, c[1], 0, alpha * 0.8);
            line(px + ox - sz, py + oy + drift, px + ox + sz, py + oy + drift);
            line(px + ox, py + oy - sz + drift, px + ox, py + oy + sz + drift);

            // Cruz B
            stroke(0, 0, c[2], alpha * 0.8);
            line(px + ox - sz + drift, py + oy - drift, px + ox + sz + drift, py + oy - drift);
            line(px + ox + drift, py + oy - sz - drift, px + ox + drift, py + oy + sz - drift);
        }
    }
}

// ── DEFAULT: polígono corrupto con vértices perturbados ──────────────────────
// Para sonidos que no son tr909: forma que parece "señal corrompida"
function dibujarDefault(anim, p, c) {
    const maxSz = map(anim.size, 0, 1, 40, 180);
    const sz    = lerp(maxSz, 0, p);

    randomSeed(anim.seed);

    translate(anim.x, anim.y);

    // Cuerpo principal: polígono con vértices ruidosos
    const n = 6;
    stroke(c[0], c[1], c[2], lerp(200, 0, p));
    strokeWeight(lerp(2, 0.5, p));
    noFill();

    beginShape();
    for (let i = 0; i < n; i++) {
        const baseAngle  = (TWO_PI / n) * i;
        const noiseAmt   = sz * random(0.2, 0.6) * (1 - p);
        const r          = sz + random(-noiseAmt, noiseAmt);
        vertex(cos(baseAngle) * r, sin(baseAngle) * r);
    }
    endShape(CLOSE);

    // Líneas internas de "corrupción" que irradian desde el centro
    const nLines = Math.floor(random(2, 5));
    strokeWeight(1);
    for (let i = 0; i < nLines; i++) {
        randomSeed(anim.seed + i * 11);
        const a  = random(TWO_PI);
        const r1 = random(sz * 0.1, sz * 0.5);
        const r2 = random(sz * 0.5, sz);
        const isWhite = random() > 0.7;
        stroke(
            isWhite ? 255 : c[0],
            isWhite ? 255 : c[1],
            isWhite ? 255 : c[2],
            lerp(180, 0, p)
        );
        line(cos(a) * r1, sin(a) * r1, cos(a + random(-0.5, 0.5)) * r2, sin(a + random(-0.5, 0.5)) * r2);
    }

    // Punto de "error" parpadeante en el centro
    if (random() > 0.5) {
        noStroke();
        fill(255, 255, 255, lerp(200, 0, p));
        const blipSz = lerp(8, 0, p);
        rect(-blipSz / 2, -blipSz / 2, blipSz, blipSz);
    }
}

// ── UTILIDADES — sin cambios ───────────────────────────────────────────────────

function getColorForSound(s) {
    const colors = {
        'tr909bd': [255, 0, 80],
        'tr909sd': [0, 200, 255],
        'tr909hh': [255, 255, 0],
        'tr909oh': [255, 150, 0]
    };
    if (colors[s]) return colors[s];
    let charCode = s.charCodeAt(0) || 0;
    return [
        (charCode * 123) % 255,
        (charCode * 456) % 255,
        (charCode * 789) % 255
    ];
}

function windowResized() {
    resizeCanvas(windowWidth, windowHeight);
}

function keyPressed() {
    if (key === 'a' || key === 'A') {
        painter.postEvent({
            type: EVENTS.DATA,
            payload: { type: "microbit", x: 0, y: 0, btnA: true, btnB: false }
        });
    }
    if (key === 'b' || key === 'B') {
        painter.postEvent({
            type: EVENTS.DATA,
            payload: { type: "microbit", x: 0, y: 0, btnA: false, btnB: true }
        });
    }
}

function keyReleased() {
    painter.postEvent({
        type: EVENTS.DATA,
        payload: { type: "microbit", x: 0, y: 0, btnA: false, btnB: false }
    });
}
```

Bridge server: 
```
const { WebSocketServer } = require("ws");
const { SerialPort } = require("serialport");
const SimAdapter = require("./adapters/SimAdapter");
const MicrobitAsciiAdapter = require("./adapters/MicrobitASCIIAdapter");
const MicrobitAscii2Adapter = require("./adapters/MicrobitASCII2Adapter");
const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");
const StrudelAdapter = require("./adapters/StrudelAdapter");
const STRUDEL_PORT = parseInt(getArg("strudelPort", "8080"), 10);
// Ya tienes esto arriba — está bien:
const OpenStageControlAdapter = require("./adapters/OpenStageControlAdapter");
const OSC_PORT = parseInt(getArg("oscPort", "8082"), 10);

const log = {
  info: (...args) => console.log(`[${new Date().toISOString()}] [INFO]`, ...args),
  warn: (...args) => console.warn(`[${new Date().toISOString()}] [WARN]`, ...args),
  error: (...args) => console.error(`[${new Date().toISOString()}] [ERROR]`, ...args)
};


function getArg(name, def = null) {
  const i = process.argv.indexOf(`--${name}`);
  if (i >= 0 && i + 1 < process.argv.length) return process.argv[i + 1];
  return def;
}

function hasFlag(name) {
  return process.argv.includes(`--${name}`);
}

function nowMs() { return Date.now(); }

function safeJsonParse(s) {
  try {
    return JSON.parse(s);

  } catch (e) {
    log.warn("Failed to parse JSON: ", s, e);
    return null;
  }
}

function broadcast(wss, obj) {
  const text = JSON.stringify(obj);
  for (const client of wss.clients) {
    if (client.readyState === 1) client.send(text);
  }
}

function status(wss, state, detail = "") {
  broadcast(wss, { type: "status", state, detail, t: nowMs() });
}

const DEVICE = (getArg("device", "sim") || "sim").toLowerCase();
const WS_PORT = parseInt(getArg("wsPort", "8081"), 10);
const SERIAL_PATH = getArg("serialPort", null);
const BAUD = parseInt(getArg("baud", "115200"), 10);
const SIM_HZ = parseInt(getArg("hz", "30"), 10);
const VERBOSE = hasFlag("verbose");

async function findMicrobitPort() {
  const ports = await SerialPort.list();
  const microbit = ports.find(p =>
    p.vendorId && parseInt(p.vendorId, 16) === 0x0D28
  );
  return microbit?.path ?? null;
}

async function createAdapter() {
  if (DEVICE === "microbit") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit found at ${path}`);
    return new MicrobitAsciiAdapter({ path, baud: BAUD, verbose: VERBOSE });
  }
  if (DEVICE === "microbit2") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit 2 found at ${path}`);
    return new MicrobitAscii2Adapter({ path, baud: BAUD, verbose: VERBOSE });
  }
  if (DEVICE === "microbit-bin") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    return new MicrobitBinaryAdapter({ path, baud: BAUD });
  }
  return new SimAdapter({ hz: SIM_HZ }); // fallback obligatorio
}

async function main() {
  const wss = new WebSocketServer({ port: WS_PORT });
  log.info(`WS listening on ws://127.0.0.1:${WS_PORT} device=${DEVICE}`);

  const adapter = await createAdapter();

  adapter.onConnected    = (detail) => { log.info(`[ADAPTER] Device Connected: ${detail}`); status(wss, "connected", detail); };
  adapter.onDisconnected = (detail) => { log.warn(`[ADAPTER] Device Disconnected: ${detail}`); status(wss, "disconnected", detail); };
  adapter.onError        = (detail) => { log.error(`[ADAPTER] Device Error: ${detail}`); status(wss, "error", detail); };

  adapter.onData = (d) => {
    if (d.type === "strudel") { broadcast(wss, d); return; }
    if (d.type === "osc")     { broadcast(wss, d); return; }
    broadcast(wss, { type: "microbit", x: d.x, y: d.y, btnA: !!d.btnA, btnB: !!d.btnB, t: nowMs() });
  };

  // OSC siempre corre en paralelo — instancia única
  const oscAdapter = new OpenStageControlAdapter({ port: OSC_PORT, verbose: VERBOSE });
  oscAdapter.onData         = (d) => broadcast(wss, d);
  oscAdapter.onConnected    = (d) => log.info(`[OSC] ${d}`);
  oscAdapter.onDisconnected = (d) => log.warn(`[OSC] ${d}`);
  oscAdapter.onError        = (d) => log.error(`[OSC] ${d}`);
  await oscAdapter.connect();

    // Strudel siempre corre en paralelo — igual que OSC
  const strudelAdapter = new StrudelAdapter({ port: STRUDEL_PORT, verbose: VERBOSE });
  strudelAdapter.onData         = (d) => broadcast(wss, d);
  strudelAdapter.onConnected    = (d) => log.info(`[STRUDEL] ${d}`);
  strudelAdapter.onDisconnected = (d) => log.warn(`[STRUDEL] ${d}`);
  strudelAdapter.onError        = (d) => log.error(`[STRUDEL] ${d}`);
  await strudelAdapter.connect();

  status(wss, "ready", `bridge up (${DEVICE})`);

  wss.on("connection", (ws, req) => {
    log.info(`[NETWORK] Remote Client connected from ${req.socket.remoteAddress}. Total clients: ${wss.clients.size}`);

    const state  = adapter.connected ? "connected" : "ready";
    const detail = adapter.connected ? adapter.getConnectionDetail() : `bridge (${DEVICE})`;
    ws.send(JSON.stringify({ type: "status", state, detail, t: nowMs() }));

    ws.on("message", async (raw) => {
      const msg = safeJsonParse(raw.toString("utf8"));
      if (!msg) return;

      if (msg.cmd === "connect") {
        log.info(`[NETWORK] Client requested adapter connect`);
        if (adapter.connected) {
          ws.send(JSON.stringify({ type: "status", state: "connected", detail: adapter.getConnectionDetail(), t: nowMs() }));
          return;
        }
        try { await adapter.connect(); }
        catch (e) { status(wss, "error", `connect failed: ${e.message || e}`); }
        return;
      }

      if (msg.cmd === "disconnect") {
        log.info(`[NETWORK] Client requested adapter disconnect`);
        if (wss.clients.size > 1) {
          ws.send(JSON.stringify({ type: "status", state: "disconnected", detail: "logical disconnect only", t: nowMs() }));
          return;
        }
        try { await adapter.disconnect(); }
        catch (e) { status(wss, "error", `disconnect failed: ${e.message || e}`); }
        return;
      }

      if (msg.cmd === "setSimHz" && adapter instanceof SimAdapter) {
        await adapter.handleCommand(msg);
        status(wss, "connected", `sim hz=${adapter.hz}`);
        return;
      }

      if (msg.cmd === "setLed") {
        try { await adapter.handleCommand?.(msg); }
        catch (e) { status(wss, "error", `command failed: ${e.message || e}`); }
        return;
      }
    });

    ws.on("close", () => {
      log.info(`[NETWORK] Remote Client disconnected. Total clients left: ${wss.clients.size}`);
      if (wss.clients.size === 0) adapter.disconnect();
    });
  });

  if (DEVICE === "sim") await adapter.connect();
}

main().catch((e) => {
  log.error("Fatal:", e);
  process.exit(1);
});
```

Bridge client:

```
if (msg.type === "microbit") {
    // sin cambios
    this._onData?.(msg);
    return;
    }

    if (msg.type === "strudel") {
    this._onData?.(msg);
    return;
    }
    
    if (msg.type === "osc") {
    this._onData?.(msg);
    return;
    }
```
## Bitácora de reflexión
