---
pubDate: 2026-04-26
tags: ["general", "astro", "web"]
---
# Olvídate de la Contraseña: Desbloqueo Automático de Disco con TPM 2.0 en Dual Boot (Fedora LUKS y Windows BitLocker)

¿Cansado de teclear contraseñas cada vez que inicias tu sistema Dual Boot con Windows y Fedora? Configurar el cifrado de disco (LUKS en Linux, BitLocker en Windows) es crucial para la seguridad, pero el proceso de desbloqueo constante puede ser tedioso. La solución está en tu chip TPM 2.0.

Este artículo te guía paso a paso para configurar el desbloqueo automático en ambos sistemas, usando tu hardware de seguridad para verificar la integridad del arranque y liberar tus claves de forma transparente.  

## **1\. Entendiendo tu Guardia de Seguridad: El Chip TPM 2.0**

El Módulo de Plataforma Confiable (TPM 2.0) es un chip de seguridad integrado que almacena tus claves criptográficas. Su función principal es asegurar que nadie haya alterado el proceso de arranque. Si el sistema operativo detecta que el firmware ha sido modificado, el TPM se "sella" y exige la clave manual para proteger tus datos.

### **Nota sobre Secure Boot y Microsoft Keys:**

Actualmente, tu sistema probablemente utiliza las **Microsoft Standard Keys**. Esto te da un **Dual Boot perfecto** con estabilidad inmediata, ya que tu BIOS confía en los certificados que permiten a Fedora (a través de un "shim" firmado) y Windows coexistir. La alternativa de usar "Custom Keys" te daría control total, pero haría que configurar el arranque Dual Boot fuera extremadamente complicado.  


## **2\. Preparación Clave en Fedora**

Antes de tocar la configuración del TPM, verifica que el sistema esté listo:

1. **Secure Boot Activo:** Confirma que el Secure Boot esté activado ejecutando el siguiente comando. El resultado debe mostrar SecureBoot enabled:

```shell
sudo bootctl status
```

2. **TPM Registrado:** Verifica que tu chip TPM 2.0 esté registrado en el sistema. Puedes comprobarlo con el siguiente comando; deberías ver una línea que mencione TPM2.0 device registered:

```shell
sudo dmesg | grep -i tpm
```

## **3\. Configurando el Desbloqueo Automático de LUKS**

En Fedora, utilizaremos la herramienta systemd-cryptenroll para vincular tu clave LUKS a los **PCRs (Platform Configuration Registers)** específicos del TPM. Esto asegura que el desbloqueo solo ocurra si los componentes críticos de tu hardware y firmware no han cambiado:

* **PCR 0**: Verifica el firmware de la placa base (BIOS/UEFI).  
* **PCR 2**: Mide cambios en el hardware o firmware de dispositivos.  
* **PCR 7**: Se fija en la política de Secure Boot.

**El Comando para Activar (Ejecutar como root):**

```shell
sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=0+2+7 /dev/nvme0n1p3
```

**(Importante:** Reemplaza /dev/nvme0n1p3 con la ruta a tu partición cifrada real).  

## **4\. Protocolo de Coexistencia (¡El Orden Sí Importa\!)**

Para evitar que BitLocker y LUKS invaliden las mediciones del otro, debes seguir este estricto orden de configuración:

1. **Fedora (Paso 1):** Aplica el comando de systemd-cryptenroll de la sección anterior y reinicia. Confirma que Fedora accede al escritorio automáticamente sin pedir la frase de paso.  
2. **Windows (Paso 2 \- Preparar):** Inicia Windows. Abre una terminal con permisos de Administrador y ejecuta:

```powershell
manage-bde -protectors -add C: -tpm
```

**¿Qué hace este comando?**  
El comando manage-bde \-protectors \-add C: \-tpm realiza la función de **añadir un protector TPM** al volumen de disco cifrado, en este caso, la unidad C:.

En esencia, le estás diciendo a BitLocker que use el Módulo de Plataforma Confiable (TPM) como la forma de desbloquear automáticamente el disco en cada arranque.

* manage-bde: Herramienta de línea de comandos para administrar BitLocker Drive Encryption.  
* \-protectors \-add C:: Indica que se añadirá un método de protección al volumen C:.  
* \-tpm: Especifica que el protector es el TPM, el cual almacenará la clave de cifrado y solo la liberará si el entorno de arranque no ha sido alterado.

Este es un paso de **preparación**; la activación final y el inicio del cifrado ocurren en el siguiente paso.

3. **Windows (Paso 3 \- Activar):** Enciende BitLocker desde el Panel de Control. **¡Asegúrate de guardar la clave de recuperación de 48 dígitos fuera del PC\!** Esto es CRÍTICO.  
4. **Sincronización Final:** Reinicia tu equipo. Ambos sistemas registrarán sus mediciones y ya deberían funcionar de forma automática.


## **5\. Mantenimiento y Recuperación**

Nunca confíes ciegamente en la automatización. Ten en cuenta estos puntos:

* **Actualizaciones de BIOS:** Si actualizas el firmware, el PCR 0 cambiará. En este caso, el TPM te pedirá la clave de recuperación o frase de paso **una sola vez** en cada sistema para re-vincular la nueva medición.  
* **Cambios en Secure Boot:** Modificar la configuración de Secure Boot en la BIOS bloqueará el TPM por seguridad.  
* **Respaldo Fundamental:** Siempre mantén a buen recaudo tu frase de recuperación de LUKS y la clave de 48 dígitos de BitLocker. Son tu plan B ante cualquier fallo de hardware o cambio de configuración.

¡Listo\! Disfruta de la comodidad del desbloqueo automático, manteniendo una seguridad robusta gracias al TPM 2.0.  
