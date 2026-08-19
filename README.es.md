# uRAD Automotive SDK

**SDK oficial del radar Automotive de [uRAD](https://urad.es) (Anteral)** —
placa de evaluación mmWave de 77 GHz basada en el **AWR1843AoP** de Texas
Instruments (antena en el encapsulado).

*Read this in [English](README.md).*

## Estructura del repositorio

| Directorio | Contenido |
|---|---|
| [`docs/`](docs) | Manual de usuario y guía del adaptador para Raspberry Pi (EN/ES) |
| [`mechanical/`](mechanical) | Modelo 3D de la placa (STEP) |
| [`firmware/`](firmware) | Guía de flasheo; los binarios están en [Releases](../../releases) |
| [`applications/`](applications) | Aplicaciones del producto (level sensing) |

## Inicio rápido (demo out-of-box)

1. Flashea el firmware out-of-box (`out_of_box_1843_aop.bin`, en
   [Releases](../../releases)) — véase [`firmware/README.md`](firmware/README.md).
2. Instala el SDK Python [urad-mmwave](https://github.com/urad-by-Anteral/urad-mmwave-core):

   ```bash
   pip install git+https://github.com/urad-by-Anteral/urad-mmwave-core.git
   ```

3. Ejecuta la demo con el perfil de este producto (identifica antes tus
   puertos COM):

   ```bash
   urad-mmwave --config profiles/automotive/config_radar.json --data-port COM7 --control-port COM8
   ```

   Añade `--gui` para el visor de nube de puntos en tiempo real. La
   referencia completa de configuración está en el README de
   [urad-mmwave-core](https://github.com/urad-by-Anteral/urad-mmwave-core).

## Aplicaciones

### Level Sensing de alta precisión

Medida de distancia con precisión milimétrica (12–150 m) con firmware
dedicado. El cliente Python forma parte del SDK común:

```bash
urad-level-sensing --model AWR --control-port COM8 --data-port COM7 --max-distance 12
```

Las implementaciones de referencia en C++ y Arduino, junto con las notas de
aplicación e informes de rendimiento, están en
[`applications/level_sensing/`](applications/level_sensing).

## Recursos de Texas Instruments

La documentación de TI que antes acompañaba a este SDK está disponible en TI:
la guía del [mmWave SDK](https://www.ti.com/tool/MMWAVE-SDK) (incluido el
formato de datos UART de la demo out-of-box) y el
[TI Resource Explorer](https://dev.ti.com).

## Licencia

El código y la documentación de Anteral se publican bajo licencia
[MIT](LICENSE). El firmware y la documentación de Texas Instruments siguen
sujetos a sus respectivas licencias de TI.
