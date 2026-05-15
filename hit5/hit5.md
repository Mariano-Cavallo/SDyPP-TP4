# Hit #5 — Filtro Chroma Key

La idea es reemplazar los píxeles verdes del video (`iChannel1`) por la imagen de la webcam (`iChannel0`). Para saber si un píxel "es verde", se calcula la distancia pitagórica en el espacio de color RGB (3 dimensiones):

```
d = sqrt( (R1-R2)² + (G1-G2)² + (B1-B2)² )
```

En GLSL eso es simplemente `length(color1.rgb - color2.rgb)`.

## Shader

```glsl
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    vec2 uv = fragCoord.xy / iResolution.xy;

    vec4 videoColor  = texture(iChannel1, uv); // video con fondo verde
    vec4 cameraColor = texture(iChannel0, uv); // webcam

    vec4 chromaColor = vec4(0.0, 1.0, 0.0, 1.0); // verde
    float threshold  = 0.4;

    float dist = length(videoColor.rgb - chromaColor.rgb);

    if (dist < threshold) {
        fragColor = cameraColor; // píxel verde → mostrar webcam
    } else {
        fragColor = videoColor;  // píxel no verde → mostrar video
    }
}
```

## Análisis del umbral

- **Umbral bajo (0.1)**: solo reemplaza los verdes exactos, quedan bordes con halo verde.
- **Umbral medio (0.4)**: buen balance, la mayoría del fondo desaparece con pocos artefactos.
- **Umbral alto (0.7+)**: empieza a reemplazar también amarillos y cian, partes del sujeto pueden desaparecer.

El truco está en encontrar el valor que limpie el fondo sin "comerse" al sujeto. En este caso con 0.4 funcionó bastante bien.
