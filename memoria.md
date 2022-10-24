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
