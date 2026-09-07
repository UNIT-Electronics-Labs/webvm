# CH552 con SDCC

La imagen `debian_sdcc_ch552` usa el SDK de UNIT Electronics MX:
`UNIT-Electronics-MX/unit_ch55x_sdk`. El SDK queda en `/home/user/ch552` y la
imagen arranca en `/home/user/ch552/examples/blink`.

Para compilar el ejemplo `blink` incluido:

```sh
make
```

También están disponibles:

```sh
make help
make hex
make bin
make flash
```

Los archivos finales se generan en `build/main.hex` y `build/main.bin`. Para
compilar otro ejemplo:

```sh
cd ../adc
make
```

`make flash` requiere que el CH552 esté conectado al equipo que ejecuta el
programador; la imagen WebVM se encarga de compilar, pero no de pasar USB al
microcontrolador.

Para crear un ejemplo nuevo, usa el `Makefile` y los headers de
`/home/user/ch552/include`. El objetivo de SDCC es `-mmcs51`; el SDK ya define
las opciones de memoria y el linker para el CH55x.
