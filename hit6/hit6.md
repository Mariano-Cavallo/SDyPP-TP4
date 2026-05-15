# Codigo para escala de grices shadertoy


'''
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    // Normalizar las coordenadas de la pantalla (de 0.0 a 1.0)
    vec2 uv = fragCoord / iResolution.xy;

    // Obtener el color de la textura de entrada (Canal iChannel0)
    vec4 texColor = texture(iChannel0, uv);
    
    // Calcular la luminancia usando los coeficientes estándar BT.601
    // Y = 0.299*R + 0.587*G + 0.114*B
    float gray = dot(texColor.rgb, vec3(0.299, 0.587, 0.114));
    
    // Asignar el valor de gris a los canales RGB, manteniendo el alfa original
    fragColor = vec4(vec3(gray), texColor.a);
}

'''

# kernel de CUDA

'''
#include <cuda_runtime.h>
#include <device_launch_parameters.h>

// Estructura básica para representar un píxel RGBA
struct Pixel {
    unsigned char r, g, b, a;
};

// Kernel CUDA para convertir la imagen a escala de grises
__global__ void grayscaleKernel(const Pixel* d_input, Pixel* d_output, int width, int height)
{
    // 1. Calcular las coordenadas globales del píxel (X, Y) para este hilo
    int x = blockIdx.x * blockDim.x + threadIdx.x;
    int y = blockIdx.y * blockDim.y + threadIdx.y;

    // 2. Verificar que el hilo no se encuentre fuera de los límites de la imagen
    if (x < width && y < height)
    {
        // Calcular el índice lineal unidimensional en el arreglo de memoria
        int idx = y * width + x;

        // Leer el píxel de la memoria global de la GPU
        Pixel p = d_input[idx];

        // 3. Aplicar los coeficientes del estándar Rec. 709
        // Se opera en punto flotante y luego se convierte a entero (0-255)
        float grayFactor = 0.2126f * p.r + 0.7152f * p.g + 0.0722f * p.b;
        unsigned char gray = static_cast<unsigned char>(grayFactor);

        // 4. Escribir el resultado en el búfer de salida, manteniendo el canal alfa
        d_output[idx].r = gray;
        d_output[idx].g = gray;
        d_output[idx].b = gray;
        d_output[idx].a = p.a;
    }
}
'''

# Función host: configura y lanza el kernel
'''
#include <iostream>

void processGrayscale(const Pixel* h_inputImage, Pixel* h_outputImage, int width, int height)
{
    // Calcular el tamaño total de la imagen en bytes
    size_t imageSize = width * height * sizeof(Pixel);

    // Punteros para la memoria de la GPU (Device)
    Pixel* d_input = nullptr;
    Pixel* d_output = nullptr;

    // 1. Reservar memoria en la GPU
    cudaMalloc(&d_input, imageSize);
    cudaMalloc(&d_output, imageSize);

    // 2. Copiar los datos de la imagen desde la CPU (Host) a la GPU (Device)
    cudaMemcpy(d_input, h_inputImage, imageSize, cudaMemcpyHostToDevice);

    // 3. Configurar la geometría de la ejecución (Hilos y Bloques)
    // Se define un bloque estándar de 16x16 hilos (256 hilos por bloque)
    dim3 blockSize(16, 16);
    
    // Calcular cuántos bloques se necesitan para cubrir toda la imagen
    dim3 gridSize((width + blockSize.x - 1) / blockSize.x, 
                  (height + blockSize.y - 1) / blockSize.y);

    // 4. Lanzar el Kernel en la GPU
    // Sintaxis <<< bloques, hilos_por_bloque >>>
    grayscaleKernel<<<gridSize, blockSize>>>(d_input, d_output, width, height);

    // Sincronizar el dispositivo para asegurar que el procesamiento terminó
    cudaDeviceSynchronize();

    // 5. Copiar el resultado de regreso de la GPU (Device) a la CPU (Host)
    cudaMemcpy(h_outputImage, d_output, imageSize, cudaMemcpyDeviceToHost);

    // 6. Liberar la memoria asignada en la GPU
    cudaFree(d_input);
    cudaFree(d_output);
}
'''

## mainImage --> kernel
En Shadertoy, mainImage es el punto de entrada que el hardware gráfico invoca automáticamente para cada píxel de la pantalla de forma paralela.

En CUDA, el equivalente es una función tipo Kernel, la cual se define con el calificador __global__. A diferencia de Shadertoy, la ejecución no es automática; debes lanzar el kernel explícitamente desde la CPU especificando la configuración de la ejecución (el número de bloques e hilos).

## fragCoord --> Índices globales de hilos (blockIdx, threadIdx, blockDim)
En Shadertoy, fragCoord contiene las coordenadas de píxel de la pantalla (por ejemplo, desde (0.5, 0.5) hasta (1919.5, 1079.5)). La GPU gestiona esto internamente.

En CUDA no existe una variable precalculada con la posición del píxel. La GPU te proporciona la posición del hilo actual dentro de una estructura de ejecución jerárquica (bloques y rejillas). Tienes que calcular la coordenada matemática manualmente combinando estas variables nativas:
'''
int x = blockIdx.x * blockDim.x + threadIdx.x; // Coordenada X del píxel
int y = blockIdx.y * blockDim.y + threadIdx.y; // Coordenada Y del píxel
'''

## iChannel0, iChannel1 --> Punteros de memoria lineal (const Pixel* d_input)
En Shadertoy, los canales son abstracciones de texturas (imágenes, videos o audio) gestionadas por la API gráfica de WebGL. Para acceder a ellos, usas la función incorporada texture(iChannel0, uv), la cual lee la textura utilizando coordenadas normalizadas de 0.0 a 1.0 e interpola los datos por hardware.

En CUDA estándar, no hay abstracción de texturas por defecto (aunque existen los Texture Objects basados en hardware, lo común en cómputo general es usar arreglos planos). Pasas los datos como un puntero a un bloque de memoria lineal en la VRAM asignado previamente con cudaMalloc. Para acceder a los datos, calculas el índice unidimensional plano:
'''
int idx = y * ancho + x; // Mapeo bidimensional a unidimensional
Pixel color = d_input[idx]; // Lectura directa de memoria
'''

## fraColor --> Escritura en el búfer de salida (d_output[idx] = ...)
En Shadertoy, fragColor es un parámetro de salida de tipo vec4 (RGBA con valores flotantes entre 0.0 y 1.0). El valor que asignes a esta variable se escribe automáticamente en el Frame Buffer para ser mostrado en pantalla.

En CUDA, no estás obligado a escribir en la pantalla ni a usar cuatro canales flotantes. El equivalente es simplemente escribir el resultado en un arreglo de salida en la memoria de la GPU utilizando el mismo índice lineal calculado. Los datos suelen guardarse en enteros de 8 bits (unsigned char de 0 a 255) para optimizar memoria si el objetivo es procesar una imagen estándar.
'''
d_output[idx].r = r;
d_output[idx].g = g;
d_output[idx].b = b;
d_output[idx].a = a;
'''