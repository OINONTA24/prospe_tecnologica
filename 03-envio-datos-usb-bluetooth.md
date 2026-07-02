---
layout: default
title: Interfaz de Usuario
nav_order: 4
---

# Interfaz de Usuario (Python USB / Bluetooth)

En esta práctica se desarrolló un sistema ciberfísico que integra una interfaz gráfica implementada en PyQt6 con un microcontrolador ESP32 mediante comunicación Bluetooth Serial Port Profile (SPP). El objetivo principal fue permitir el control de un actuador físico (LED del ESP32) desde una aplicación de escritorio, manteniendo sincronización entre el estado físico del sistema y su representación digital.

Esta práctica demuestra la interacción entre software de alto nivel y hardware embebido mediante un canal de comunicación inalámbrico, destacando conceptos fundamentales como comunicación serial virtual, programación orientada a eventos y retroalimentación del sistema.

---

## 2. Instalación y Configuración del Entorno

El primer paso consistió en instalar la librería PyQt6 mediante:

```bash
pip install PyQt6
````

Esta librería provee los componentes necesarios para crear interfaces gráficas en Python utilizando el framework Qt. Sin esta instalación no sería posible desarrollar la GUI requerida para interactuar con el ESP32.

Posteriormente se verificó la instalación con:

```bash
pip show PyQt6
```

Esta verificación permite confirmar que los módulos fueron correctamente instalados y evita errores de importación durante la ejecución del programa.

---

## 3. Diseño de la Interfaz Gráfica

La interfaz gráfica fue desarrollada en Visual Studio Code utilizando PyQt6. Se optó por esta herramienta debido a su robustez, soporte multiplataforma y modelo basado en eventos, el cual es ideal para aplicaciones que interactúan con hardware en tiempo real.

La GUI se organizó en tres bloques funcionales:

1. Gestión de conexión Bluetooth.
2. Control del LED.
3. Registro de comunicación.

Esta separación modular permite claridad en la interacción y facilita el mantenimiento del código.

![Interfaz](assets/img/01-publicar/InterfazPyQt6.png)
*Figura 1:* Interfaz gráfica desarrollada en PyQt6 para el control del LED del ESP32 mediante comunicación Bluetooth.


La interfaz incluye un LED virtual implementado con `QPainter`, el cual refleja visualmente el estado del LED físico del ESP32. Esto mejora la interacción humano-máquina y permite validar visualmente la comunicación.

---

## 4. Comunicación Serial sobre Bluetooth

El ESP32 utiliza Bluetooth clásico con perfil SPP, el cual crea un puerto serial virtual en el sistema operativo. Esto permite que la comunicación se maneje como si fuera UART tradicional.

Para interactuar con este puerto se utilizó la librería PySerial:

```python
serial.Serial(port, 115200, timeout=0.1)
```

Aunque Bluetooth no depende del baudrate físico, PySerial requiere este parámetro para configurar la conexión. Se utiliza un timeout corto para evitar bloqueos en la lectura.

La selección del puerto COM correcto es fundamental, ya que el sistema operativo crea dos:

* COM entrante
* COM saliente

Solo el COM saliente permite iniciar comunicación desde la PC.

![Puertos](assets/img/01-publicar/PyQt_configuracion.png)
*Figura 2:* Identificación del puerto COM saliente asignado al ESP32 para la comunicación serial Bluetooth.

![Emparejamiento](assets/img/01-publicar/configu_COM.png)

*Figura 3:* Emparejamiento del dispositivo ESP32 mediante Bluetooth en el sistema operativo.

---

## 5. Programación del ESP32

El ESP32 fue programado mediante Arduino IDE utilizando la librería BluetoothSerial.

```cpp
#include "BluetoothSerial.h"
BluetoothSerial SerialBT;
```

Se inicializa el dispositivo con:

```cpp
SerialBT.begin("ESP32_LED");
```

Esto permite que la computadora detecte el dispositivo durante el emparejamiento.

El LED se configura como salida digital:

```cpp
pinMode(LED_PIN, OUTPUT);
```

El ESP32 interpreta comandos de texto enviados desde la GUI:

```cpp
if (cmd == "LED_ON") {
 digitalWrite(LED_PIN, HIGH);
 SerialBT.println("LED_ON_OK");
}
```

La confirmación enviada de regreso es crucial para garantizar sincronización.

## Descargar

[Descargar código Arduino (Esp32PyQt6.ino)]({{ site.baseurl }}/assets/files/Esp32PyQt6.ino)

## 6. Funcionamiento del Código PyQt6

La aplicación desarrollada en PyQt6 se basa en un modelo de programación orientado a eventos, el cual es característico de las interfaces gráficas modernas. En este paradigma, el flujo del programa no se ejecuta de manera secuencial continua, sino que responde a eventos generados por el usuario (como clics en botones) o por el sistema (como temporizadores o llegada de datos desde un puerto de comunicación).

Este enfoque resulta particularmente adecuado para aplicaciones que interactúan con hardware, ya que permite mantener la interfaz responsiva mientras se realizan tareas de comunicación asincrónica.

### 6.1 Lectura No Bloqueante del Puerto Serial

Para la recepción de datos provenientes del ESP32 se emplea un temporizador (`QTimer`) que ejecuta periódicamente una función encargada de revisar el buffer del puerto serial. Este mecanismo implementa una estrategia de lectura no bloqueante, lo que significa que la aplicación no se detiene esperando datos, sino que consulta de forma periódica si existen nuevos bytes disponibles.

Desde el punto de vista del diseño de software, esta decisión es fundamental porque el hilo principal de PyQt6 está dedicado al manejo de la interfaz gráfica. Si se utilizara una lectura bloqueante del puerto serial, el hilo principal quedaría suspendido hasta recibir datos, provocando congelamientos en la interfaz y una mala experiencia de usuario.

El uso de `QTimer` permite desacoplar la comunicación serial del renderizado de la interfaz, garantizando que la GUI continúe respondiendo a eventos del usuario mientras se monitorea la llegada de información desde el ESP32.

Además, este enfoque facilita la escalabilidad del sistema, ya que se pueden agregar más tareas periódicas sin afectar la estabilidad general del programa.

### 6.2 Confirmación de Estados del Sistema

El sistema implementa un mecanismo de confirmación basado en mensajes de retorno enviados por el ESP32 (`LED_ON_OK` y `LED_OFF_OK`). La interfaz gráfica no actualiza el estado del LED virtual inmediatamente después de enviar un comando, sino únicamente cuando recibe la confirmación correspondiente.

Esta decisión responde a principios fundamentales de sistemas distribuidos y sistemas ciberfísicos, donde la consistencia entre el estado físico y su representación digital debe garantizarse explícitamente.

Si la GUI cambiara el estado visual sin confirmación:

- Podría mostrarse un LED encendido cuando físicamente no lo está.
- Se perdería coherencia entre dominios.
- No se podrían detectar errores de comunicación.
- Se comprometería la confiabilidad del sistema.

El mecanismo de confirmación convierte la comunicación en un proceso robusto basado en solicitud–respuesta, mejorando la tolerancia a fallos.

### 6.3 Manejo de Eventos de Usuario

Los botones de la interfaz generan eventos que son capturados por PyQt6 y asociados a funciones específicas mediante señales y slots. Cuando el usuario interactúa con un botón, se emite una señal que activa la función encargada de enviar el comando correspondiente al ESP32.

Este modelo desacopla la lógica de interfaz de la lógica de comunicación, lo cual mejora la organización del código y facilita su mantenimiento. Además, permite extender la funcionalidad del sistema sin modificar la estructura base de la aplicación.

Desde la perspectiva de sistemas ciberfísicos, estos eventos representan la interacción humano–máquina, donde una acción del usuario desencadena una modificación en el sistema físico a través de la capa de comunicación.

### 6.4 Integración de Componentes

El funcionamiento completo del código PyQt6 se puede entender como la interacción coordinada de tres subsistemas:

1. Interfaz gráfica → Manejo de eventos del usuario.
2. Comunicación serial → Envío y recepción de datos.
3. Representación visual → Reflejo del estado físico.

La correcta integración de estos elementos permite que el sistema opere de manera estable y coherente, asegurando que la información fluya correctamente entre el usuario y el dispositivo embebido.

En conjunto, el diseño implementado garantiza:

- Responsividad de la interfaz.
- Consistencia de estados.
- Comunicación confiable.
- Modularidad del código.
- Escalabilidad del sistema.

Estos principios son esenciales en el desarrollo de aplicaciones que interactúan con hardware en tiempo real.

## Descargar
[Descargar código PyQt6(PyQt6.py)]({{ site.baseurl }}/assets/files/PyQt6.py)

## 7. Resultados

El sistema logró establecer comunicación bidireccional estable entre la interfaz gráfica y el ESP32. El control del LED físico se realiza en tiempo real y el LED virtual refleja correctamente el estado del hardware.

La implementación demostró:

* Comunicación inalámbrica funcional.
* Sincronización correcta.
* Interfaz responsiva.
* Confirmación confiable.
* Integración hardware-software.

### 7.1 Video
<video controls width="720">
  <source src="{{ '/assets/videos/PyQt6.mp4' | relative_url }}" type="video/mp4">
  Tu navegador no soporta video HTML5.
</video>