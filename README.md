[![Open in Visual Studio Code](https://classroom.github.com/assets/open-in-vscode-2e0aaae1b6195c2367325f4f02e2d04e9abb55f0b24a779b69b11b9e10269abc.svg)](https://classroom.github.com/online_ide?assignment_repo_id=20745350&assignment_repo_type=AssignmentRepo)

# Lab 4 — Visualización de datos en Raspberry Pi con Node-RED y Python

## Integrantes
- [David Santiago Puentes Cárdenas — 99225](https://github.com/Monstertrox)  
- [Juan David Arias Bojacá — 107394](https://github.com/juandariasb-ai)

---

## 1) Descripción general

Este laboratorio implementa un **sistema de visualización y registro de colores** en una **Raspberry Pi** usando **Node-RED** (dashboard web) y **Python** (procesamiento de datos). El flujo permite:

- Elegir un color en un **Color Picker** (dashboard Node-RED).  
- Ver y editar el color en **RGB** de forma manual.  
- **Convertir** entre **Hex ↔ RGB** en tiempo real.  
- **Guardar** los datos (con timestamp) en un archivo local.  
- **Procesar** esos registros con un script de **Python**.

> Objetivo: construir un pipeline simple, reproducible y robusto de **captura → transformación → almacenamiento → análisis** sobre Raspberry Pi.

---

## 2) Requisitos

- **Hardware**: Raspberry Pi (3B+ o superior recomendado), microSD ≥16 GB, red local.  
- **SO**: Raspberry Pi OS (32/64 bits).  
- **Software**:  
  - Node.js + Node-RED (instalador oficial de Node-RED para Pi).  
  - Python 3 con `pip` (incluye `venv` si se desea aislamiento).  
- **Red**: IP de la Pi accesible desde el navegador del cliente.

---

## 3) Arquitectura y diagrama

Flujo de datos: **Usuario (web) → Node-RED (dashboard) → conversión → archivo → Python (procesamiento)**.


![Diagrama del flujo](src/flujo.png)


---

## 4) Estructura sugerida del repositorio

```
.
├─ README.md
├─ src/
│  ├─ flujo.png                 # Diagrama del sistema (esta imagen)
│  ├─ flow_lab4.json            # Export del flujo Node-RED (opcional)
│  └─ color_data.txt            # Archivo de datos (generado por Node-RED)
└─ python/
   └─ procesar_colores.py       # Script de ejemplo en Python
```

---

## 5) Instalación y configuración

### 5.1. Acceso por SSH
```bash
ssh usuario@IP_DE_LA_PI
# Tip: para conocer la IP en la Pi
hostname -I
```

### 5.2. Instalar/actualizar Node-RED en Raspberry Pi
```bash
bash <(curl -sL https://raw.githubusercontent.com/node-red/linux-installers/master/deb/update-nodejs-and-nodered)
```
Durante la instalación responde:  
- **Are you really sure you want to do this? [y/N] →** `y`  
- **Would you like to install the Pi-specific nodes? [y/N] →** `y`

### 5.3. Habilitar Node-RED como servicio
```bash
sudo systemctl enable nodered.service
sudo systemctl start nodered.service
# Ver estado y logs si algo falla:
systemctl status nodered.service
journalctl -u nodered -f
```

### 5.4. Instalar el Dashboard en Node-RED
En el editor web de Node-RED: **Menu → Manage palette → Install**  
Buscar e instalar: **`node-red-dashboard`**

---

## 6) Flujo en Node-RED

### 6.1. Nodos y configuración

**Color Picker (dashboard)**
- Grupo: `Default`
- Etiqueta: `Selector de color`
- Formato: **Hexadecimal** (ej. `#80FF00`)

**Text Input (dashboard)**
- Grupo: `Default`
- Etiqueta: `Valores RGB`
- Placeholder: `Ej: 255,128,64`
- Validar que el texto cumpla `R,G,B` con `0–255`.

**Function (Procesamiento y conversiones)**  
_Conversión Hex → RGB y RGB → Hex; salida unificada con timestamp._
```js
// Utilidades
function clamp(v){ v = Number(v); return Math.max(0, Math.min(255, v||0)); }
function hexToRgb(hex){
    const m = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec((hex||"").trim());
    if(!m) return null;
    return { r: parseInt(m[1],16), g: parseInt(m[2],16), b: parseInt(m[3],16) };
}
function rgbToHex(r,g,b){
    const toHex = (n)=>clamp(n).toString(16).padStart(2,"0").toUpperCase();
    return `#${toHex(r)}${toHex(g)}${toHex(b)}`;
}

// Entradas posibles:
// - Desde Color Picker: msg.payload = "#RRGGBB"
// - Desde Text Input:   msg.payload = "R,G,B"
const now = new Date().toISOString();
let hex, rgb;

if (typeof msg.payload === "string" && msg.payload.trim().startsWith("#")) {
    // Viene del Color Picker
    rgb = hexToRgb(msg.payload.trim());
    if(!rgb) { node.error("Hex inválido", msg); return null; }
    hex = msg.payload.trim().toUpperCase();
} else if (typeof msg.payload === "string") {
    // Viene de Text Input
    const parts = msg.payload.split(",").map(s=>s.trim());
    if (parts.length !== 3) { node.error("RGB inválido", msg); return null; }
    rgb = { r: clamp(parts[0]), g: clamp(parts[1]), b: clamp(parts[2]) };
    hex = rgbToHex(rgb.r, rgb.g, rgb.b);
} else {
    node.error("Tipo de payload no soportado", msg);
    return null;
}

// Estructura canónica
msg.payload = {
    timestamp: now,
    hex,
    rgb
};

// Para escritura en archivo como línea JSON
msg.line = JSON.stringify(msg.payload) + "\n";
return msg;
```

**File (Almacenamiento persistente)**
- Filename: `/home/pi/color_data.txt` (o `src/color_data.txt` dentro del repo si tienes permisos).  
- Acción: **`Append to file`**  
- **Contenido a escribir**: usa `msg.line` (línea JSON con salto de línea).  
  - En el nodo File: setea **`msg.line`** como entrada (habilita “Append” y “utf8”).

**Debug (Monitoreo)**
- Nivel: `complete msg` para ver estructura y depurar.

### 6.2. Conexiones del flujo

```
Color Picker  ─┐
               ├─> Function ──> Debug
Text Input   ──┘               └─> File (append JSONL)
```

---

## 7) Script de Python (procesamiento)

Ejemplo: lectura de **JSONL** (`color_data.txt`), estadísticas simples y generación de colores complementarios.

`python/procesar_colores.py`
```python
#!/usr/bin/env python3
import json
from pathlib import Path
from datetime import datetime

DATA_FILE = Path(__file__).resolve().parents[1] / "src" / "color_data.txt"

def hex_to_rgb(hex_str: str):
    hex_str = hex_str.strip().lstrip("#")
    if len(hex_str) != 6:
        return None
    r = int(hex_str[0:2], 16)
    g = int(hex_str[2:4], 16)
    b = int(hex_str[4:6], 16)
    return (r, g, b)

def rgb_to_hex(r: int, g: int, b: int) -> str:
    clamp = lambda v: max(0, min(255, int(v)))
    return "#{:02X}{:02X}{:02X}".format(clamp(r), clamp(g), clamp(b))

def complement(hex_str: str) -> str:
    r, g, b = hex_to_rgb(hex_str)
    return rgb_to_hex(255 - r, 255 - g, 255 - b)

def main():
    if not DATA_FILE.exists():
        print(f"[WARN] No existe el archivo: {DATA_FILE}")
        return

    registros = []
    with open(DATA_FILE, "r", encoding="utf-8") as f:
        for line in f:
            line = line.strip()
            if not line:
                continue
            try:
                item = json.loads(line)
                # Validaciones mínimas
                if "hex" in item and "rgb" in item and "timestamp" in item:
                    registros.append(item)
            except json.JSONDecodeError:
                # Línea corrupta o previa al cambio a JSONL: ignorar
                continue

    if not registros:
        print("[INFO] No hay registros válidos.")
        return

    # Reporte simple
    latest = max(registros, key=lambda x: x["timestamp"])
    comp = complement(latest["hex"])
    print("=== Resumen ===")
    print(f"Total de registros: {len(registros)}")
    print(f"Último registro: {latest['timestamp']} — {latest['hex']} {latest['rgb']}")
    print(f"Complementario: {comp}")

if __name__ == "__main__":
    main()

## 8) Acceso y operación

- **Editor Node-RED**: `http://IP_DE_LA_PI:1880`  
- **Dashboard**: `http://IP_DE_LA_PI:1880/ui`

**Flujo típico**  
1. Abre el dashboard y selecciona un color con el **Color Picker**.  
2. Observa/edita los valores en **RGB** (campo de texto).  
3. El Function convierte y **uniformiza** la salida (`{ timestamp, hex, rgb }`).  
4. El nodo File **anexa** una línea JSON al archivo.  
5. Ejecuta Python para **analizar** lo registrado:
   ```bash
   cd python
   python3 procesar_colores.py
   ```

---

## 9) Validaciones y manejo de errores

**En Node-RED**
- **Hex válido**: `#RRGGBB` (6 dígitos hex).  
- **RGB válido**: `R,G,B` con `0–255`.  
- **Permisos de archivo**: si `/home/pi/color_data.txt` no se crea, revisa permisos del usuario que ejecuta el servicio.

**En Python**
- Verifica existencia del archivo.  
- Ignora líneas corruptas (pre-JSONL) sin romper la ejecución.  
- “Clampea” valores al rango `0–255`.

**Servicio**
```bash
systemctl status nodered.service
journalctl -u nodered -f
```

---

## 10) Pruebas rápidas (E2E)

1. **Conectividad**: `ping IP_DE_LA_PI`  
2. **Editor**: abre `http://IP_DE_LA_PI:1880` y despliega el flujo sin errores.  
3. **Dashboard**: cambia el color; verifica en Node-RED **Debug** la estructura de `msg.payload`.  
4. **Archivo**: confirma que `src/color_data.txt` o `/home/pi/color_data.txt` **crece** y contiene líneas JSON válidas.  
5. **Python**: corre `procesar_colores.py` y revisa el resumen.

---

## 11) Mejoras opcionales

- **MQTT (Mosquitto)**: publicar `hex/rgb` en un tópico (`vision/color`) para integrar otros nodos/procesos.  
- **HTTPS/Reverse Proxy**: exponer Node-RED detrás de Nginx con TLS (mejor para redes abiertas).  
- **Cambio de puerto**: si 1880 está ocupado, define `uiPort` en `~/.node-red/settings.js`.  
- **Autenticación**: habilitar credenciales en el editor y en dashboard.  
- **File rollover**: separar registros por día (`color_data_YYYYMMDD.txt`).  
- **Análisis avanzado**: Python puede transformar a HSV/HSL, generar paletas o disparar alertas si se cumple cierta condición.

---

## 12) Conclusiones

- **Acceso correcto** = mitad del éxito: IP válida + `:1880` para editor y `/ui` para dashboard.  
- **Datos coherentes**: unificar la **salida canónica** (`{timestamp, hex, rgb}`) simplifica depuración y análisis.  
- **Persistencia simple pero eficaz**: JSONL + `append` permite historiales, auditoría y fácil parsing.  
- **Escalabilidad**: con MQTT y rotación de archivos, el flujo crece sin volverse frágil.  
- **Mantenibilidad**: correr Node-RED como **servicio systemd** garantiza reinicio automático y observabilidad con `journalctl`.

---

## 13) Créditos y referencias

- Node-RED y Node-RED Dashboard.  
- Python 3 (estándar).  
- Raspberry Pi OS.  


