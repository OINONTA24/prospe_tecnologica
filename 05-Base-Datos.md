---
layout: default
title: Servicios de almacenamiento en la nube
nav_order: 6
---


# ESP32 con botón, servidor Flask y Firebase

## Objetivo de la práctica

El objetivo de esta práctica fue implementar una comunicación entre una **ESP32** y un **servidor desarrollado en Flask**, para que la ESP32 pudiera enviar información mediante una petición **HTTP POST** cuando se presionara un botón físico. Posteriormente, el servidor Flask recibiría esa información en formato **JSON** y la enviaría a una base de datos en la nube usando **Firebase Firestore**.

La idea general fue simular un sistema de monitoreo remoto, donde un dispositivo físico pudiera reportar un evento hacia un servidor local y después almacenar ese evento en una base de datos en la nube para su consulta posterior.

---

## Descripción general del sistema

El sistema se compone de tres partes principales:

1. Una **ESP32** conectada a un botón físico.
2. Un **servidor Flask** ejecutándose en una computadora.
3. Una **base de datos en Firebase Firestore**, utilizada para guardar los eventos recibidos.

La ESP32 se conecta a una red WiFi y permanece revisando constantemente el estado de un botón conectado al pin **GPIO 4**. Cuando el botón es presionado, la ESP32 genera un mensaje en formato JSON y lo envía al servidor Flask mediante una petición HTTP POST.

El servidor Flask recibe esa petición en el endpoint `/evento`, interpreta el contenido del JSON y guarda la información recibida en una colección de Firebase Firestore llamada `eventos`.

---

## Flujo general de funcionamiento

El funcionamiento general del sistema es el siguiente:

```text
Botón físico
    ↓
ESP32 detecta que el botón fue presionado
    ↓
ESP32 genera un JSON con el evento
    ↓
ESP32 realiza una petición HTTP POST al servidor Flask
    ↓
Flask recibe el JSON en el endpoint /evento
    ↓
Flask guarda el evento en Firebase Firestore
    ↓
El evento queda disponible en la nube para consulta posterior
````

---

## Idea inicial del proyecto

La idea inicial era que la ESP32 tuviera conectado un botón y que, dependiendo de si el botón estaba presionado o no, enviara al servidor un estado, por ejemplo:

```json
{
  "boton": 1
}
```

o:

```json
{
  "boton": 0
}
```

Esto permitiría representar directamente el estado del botón como un valor binario, donde:

| Valor | Significado         |
| ----- | ------------------- |
| `1`   | Botón presionado    |
| `0`   | Botón no presionado |

Sin embargo, en el código implementado se decidió enviar un payload más descriptivo, indicando el nombre del dispositivo y el tipo de evento ocurrido. Por eso, cuando se presiona el botón, la ESP32 envía un JSON como el siguiente:

```json
{
  "dispositivo": "ESP32_WROOM",
  "evento": "boton_presionado"
}
```

Este formato permite que el servidor identifique qué dispositivo envió la información y qué evento ocurrió.

---

## Aclaración sobre el endpoint usado

Inicialmente se consideró usar una ruta como `/estado`, ya que la intención era enviar el estado del botón. Sin embargo, en el código final se utilizó el endpoint:

```text
/evento
```

Esto tiene sentido porque el servidor no solamente recibe un estado numérico, sino que registra un evento específico: la presión del botón.

Por lo tanto, el flujo final quedó de la siguiente manera:

```text
ESP32 → POST /evento → Flask → Firebase
```

---

# Código del servidor Flask

El servidor fue desarrollado en Python utilizando Flask. Además, se utilizó el SDK de Firebase Admin para conectar el servidor con Firebase Firestore.

El archivo principal del servidor se llama:

```text
app.py
```

---

## Código `app.py`

```python
# app.py
# API Flask + Firebase Firestore
# - GET  /            -> prueba rápida
# - GET  /inventario  -> lee colección 'sensores'
# - POST /inventario  -> agrega sensor a colección 'sensores'
# - POST /evento      -> guarda evento (ej. botón ESP32) en colección 'eventos'
# - GET  /eventos     -> lista últimos eventos (opcional, útil para verificar)

import firebase_admin
from firebase_admin import credentials, firestore
from flask import Flask, jsonify, request

# -----------------------
# 1) Firebase init
# -----------------------
# Coloca tu archivo JSON en la misma carpeta y nómbralo EXACTAMENTE así
SERVICE_ACCOUNT_FILE = "databasep1-fd3bc-firebase-adminsdk-fbsvc-11e1a7b47a.json"

cred = credentials.Certificate(SERVICE_ACCOUNT_FILE)
firebase_admin.initialize_app(cred)
db = firestore.client()

# -----------------------
# 2) Flask init
# -----------------------
app = Flask(__name__)


@app.route("/", methods=["GET"])
def home():
    return "API de Inventario/Producción funcionando"


# =========================================================
# INVENTARIO (colección: 'sensores')
# =========================================================

@app.route("/inventario", methods=["GET"])
def get_inventario():
    """Lee todos los documentos de la colección 'sensores'."""
    try:
        sensores_ref = db.collection("sensores")
        docs = sensores_ref.stream()

        inventario = []
        for doc in docs:
            inventario.append(doc.to_dict())

        return jsonify(inventario), 200

    except Exception as e:
        return jsonify({"error": str(e)}), 500


@app.route("/inventario", methods=["POST"])
def add_inventario():
    """Agrega un documento a la colección 'sensores'."""
    try:
        data = request.json or {}

        nuevo_sensor = {
            "nombre": data.get("nombre"),
            "tipo": data.get("tipo"),
            "cantidad": data.get("cantidad")
        }

        # Validación mínima
        if not nuevo_sensor["nombre"] or not nuevo_sensor["tipo"]:
            return jsonify({"error": "Faltan campos: 'nombre' y/o 'tipo'"}), 400

        db.collection("sensores").add(nuevo_sensor)

        return jsonify({"mensaje": "Sensor agregado correctamente"}), 201

    except Exception as e:
        return jsonify({"error": str(e)}), 500


# =========================================================
# EVENTOS (colección: 'eventos') -> para botón ESP32
# =========================================================

@app.route("/evento", methods=["POST"])
def registrar_evento():
    """
    Guarda un evento en Firestore.
    Ejemplo body JSON:
    {
      "dispositivo": "ESP32_WROOM",
      "evento": "boton_presionado"
    }
    """
    try:
        data = request.json or {}

        evento = {
            "dispositivo": data.get("dispositivo", "desconocido"),
            "evento": data.get("evento", "sin_evento"),
            "timestamp": firestore.SERVER_TIMESTAMP
        }

        db.collection("eventos").add(evento)

        return jsonify({"mensaje": "Evento guardado"}), 200

    except Exception as e:
        return jsonify({"error": str(e)}), 500


@app.route("/eventos", methods=["GET"])
def listar_eventos():
    """Lista los últimos eventos (útil para verificar que sí se guardan)."""
    try:
        # Ordena por timestamp descendente (últimos primero)
        docs = (
            db.collection("eventos")
            .order_by("timestamp", direction=firestore.Query.DESCENDING)
            .limit(50)
            .stream()
        )

        eventos = []
        for d in docs:
            item = d.to_dict()
            eventos.append(item)

        return jsonify(eventos), 200

    except Exception as e:
        return jsonify({"error": str(e)}), 500


# -----------------------
# Run server
# -----------------------
if __name__ == "__main__":
    # IMPORTANTE:
    # host="0.0.0.0" permite que el ESP32 (en la misma red) pueda acceder a tu PC
    app.run(host="0.0.0.0", debug=True, port=5000)
```

---

# Explicación del código Flask

## Inicialización de Firebase

La primera parte del código se encarga de conectar Flask con Firebase:

```python
SERVICE_ACCOUNT_FILE = "databasep1-fd3bc-firebase-adminsdk-fbsvc-11e1a7b47a.json"

cred = credentials.Certificate(SERVICE_ACCOUNT_FILE)
firebase_admin.initialize_app(cred)
db = firestore.client()
```

Aquí se carga el archivo JSON de credenciales de Firebase. Este archivo permite que el servidor Flask tenga autorización para conectarse con el proyecto de Firebase y escribir datos en Firestore.

La variable `db` representa la conexión con Firestore. A través de esta variable se pueden crear colecciones, agregar documentos y consultar información almacenada.

---

## Endpoint principal `/`

El endpoint principal se usa para verificar rápidamente que el servidor está funcionando:

```python
@app.route("/", methods=["GET"])
def home():
    return "API de Inventario/Producción funcionando"
```

Cuando se abre en el navegador la dirección del servidor, por ejemplo:

```text
http://172.22.26.115:5000/
```

debe mostrarse el mensaje:

```text
API de Inventario/Producción funcionando
```

Esto confirma que Flask está corriendo correctamente.

---

## Endpoint `/inventario`

El servidor también incluye endpoints para manejar una colección llamada `sensores`.

El método `GET /inventario` permite leer los sensores guardados en Firebase:

```python
@app.route("/inventario", methods=["GET"])
def get_inventario():
```

Este endpoint consulta todos los documentos de la colección `sensores` y los devuelve como una lista en formato JSON.

---

## Endpoint `POST /inventario`

El método `POST /inventario` permite agregar un sensor a la colección `sensores`:

```python
@app.route("/inventario", methods=["POST"])
def add_inventario():
```

Este endpoint espera recibir un JSON con campos como:

```json
{
  "nombre": "Sensor de temperatura",
  "tipo": "DHT11",
  "cantidad": 1
}
```

Después, el servidor guarda esa información en Firebase dentro de la colección `sensores`.

Aunque este endpoint no forma parte directa del botón de la ESP32, sirve como ejemplo adicional para comprobar que Flask puede recibir datos y almacenarlos en Firebase.

---

## Endpoint `POST /evento`

Este es el endpoint más importante para la comunicación con la ESP32:

```python
@app.route("/evento", methods=["POST"])
def registrar_evento():
```

Cuando la ESP32 detecta que el botón fue presionado, manda una petición HTTP POST a esta ruta.

El servidor recibe un JSON con esta estructura:

```json
{
  "dispositivo": "ESP32_WROOM",
  "evento": "boton_presionado"
}
```

Después, Flask construye un nuevo documento con la información recibida:

```python
evento = {
    "dispositivo": data.get("dispositivo", "desconocido"),
    "evento": data.get("evento", "sin_evento"),
    "timestamp": firestore.SERVER_TIMESTAMP
}
```

Finalmente, guarda ese documento en Firebase:

```python
db.collection("eventos").add(evento)
```

Esto crea automáticamente un nuevo documento dentro de la colección `eventos`.

---

## Endpoint `GET /eventos`

Este endpoint permite consultar los últimos eventos guardados:

```python
@app.route("/eventos", methods=["GET"])
def listar_eventos():
```

La consulta se ordena por `timestamp` de forma descendente, es decir, muestra primero los eventos más recientes:

```python
docs = (
    db.collection("eventos")
    .order_by("timestamp", direction=firestore.Query.DESCENDING)
    .limit(50)
    .stream()
)
```

Este endpoint es útil para verificar desde el navegador si los eventos enviados por la ESP32 sí se están guardando correctamente en Firebase.

Por ejemplo, se puede acceder a:

```text
http://172.22.26.115:5000/eventos
```

y el servidor devolverá una lista de los últimos eventos registrados.

---

# Código de la ESP32

El código de la ESP32 se encarga de conectarse a la red WiFi, leer el estado del botón y enviar una petición HTTP POST al servidor Flask cuando el botón sea presionado.

---

## Código ESP32

```cpp
#include <WiFi.h>
#include <HTTPClient.h>

const char* ssid = "Primavera26";
const char* password = "TU_CONTRASEÑA_WIFI";

const char* serverName = "http://172.22.26.115:5000/evento"; 
// Cambia por la IP de tu computadora

const int boton = 4;

void setup() {

  Serial.begin(115200);

  pinMode(boton, INPUT_PULLUP);

  WiFi.begin(ssid, password);

  Serial.print("Conectando WiFi");

  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }

  Serial.println("Conectado");
}

void loop() {

  if (digitalRead(boton) == LOW) {

    Serial.println("Boton presionado");

    if (WiFi.status() == WL_CONNECTED) {

      HTTPClient http;

      http.begin(serverName);

      http.addHeader("Content-Type", "application/json");

      String json = "{\"dispositivo\":\"ESP32_WROOM\",\"evento\":\"boton_presionado\"}";

      int httpResponseCode = http.POST(json);

      Serial.print("Respuesta: ");
      Serial.println(httpResponseCode);

      http.end();
    }

    delay(2000); // evitar múltiples envíos
  }
}
```

---

# Explicación del código ESP32

## Librerías utilizadas

El código comienza incluyendo dos librerías:

```cpp
#include <WiFi.h>
#include <HTTPClient.h>
```

La librería `WiFi.h` permite que la ESP32 se conecte a una red inalámbrica. La librería `HTTPClient.h` permite realizar peticiones HTTP, como `GET` o `POST`, hacia un servidor.

En esta práctica se utiliza una petición `POST`, porque la ESP32 necesita enviar información al servidor.

---

## Configuración de la red WiFi

En esta parte se declaran el nombre y la contraseña de la red WiFi:

```cpp
const char* ssid = "Primavera26";
const char* password = "TU_CONTRASEÑA_WIFI";
```

La ESP32 usa estos datos para conectarse a la misma red donde se encuentra la computadora que ejecuta el servidor Flask.

Es importante que la ESP32 y la computadora estén conectadas a la misma red, ya que de lo contrario la ESP32 no podrá comunicarse con Flask usando la IP local.

---

## Dirección del servidor Flask

La siguiente línea define la dirección a la que la ESP32 enviará la información:

```cpp
const char* serverName = "http://172.22.26.115:5000/evento";
```

Esta dirección se compone de varias partes:

| Parte           | Significado                            |
| --------------- | -------------------------------------- |
| `http://`       | Protocolo de comunicación              |
| `172.22.26.115` | IP de la computadora donde corre Flask |
| `5000`          | Puerto donde Flask está escuchando     |
| `/evento`       | Endpoint que recibe el evento          |

Si la IP de la computadora cambia, también se debe cambiar esta línea en el código de la ESP32.

---

## Configuración del botón

El botón se declaró en el pin GPIO 4:

```cpp
const int boton = 4;
```

Dentro de la función `setup()`, el pin se configura como entrada con resistencia interna pull-up:

```cpp
pinMode(boton, INPUT_PULLUP);
```

Esto significa que el pin normalmente se encuentra en estado `HIGH`, y cuando se presiona el botón, el pin pasa a estado `LOW`.

La conexión física del botón es:

```text
GPIO 4 ---- Botón ---- GND
```

Por esta razón, el código detecta que el botón fue presionado cuando la lectura es `LOW`:

```cpp
if (digitalRead(boton) == LOW)
```

---

## Conexión a WiFi

Dentro del `setup()`, la ESP32 intenta conectarse a la red WiFi:

```cpp
WiFi.begin(ssid, password);
```

Mientras no se haya conectado, imprime puntos en el monitor serial:

```cpp
while (WiFi.status() != WL_CONNECTED) {
  delay(500);
  Serial.print(".");
}
```

Cuando la conexión se realiza correctamente, se imprime:

```text
Conectado
```

Esto indica que la ESP32 ya está lista para enviar datos al servidor Flask.

---

## Detección del botón

En la función `loop()`, la ESP32 revisa constantemente el estado del botón:

```cpp
if (digitalRead(boton) == LOW)
```

Cuando el botón es presionado, se imprime en el monitor serial:

```text
Boton presionado
```

Después, el programa verifica que la ESP32 siga conectada al WiFi:

```cpp
if (WiFi.status() == WL_CONNECTED)
```

Si la ESP32 tiene conexión, entonces prepara la petición HTTP.

---

## Creación del JSON

El mensaje que se manda al servidor se construye como una cadena de texto en formato JSON:

```cpp
String json = "{\"dispositivo\":\"ESP32_WROOM\",\"evento\":\"boton_presionado\"}";
```

Este texto representa el siguiente objeto JSON:

```json
{
  "dispositivo": "ESP32_WROOM",
  "evento": "boton_presionado"
}
```

Este JSON indica que el dispositivo llamado `ESP32_WROOM` detectó el evento `boton_presionado`.

---

## Envío de la petición HTTP POST

La ESP32 crea una conexión HTTP con el servidor:

```cpp
HTTPClient http;
http.begin(serverName);
```

Después, agrega el encabezado que indica que el contenido enviado está en formato JSON:

```cpp
http.addHeader("Content-Type", "application/json");
```

Luego se envía la petición POST:

```cpp
int httpResponseCode = http.POST(json);
```

El servidor responde con un código HTTP. Si el código es `200`, significa que el evento se recibió y se guardó correctamente.

---

## Prevención de múltiples envíos

Al final del bloque se utiliza un retardo de 2 segundos:

```cpp
delay(2000);
```

Esto se hizo para evitar que una sola presión del botón genere muchos envíos consecutivos al servidor. Sin este retardo, la ESP32 podría mandar muchas peticiones mientras el botón permanezca presionado.

---

# Funcionamiento esperado

Cuando todo el sistema está funcionando correctamente, el proceso esperado es el siguiente:

1. Se ejecuta el servidor Flask en la computadora.
2. La ESP32 se conecta al WiFi.
3. El usuario presiona el botón conectado al GPIO 4.
4. La ESP32 imprime en el monitor serial que el botón fue presionado.
5. La ESP32 envía un JSON al endpoint `/evento`.
6. Flask recibe el JSON.
7. Flask guarda el evento en Firebase Firestore.
8. Se puede consultar el evento desde Firebase o desde el endpoint `/eventos`.

---

# Evidencia de almacenamiento en Firebase

Después de ejecutar el servidor Flask y presionar el botón físico conectado a la ESP32, se verificó que los datos enviados por el microcontrolador fueran almacenados correctamente en Firebase Firestore.

En la base de datos se creó una colección llamada:

```text
eventos
```

Dentro de esta colección, Firebase generó automáticamente documentos con identificadores únicos. Cada documento representa un evento enviado por la ESP32 cada vez que el botón fue presionado.

En uno de los documentos registrados se observan los siguientes campos:

```json
{
  "dispositivo": "ESP32_WROOM",
  "evento": "boton_presionado",
  "timestamp": "fecha y hora generada por Firebase"
}
```

El campo `dispositivo` indica que el evento fue enviado por la ESP32. El campo `evento` muestra que la acción detectada fue la presión del botón. Finalmente, el campo `timestamp` registra la fecha y hora en la que el evento fue almacenado en la base de datos.

Esta evidencia confirma que el flujo completo funcionó correctamente:

```text
ESP32
↓
Botón presionado
↓
HTTP POST
↓
Servidor Flask
↓
Firebase Firestore
↓
Colección eventos
```

Por lo tanto, se logró conectar un dispositivo físico con una base de datos en la nube, permitiendo registrar eventos de hardware en tiempo real.

---

## Imagen de evidencia

Puedes agregar tu captura de Firebase de la siguiente manera:

```markdown
![Eventos registrados en Firebase Firestore](../assets/images/firebase-eventos.png)
```

También puedes agregar un pie de figura debajo:

```markdown
**Figura 1. Eventos del botón de la ESP32 almacenados en Firebase Firestore.**

En la figura se observa la colección `eventos` dentro de Firebase Firestore. Cada documento corresponde a una activación del botón físico conectado a la ESP32. El documento seleccionado contiene los campos `dispositivo`, `evento` y `timestamp`, lo cual confirma que el servidor Flask recibió correctamente el JSON enviado por la ESP32 y posteriormente lo almacenó en la base de datos en la nube.
```

---

# Ejemplo de dato guardado en Firebase

Cuando se presiona el botón, Firebase guarda un documento similar a este:

```json
{
  "dispositivo": "ESP32_WROOM",
  "evento": "boton_presionado",
  "timestamp": "fecha y hora generada por Firebase"
}
```

La colección donde se guarda la información se llama:

```text
eventos
```

Cada vez que se presiona el botón, se agrega un nuevo documento a esta colección.

---

# Prueba del sistema

Para probar el sistema, primero se debe ejecutar Flask:

```bash
python app.py
```

Después, en el navegador se puede entrar a:

```text
http://172.22.26.115:5000/
```

Si el servidor está funcionando, debe aparecer:

```text
API de Inventario/Producción funcionando
```

Luego se carga el programa en la ESP32 y se abre el monitor serial. Cuando la ESP32 se conecte al WiFi, debe aparecer:

```text
Conectado
```

Al presionar el botón, debe aparecer:

```text
Boton presionado
Respuesta: 200
```

El código `200` indica que Flask recibió correctamente la petición.

Finalmente, se puede revisar el endpoint:

```text
http://172.22.26.115:5000/eventos
```

Ahí deben aparecer los últimos eventos guardados.

---

# Problemas comunes

## La ESP32 no se conecta al WiFi

Si la ESP32 no se conecta, se debe revisar que el nombre de la red y la contraseña sean correctos.

También es importante verificar que la red sea compatible con la ESP32. Normalmente, la ESP32 trabaja con redes WiFi de 2.4 GHz.

---

## La ESP32 muestra respuesta `-1`

Si en el monitor serial aparece:

```text
Respuesta: -1
```

significa que la ESP32 no pudo conectarse al servidor Flask.

Las causas más comunes son:

* La IP del servidor es incorrecta.
* Flask no está corriendo.
* La computadora y la ESP32 no están en la misma red.
* El firewall de la computadora está bloqueando el puerto `5000`.
* El servidor Flask no está usando `host="0.0.0.0"`.

---

## La ESP32 muestra respuesta `404`

Si aparece un código `404`, significa que la ruta no existe.

En este caso, se debe revisar que la ESP32 esté enviando la petición a:

```text
/evento
```

y no a otra ruta como:

```text
/estado
```

---

## La ESP32 muestra respuesta `500`

Si aparece un código `500`, significa que hubo un error interno en Flask.

Las causas más comunes son:

* El archivo JSON de Firebase no está en la misma carpeta que `app.py`.
* El nombre del archivo JSON no coincide exactamente.
* Firebase no fue inicializado correctamente.
* Hay un problema con los permisos de Firestore.

---

# Conclusión

En esta práctica se logró comunicar una ESP32 con un servidor Flask mediante una petición HTTP POST. La ESP32 detecta la presión de un botón físico conectado al GPIO 4 y, al detectar el evento, envía un mensaje JSON al servidor.

El servidor Flask recibe ese JSON en el endpoint `/evento`, interpreta los datos recibidos y los guarda en Firebase Firestore dentro de la colección `eventos`. De esta forma, se consiguió conectar un dispositivo físico con una base de datos en la nube.

Aunque inicialmente la idea era enviar simplemente el estado del botón como `1` o `0`, el código final envía un evento más descriptivo, indicando el dispositivo y el tipo de acción realizada. Esto hace que el sistema sea más claro, escalable y fácil de consultar posteriormente.

Este tipo de arquitectura puede utilizarse como base para proyectos de IoT, monitoreo remoto, control de dispositivos, registro de eventos, sistemas de producción o inventario conectado a la nube.

```