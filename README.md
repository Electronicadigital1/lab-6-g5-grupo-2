[![Open in Visual Studio Code](https://classroom.github.com/assets/open-in-vscode-2e0aaae1b6195c2367325f4f02e2d04e9abb55f0b24a779b69b11b9e10269abc.svg)](https://classroom.github.com/online_ide?assignment_repo_id=24062537&assignment_repo_type=AssignmentRepo)
# Visualización usando pantalla LCD 16x2

# Integrantes

- Jennifer Zulemay Bautista Quiñonez - 1116775152
- Andrés Camilo Bustamante Guzman - 1107976287
- Brayan Steven Galindo Pardo - 1016714138
  
# Informe

Indice:

1. [Diseño implementado](#diseño-implementado)
2. [Simulaciones](#simulaciones)
3. [Implementación](#implementación)
4. [Conclusiones](#conclusiones)
5. [Referencias](#referencias)

## Diseño implementado
### Descripción

El sistema implementado corresponde a un controlador digital para una pantalla LCD de 16 columnas y 2 filas, implementado mediante Verilog sobre una FPGA Altera Cyclone IV. Su función es generar de manera organizada los comandos, caracteres ASCII y señales de control requeridos para inicializar la pantalla y visualizar información.

El diseño original permite mostrar texto estático almacenado en un archivo hexadecimal. Este archivo contiene 32 bytes, correspondientes a los 16 caracteres de la primera fila y los 16 caracteres de la segunda fila. En este caso, los textos almacenados son “Bateria 1” y “Bateria 2”, seguidos por caracteres de espacio para completar la capacidad de cada línea.

El funcionamiento se controla mediante una máquina de estados finitos, FSM, encargada de organizar la secuencia de inicialización, escritura de la primera línea, posicionamiento del cursor y escritura de la segunda línea. Adicionalmente, se utiliza un divisor de frecuencia para reducir la velocidad del reloj de la FPGA y garantizar que los comandos y datos permanezcan estables durante el tiempo requerido por la pantalla.

### Diagramas

##### Instancias y bloques funcionales

El controlador se encuentra compuesto por los siguientes bloques:

###### Divisor de frecuencia:
reduce la frecuencia del reloj principal de la FPGA y genera una señal más lenta utilizada para controlar los tiempos de la pantalla.

###### Justificación del divisor de frecuencia

La FPGA trabaja con un reloj principal de:

```text
f_clk = 50 MHz = 50 000 000 Hz
```

El periodo del reloj de entrada es:

```text
T_clk = 1 / f_clk

T_clk = 1 / 50 000 000

T_clk = 20 ns
```

Para reducir la velocidad de operación de la pantalla LCD se seleccionó:

```text
COUNT_MAX = 800 000
```

El tiempo que tarda la señal `clk_16ms` en cambiar de estado es:

```text
t_cambio = COUNT_MAX / f_clk

t_cambio = 800 000 / 50 000 000

t_cambio = 0.016 s = 16 ms
```

Como la señal cambia de nivel cada 16 ms, necesita dos cambios para completar un periodo:

```text
T_LCD = 2 × t_cambio

T_LCD = 2 × 16 ms

T_LCD = 32 ms
```

La frecuencia resultante es:

```text
f_LCD = 1 / T_LCD

f_LCD = 1 / 0.032

f_LCD = 31.25 Hz
```

Por lo tanto, la señal `enable` cambia de estado cada **16 ms** y tiene una frecuencia aproximada de **31.25 Hz**. Este tiempo permite mantener estables los comandos y caracteres enviados a la LCD, garantizando que sean capturados correctamente en el flanco de bajada de `enable`.

La expresión general para seleccionar el divisor es:

```text
COUNT_MAX = f_clk × t_cambio
```


###### Máquina de estados finitos:
determina la operación que debe ejecutarse en cada instante, incluyendo la configuración, escritura estática, posicionamiento del cursor y actualización de los valores dinámicos.

###### Memoria de configuración:
almacena los comandos utilizados durante la inicialización de la pantalla, correspondientes a 0x38, 0x06, 0x0C y 0x01.

###### Memoria de texto estático:
almacena los 32 caracteres ASCII cargados desde el archivo hexadecimal mediante la instrucción $readmemh.

###### Conversor binario a ASCII: 
transforma cada entrada numérica de 8 bits en tres caracteres decimales correspondientes a centenas, decenas y unidades.

###### Contadores:
permiten recorrer la memoria de comandos, los caracteres estáticos y los dígitos dinámicos.

###### Multiplexor de salida:
selecciona el byte que debe enviarse a la pantalla dependiendo del estado actual de la FSM.

###### Interfaz LCD:
genera las señales rs, rw, enable y data[7:0].

##### Variables inmersas 

Las entradas principales del sistema son:

- clk: reloj principal de la FPGA.
- reset: reinicio activo en nivel bajo.
- ready_i: señal que inicia la configuración y escritura.
- value_1[7:0]: primer valor numérico dinámico.
- value_2[7:0]: segundo valor numérico dinámico.

Las salidas hacia la LCD son:

- rs: selecciona entre comando y dato.
- rw: selecciona lectura o escritura y permanece en cero.
- enable: señal de habilitación de la pantalla.
- data[7:0]: bus que transporta los comandos y caracteres ASCII.

Entre las variables internas se encuentran fsm_state, command_counter, data_counter, static_data_mem y config_mem.

#### Maquina de estados caso estatico

<img width="1448" height="1086" alt="ChatGPT Image 15 jun 2026, 02_48_35 p m" src="https://github.com/user-attachments/assets/8a414037-8b61-402f-a36e-3cfba8b79d3d" />

La máquina de estados finitos correspondiente al funcionamiento estático comienza en el estado IDLE, donde se reinician los contadores de comandos y datos, y el sistema permanece esperando que la señal ready_i tome el valor lógico uno. Cuando se activa esta señal, la FSM pasa al estado CONFIG_CMD1, en el cual se transmiten de manera secuencial los comandos necesarios para inicializar la pantalla LCD, configurarla en modo de 8 bits, habilitar sus dos líneas, definir el desplazamiento del cursor, encender la visualización y limpiar el contenido anterior. Una vez enviados todos los comandos, se ingresa al estado WR_STATIC_TEXT_1L, encargado de recorrer las primeras 16 posiciones de la memoria static_data_mem y enviar los caracteres ASCII correspondientes a la primera fila. Al finalizar esta línea, la máquina pasa a CONFIG_CMD2, donde se envía el comando hexadecimal 0xC0 para posicionar el cursor al inicio de la segunda fila. Posteriormente, en el estado WR_STATIC_TEXT_2L, se transmiten los 16 caracteres restantes almacenados en la memoria para completar la segunda línea. Cuando termina la escritura, la máquina regresa al estado IDLE, quedando preparada para repetir el proceso cuando se active nuevamente la señal de inicio.


#### Maquina de estados caso dinamico

<img width="1448" height="1086" alt="ChatGPT Image 15 jun 2026, 02_48_40 p m" src="https://github.com/user-attachments/assets/8451e002-6fe9-4ac8-adff-0ebbe9688e8a" />

La máquina de estados del controlador dinámico inicia igualmente en IDLE, donde se ponen en cero los contadores y se espera la activación de ready_i. Después, en CONFIG_LCD, se envían los comandos de inicialización de la pantalla. Los estados WRITE_STATIC_L1, SET_LINE_2 y WRITE_STATIC_L2 se encargan de escribir inicialmente los textos fijos de la primera y segunda fila, utilizando el contenido almacenado en static_data_mem. Una vez presentado el texto estático, la FSM entra en la etapa de actualización dinámica. En SET_DYNAMIC_L1 se envía el comando 0x8A para posicionar el cursor en la columna seleccionada de la primera fila, y en WRITE_DYNAMIC_L1 se convierte el primer valor binario de 8 bits en tres caracteres ASCII correspondientes a centenas, decenas y unidades, los cuales son transmitidos a la pantalla. Después, el estado SET_DYNAMIC_L2 envía el comando 0xCA para ubicar el cursor en la misma columna de la segunda fila, mientras que WRITE_DYNAMIC_L2 escribe los tres caracteres asociados al segundo valor numérico. Al finalizar la escritura del segundo dato, la máquina regresa a SET_DYNAMIC_L1, formando un ciclo continuo que permite actualizar ambos valores sin borrar ni volver a escribir todo el texto estático de la pantalla. En cualquier momento, si la señal reset se lleva a nivel bajo, la FSM regresa inmediatamente al estado IDLE.


## Simulaciones 

<!-- (Incluir las de Digital si hicieron uso de esta herramienta, pero también deben incluir simulaciones realizadas usando un simulador HDL como por ejemplo Icarus Verilog + GTKwave) -->


## Implementación


## Conclusiones


## Referencias

