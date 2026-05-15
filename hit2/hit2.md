# Hit #2 — Painting a Landscape with Maths (Inigo Quilez)

Inigo Quilez pinta un paisaje completo (montañas, nubes, agua, cielo) usando solo matemáticas en un shader. Sin modelos 3D, sin texturas cargadas: todo se genera píxel a píxel con funciones.

Las técnicas principales que usa son:
- **Ray Marching**: lanza un rayo por cada píxel y avanza hasta tocar una superficie
- **SDF (Signed Distance Functions)**: funciones que describen la geometría sin polígonos
- **Noise fractal (fBm)**: sumas de ruido a distintas frecuencias para generar el terreno y las nubes
- **Iluminación procedural**: luz, sombras y niebla calculados matemáticamente

## Conclusiones

Lo más llamativo es que la escena no "existe" en memoria — se calcula en el momento para cada píxel. Esto refuerza el concepto de paralelismo: cada píxel es independiente del resto, exactamente igual a como funciona un kernel de CUDA donde cada hilo procesa su propio dato sin depender de los demás.
