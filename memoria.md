# Memoria Practica 2

## Ejercicio 1
En este ejercicio debemos paralelizar el bucle interno de la primer parte. 

**Bucle interno de la parte 1**
````c
for ( off = 1 ; off < w ; off++ ) {
      d  = distance( w-off, &a[3*(y-1)*w], &a[3*(y*w+off)], dmin );
      d += distance( off, &a[3*(y*w-off)], &a[3*y*w], dmin-d );
      // Update minimum distance and corresponding best offset
      if ( d < dmin ) { dmin = d; bestoff = off; }
    }
````

```c
#pragma omp parallel for private(d, id_thread) lastprivate(num_threads)
for ( off = 1 ; off < w ; off++ ) {
    d  = distance( w-off, &a[3*(y-1)*w], &a[3*(y*w+off)], dmin );
    d += distance( off, &a[3*(y*w-off)], &a[3*y*w], dmin-d );
    num_threads = omp_get_num_threads();
// Update minimum distance and corresponding best offset
    if(d < dmin) {
        #pragma omp critical
        if (d < dmin) {
            dmin = d;
            bestoff = off;
        }
    }
}
voff[y] = bestoff;
```

En este bucle intervienen diferentes variables que son: `off`, `d`, `dmin` y `bestoff`. De los cuales, la variable `off`
es privada por defecto por la directiva `#pragma omp parallel for`. Pero observando en las operaciones del bucle vemos 
en cada iteración se le asigna nuevo valor a la variale `d`, si no hacemos nada esta variable seria compartida cuando 
paralelizasemos el bucle, de manera que habria condiciones de carrera. La solución de esto es que sea `private`.
La dificultad del bucle viene dada por el `if`, ya que la variable `dmin` debe ser el desplazamiento mínimo de entre todos. 
Esto se podría solucionarse mediante `reduction(min: dmin)`, sin embargo la variable `bestoff` daría problemas debido a
que sólo se actualiza cuando `d < dmin`. Si no hiciesemos nada, `bestoff` sería compartidad y daría problemas de condición 
de carrera, pero tampoco tendría sentido hacerlo `reduction`, ya que no sabemos cómo queremos que sea el valor final de
ella. Como solución definitiva, nos hemos decantado por hacer que todo el `if` este dentro de otro `if` que comprueba la
la mismma condicion y sea la interna una sección crítica, resuelve tanto la actualización de `dmin` como la de `bestoff`, de esta manera,aunque sean compartidas, no hay condición de carrera y 
aseguramos que tengan los valores que deben tener.

**Bucle parte 3**

````c
v = malloc(3 * max * sizeof(Byte));
      if (v == NULL)
            fprintf(stderr, "ERROR: Not enough memory for v\n");
      else {
          #pragma omp for
          for (y = 1; y < h; y++) {
                cyclic_shift(w, &a[3 * y * w], voff[y], v);
          }
          free(v);
      }
````

````c
  #pragma omp parallel private(v)
  {
      v = malloc(3 * max * sizeof(Byte));
      if (v == NULL)
            fprintf(stderr, "ERROR: Not enough memory for v\n");
      else {
          #pragma omp for
          for (y = 1; y < h; y++) {
                cyclic_shift(w, &a[3 * y * w], voff[y], v);
          }
          free(v);
      }
  }
````

En este caso, para paralelizar el bucle hemos tenido que encerrarlo todo en un bloque, ya que cada hilo debe tener una copia privada del 
vector `v`, `#pragma omp parallel private(v)`. Después distribuir las iteraciones entre los hilos mediante 
`#pragma omp for` de esta manera cada hilo realizara una serie de iteraciones.

---
## Ejercicio 2

````c
void realign( int w,int h,Byte a[] ) {
  int y, off,bestoff,dmin,max, d, *voff;
  Byte *v;
  int num_threads = 0, id_thread = 0;

  voff = malloc( h * sizeof(int) );
  if ( voff == NULL ) {
    fprintf(stderr,"ERROR: Not enough memory for voff\n");
    return;
  }
  #pragma omp parallel private(v)
  {
    num_threads = omp_get_num_threads();
    // Part 1. Find optimal offset of each line with respect to the previous line
    #pragma omp for private(off, d, dmin, bestoff)
    for (y = 1; y < h; y++) {
        // Find offset of line y that produces the minimum distance between lines y and y-1
        dmin = distance(w, &a[3 * (y - 1) * w], &a[3 * y * w], INT_MAX); // offset=0
        bestoff = 0;

        for (off = 1; off < w; off++) {
            d = distance(w - off, &a[3 * (y - 1) * w], &a[3 * (y * w + off)], dmin);
            d += distance(off, &a[3 * (y * w - off)], &a[3 * y * w], dmin - d);

            // Update minimum distance and corresponding best offset

            if (d < dmin) {
                dmin = d;
                bestoff = off;
            }
        }

        voff[y] = bestoff;
    }

    // Part 2. Convert offsets from relative to absolute and find maximum offset of any line
    #pragma omp single
    {
        max = 0;
        voff[0] = 0;
        for (y = 1; y < h; y++) {
            voff[y] = (voff[y - 1] + voff[y]) % w;
            d = voff[y] <= w / 2 ? voff[y] : w - voff[y];
            if (d > max) max = d;
        }
    }
    // Part 3. Shift each line to its place, using auxiliary buffer v

    v = malloc(3 * max * sizeof(Byte));
    if (v == NULL)
        fprintf(stderr, "ERROR: Not enough memory for v\n");
    else {
        #pragma omp for
        for (y = 1; y < h; y++) {
            cyclic_shift(w, &a[3 * y * w], voff[y], v);
        }
        free(v);
    }
  }
  printf("Numero de hilos usados: %d\n", num_threads);
  free(voff);
}
````
En esta parte debemos cerrar los tres bucles en un bloque mediante `#pragam omp parallel {...}`. Como el segundo bucle 
no se puede paralelizar, una solución que hemos pensado es ponerlo dentro de un bloque `#pragma omp single`, de esta 
manera solo un hilo ejecutará esa parte y los demás esperarán a que termine, ya que la tercera parte depende de esta 
segunda. En cuanto al bucle externo de la parte 1, usamos la directiva `#pragma omp for private(off, d, dmin, bestoff)` 
para distribuir las iteraciones entre los hilos. Como cada hilo tendrá que calcular el desplazamiento óptimo de las 
lineas que les ha tocado, debemos hacer private las variables `off, d, dmin, bestoff` para que cada hilo tenga una copia
para el mismo. El bucle de la tercera parte es casi igual que el ejercicio anterior, solo que hemos declarado private `v`
al principio de todo el bloque paralelo.

---

## Ejercicio 6
### Paralelización de distance

#### Versión a: d compartido
````c
int distance( int n, Byte a1[], Byte a2[], int c ) {
  int i, d,e;
  n *= 3; // 3 bytes per pixel (red, green, blue)
  d = 0;
  #pragma omp parallel private(i, e)
  {
      int hilo = omp_get_thread_num();
      int nhilos = omp_get_num_threads();
      for ( i = hilo ; i < n && d < c ; i+=nhilos) {
          e = (int)a1[i] - a2[i];
          if ( e >= 0 ) {
              #pragma omp atomic
              d += e;
          }
          else {
              #pragma omp atomic
              d -= e;
          }

      }

  }

  return d;
}
````
Para paralelizar este método, la dificultad esta principalmente en la guarda del bucle for. Al existir una
condición extra, `d < c`, esto hace que no se pueda paralelizar de forma habitual. Como solución a este problema,
es necesario repartir manualmente las iteraciones del bucle entre los hilos. Es por eso que se declara dos variables
privadas para cada hilo, `hilo` es el identificador del hilo y `nhilo`es el número de hilos. En cada iteración se
modifican las variables `i` y `e`, es por eso que necesitamos declararlos como privadas para que cada hilo tenga
su propia copia. En cuanto a la variable `d` al ser compartida, la actualización de su valor por cada hilo debe 
hacerse en exclusión mútua. Por este motivo, usamos la directiva `#praga omp atomic` para la expresión `d+=e`. 

#### Versión b: d privado
````c
int distance( int n, Byte a1[], Byte a2[], int c ) {
  int i, d,e;
  n *= 3; // 3 bytes per pixel (red, green, blue)
  d = 0;
  #pragma omp parallel private(i, e) reduction(+:d)
  {
      int hilo = omp_get_thread_num();
      int nhilos = omp_get_num_threads();
      for ( i = hilo ; i < n && d < c ; i+=nhilos) {
          e = (int)a1[i] - a2[i];
          if ( e >= 0 ) {

              d += e;
          }
          else {

              d -= e;
          }

      }

  }

  return d;
}
````
La diferencia principal comparado con la versión anterior es que en este caso la variable d es privada, es
decir, para cada hilo existirá una copia. Después de que todos los hilos terminen sus iteraciones, para conseguir
el valor verdadero de d, es necesario sumar todos esas copias de los hilos. Para conseguir este resultado se ha 
utilizado la `reduction(+:d)`.

### Paralelización de cyclic_shift

````c
void cyclic_shift( int n, Byte a[], int p, Byte v[] ) {
  int i,d;

  if ( p != 0 ) {
    n *= 3; p *= 3;
    d = n - p;
    // Shift is done from right to left of from left to right
    // depending on which alternative requires less space in the auxiliary
    // array v
    if ( p <= n / 2 ) { // right to left
        #pragma omp parallel
        {
            #pragma omp for
            for (i = 0; i < p; i++) v[i] = a[i];
            #pragma omp single
            for (i = p; i < n; i++) a[i - p] = a[i];
            #pragma omp for
            for (i = 0; i < p; i++) a[d + i] = v[i];
        }
    } else { // left to right
        #pragma omp parallel
        {
            #pragma omp for
            for (i = 0; i < d; i++) v[i] = a[p + i];
            #pragma omp single
            for (i = p - 1; i >= 0; i--) a[i + d] = a[i];
            #pragma omp for
            for (i = 0; i < d; i++) a[i] = v[i];
        }
    }
  }
}
```` 
Para paralelizar el método se ha definido dos regiones paralelas. Una para los desplazamientos hacia izquierda, 
mientras que la otra región es para los desplazamientos hacia derecha. En las dos regiones, los primeros bucles y 
los últimos se paralelizan sin dificultad con una directiva `#pragma omp for`. Sin embargo, los bucles del medio 
no se pueden paralelizar debido a la existencia de dependencias entre iteraciones. En los desplazamientos hacia la 
izquierda, si `p < d` habrá dependiencias entre iteraciones. Mientras que en los desplazamientos hacia la derecha, 
si `p > d` habrá dependencias. 

En cuanto a las prestaciones de las dos versiones, es más rapida la versión cuando la variable `d` en la función 
`distance` es privada. La principal razón es que si fuera compartida, los demás hilos tendrían que esperar a que 
el hilo que estuviese ejecutando la expresión `d+=a` termine para que otro pueda ejecutarlo. Esto no sucede cuando 
la variable es privada, ya que cada hilo por separado realizará la actualización y al final de todo se unen todas 
las copias. 