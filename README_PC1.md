Informe PC1

Curso: Algoritmos y Estructuras de Datos
Proyecto base:VideoClipMultiImagenThreads_BaseEstudiantes
Equipo:Juarez Garcia Jorge Alejandro


1. Descripción del proyecto

El proyecto genera un clip de video en formato MP4 a partir de:

- Un archivo de audio en formato WAV (`HONDA.wav`), cuyo volumen controla la animación.
- Un conjunto de 10 imágenes de un Honda Civic en carretera (`H1.png` a `H10.png`), ubicadas en `input/images/`.

El programa recorre el audio en bloques, calcula el nivel de volumen de cada bloque, y en base a ese nivel decide qué imagen mostrar y qué efecto visual aplicar en cada frame del video. El proceso se ejecuta primero en **modo serial** (un solo hilo) y luego en **modo paralelo** (varios hilos trabajando al mismo tiempo), para comparar el tiempo que toma cada uno.

2. Cambios realizados en `StudentWork.java`

TODO 1. `calculateAudioLevel`

Se implementó el recorrido del arreglo `short[]` correspondiente al bloque de audio del frame. Se calcula el promedio del valor absoluto de las muestras (`Math.abs(samples[i])`) entre las posiciones `start` y `end`, y se normaliza dividiendo entre 32768 (el valor máximo que puede tomar un `short`), de forma que el resultado siempre quede entre 0.0 y 1.0.

java
long suma = 0;
for (int i = inicio; i < fin; i++) {
    suma += Math.abs((int) samples[i]);
}
double promedio = (double) suma / cantidad;
double nivel = promedio / 32768.0;

TODO 2. `chooseImageIndex`

Se combinó el nivel de audio con el número de frame para elegir la imagen: el volumen determina un "grupo" de imagen (`level * imageCount`), y el número de frame agrega un desplazamiento dentro de ese grupo. Así, la imagen no depende únicamente del volumen ni únicamente del tiempo, sino de ambos a la vez, tal como pedía la consigna.

java
int grupo = (int) Math.floor(level * imageCount);
int desplazamiento = frameNumber % imageCount;
int indice = (grupo + desplazamiento) % imageCount;

TODO 3. `applyEffects`

Se aplican dos tipos de transformación sobre cada frame:

- Transformación geométrica: una rotación suave y oscilante (`base.rotate(angle)`), calculada con una función seno sobre el número de frame.
- Filtro por matriz/convolución: según el nivel de audio, se elige entre 4 estados visuales distintos:
  - Volumen bajo (`< 0.25`): `brighten(0.8)` — oscurece la imagen.
  - Volumen medio-bajo (`< 0.50`): `blur()` — difumina la imagen (convolución 3×3).
  - Volumen medio-alto (`< 0.75`): `sharpen()` — realza bordes (convolución 3×3).
  - Volumen alto (`≥ 0.75`): `sobel()` — detecta bordes, simulando un efecto de "impacto" o velocidad.

Esto cumple con el mínimo pedido: 1 transformación geométrica, 1 filtro de convolución, y 3 (en este caso 4) estados visuales diferentes.

3. Imágenes y audio utilizados

- Imágenes: 10 fotografías de un Honda Civic en carretera (`H1.png` … `H10.png`), colocadas en `input/images/`.
- Audio: archivo `HONDA.wav`, colocado en `input/audio/`.
- Configuración usada (`config.properties`): `fps=12`, `durationSeconds=10` → 120 frames en total, cumpliendo el mínimo de 96 frames y la duración de 8–12 segundos pedida.

4. Resultados: serial vs. paralelo

| Modo               | Tiempo (s) | Speedup | Eficiencia |
|---------------------|-----------:|--------:|-----------:|
| Serial               | 7.451      | 1.00x   | —          |
| Paralelo (2 threads) | 5.115      | 1.439x  | 71.94%     |
| Paralelo (4 threads) | 3.859      | 1.931x  | 48.27%     |

Observaciones

- El modo paralelo con 4 threads fue **1.93 veces más rápido** que el serial, pero no llegó a ser 4 veces más rápido (que sería el caso ideal). La eficiencia de 48.27% muestra que no todo el trabajo se aprovecha perfectamente al repartirlo entre hilos.
- Con 2 threads, el speedup fue de **1.44x**, con una eficiencia de **71.94%** — más alta que con 4 threads. Esto es un patrón típico: al usar menos hilos, cada uno tiene más trabajo real que hacer en proporción al overhead de crear y coordinar hilos, por lo que se aprovecha mejor cada uno. Al aumentar a 4 threads se gana velocidad absoluta (más rápido en segundos), pero se pierde eficiencia relativa (no se duplica la ganancia).
- Esto ocurre porque hay una parte del trabajo que no se puede paralelizar (por ejemplo, leer el audio una sola vez al inicio), y porque crear y coordinar varios hilos tiene un costo (overhead) que reduce la ganancia total. Ese costo pesa más cuantos más hilos se usan.
- **Conclusión:** más threads no siempre es mejor de forma proporcional — hay que balancear velocidad total contra eficiencia por hilo, y ese balance depende de cuántos núcleos reales tiene el procesador usado.

5. Preguntas de la defensa

1. ¿Qué representa `short[]`?
Es el arreglo que contiene las muestras de audio leídas del archivo WAV. Cada `short` es un número entre -32768 y 32767 que representa la amplitud de la onda de sonido en un instante muy específico del tiempo.

2. ¿Qué representa `int[][]`?
Es la matriz que representa una imagen en escala de grises: cada posición `[fila][columna]` guarda el valor de intensidad (de 0 a 255) del píxel correspondiente.

3. ¿Cómo se elige una imagen para cada frame?
Se combina el nivel de volumen del audio en ese frame con el número de frame: el volumen decide un grupo de imagen, y el número de frame agrega variación dentro de ese grupo, de modo que la imagen cambia tanto por el sonido como por el paso del tiempo.

4. ¿Por qué distintos frames pueden ejecutarse en distintos threads?
Porque el cálculo de cada frame (leer su bloque de audio, elegir imagen, aplicar efectos) es independiente del cálculo de los demás frames — ningún frame necesita el resultado de otro para poder generarse. Eso permite repartir los frames entre varios hilos sin que se interfieran entre sí.

5. ¿Qué cambiaría si dos threads escribieran el mismo archivo?
Podría producirse una condición de carrera (race condition): ambos threads intentarían escribir al mismo tiempo, lo que podría corromper el archivo, provocar que uno sobrescriba el trabajo del otro, o generar errores de acceso al archivo. Por eso cada thread en este proyecto escribe su propio archivo de frame (`frame_XXX.png`), evitando ese conflicto.

6. ¿Cuál fue su speedup?
1.931x con 4 threads (tiempo serial de 7.451 s contra 3.859 s en paralelo).

7. ¿Más threads siempre significa más velocidad?
No necesariamente. A partir de cierto número de threads, el beneficio empieza a disminuir porque: (a) la cantidad de núcleos físicos del procesador es limitada, (b) crear y coordinar más hilos tiene un costo adicional (overhead), y (c) puede haber partes del programa que no se benefician de la paralelización. Por eso la eficiencia (48.27% en este caso) baja a medida que se agregan más threads, en vez de mantenerse en 100%.

8. Cambios en `StudentWork.java`
Ver sección 2 de este informe para el detalle de los 3 TODOs implementados.

6. Estructura de entrega

├── proyecto/
├── videoclip_serial.mp4
├── videoclip_parallel.mp4
├── output/metrics.csv
├── README_EQUIPO.md
└── capturas/
```
