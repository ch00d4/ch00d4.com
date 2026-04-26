---
pubDate: 2026-04-26
description: "Guía técnica sobre la implementación de Snapper para la gestión de snapshots Btrfs en Fedora, permitiendo la creación de puntos de restauración del sistema."
slug: "guia-snapper-btrfs-fedora-restauracion"
draft: true
---
# Implementación de Snapper en Fedora: Gestión Avanzada de Snapshots Btrfs

## Resumen Ejecutivo
Esta guía técnica detalla el despliegue de **Snapper** en Fedora Linux para la gestión de instantáneas (snapshots) a nivel de sistema de archivos. A diferencia de los backups tradicionales, Snapper aprovecha las capacidades de *Copy-on-Write* (CoW) de **Btrfs** para crear puntos de restauración instantáneos con un consumo de almacenamiento mínimo, permitiendo revertir cambios fallidos en la configuración o actualizaciones del sistema de forma atómica.

## Contexto y Planteamiento
En entornos de administración de sistemas y desarrollo, la mutabilidad del sistema operativo representa un riesgo operativo. Fedora utiliza por defecto el sistema de archivos Btrfs, el cual soporta snapshots de forma nativa. Sin embargo, la gestión manual de estos subvolúmenes es propensa a errores. **Snapper** automatiza este proceso, proporcionando un marco de trabajo para capturar el estado del sistema (`/`) antes y después de operaciones críticas (como transacciones de DNF), facilitando la recuperación ante desastres sin necesidad de reinstalar el sistema.

## Análisis Técnico Profundo
El funcionamiento de Snapper se basa en la jerarquía de subvolúmenes de Btrfs. Cuando se crea un snapshot, Btrfs no copia los datos, sino que crea una nueva raíz de árbol que apunta a los mismos bloques de datos existentes. Solo cuando un bloque se modifica, se escribe la nueva versión en un espacio distinto (principio CoW).

### Flujo de Operación
1. **Configuración:** Snapper crea un archivo de configuración en `/etc/snapper/configs/` que define el punto de montaje y las políticas de retención.
2. **Snapshot Pre/Post:** Durante una actualización, se genera un snapshot *pre* (estado inicial) y un snapshot *post* (estado final).
3. **Diferenciación:** Snapper permite comparar ambos estados mediante el análisis de metadatos de los bloques.


## Implementación Práctica

### 1. Instalación y Preparación
Primero, es imperativo instalar el paquete principal y el plugin para el gestor de paquetes DNF, lo que automatizará la creación de snapshots en cada instalación o borrado de software.

```shell
sudo dnf install snapper python3-dnf-plugin-snapper
```
### 2. Configuración del Subvolumen para la Raíz
Para que Snapper gestione la raíz del sistema, debemos crear una configuración específica llamada `root`.

```shell
sudo snapper -c root create-config /
```

### 3. Ajuste de Permisos y Políticas
Por defecto, solo el usuario root puede interactuar con Snapper. Para permitir el uso al usuario actual y ajustar la retención de snapshots (para evitar saturar el disco):

```shell
sudo nano /etc/snapper/configs/root
```

Modifique los siguientes parámetros para optimizar el espacio:
* `ALLOW_USERS="tu_usuario"`
* `NUMBER_LIMIT="10"` (Snapshots ordinarios)
* `NUMBER_LIMIT_IMPORTANT="5"` (Snapshots de transacciones críticas)

### 4. Creación Manual de un Punto de Restauración
Para crear un snapshot manual antes de modificar archivos de configuración sensibles en `/etc`:

```shell
snapper -c root create --description "Antes de modificar SSHD" --userdata "type=manual"
```

## Estudio de Caso / Escenario
**Escenario:** Un ingeniero actualiza el kernel y los drivers de video, resultando en un sistema inestable.

**Resolución:**
1. Listar los snapshots disponibles para identificar el ID previo al fallo:
   ```shell
   snapper -c root list
   ```
2. Identificar que el snapshot ID 45 es el "Pre" y el 46 es el "Post" de la actualización.
3. Comparar cambios en archivos específicos:
   ```shell
   snapper -c root diff 45..46 /etc/X11/xorg.conf
   ```
4. Revertir el sistema al estado del snapshot 45:
   ```shell
   sudo snapper -c root undochange 45..46
   ```

## Pros, Contras y Compensaciones

| Característica | Ventaja / Desventaja | Descripción Técnica |
| :--- | :--- | :--- |
| **Rendimiento** | Pros | La creación de snapshots es casi instantánea independientemente del tamaño del disco. |
| **Espacio** | Compensación | Aunque el snapshot inicial es ligero, la divergencia de datos (escrituras intensivas) aumenta el uso de disco gradualmente. |
| **Recuperación** | Contras | Si el subvolumen de logs (`/var/log`) está incluido, la reversión puede causar pérdida de registros históricos de depuración. |
| **Integridad** | Pros | Garantiza consistencia a nivel de bloque en el sistema de archivos. |

## Conclusión
Snapper transforma la gestión de Fedora de un modelo de "esperar lo mejor" a uno de **resiliencia determinista**. Su integración con Btrfs ofrece una capa de seguridad técnica superior a las herramientas de respaldo a nivel de archivo. Para una implementación profesional, se recomienda monitorizar periódicamente el espacio de los subvolúmenes mediante 
```shell
btrfs filesystem usage /
```
para asegurar que las políticas de retención de Snapper sean adecuadas para la carga de trabajo del sistema.