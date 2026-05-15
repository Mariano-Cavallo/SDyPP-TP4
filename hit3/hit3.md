# Hit #3 — Copiar píxeles desde iChannel0

Para este hit configuré `iChannel0` con la webcam (también puede ser una imagen o video de ejemplo). Se hace clickeando el recuadro de `iChannel0` debajo del editor en ShaderToy.

## Shader

```glsl
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    vec2 uv = fragCoord.xy / iResolution.xy;
    fragColor = texture(iChannel0, uv);
}
```

**`vec2 uv`**: normaliza las coordenadas del píxel al rango `[0.0, 1.0]` dividiendo por la resolución. Esto hace que el shader funcione igual sin importar el tamaño de pantalla.

**`texture(iChannel0, uv)`**: lee el color del canal en esas coordenadas y lo escribe directamente en la salida. El resultado es la textura mostrada sin ninguna modificación.
