# Hit #4 — Coordenadas UV y Flip

La clave de este hit es que manipulando las coordenadas UV *antes* de leer la textura podemos transformar la imagen sin tocar los datos originales.

## FLIP-Y (imagen cabeza abajo)

```glsl
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    vec2 uv = fragCoord.xy / iResolution.xy;
    uv.y = 1.0 - uv.y;
    fragColor = texture(iChannel0, uv);
}
```

Al hacer `1.0 - uv.y`, el borde inferior pasa a leer lo que estaba arriba y viceversa. El centro (0.5) no cambia.

## FLIP-X (espejo horizontal)

```glsl
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    vec2 uv = fragCoord.xy / iResolution.xy;
    uv.x = 1.0 - uv.x;
    fragColor = texture(iChannel0, uv);
}
```

Mismo concepto pero en el eje horizontal.

## Potencialidad de UV

Con simples operaciones matemáticas sobre UV se pueden lograr efectos como zoom, repetición (tiling) o rotación, todo sin costo adicional de memoria y de forma paralela por píxel.
