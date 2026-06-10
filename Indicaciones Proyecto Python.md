# Control de NeoPixel

Este proyecto permite controlar un módulo o tira **NeoPixel / WS2812 / WS2812B** desde una página web local creada por el **Raspberry Pi Pico W**.

No usa Adafruit IO ni paneles externos. El Pico W se conecta al WiFi, crea un servidor web y desde el navegador se pueden cambiar los colores.

---

## 1. Componentes necesarios

| Cantidad | Componente |
|---:|---|
| 1 | Raspberry Pi Pico W |
| 1 | Módulo o tira NeoPixel / WS2812 / WS2812B |
| 1 | Resistencia de 330 Ω para la línea de datos |
| 1 | Protoboard |
| 3 | Cables Dupont |

---

## 2. Conexión del NeoPixel

Este proyecto usa el pin **GP28**, igual que el código original:

```python
np = NeoPixel(Pin(28), 8)
```

| NeoPixel | Raspberry Pi Pico W | Nota |
|---|---|---|
| VCC / 5V / + | VBUS | Alimentación desde USB |
| GND / - | GND | Tierra común |
| DIN / IN / DI / DATA | GP28 | Señal de datos |

Conexión recomendada:

```text
NeoPixel VCC / 5V / +  -> VBUS del Pico W
NeoPixel GND / -       -> GND del Pico W
NeoPixel DIN / IN / DI -> resistencia 330 Ω -> GP28
```

![alt text](image.png)

---

## 3. Importante: DIN no es DOUT

El cable de datos del Pico W debe ir al pin:

```text
DIN / DI / IN
```

No debe ir al pin:

```text
DOUT / DO / OUT
```

Si se conecta al lado incorrecto, la web puede funcionar, pero el NeoPixel no prenderá.

---

## 4. Cantidad de NeoPixels

En el código se usa:

```python
NUM_PIXELS = 8
```

Si tienes solo 1 NeoPixel, cambia a:

```python
NUM_PIXELS = 1
```

Si tienes una tira de 12 LEDs, cambia a:

```python
NUM_PIXELS = 12
```

---

## 5. Código completo

Guarda este código en Thonny como `main.py`.

```python
from machine import Pin
from neopixel import NeoPixel
import network
import socket
import time
import machine

ssid = 'NOMBRE-DEL-WIFI'
password = 'CONTRASEÑA-DEL-WIFI'

wlan = network.WLAN(network.STA_IF)
wlan.active(True)
wlan.connect(ssid, password)

NUM_PIXELS = 8
NEOPIXEL_PIN = 28

np = NeoPixel(Pin(NEOPIXEL_PIN), NUM_PIXELS)

rojo = 0
verde = 0
azul = 0


def aplicar_color(r, g, b):
    global rojo, verde, azul

    rojo = max(0, min(255, int(r)))
    verde = max(0, min(255, int(g)))
    azul = max(0, min(255, int(b)))

    for i in range(NUM_PIXELS):
        np[i] = (rojo, verde, azul)

    np.write()


def apagar():
    aplicar_color(0, 0, 0)


def probar_neopixel():
    aplicar_color(255, 0, 0)
    time.sleep(0.4)
    aplicar_color(0, 255, 0)
    time.sleep(0.4)
    aplicar_color(0, 0, 255)
    time.sleep(0.4)
    apagar()


print("Conectando a WiFi...")

espera = 15
while espera > 0:
    if wlan.status() < 0 or wlan.status() >= 3:
        break

    espera -= 1
    print("Esperando conexion...")
    time.sleep(1)

if wlan.status() != 3:
    raise RuntimeError("Error de conexion WiFi")
else:
    print("Conectado")
    ip = wlan.ifconfig()[0]
    print("IP:", ip)


def obtener_parametro(request, nombre):
    try:
        request = request.decode("utf-8")

        if "?" not in request:
            return None

        inicio = request.find("?") + 1
        fin = request.find(" ", inicio)

        query = request[inicio:fin]
        partes = query.split("&")

        for parte in partes:
            if "=" in parte:
                clave, valor = parte.split("=", 1)

                if clave == nombre:
                    valor = valor.replace("+", " ")
                    valor = valor.replace("%23", "#")
                    return valor

    except:
        return None

    return None


def hex_a_rgb(hex_color):
    hex_color = hex_color.replace("#", "")

    if len(hex_color) != 6:
        return 0, 0, 0

    r = int(hex_color[0:2], 16)
    g = int(hex_color[2:4], 16)
    b = int(hex_color[4:6], 16)

    return r, g, b


def pagina_web():
    html = """<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Control NeoPixel Local</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <style>
        body {
            font-family: Arial, sans-serif;
            background: #111827;
            color: white;
            text-align: center;
            padding: 25px;
        }

        .card {
            background: #1f2937;
            max-width: 450px;
            margin: auto;
            padding: 25px;
            border-radius: 15px;
        }

        h1 {
            color: #38bdf8;
        }

        .preview {
            width: 160px;
            height: 160px;
            margin: 20px auto;
            border-radius: 50%;
            border: 4px solid white;
            background: rgb(""" + str(rojo) + "," + str(verde) + "," + str(azul) + """);
        }

        .dato {
            background: #374151;
            padding: 10px;
            border-radius: 8px;
            margin: 10px;
        }

        button {
            width: 90%;
            padding: 14px;
            margin: 7px;
            border: none;
            border-radius: 10px;
            font-size: 18px;
            color: white;
            cursor: pointer;
        }

        input[type="range"] {
            width: 90%;
        }

        input[type="color"] {
            width: 90%;
            height: 55px;
            margin: 10px;
            border: none;
            border-radius: 8px;
        }

        .red { background: #ef4444; }
        .green { background: #22c55e; }
        .blue { background: #3b82f6; }
        .white { background: #e5e7eb; color: black; }
        .off { background: #374151; }
        .send { background: #f59e0b; }
    </style>
</head>

<body>
    <div class="card">
        <h1>Control NeoPixel Local</h1>

        <div class="preview"></div>

        <div class="dato">
            RGB actual: """ + str(rojo) + """, """ + str(verde) + """, """ + str(azul) + """
        </div>

        <form action="/set" method="get">
            <input type="hidden" name="r" value="255">
            <input type="hidden" name="g" value="0">
            <input type="hidden" name="b" value="0">
            <button class="red" type="submit">Rojo</button>
        </form>

        <form action="/set" method="get">
            <input type="hidden" name="r" value="0">
            <input type="hidden" name="g" value="255">
            <input type="hidden" name="b" value="0">
            <button class="green" type="submit">Verde</button>
        </form>

        <form action="/set" method="get">
            <input type="hidden" name="r" value="0">
            <input type="hidden" name="g" value="0">
            <input type="hidden" name="b" value="255">
            <button class="blue" type="submit">Azul</button>
        </form>

        <form action="/set" method="get">
            <input type="hidden" name="r" value="255">
            <input type="hidden" name="g" value="255">
            <input type="hidden" name="b" value="255">
            <button class="white" type="submit">Blanco</button>
        </form>

        <form action="/set" method="get">
            <input type="hidden" name="r" value="0">
            <input type="hidden" name="g" value="0">
            <input type="hidden" name="b" value="0">
            <button class="off" type="submit">Apagar</button>
        </form>

        <hr>

        <h2>Color personalizado</h2>

        <form action="/color" method="get">
            <input type="color" name="hex" value="#ff0000">
            <button class="send" type="submit">Aplicar color</button>
        </form>

        <hr>

        <h2>RGB manual</h2>

        <form action="/set" method="get">
            <p>Rojo</p>
            <input type="range" name="r" min="0" max="255" value="""" + str(rojo) + """">

            <p>Verde</p>
            <input type="range" name="g" min="0" max="255" value="""" + str(verde) + """">

            <p>Azul</p>
            <input type="range" name="b" min="0" max="255" value="""" + str(azul) + """">

            <br><br>
            <button class="send" type="submit">Aplicar RGB</button>
        </form>
    </div>
</body>
</html>
"""
    return html


def iniciar_servidor():
    addr = socket.getaddrinfo("0.0.0.0", 80)[0][-1]

    s = socket.socket()
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    s.bind(addr)
    s.listen(1)

    print("Servidor listo")
    print("Abre en navegador: http://" + ip)

    while True:
        conn, addr = s.accept()
        request = conn.recv(1024)

        print(request)

        if b"GET /set" in request:
            r = obtener_parametro(request, "r")
            g = obtener_parametro(request, "g")
            b = obtener_parametro(request, "b")

            if r is not None and g is not None and b is not None:
                aplicar_color(int(r), int(g), int(b))

            conn.sendall(
                b"HTTP/1.1 303 See Other\r\n"
                b"Location: /\r\n"
                b"Connection: close\r\n\r\n"
            )

        elif b"GET /color" in request:
            color_hex = obtener_parametro(request, "hex")

            if color_hex is not None:
                r, g, b = hex_a_rgb(color_hex)
                aplicar_color(r, g, b)

            conn.sendall(
                b"HTTP/1.1 303 See Other\r\n"
                b"Location: /\r\n"
                b"Connection: close\r\n\r\n"
            )

        elif b"GET /favicon.ico" in request:
            conn.sendall(
                b"HTTP/1.1 204 No Content\r\n"
                b"Connection: close\r\n\r\n"
            )

        else:
            html = pagina_web()

            conn.sendall(b"HTTP/1.1 200 OK\r\n")
            conn.sendall(b"Content-Type: text/html\r\n")
            conn.sendall(b"Connection: close\r\n\r\n")
            conn.sendall(html.encode("utf-8"))

        conn.close()


try:
    probar_neopixel()
    iniciar_servidor()

except KeyboardInterrupt:
    apagar()
    machine.reset()
```

---

## 6. Cómo usar

1. Cargar el código en Thonny.
2. Guardarlo como `main.py`.
3. Ejecutarlo.
4. Esperar la IP en consola.

Ejemplo:

```text
IP: 192.168.1.147
```

5. Abrir en el navegador:

```text
http://192.168.1.147
```

---

## 7. Problemas comunes

### No prende nada

Revisar:

```text
VCC -> VBUS
GND -> GND
DIN -> GP28
```

También verificar que sea **DIN**, no **DOUT**.

---

### La página abre pero no cambia el color

Si en consola aparece algo como:

```text
GET /set?r=255&g=0&b=0
```

entonces la página sí está funcionando. El problema estaría en el cableado o en el pin de datos.

---

### El azul no funciona

Verificar que el botón azul tenga esta línea:

```html
<input type="hidden" name="b" value="255">
```

En este README ya está corregido.

---

### Los colores salen cambiados

Algunos NeoPixel usan otro orden de color. Si rojo y verde salen cambiados, modifica:

```python
np[i] = (rojo, verde, azul)
```

por:

```python
np[i] = (verde, rojo, azul)
```

---

## 8. Notas finales

- Este proyecto funciona en red local.
- No requiere Adafruit IO.
- El celular o computadora debe estar en la misma red WiFi que el Pico W.
- Para tiras grandes, usar fuente externa de 5V y unir GND de la fuente con GND del Pico W.
