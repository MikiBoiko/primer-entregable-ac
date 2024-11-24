# Primer entregable de la asignatura AC

Miguel Montero Boiko

## El caso: procesador escalar con planificación dinámica y el código ensamblador que se ejecuta

### **Características del procesador**

1. **Procesador y planificación**:  
   - Se trata de un procesador **escalar**.  
   - Aplica **planificación dinámica** basada en el algoritmo de **Tomasulo con especulación**.  

2. **Unidades funcionales y sus latencias**:

| **Tipo de unidad**    | **Segmentada** | **Latencia** | **Estaciones de reserva** |  
|-----------------------|----------------|--------------|---------------------------|  
| **Suma CF**           | Sí             | 4 ciclos     | 1                         |  
| **Multiplicación CF** | Sí             | 6 ciclos     | 1                         |  
| **División CF**       | No             | 12 ciclos    | 1                         |  
| **Enteros**           | No aplica      | 1 ciclo      | 2                         |

3. **Reorder Buffer (ROB)**:  
   - El procesador tiene un ROB con **5 entradas**.  

4. **Load buffers y acceso a memoria**:  
   - El procesador dispone de **2 load buffers**.  
   - El acceso a memoria (lectura y escritura) se realiza a través de una unidad:  
     - Es **segmentada**.  
     - Latencia de **2 ciclos**.  

5. **Restricción adicional**:  
   - No es posible escribir un dato desde el único **Bus de Datos Común (BCD)** e iniciar en el mismo ciclo una operación que dependa de ese dato.

### **Código de alto nivel**

``` c
for (i=0; i<100; i++)
    A[i] = A[i]/B[i] + C[i]*D[i] + A[i]*C[i]
```

### **Código ensamblador de RISC-V**

``` risc-v
    addi x11, x0, 100
L1:	flw f0, 0(x7)
    flw f2, 0(x8)
    fdiv.d f4, f0, f2		
    flw f6, 0(x9)
    flw f8, 0(x10)
    fmul.d f10, f6, f8		
    fadd.d f10, f4, f10		
    fmul.d f4, f0, f6		
    fadd.d f10, f4, f10		 
    fsd f10, 0(x7)
    addi x7, x7, 8
    addi x8, x8, 8
    addi x9, x9, 8
    addi x10, x10, 8
    addi x11, x11, -1
    bne x11, x0, L1
```

## **Terminología y conceptos con los que trabajaremos**

### **Terminología y acrónimos**  

- **RAW/LDE (Read After Write / Lectura Después de Escritura)**: Dependencia donde una instrucción necesita leer un valor que aún no ha sido escrito por otra.  

- **WAW/EDE (Write After Write / Escritura Después de Escritura)**: Conflicto cuando dos instrucciones escriben en el mismo registro y deben respetar el orden de escritura.  

- **WAR/EDL (Write After Read / Escritura Después de Lectura)**: Conflicto cuando una instrucción escribe en un registro que otra aún necesita leer.  

- **ROB (Reorder Buffer)**: Estructura que garantiza la **finalización en orden** de las instrucciones en procesadores con ejecución fuera de orden.  

- **BCD (Bus de Datos Común)**: Canal único que permite a las unidades funcionales escribir resultados y distribuirlos a otras instrucciones o registros.  

- **Estaciones de Reserva**: Buffers que almacenan instrucciones y sus operandos mientras esperan su turno en las unidades funcionales.  

- **Unidades segmentadas**: Componentes divididos en varias etapas, permitiendo procesar varias instrucciones simultáneamente.  

- **Dependencias de datos**: Restricciones entre instrucciones debido a lecturas/escrituras en registros o memoria.  

- **Stalls**: Ciclos vacíos en el pipeline causados por dependencias o conflictos de recursos.

### **Tomasulo con especulación**

#### **Desglose de las instrucciones por unidad funcional**

##### **1. Unidad de Enteros**  
| **Instrucción**          | **Explicación** |
|--------------------------|-----------------|
| `addi x_dest, x_src, imm` | Realiza una suma entre el contenido del registro fuente `x_src` y un valor constante (inmediato) `imm`. El resultado se almacena en el registro destino `x_dest`. |
| `bne x1, x2, label`       | Compara los registros `x1` y `x2`. Si sus valores son distintos, salta a la instrucción indicada por `label`. |

---

##### **2. Unidad de Memoria Segmentada**  
| **Instrucción**          | **Explicación** |
|--------------------------|-----------------|
| `flw f_dest, offset(x)`   | Carga un valor de coma flotante desde la dirección de memoria calculada como `offset + x` al registro `f_dest`. |
| `fsd f_src, offset(x)`    | Almacena el valor del registro de coma flotante `f_src` en la dirección de memoria calculada como `offset + x`. |

---

##### **3. Unidades de Punto Flotante (suma, multiplicación, división)**  
| **Instrucción**          | **Explicación** |
|--------------------------|-----------------|
| `fadd.d f_dest, f_src1, f_src2` | Suma los valores de los registros de coma flotante `f_src1` y `f_src2`. Almacena el resultado en `f_dest`. |
| `fmul.d f_dest, f_src1, f_src2` | Multiplica los valores de los registros de coma flotante `f_src1` y `f_src2`. Almacena el resultado en `f_dest`. |
| `fdiv.d f_dest, f_src1, f_src2` | Divide el valor del registro `f_src1` entre el valor del registro `f_src2`. Almacena el resultado en `f_dest`. |

#### **Etapas del algoritmo**

1. **Issue**: La instrucción se emite si hay espacio en la ROB y en la estación de reserva.
1. **Ejecución**: Cuado se inicializa los operandos están listos y la unidad funcional correspondiente está libre.
1. **Escritura de resultados**: Se emite el resultado a la **BCD** de que la instrucción está lista.
1. **Commit**: El resultado se escribe en el destino final: registro o memoria según lo programado.

#### **Diagrama del procesador propuesto en este caso**

![SVG TOMASULO CASO](tomasulo-caso.svg)

Utilizaré este esquema para instanciar diferentes ciclos de del procesador a medida que completo la tabla. He de anotar que todos los registros `xi` han sido ignorados por brevedad.

## **Enunciados del entregable**

### **(Apartado A) Tomasulo - primera iteración del bucle**

|       | Instrucción            | Issue | Ejecución  | Escritura | Commit |
|-------|------------------------|-------|------------|-----------|--------|
|       | `addi x11, x0, 100`    |     1 |          1 |         2 |      3 |
| `L1:` | `L1: flw f0, 0(x7)`    |     2 |          2 |         4 |      5 |
|       | `flw f2, 0(x8)`        |     3 |          3 |         5 |      6 |
|       | `fdiv.d f4, f0, f2`    |     4 |          6 |        18 |     19 |
|       | `flw f6, 0(x9)`        |     5 |          5 |         7 |      8 |
|       | `flw f8, 0(x10)`       |     6 |          6 |         8 |      9 |
|       | `fmul.d f10, f6, f8`   |     7 |          9 |        15 |     16 |
|       | `fadd.d f10, f4, f10`  |     8 |         19 |        23 |     24 |
|       | `fmul.d f4, f0, f6`    |     9 |         19 |        25 |     26 |
|       | `fadd.d f10, f4, f10`  |    10 |         26 |        30 |     31 |
|       | `fsd f10, 0(x7)`       |    16 |         31 |        33 |     34 |
|       | `addi x7, x7, 8`       |    19 |         34 |        35 |     36 |
|       | `addi x8, x8, 8`       |    24 |         24 |        25 |     26 |
|       | `addi x9, x9, 8`       |    25 |         25 |        26 |     27 |
|       | `addi x10, x10, 8`     |    26 |         26 |        27 |     28 |
|       | `addi x11, x11, -1`    |    27 |         27 |        28 |     29 |
|       | `bne x11, x0, L1`      |    28 |         29 |         - |      - |

---

### **(Apartado B) Registros y ROB - estado justo antes del ciclo 22**

#### **Registros**

| Registro  | `f0` | `f2` | `f4`      | `f6` | `f8` | `f10`     |
|-----------|------|------|-----------|------|------|-----------|
| **TAG**   |    2 |    3 |         1 |    2 |    3 |         4 |
| **Valor** | A[i] | B[i] | A[i]/B[i] | C[i] | D[i] | C[i]*D[i] |

Que será el mismo estado que en la última transformación del ciclo 19, ya que se están esperando todas las instrucciones:

![SVG TOMASULO CICLOS](tomasulo-ciclos.svg)

#### **ROB**

| **Entrada** | **Destino** | **Valor** |  **Tipo**           | **Ready** |
|-------------|-------------|-----------|---------------------|-----------|
| **1**       | x7          | <waiting> | addi x7, x7, 8      | N         |
| **2**       | f10         | <waiting> | fadd.d f10, f4, f10 | N         |
| **3**       | f4          | <waiting> | fmul.d f4, f0, f6   | N         |
| **4**       | 0(x7)       |         - | fsd f10, 0(x7)      | N         |
| **5**       | f10         | <waiting> | fadd.d f10, f4, f10 | N         |

---

### **(Apartado C) Preguntas breves - variaciones y comportamientos del procesador**

1. #### ¿Qué cambiaría en la tabla desarrollada en el **Apartado A** si la unidad funcional de división fuese segmentada?

En nuestro caso, como solo se realiza una división, no afectaría a los tiempos. Si se realizase más de una iteración, imagino que veríamos una gran diferencia ya que se podrían realizar la segmentación que permitiera realizar una división detras de otra, que aunque con latencia, nos dejaría llegar a un CPI de 1 en cuestión de la división.

2. #### ¿Y si en lugar de una unidad funcional de multiplicación en punto flotante tuviésemos dos (manteniendo una estación de reserva por cada unidad funcional)?

Si estuviera correctamente desarrollado permitiría realizar un issue y ejecución al mismo tiempo de la sección:

``` risc-v
fmul.d f10, f6, f8
fadd.d f10, f4, f10
fmul.d f4, f0, f6
```

3. #### ¿Y si el acceso a memoria fuese no segmentado?

Habría muchos más stalls esperando a cada lectura y escritura de memoria, sobretodo al principio de cada bucle (cuado se obtiene o se escribe `A[i]`, `B[i]`, `C[i]` y `D[i]`, es decir, cada instrucción `flw` y `fsd`).

4. #### ¿Qué sucede exactamente en el ROB cuando la instrucción `fadd.d f10, f4, f10` realiza su fase de escritura?

Indicará al siguiente ciclo que se escriba en el banco de reegistros y se emitirá el resultado a la BCD de que la instrucción está lista, en este caso 

5. #### ¿Cómo se evita la dependencia de escritura después de escritura entre las dos instrucciones consecutivas que tienen como registro destino `f10`?

Como la instrucción ha sido escrita con la dependencia a un resultado ROB, al completarse la instrucción previa, se le sobreescribirá en fase de escritura por la `CDB` directamente el valor resultado (aunque se ejecute en el siguiente ciclo).

6. #### ¿Qué sucedería si tuviésemos un predictor de saltos que para el salto de la primera iteración predijese el salto como tomado? ¿Y si la predicción fuese no tomado?

| **Perdicción**            | **Correcta**                               | **Incorrecta**                                   |
|---------------------------|--------------------------------------------|--------------------------------------------------|
|**Tomado**                 | Se ejecuta el salto a `L1`                 | Se borra toda la ROB y los tags de registro      |
| **No tomado**             | Se ejecutan las instrucciones secuenciales | Se descartan las instrucciones y se salta a `L1` |

---

### **(Apartado D) Mismas características de procesador excepto que es superescalar de dos de ancho y con 8 entradas en la ROB - primera iteración del bucle**

Podemos hacer issue the dos instrucciones a la vez siempre que no tenga conflictos (realmente es lo que he estado haciendo en el apartado anterior porque pensaba que era superescalar de ancho infinito... y he tenido que rehacerlo todo de nuevo...).

Por carencia de tiempo he hecho el cálculo de este apartado con la cabeza. La metodología ha sido coger instrucciones de dos en dos siempre que no haya dependencias y haya entradas RE y ROB libres para la operación. Luego es ver si se puede ejecutar y las dependencias están resueltas entre las operaciones, finalmente simplemente se hace la escritura y se libera en el commit (resolviendo cualquier dependencia).

|       | Instrucción            | Issue           | Ejecución          | Escritura | Commit |
|-------|------------------------|-----------------|--------------------|-----------|--------|
|       | `addi x11, x0, 100`    |               1 |                  1 |         2 |      3 |
| `L1:` | `flw f0, 0(x7)`        |               1 |                  1 |         3 |      4 |
|       | `flw f2, 0(x8)`        |               2 |                  2 |         4 |      5 |
|       | `fdiv.d f4, f0, f2`    |               2 |                  5 |        17 |     18 |
|       | `flw f6, 0(x9)`        |               3 |                  3 |         5 |      6 |
|       | `flw f8, 0(x10)`       |               3 |                  4 |         6 |      7 |
|       | `fmul.d f10, f6, f8`   |               4 |                  7 |        13 |     14 |
|       | `fadd.d f10, f4, f10`  |               4 |                 18 |        22 |     23 |
|       | `fmul.d f4, f0, f6`    |               5 |                 18 |        24 |     25 |
|       | `fadd.d f10, f4, f10`  |               5 |                 25 |        29 |     30 |
|       | `fsd f10, 0(x7)`       |               6 |                 30 |        32 |     33 |
|       | `addi x7, x7, 8`       |               6 |                 33 |        35 |     36 |
|       | `addi x8, x8, 8`       |               7 |                  7 |         8 |      9 |
|       | `addi x9, x9, 8`       |               7 |                  7 |         8 |      9 |
|       | `addi x10, x10, 8`     |               8 |                  8 |         9 |     10 |
|       | `addi x11, x11, -1`    |               8 |                  8 |         9 |     10 |
|       | `bne x11, x0, L1`      |               9 |                 10 |         - |      - |

Se puede observar como los issues se ejecutan pronto pero por tiempos largos de latencia, igualmente tenemos que esperar practicamente lo mismo por los resultados finales de este caso en particular. Es verdad que la iteración siguiente podría ser planificación antes, sin embargo haber subido tres entradas no nos salvará del eventual cuello de botella con estas especificaciones.
