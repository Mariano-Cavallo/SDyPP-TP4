# Clasificación y Funciones de Shaders

## 1. Vertex Shader
* **Frecuencia:** Se ejecuta una vez por cada vértice.
* **Función principal:** Transforma posiciones 3D del espacio del modelo al espacio de pantalla (proyección).
* **Tareas secundarias:** Cálculo de normales, coordenadas UV y coeficientes de iluminación base.

## 2. Geometry Shader
* **Entrada:** Toma primitivas completas (triángulos, líneas o puntos).
* **Capacidad:** Puede generar nuevas primitivas o eliminar las existentes en tiempo real.
* **Aplicaciones:** Creación de partículas, sombras volumétricas, extrusión de siluetas o renderizado de pelo/césped.

## 3. Tessellation Shaders
* **Mecanismo:** Subdividen mallas simples en geometrías complejas dinámicamente.
* **Componentes:** Incluye las etapas de **Control** (Hull Shader) y **Evaluación** (Domain Shader).
* **Optimización:** Ajusta el Nivel de Detalle según la distancia de la cámara. Reduce el uso de ancho de banda al generar relieve en la GPU en lugar de cargar modelos de alta densidad desde la memoria.

## 4. Compute Shader (GPGPU)
* **Naturaleza:** No forma parte del pipeline gráfico tradicional de rasterización.
* **Propósito:** Ejecuta cálculos de propósito general utilizando la arquitectura paralela de la GPU.
* **Relación Técnica:** Actúa como el puente conceptual hacia tecnologías como CUDA o OpenCL para procesamiento de datos no visuales.

## 5. Pixel / Fragment Shader
* **Frecuencia:** Se ejecuta una vez por cada fragmento (píxel candidato).
* **Función principal:** Determina el color final y los atributos del píxel en el *framebuffer*.

### Características y Operación
* **Ejecución Paralela:** Se invoca masivamente para cada fragmento generado por el rasterizador.
* **Procesamiento Visual:** Responsable de texturizado, iluminación avanzada (difusa, especular), sombras y efectos de materiales (transparencias, reflexiones).
* **Flujo de Datos:** Recibe datos interpolados del Vertex Shader (UVs, normales) y devuelve un vector de color, generalmente en formato RGBA.
* **Espacio 2D:** Opera sobre la imagen ya rasterizada, permitiendo técnicas como *normal mapping* para simular detalle superficial sin aumentar la carga poligonal.


# Pipeline de renderizado GPU (WebGL)
WebGL ejecuta pares de funciones (vertex + fragment shader) sobre la GPU. El pipeline completo tiene 6 etapas principales:

## Etapas 3D
1. Vertex Shader — transforma vértices de coordenadas del objeto al clip space.
2. Primitive Assembly — agrupa vértices en triángulos/líneas/puntos y hace culling/clipping.
3. Rasterización — convierte primitivas 3D en fragmentos 2D; el puente entre ambos mundos.

## Etapas 2D
4. Fragment Shader — calcula el color de cada pixel (texturizado, iluminación, efectos).
5. Testing & Blending — depth test, stencil test, alpha blending.
6. Framebuffer — escribe el resultado final en memoria de pantalla.

# Post-processing
El post-processing es la aplicación de efectos visuales sobre la imagen ya renderizada (el framebuffer). Opera completamente en 2D, píxel a píxel, una vez que la escena 3D fue convertida a imagen. Ejemplos: bloom, motion blur, color grading, FXAA, depth of field, chroma key.

Etapa del pipeline Se ejecuta después del Fragment Shader y el Framebuffer — etapa 4/5/6 — típicamente como un pase adicional donde se usa el framebuffer de la escena como textura de entrada de un nuevo fragment shader.

# Entradas del shader (ShaderToy uniforms)

| Nombre | Tipo | Descripción |
| :--- | :--- | :--- |
| **iResolution** | `vec3` | Resolución del viewport en píxeles (x=ancho, y=alto, z=1.0). |
| **iTime** | `float` | Tiempo de reproducción del shader en segundos. |
| **iTimeDelta** | `float` | Tiempo que tardó en renderizarse el frame anterior (delta time). |
| **iFrame** | `int` | Número de frame actual desde el inicio del shader. |
| **iFrameRate** | `float` | FPS actuales del shader. |
| **iMouse** | `vec4` | xy: posición actual del mouse; zw: posición del último click (si se mantiene presionado). |
| **iChannel0..3** | `sampler2D/Cube` | Texturas de entrada configurables (imagen, video, webcam, ruido, audio). |
| **iChannelResolution** | `vec3[4]` | Resolución de cada canal `iChannel` en píxeles. |
| **iChannelTime** | `float[4]` | Tiempo de reproducción de cada canal (especialmente para videos). |
| **iDate** | `vec4` | Fecha actual: x=año, y=mes, z=día, w=segundos desde medianoche. |
| **iSampleRate** | `float` | Tasa de muestreo de audio (generalmente 44100 Hz). |


#  Salidas del Pixel Shader

| Nombre | Tipo | Descripción |
| :--- | :--- | :--- |
| **fragColor** | `out vec4` | Color final del pixel: (R, G, B, Alpha) en rango [0.0, 1.0] |


## ¿Qué es uv?
Son las coordenadas UV (Texture Coordinates): fragCoord / iResolution.xy normaliza las coordenadas del pixel al rango [0.0, 1.0]. El pixel inferior izquierdo queda en (0,0) y el superior derecho en (1,1).

## ¿Por qué UV y no XY?
XY depende de la resolución de pantalla (p. ej. 1920×1080 px). UV es independiente de la resolución: el mismo valor UV funciona en cualquier tamaño de viewport. Permite escribir shaders portables y hacer cálculos matemáticos limpios en el rango [0,1].

## ¿Cómo logra animación con entradas "estáticas"?
La entrada iTime es un float que aumenta cada frame. Al usarlo dentro de cos(), el valor del coseno cambia en cada frame, produciendo colores que varían en el tiempo → animación.

## ¿Cómo col es vec3 si se iguala a flotantes?
GLSL permite swizzling y broadcasting: 0.5 + 0.5*cos(...) produce un escalar que se suma a un vec3. El escalar se expande automáticamente a vec3 (cada componente recibe el mismo valor). El resultado es un vec3.

## Constructores de vec4 y swizzling
vec4(col, 1.0) construye un vec4 desde un vec3 + un float. Los componentes son (R, G, B, A). uv.xyx es swizzling: toma los componentes x, y, x del vec2 y forma un vec3. vec2 tiene: x/y, r/g, s/t. vec3 agrega z/b/p. vec4 agrega w/a/q.

## ¿Qué hace vec3(0, 2, 4)?
Introduce un desfase de fase distinto para cada canal de color (R, G, B). Sin él, los tres canales serían iguales y se vería gris. Con el desfase, cada canal del coseno oscila a distintas fases → colores que cambian y varían entre sí.



