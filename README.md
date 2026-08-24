# Daheng MER2-2000-19U3C — Python / Ubuntu 24.04

Configuración y pruebas de una cámara industrial **Daheng Imaging MER2-2000-19U3C** utilizando el **Galaxy SDK / gxipy** en **Ubuntu 24.04** con Python 3.12.

Este repositorio documenta el proceso de instalación, compatibilidad y primeras pruebas realizadas con el SDK Python de Daheng Imaging.

---

## 1. Hardware

### Cámara

- **Fabricante:** Daheng Imaging
- **Modelo:** `MER2-2000-19U3C`
- **Número de serie:** `FCE25050396`
- **Interfaz detectada por el SDK:** USB3
- **`device_class`:** `3`

La cámara es detectada correctamente por el Galaxy SDK.

> 💡 **Nota:** El número de serie se incluye aquí únicamente como referencia del dispositivo utilizado durante las pruebas. Si este repositorio se hace público, puede ser conveniente eliminarlo.

---

## 2. Sistema operativo

### Sistema utilizado
- **SO:** Ubuntu 24.04
- **Arquitectura:** x86_64
- **Kernel utilizado durante las pruebas:** 7.0.0-30-generic
- **Python del sistema:** Python 3.12.3

### Comprobaciones realizadas

*   **Comando:** `uname -r`
    *   **Resultado:** `7.0.0-30-generic`
*   **Comando:** `uname -m`
    *   **Resultado:** `x86_64`
*   **Comando:** `python3 --version`
    *   **Resultado:** `Python 3.12.3`

---

## 3. Galaxy SDK

Antes de utilizar `gxipy` es necesario tener instalado el Galaxy Linux SDK.

En este equipo el SDK nativo ya estaba instalado antes de comenzar estas pruebas.

### Comprobación de `libgxiapi.so`

Se comprobó la existencia de la librería mediante el comando:
```bash
ldconfig -p | grep -E 'gxiapi|dximageproc'
```

*   **Resultado:**
    ```text
    libgxiapi.so (libc6,x86-64) => /lib/libgxiapi.so
    ```

También se realizó la búsqueda manual:
```bash
find /usr -name "libgxiapi.so*" 2>/dev/null
```

*   **Resultado:**
    ```text
    /usr/lib/libgxiapi.so
    ```

**Conclusión:** La biblioteca encontrada y activa es `/usr/lib/libgxiapi.so`.

### 3.1 libdximageproc.so

Durante la comprobación inicial no se encontró esta librería.

*   **Comando utilizado:**
    ```bash
    find /usr -name "libdximageproc.so*" 2>/dev/null
    ```
*   **Resultado:** *(No se encontró ninguna coincidencia)*

Por el momento esto no ha impedido detectar la cámara, por lo que no se ha instalado ni modificado ninguna biblioteca adicional relacionada con `libdximageproc.so`.

## 4. Galaxy Python SDK

Se utilizó el paquete: `Galaxy_Linux_Python_2.6.2606.9081`

### Estructura principal del paquete
```text
Galaxy_Linux_Python_2.6.2606.9081/
│
├── api/
│   ├── gxipy/
│   └── setup.py
│
├── sample/
│   ├── GxSingleCamColor
│   ├── GxSingleCamMono
│   ├── GxAcquireSoftTrigger
│   └── GxAcquireCallback
│
├── doc_en/
├── doc_cn/
└── README
```

El paquete Python depende directamente del Galaxy Linux SDK nativo.

### Arquitectura de comunicación
```text
  Python
    │
    ▼
  gxipy
    │
    ▼
Galaxy Linux SDK
    │
    ▼
  libgxiapi.so
    │
    ▼
Cámara Daheng
```

---

## 5. Entorno virtual Python

Para evitar modificar las librerías Python integradas en el sistema operativo, se configuró un entorno virtual aislado.

Desde el directorio raíz:
```bash
cd ~/Galaxy_Linux_Python_2.6.2606.9081
```

Al intentar la inicialización:
```bash
python3 -m venv daheng
```

El gestor de paquetes de Ubuntu arrojó el siguiente error por falta de `ensurepip`:
> *The virtual environment was not created successfully because ensurepip is not available.*

Se solucionó instalando el paquete faltante desde los repositorios oficiales:
```bash
sudo apt update
sudo apt install python3.12-venv
```

Posteriormente, se eliminó el directorio residual del entorno fallido y se generó nuevamente de forma limpia:
```bash
rm -rf daheng
python3 -m venv daheng
```

El entorno se activa ejecutando:
```bash
source daheng/bin/activate
```

Al estar operativo, la consola antepone el prefijo indicativo: `(daheng)`.

---

## 6. Python utilizado

Dentro del entorno virtual se trabaja con **Python 3.12.3**.

El propósito inicial de la prueba era determinar si la versión previa de `gxipy` lograba ejecutarse sobre Python 3.12, evitando la necesidad de forzar una degradación hacia Python 3.8.

**Estado actual:** Python 3.12 responde de manera correcta con `gxipy` aplicando únicamente una adaptación de código mínima. No se requirió instalar software adicional o versiones anteriores de Python.

## 7. NumPy

Inicialmente se intentó validar la importación de la librería mediante el comando:
```bash
PYTHONPATH=. python -c "import gxipy; print('gxipy importado correctamente')"
```

*   **Primer error detectado:** 
    ```text
    ModuleNotFoundError: No module named 'numpy'
    ```

Para solucionarlo, se instaló NumPy dentro del entorno virtual activo:
```bash
python -m pip install numpy
```

Al volver a ejecutar la prueba, apareció una segunda incompatibilidad de dependencias:
*   **Segundo error detectado:** 
    ```text
    ModuleNotFoundError: No module named 'numpy.compat'
    ```
*   **Origen del problema:** El error procedía del archivo `gxipy/DeviceManager.py`, específicamente en la línea:
    ```python
    from numpy.compat import long
    ```

---

## 8. Adaptación de gxipy para Python 3.12 / NumPy moderno

Se realizó una modificación mínima en el script fuente `api/gxipy/DeviceManager.py`.

*   **Línea original (Antes):** `from numpy.compat import long`

Esta importación fue eliminada por completo. El propio código fuente del SDK ya contempla una validación interna específica para la compatibilidad con Python 3 líneas más abajo:
```python
if sys.version_info.major > 2:
    INT_TYPE = int
else:
    INT_TYPE = (int, long)
```
Por lo tanto, al ejecutar el script bajo un entorno de Python 3, la variable obsoleta `long` nunca llega a declararse ni utilizarse, haciendo innecesaria su importación.

### 8.1 Copia de seguridad
Antes de aplicar cualquier cambio directo sobre el módulo, se generó un respaldo de seguridad del archivo:
```bash
cp gxipy/DeviceManager.py gxipy/DeviceManager.py.orig
```

La estructura local cuenta con los siguientes archivos:
*   `api/gxipy/DeviceManager.py` (Archivo de trabajo modificado)
*   `api/gxipy/DeviceManager.py.orig` (Archivo original sin cambios)

*(En caso de fallo, el archivo original puede restaurarse en cualquier momento ejecutando `cp gxipy/DeviceManager.py.orig gxipy/DeviceManager.py`)*.

### 8.2 Modificación realizada
Se automatizó la remoción de la cabecera obsoleta utilizando el editor de flujo `sed`:
```bash
sed -i '/from numpy.compat import long/d' gxipy/DeviceManager.py
```
La cabecera del archivo quedó libre de la línea conflictiva de manera exitosa.

---

## 9. Comprobación de compatibilidad de gxipy

Tras eliminar la importación incompatible de NumPy, se repitió el comando de diagnóstico:
```bash
PYTHONPATH=. python -c "import gxipy; print('gxipy importado correctamente')"
```

*   **Resultado obtenido:**
    ```text
    gxipy importado correctamente
    ```

El éxito de la prueba ratifica y demuestra la compatibilidad operativa de la siguiente cadena de entorno:

```text
Ubuntu 24.04
    │
    ▼
Python 3.12.3
    │
    ▼
Entorno virtual daheng
    │
    ▼
NumPy
    │
    ▼
gxipy 2.6.2606.9081
```
## 10. Detección de la cámara

Una vez que `gxipy` pudo importarse correctamente, se realizó la primera prueba de detección:
```bash
PYTHONPATH=. python -c "import gxipy as gx; dm=gx.DeviceManager(); print(dm.update_device_list())"
```

### Resultado obtenido:
```json
(1, [{
    'index': 1,
    'vendor_name': 'Daheng Imaging',
    'model_name': 'MER2-2000-19U3C',
    'sn': 'FCE25050396',
    'display_name': 'MER2-2000-19U3C(FCE25050396)',
    'device_id': 'MER2-2000-19U3C(FCE25050396)',
    'user_id': '',
    'access_status': 0,
    'device_class': 3,
    'mac': '',
    'ip': '',
    'subnet_mask': '',
    'gateway': '',
    'nic_mac': '',
    'nic_ip': '',
    'nic_subnet_mask': '',
    'nic_gateWay': '',
    'nic_description': ''
}])
```

---

## 11. Estado actual

La cámara está siendo detectada correctamente por el sistema.

| Componente | Estado |
| :--- | :--- |
| Ubuntu 24.04 | **OK** |
| Arquitectura x86_64 | **OK** |
| Python 3.12.3 | **OK** |
| Entorno virtual daheng | **OK** |
| NumPy | **OK** |
| Galaxy Linux SDK | **OK** |
| libgxiapi.so | **OK** |
| gxipy | **OK** (después de una adaptación mínima) |
| Detección de cámara | **OK** |
| MER2-2000-19U3C | **Detectada** |
| Captura de imagen | *Pendiente* |
| Streaming | *Pendiente* |
| Trigger | *Pendiente* |
| Callback | *Pendiente* |
| OpenCV | *Pendiente* |

---

## 12. Estructura actual del proyecto

La distribución del directorio de trabajo se encuentra configurada de la siguiente manera:

```text
Galaxy_Linux_Python_2.6.2606.9081/
│
├── daheng/
│   └──                 # Entorno virtual Python
│
├── api/
│   ├── gxipy/
│   │   ├── DeviceManager.py
│   │   ├── DeviceManager.py.orig
│   │   ├── Device.py
│   │   ├── Feature.py
│   │   ├── ImageProc.py
│   │   ├── ImageFormatConvert.py
│   │   ├── gxwrapper.py
│   │   ├── dxwrapper.py
│   │   └── ...
│   │
│   └── setup.py
│
├── sample/
│   ├── GxSingleCamColor/
│   ├── GxSingleCamMono/
│   ├── GxAcquireSoftTrigger/
│   └── GxAcquireCallback/
│
├── doc_en/
├── doc_cn/
└── README.md
```

## 13. Comandos para trabajar con el entorno

**Activar el entorno:**
```bash
cd ~/Galaxy_Linux_Python_2.6.2606.9081
source daheng/bin/activate
```

**Comprobar la versión de Python activa:**
```bash
python --version
```

**Comprobar la ruta del binario de Python utilizado:**
```bash
which python
```

**Entrar en el directorio de la API:**
```bash
cd ~/Galaxy_Linux_Python_2.6.2606.9081/api
```

**Probar la importación del módulo gxipy:**
```bash
PYTHONPATH=. python -c "import gxipy; print('gxipy importado correctamente')"
```

**Detectar cámaras conectadas en el bus:**
```bash
PYTHONPATH=. python -c "import gxipy as gx; dm=gx.DeviceManager(); print(dm.update_device_list())"
```

---

## 14. Próximo paso

El siguiente objetivo técnico es comprobar que el software puede abrir la cámara e inicializar la comunicación. 

### Flujo lógico de la prueba de captura:
```text
Cámara
   │
   ▼
DeviceManager
   │
   ▼
open_device_by_index()
   │
   ▼
stream_on()
   │
   ▼
get_image()
   │
   ▼
Frame
   │
   ▼
NumPy
```

### Objetivos específicos de la primera prueba:
*   Apertura del dispositivo.
*   Lectura de parámetros básicos de configuración.
*   Inicio del stream de datos.
*   Captura de un frame individual de prueba.
*   Conversión de la estructura del frame nativo a una matriz NumPy.
*   Guardado de la imagen resultante en disco.
*   Cierre correcto y liberación de los recursos de la cámara.

---

## 15. Próximas pruebas previstas

Una vez validada con éxito la rutina de captura básica, se procederá al estudio y desarrollo de las siguientes funcionalidades:

### Captura continua
```text
Camera ➔ Stream ➔ Frames ➔ NumPy
```

### Exposición
Pruebas de calibración variando dinámicamente el parámetro:
*   `ExposureTime`

### Ganancia
Pruebas de ajuste de iluminación variando el parámetro:
*   `Gain`

### Trigger por software
Envío de señales de sincronización de manera lógica:
```text
Python ➔ Software Trigger ➔ Frame
```

### Callback
Captura asíncrona controlada por eventos de hardware:
```text
Camera ➔ New Frame ➔ Callback ➔ Procesamiento
```

### Procesamiento de imagen
Integración posterior del flujo de matrices de datos con las librerías:
*   **NumPy** (Cálculo numérico y manipulación de arreglos)
*   **OpenCV** (Visión artificial y visualización en tiempo real)
*   **SciPy** (Algoritmos avanzados y procesamiento estadístico de señales)

## 16. Objetivo final

El objetivo del proyecto es disponer de un entorno reproducible para trabajar con la cámara Daheng desde Python en Ubuntu 24.04:

```text
┌───────────────────────────┐
│       Ubuntu 24.04        │
├───────────────────────────┤
│ Python 3.12               │
│ Virtual Environment       │
│ NumPy                     │
│ gxipy 2.6.2606.9081       │
├───────────────────────────┤
│ Galaxy Linux SDK          │
│ libgxiapi.so              │
├───────────────────────────┤
│ Daheng MER2-2000-19U3C    │
└───────────────────────────┘
```

### Estado del proyecto

**Estado actual:** Cámara detectada correctamente.

- [x] Ubuntu 24.04
- [x] x86_64
- [x] Galaxy Linux SDK
- [x] `libgxiapi.so`
- [x] Python 3.12
- [x] Entorno virtual
- [x] NumPy
- [x] `gxipy`
- [x] Compatibilidad con NumPy corregida
- [x] Cámara MER2-2000-19U3C detectada
- [ ] Abrir cámara
- [ ] Capturar imagen
- [ ] Guardar imagen
- [ ] Streaming
- [ ] Trigger
- [ ] Callback
- [ ] Procesamiento OpenCV

---

### Nota

Este README documenta el estado de desarrollo y las pruebas realizadas sobre el equipo utilizado durante el proyecto.

El SDK Python utilizado es una versión antigua (`2.6.2606.9081`) y se ha realizado una adaptación mínima para poder importarlo correctamente con Python 3.12 y NumPy moderno. 

La biblioteca nativa del Galaxy SDK (`libgxiapi.so`) no ha sido modificada.
