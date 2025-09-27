[![Open in Visual Studio Code](https://classroom.github.com/assets/open-in-vscode-2e0aaae1b6195c2367325f4f02e2d04e9abb55f0b24a779b69b11b9e10269abc.svg)](https://classroom.github.com/online_ide?assignment_repo_id=20745350&assignment_repo_type=AssignmentRepo)

# Lab 4 — Visualización de datos en Raspberry Pi con Node-RED y Python

## Integrantes

[David Santiago Puentes Cárdenas - 99225](https://github.com/Monstertrox)

[Juan David Arias Bojacá - 107394](https://github.com/juandariasb-ai)

## Documentación

1. Descripción
   Sistema de visualización de datos implementado en Raspberry Pi utilizando Node-RED y Python. Incluye:

Flujo Node-RED con selector de color, visualización de valores RGB y almacenamiento en archivo.

Script Python para lectura y procesamiento de valores RGB desde archivo.

Configuración automática de Node-RED como servicio systemd.

Dashboard interactivo para control y monitoreo.

## 2. Diagrama del Flujo (Simplificado)

Flujo del sistema implementado (conexión entre Raspberry Pi, Node-RED y Python):

![flujo]("C:\Users\DELL\Downloads\VPN.png")

3. Componentes Principales

3.1. Node-RED Flow
Color Picker: Selector gráfico de color hexadecimal

Text Input: Campo para visualización/edición manual de valores RGB

Function Node: Conversión y procesamiento de datos de color

File Node: Almacenamiento persistente en archivo de texto

Debug Node: Monitoreo en tiempo real del flujo de datos

4. Instrucciones de Instalación y Configuración

4.1. Conexión SSH a Raspberry Pi

ssh usuario_raspberry@ip_raspberry

4.2. Instalación de Node-RED

bash <(curl -sL https://raw.githubusercontent.com/node-red/linux-installers/master/deb/update-nodejs-and-nodered)

Respuestas durante instalación:

Are you really sure you want to do this? [y/N] → y

Would you like to install the Pi-specific nodes? [y/N] → y

4.3. Configuración como Servicio Systemd

sudo systemctl enable nodered.service
sudo reboot

4.4. Instalación del Dashboard
En interfaz Node-RED: Menu > Manage palette > Install

Buscar e instalar: node-red-dashboard

5. Flujo de Node-RED
   5.1. Configuración de Nodos
   Color Picker Node:

Grupo: Default

Etiqueta: Selector de Color

Formato: Hexadecimal

Text Input Node:

Grupo: Default

Etiqueta: Valores RGB

Placeholder: Ej: 255,128,64

Function Node (Procesamiento):

// Conversión hexadecimal a RGB
msg.payload = {
hex: msg.payload,
rgb: hexToRgb(msg.payload)
};
return msg;

File Node (Almacenamiento):

Filename: /home/pi/color_data.txt

Acción: Append to file

5.2. Conexiones del Flujo

Color Picker → Function → [Text Input, Debug, File]
Text Input → Function → [Debug, File]

6. Acceso y Operación
   6.1. Interfaz Web
   Node-RED Editor: http://ip_raspberry:1880

Dashboard: http://ip_raspberry:1880/ui

6.2. Flujo de Operación
Selección de Color: Usar color picker en el dashboard

Visualización: Valores RGB mostrados en campo de texto

Almacenamiento: Datos guardados automáticamente en archivo

Procesamiento: Script Python lee y procesa los datos almacenados

7. Validaciones y Manejo de Errores

7.1. Validaciones en Node-RED
Formato hexadecimal válido en color picker

Validación de formato RGB en text input

Manejo de errores en escritura de archivo

7.2. Validaciones en Python
Verificación de existencia de archivo

Parsing robusto de formatos de color

Manejo de valores fuera de rango (0-255)

8. Ejemplos de Uso
   8.1. Caso Normal
   Usuario selecciona color #4A90E2 en el picker

Campo de texto muestra 74, 144, 226

Datos se guardan en archivo con timestamp

Script Python procesa los valores para aplicación específica

8.2. Entrada Manual
Usuario escribe 128, 255, 0 en campo de texto

Function node convierte a hexadecimal #80FF00

Flujo continúa igual que con selección por picker

## Conclusiones

1. Configuración Correcta de Acceso Remoto - Elemento Crítico
   Se demostró que la configuración precisa de la dirección IP y puertos es fundamental para el éxito del proyecto. Aspectos clave identificados:

IP Exacta de la Raspberry Pi: El uso de la dirección IP correcta es indispensable para la conexión SSH inicial y posterior acceso web

Puerto 1880 Obligatorio: La URL http://[100.97.75.115]:1880 no es opcional - sin el puerto 1880 explícito, el acceso a Node-RED es imposible

Dashboard en /ui: La ruta específica http://[100.97.75.115]:1880/ui es esencial para visualizar la interfaz del usuario final

2. Consecuencias de Configuraciones Incorrectas
   Se evidenció que errores en estos parámetros generan problemas inmediatos:

Conexión SSH Fallida: IP incorrecta → imposibilidad de administración remota

Node-RED Inaccesible: Omisión del puerto 1880 → error de conexión en el navegador

Dashboard No Visible: Falta de /ui → interfaz de usuario no disponible, aunque Node-RED esté funcionando

3. Verificación en Capas del Acceso
   El proyecto implementó una estrategia de verificación escalonada:

Conexión SSH Exitosa (IP correcta)
↓
Node-RED Accesible (IP:1880)
↓
Dashboard Funcional (IP:1880/ui)
↓
Sistema Operativo Completamente

4. Importancia del Puerto 1880 en la Arquitectura
   Puerto Predeterminado: 1880 no es aleatorio - es el puerto estándar asignado oficialmente para Node-RED

Avoids Port Conflicts: Evita conflictos con otros servicios web (puerto 80) o aplicaciones

Security Aspect: Puerto no estándar proporciona mínima seguridad por ofuscación
