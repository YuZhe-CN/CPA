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
#pragma omp parallel for private(d)
    for ( off = 1 ; off < w ; off++ ) {
      d  = distance( w-off, &a[3*(y-1)*w], &a[3*(y*w+off)], dmin );
      d += distance( off, &a[3*(y*w-off)], &a[3*y*w], dmin-d );
      // Update minimum distance and corresponding best offset
      #pragma omp critical
      if ( d < dmin ) { dmin = d; bestoff = off; }
    }
```

En este bucle intervienen diferentes variables que son: `off`, `d`, `dmin` y `bestoff`. De los cuales, la variable `off`
es privada por defecto por la directiva `#pragma omp parallel for`. Pero observando en las operaciones del bucle vemos 
en cada iteración se le asigna nuevo valor a la variale `d`, si no hacemos nada esta variable seria compartida cuando 
paralelizasemos el bucle, de manera que habria condiciones de carrera. La solución de esto es que sea `private`.
La dificultad del bucle viene dada por el `if`, ya que la variable `dmin` debe ser el desplazamiento mínimo de entre todos. 
Esto se podría solucionarse mediante `reduction(min: dmin)`, sin embargo la variable `bestoff` daría problemas debido a
que sólo se actualiza cuando `d < dmin`. Si no hiciesemos nada, `bestoff` sería compartidad y daría problemas de condición 
de carrera, pero tampoco tendría sentido hacerlo `reduction`, ya que no sabemos cómo queremos que sea el valor final de
ella. Como solución definitiva, nos hemos decantado por hacer que todo el if sea una sección crítica, resuelve tanto la 
actualización de `dmin` como la de `bestoff`, de esta manera, aunque sean compartidas, no hay condición de carrera y 
aseguramos que tengan los valores que deben tener.

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
**Bucle parte 3**
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
En este caso, para paralelizar el bucle hemos tenido que un bloque ya que cada hilo debe tener una copia privada del 
vector `v`, `#pragma omp parallel private(v)`. Después distribuir las iteraciones entre los hilos mediante 
`#pragma omp for`.

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
para el mismo. 