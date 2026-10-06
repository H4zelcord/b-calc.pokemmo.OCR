# Hoja de Ruta: OCR breeding-calc.pokemmo

---

## Índice

- [Estructura de proyecto](#estructura-de-proyecto)
- [Detección de ventana y captura de región](#detección-de-ventana-y-captura-de-región)
- [Preprocesamiento de imagen](#preprocesamiento-de-imagen)
- [Reconocimiento de texto con EasyOCR](#reconocimiento-de-texto-con-easyocr)
- [Estructuración de datos](#estructuración-de-datos)
- [Lógica de crianza](#lógica-de-crianza)
- [Interfaz de usuario](#interfaz-de-usuario)
- [Empaquetado y distribución](#empaquetado-y-distribución)
- [Recursos generales](#recursos-generales)

---

## Estructura de proyecto

Se define la organización de carpetas, archivos y módulos del proyecto antes de escribir código. Se establece la carpeta raíz `b-calc.pokemmo.ocr` con una separación clara de responsabilidades por fase: `src/capture/`, `src/preprocessing/`, `src/ocr/`, `src/parsing/`, `src/breeding/` y `src/ui/`. Se añaden carpetas auxiliares: `data/` para configuración (mapa de regiones y esquema de datos), `tests/` para pruebas unitarias, `docs/` para documentación y `assets/` para iconos e imágenes. Se crea `requirements.txt` con las dependencias del proyecto y un archivo `.gitignore` que excluye `venv/`, `__pycache__/`, `dist/` y `build/`. Se definen también los archivos de configuración iniciales: `data/regions.json` con el mapa de regiones relativas y `data/schema.json` con el esquema de datos del Pokémon.

El proyecto se desarrolla en Linux, por lo que las dependencias y herramientas deben funcionar de forma nativa en ese entorno. Al mismo tiempo, se busca compatibilidad con Windows, lo que implica abstraer las diferencias entre sistemas en una capa común. En Linux, la detección de ventanas se apoya en herramientas propias del entorno gráfico (X11 o Wayland) y en utilidades como `xdotool` o `wmctrl`, mientras que en Windows se usa `win32gui`. La captura de pantalla con `mss` funciona de forma idéntica en ambos sistemas, lo que simplifica esa parte. La capa de abstracción de plataforma se sitúa en `src/capture/platform/`, con implementaciones separadas para Linux y Windows, seleccionadas en tiempo de ejecución según `sys.platform`.

**Documentación:**

- Entornos virtuales: [docs.python.org/es/3/tutorial/venv.html](https://docs.python.org/es/3/tutorial/venv.html)
- Detección de ventanas en Linux con `wmctrl` y `xdotool`: consultar `man wmctrl` y `man xdotool`
- Detección de ventanas en Windows con `win32gui`: buscar "win32gui FindWindow GetClientRect" en Stack Overflow

---

## Detección de ventana y captura de región

Se localiza la ventana de PokeMMO, se obtienen sus coordenadas y se captura una imagen recortada de una zona concreta, adaptándose al tamaño y posición de la ventana. El trabajo se apoya en tres conceptos clave: región de pantalla (X, Y, ancho, alto), diferencia entre coordenadas absolutas y relativas a la ventana, y uso del área cliente de la ventana (sin bordes ni barra de título) para que la captura sea precisa.

La detección de la ventana se abstrae en una capa de plataforma para que el resto del proyecto no dependa del sistema operativo. En Linux, se obtiene la geometría de la ventana mediante `wmctrl` o `xdotool` (bajo X11) o mediante `pygetwindow` con soporte limitado en Wayland. En Windows, se usa `win32gui` para obtener el área cliente exacta con `GetClientRect` y `ClientToScreen`. La captura de región se realiza siempre con `mss`, que es multiplataforma y ofrece el mismo comportamiento en ambos entornos.

Se definen las regiones relativas en porcentajes para cada campo, se calculan las coordenadas absolutas a partir de las dimensiones de la ventana, se captura la región con `mss` y se guarda como PNG. Se verifica el comportamiento moviendo y redimensionando la ventana. El mapa de regiones se persiste en `data/regions.json` para no tenerlo hardcodeado.

**Librerías:**

- `mss` (multiplataforma): [documentación](https://python-mss.readthedocs.io/)
- `pygetwindow` (multiplataforma, con limitaciones en Wayland): [documentación](https://pygetwindow.readthedocs.io/en/latest/)
- `wmctrl` / `xdotool` (Linux, X11): consultar `man wmctrl` y `man xdotool`
- `win32gui` (Windows): buscar "win32gui GetClientRect ClientToScreen" en Stack Overflow

**Documentación:**

- Captura de regiones con mss: [python-mss.readthedocs.io/en/latest/examples.html](https://python-mss.readthedocs.io/en/latest/examples.html)
- pygetwindow quickstart: [pygetwindow.readthedocs.io/en/latest/#quickstart](https://pygetwindow.readthedocs.io/en/latest/#quickstart)

---

## Preprocesamiento de imagen

Se transforma la captura cruda en una imagen limpia en blanco y negro, con el texto bien definido, para maximizar la precisión del OCR. Los conceptos que intervienen son escala de grises, umbralización (thresholding), reducción de ruido y escalado previo al OCR.

Se utiliza OpenCV para cargar la imagen con `cv2.imread()`, convertirla a escala de grises con `cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)`, aplicar `cv2.adaptiveThreshold()`, reducir ruido con `cv2.medianBlur()` o `cv2.bilateralFilter()`, y escalar 2x–3x con `cv2.resize()` usando interpolación cúbica. Se guardan las imágenes preprocesadas para inspección visual y se ajustan los parámetros hasta que el texto se vea nítido. OpenCV funciona de forma idéntica en Linux y Windows, por lo que no requiere capa de abstracción.

**Librería:**

- OpenCV (cv2): [docs.opencv.org/4.x/d6/d00/tutorial_py_root.html](https://docs.opencv.org/4.x/d6/d00/tutorial_py_root.html)

**Documentación:**

- Tutorial oficial de OpenCV-Python: [docs.opencv.org/4.x/d6/d00/tutorial_py_root.html](https://docs.opencv.org/4.x/d6/d00/tutorial_py_root.html)
- Umbralización adaptativa: buscar "OpenCV adaptive threshold Python".

---

## Reconocimiento de texto con EasyOCR

Se extrae el texto de la imagen preprocesada, obteniendo texto y valor de confianza por campo. Intervienen conceptos como OCR (Reconocimiento Óptico de Caracteres), confianza (0–1) e inicialización del modelo, que descarga los modelos en la primera ejecución y los carga desde caché en las siguientes.

Se inicializa el lector con `reader = easyocr.Reader(['es', 'en'])`, se ejecuta `reader.readtext(imagen)` y se extrae texto y confianza de cada resultado (`[coordenadas, texto, confianza]`). Se filtran resultados con confianza menor a 0.6 y se post-procesan confusiones comunes: `O` → `0`, `l` → `1`, `S` → `5`. Se prueba con distintas resoluciones y tamaños de ventana para verificar que la precisión se mantiene.

EasyOCR funciona en Linux y Windows, pero en Linux se instala sobre PyTorch con soporte CUDA opcional para GPU NVIDIA. En Windows también soporta CUDA. Se documenta la configuración de ambos entornos, incluyendo el uso de CPU como alternativa cuando no hay GPU compatible.

**Librería:**

- EasyOCR: [github.com/JaidedAI/EasyOCR](https://github.com/JaidedAI/EasyOCR)

**Documentación:**

- README oficial con ejemplos: [github.com/JaidedAI/EasyOCR](https://github.com/JaidedAI/EasyOCR)
- Instalación de PyTorch según plataforma: [pytorch.org/get-started/locally](https://pytorch.org/get-started/locally/)

---

## Estructuración de datos

Se convierte el texto crudo en un objeto Pokémon con campos definidos y validados. Intervienen expresiones regulares (regex), validación de rangos y mapeo de nombres a IDs internos.

Se define el esquema de datos del Pokémon:

```python
{
    "especie": str,
    "nivel": int,
    "ivs": {"hp": int, "atk": int, "def": int, "spa": int, "spd": int, "spe": int},
    "evs": {"hp": int, "atk": int, "def": int, "spa": int, "spd": int, "spe": int},
    "naturaleza": str,
    "habilidad": str,
    "movimientos": [str, str, str, str],
    "sexo": str
}
```

Se crean expresiones regulares por campo, se validan rangos (IVs 0–31, EVs 0–510, nivel 1–100) y se marcan los campos fallidos como "desconocido" para corrección manual. Se carga `monsters.json` (obtenido del Dump del juego) para mapear nombres a IDs internos y se normalizan naturalezas y movimientos.

**Documentación:**

- Expresiones regulares en Python: [docs.python.org/es/3/howto/regex.html](https://docs.python.org/es/3/howto/regex.html)
- Datos de PokeMMO: utilidad **"Settings → Utility → Dump Moddable Resources"** del juego, que genera `monsters.json`, `moves.json` y `skills.json`.

---

## Lógica de crianza

Se programan las reglas de crianza de PokeMMO para calcular resultados y costes. Intervienen los grupos huevo y la compatibilidad, la herencia de IVs mediante objetos de crianza, la naturaleza y la Piedra Eterna (50% de probabilidad), el consumo de padres (ambos desaparecen tras criar) y los costes económicos y de objetos.

Se documentan las reglas exactas de crianza, se implementa la comprobación de compatibilidad entre dos padres, el cálculo de herencia de IVs según los objetos equipados, el cálculo de probabilidad de naturaleza, el coste total en dinero y objetos, y opcionalmente un optimizador de ruta para IVs objetivo. Se escriben tests unitarios con `pytest`, que funciona igual en Linux y Windows.

**Documentación:**

- Guía de crianza (ShoutWiki): [pokemmo.shoutwiki.com/wiki/Breeding](https://pokemmo.shoutwiki.com/wiki/Breeding)
- Guía de crianza (Fandom): [pokemmo.fandom.com/wiki/Breeding](https://pokemmo.fandom.com/wiki/Breeding)
- Artículos sobre herencia y costes: [mmokb.com](https://mmokb.com/)

---

## Interfaz de usuario

Se construye una ventana donde el usuario ve los datos capturados, los corrige si es necesario y obtiene los resultados de la crianza. Intervienen widgets, layouts, señales y slots, y atajos de teclado globales.

Se crea la ventana principal con dos paneles (padre 1 / padre 2), botones "Capturar desde PokeMMO" por panel, visualización de la captura usada y del texto reconocido para verificación, corrección manual de cualquier campo, atajos globales (`Ctrl+Alt+1`, `Ctrl+Alt+2`) con `pynput`, presentación de los resultados de crianza (IVs del huevo, costes, probabilidades), sección de configuración (ruta, idioma, umbral de confianza) y guardado del historial de capturas y resultados.

PyQt6 y pynput funcionan en Linux y Windows. En Linux, los atajos globales pueden requerir permisos adicionales según el entorno de escritorio (X11 funciona de forma directa; Wayland puede requerir configuración extra). En Windows funcionan sin configuración adicional. La sección de configuración debe permitir al usuario ajustar la ruta del ejecutable de PokeMMO o su título de ventana según el sistema.

**Librerías:**

- PyQt6: [doc.qt.io/qtforpython-6](https://doc.qt.io/qtforpython-6/)
- pynput (atajos globales): [pynput.readthedocs.io](https://pynput.readthedocs.io/)

**Documentación:**

- Tutorial de PyQt6 en español: [ellibrodepython.com/interfaz-grafica-python](https://ellibrodepython.com/interfaz-grafica-python)
- Documentación oficial PyQt6: [doc.qt.io/qtforpython-6](https://doc.qt.io/qtforpython-6/)

---

## Empaquetado y distribución

Se convierte el script en un ejecutable que cualquiera pueda usar sin instalar Python. En Linux se distribuye como binario con PyInstaller, y en Windows como `.exe` con la misma herramienta.

Se empaqueta con `pyinstaller --onefile --windowed src/main.py`, se verifica el ejecutable generado en `dist/`, se prueba en una máquina limpia sin Python instalado y se añade un icono con `--icon=logo.ico` (Windows) o `--icon=logo.png` (Linux). Se redacta un README con instalación, uso y limitaciones para cada plataforma, y se incluye un aviso sobre los Términos de Servicio de PokeMMO: la herramienta solo captura la pantalla, no lee memoria ni modifica el cliente. Se documentan también las dependencias del sistema en Linux (por ejemplo, `libxcb` para Qt, `wmctrl` o `xdotool` para detección de ventanas) y las alternativas si no están disponibles.

**Documentación:**

- Manual oficial de PyInstaller: [pyinstaller.org](https://pyinstaller.org/)
- Comando básico: `pyinstaller --onefile --windowed tu_script.py`
- Empaquetado en Linux: [pyinstaller.org/en/stable/usage.html](https://pyinstaller.org/en/stable/usage.html)

---

## Recursos generales

**Foros y comunidades:**

- Foro oficial de PokeMMO: [forums.pokemmo.com](https://forums.pokemmo.com/)
- ShoutWiki de PokeMMO: [pokemmo.shoutwiki.com](https://pokemmo.shoutwiki.com/)

**Repositorios de datos:**

- PokeMMO-Data en GitHub: [github.com/PokeVengers/PokeMMO-Data](https://github.com/PokeVengers/PokeMMO-Data)
- Utilidad de Dump del juego: "Settings → Utility → Dump Moddable Resources".

**Herramientas similares:**

- PokeMMOHub: [pokemmohub.com](https://pokemmohub.com/)
- Búsqueda en GitHub: "game OCR screen reader Python".

**Compatibilidad Linux / Windows:**

- Detección de ventanas en Linux (X11): `wmctrl`, `xdotool`
- Detección de ventanas en Windows: `pywin32` (`win32gui`)
- Captura multiplataforma: `mss`
- Empaquetado multiplataforma: PyInstaller (genera binario en Linux y `.exe` en Windows)
