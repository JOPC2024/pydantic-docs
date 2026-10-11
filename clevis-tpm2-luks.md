# Configuración de Clevis con TPM2 y LUKS en Linux

## Tabla de Contenidos

- [Introducción](#introducción)
- [Arquitectura General](#arquitectura-general)
- [Requisitos Previos](#requisitos-previos)
- [Instalación](#instalación)
- [Configuración Paso a Paso](#configuración-paso-a-paso)
  - [Paso 1: Preparar el sistema](#paso-1-preparar-el-sistema)
  - [Paso 2: Instalar TPM2 y LUKS](#paso-2-instalar-tpm2-y-luks)
  - [Paso 3: Crear un archivo de cifrado (keyfile)](#paso-3-crear-un-archivo-de-cifrado-keyfile)
  - [Paso 4: Configurar Clevis con TPM2](#paso-4-configurar-clevis-con-tpm2)
  - [Paso 5: Crear partición LUKS protegida por TPM2](#paso-5-crear-partición-luks-protegida-por-tpm2)
  - [Paso 6: Configurar el initramfs](#paso-6-configurar-el-initramfs)
  - [Paso 7: Verificar la configuración](#paso-7-verificar-la-configuración)
- [Comandos de Referencia](#comandos-de-referencia)
- [Solución de Problemas](#solución-de-problemas)

---

## Introducción

**Clevis** es una herramienta de cifrado en reposo que permite cifrar dispositivos de almacenamiento (LUKS, dm-crypt, etc.) usando múltiples métodos de autenticación: TPM2, LUKS passphrase, keyfiles, o tokens.

Esta guía cubre la configuración de **TPM2 + LUKS** con Clevis, proporcionando una capa adicional de seguridad mediante el Trusted Platform Module (TPM 2.0).

---

## Arquitectura General

```
┌─────────────────────────────────────────────────────────┐
│                    Sistema Operativo                       │
│                                                             │
│  ┌──────────────┐    ┌──────────────┐    ┌────────────┐ │
│  │   TPM2       │◄──►│   Clevis     │◄──►│  LUKS      │ │
│  │  (chip físico)│    │  (daemon)    │    │  (partición)│ │
│  └──────────────┘    └──────────────┘    └────────────┘ │
│                                                             │
│  ┌──────────────┐    ┌──────────────┐                     │
│  │   Passphrase │    │   Keyfile     │                     │
│  │   (backup)   │◄──►│   (fallback) │                     │
│  └──────────────┘    └──────────────┘                     │
└─────────────────────────────────────────────────────────┘
```

**Flujo de arranque:**
1. El initramfs monta el TPM2 en `/dev/tpm0` (o similar).
2. Clevis verifica la integridad del sistema mediante TPM2.
3. Si la verificación es exitosa, se libera la clave de cifrado LUKS.
4. El sistema monta la partición raíz y continúa el arranque.

---

## Requisitos Previos

| Componente | Versión Mínima |
|------------|---------------|
| Kernel Linux | 5.10+ (TPM2.0) |
| TPM2 | Chip físico con firmware TPM 2.0 |
| Clevis | 97+ |
| LUKS2 | 2.0+ |
| dm-crypt | 1.02+ |

**Verificación de TPM2:**
```bash
# Verificar si el sistema tiene un chip TPM2
ls -l /dev/tpm*
# Salida esperada: crw-rw---- 1 root tpm 10, 236 ... /dev/tpm0

# Verificar versión del firmware TPM
sudo tpm2_getcap -c NVIndex -t nvram -i 0x40000000
```

---

## Instalación

### Arch Linux

```bash
sudo pacman -S clevis clevis-tss clevis-luks tpm2-tools
```

### Debian/Ubuntu

```bash
sudo apt install clevis clevis-luks tpm2-tpm2.0-tools
```

### Fedora/RHEL/CentOS

```bash
sudo dnf install clevis clevis-luks tpm2-tss tpm2-pkcs11
```

### openSUSE

```bash
sudo zypper install clevis clevis-luks tpm2-tools
```

---

## Configuración Paso a Paso

### Paso 1: Preparar el sistema

```bash
# Verificar que TPM2 está disponible
ls -l /dev/tpm*

# Si no existe, habilitar el módulo del kernel (en sistemas con TPM físico)
sudo modprobe tpm_tis
sudo modprobe tpm_tis_bios

# Crear un archivo de registro para ver los eventos TPM
echo "1" | sudo tee /sys/firmware/tpm/enable
```

### Paso 2: Instalar TPM2 y LUKS

```bash
# Arch Linux
sudo pacman -S clevis clevis-tss clevis-luks tpm2-tools

# Debian/Ubuntu
sudo apt install clevis clevis-luks tpm2-tpm2.0-tools

# Fedora/RHEL/CentOS
sudo dnf install clevis clevis-luks tpm2-tss tpm2-pkcs11

# openSUSE
sudo zypper install clevis clevis-luks tpm2-tools
```

### Paso 3: Crear un archivo de cifrado (keyfile) para fallback

El keyfile actúa como método de recuperación si el TPM falla o no está presente.

```bash
# Generar un keyfile seguro con una contraseña fuerte
sudo dd if=/dev/urandom of=/etc/clevis/keyfile bs=4096 count=1
chmod 600 /etc/clevis/keyfile

# O usar una contraseña (menos seguro pero más portable)
echo -n "tu-contraseña-muy-segura" | sudo dd of=/etc/clevis/keyfile bs=4096 count=1 status=none
chmod 600 /etc/clevis/keyfile
```

### Paso 4: Configurar Clevis con TPM2

Crear la configuración de Clevis para usar TPM2 como método primario y keyfile como fallback.

```bash
# Crear el archivo de configuración de Clevis
sudo clevis --set-initramfs-configuration-file /etc/clevis/config.json << 'EOF'
{
  "control": {
    "tpm2": {
      "pcr": [0,1,3,4,11],
      "policy": "allow"
    }
  },
  "initramfs": {
    "clevis": "/usr/lib/clevis/initramfs/ctrl",
    "tpm2": "/dev/tpm0",
    "keyfile": "/etc/clevis/keyfile"
  }
}
EOF

# Alternativamente, usar el comando directo:
sudo clevis --set-initramfs-configuration-file /etc/clevis/config.json \
  tpm2,policy=allow,keyfile=/etc/clevis/keyfile
```

### Paso 5: Crear partición LUKS protegida por TPM2

```bash
# Identificar la partición a cifrar (ej. /dev/nvme0n1p3)
lsblk

# Crear un nuevo volumen LUKS con Clevis + TPM2
sudo clevis luks create \
  --device /dev/nvme0n1p3 \
  --tpm2-pcr [0,1,3,4,11] \
  --keyfile /etc/clevis/keyfile \
  --label "root"

# Alternativamente, crear manualmente:
sudo cryptsetup luksFormat /dev/nvme0n1p3 \
  --type luks2 \
  --cipher aes-xts-plain64 \
  --hash sha256 \
  --key-size 512 \
  --pebs 512 \
  --sector-size 512 \
  --iter-time 0 \
  --time-based-keygen \
  --label "root"

# Abrir el volumen LUKS con TPM2
sudo clevis luks open \
  --device /dev/nvme0n1p3 \
  --tpm2-pcr [0,1,3,4,11] \
  --keyfile /etc/clevis/keyfile

# O abrir manualmente (si Clevis no está disponible en el initramfs):
sudo cryptsetup open /dev/nvme0n1p3 root_crypt \
  --type luks2 \
  --tpm2-device /dev/tpm0 \
  --tpm2-policy allow,pcr=[0,1,3,4,11] \
  --keyfile /etc/clevis/keyfile

# Crear un sistema de archivos en el volumen LUKS
sudo mkfs.ext4 /dev/mapper/root_crypt
```

### Paso 6: Configurar el initramfs

#### Para Arch Linux (systemd-boot):

```bash
# Instalar los módulos necesarios en el initramfs
sudo mkinitcpio -P

# Editar /etc/mkinitcpio.conf y asegurarse de incluir:
MODULES=(crypto tpm_tis tpm_tis_bios dm_crypt)
HOOKS=(base udev autodetect keyboard keytable modconf block filesystems fsck)
```

#### Para Debian/Ubuntu (dracut):

```bash
# Instalar los módulos necesarios
sudo apt install dracut clevis-luks

# Editar /etc/dracut.conf.d/clevis.conf:
cat > /etc/dracut.conf.d/clevis.conf << 'EOF'
install_items+=" tpm_tis tpm_tis_bios dm_crypt"
add_drivers+=" tpm_tis tpm_tis_bios"
EOF

# Regenerar el initramfs
sudo dracut -f
```

#### Para Fedora/RHEL (dracut):

```bash
# Editar /etc/dracut.conf.d/clevis.conf:
cat > /etc/dracut.conf.d/clevis.conf << 'EOF'
install_items+=" tpm_tis tpm_tis_bios dm_crypt"
add_drivers+=" tpm_tis tpm_tis_bios"
EOF

# Regenerar el initramfs
sudo dracut -f
```

#### Para openSUSE (initrd):

```bash
# Editar /etc/initrd.d/clevis.conf:
cat > /etc/initrd.d/clevis.conf << 'EOF'
MODULES="tpm_tis tpm_tis_bios dm_crypt"
EOF

# Regenerar el initramfs
sudo mkinitrd -f /boot/initrd
```

### Paso 7: Verificar la configuración

```bash
# Verificar que Clevis está configurado correctamente
clevis show

# Salida esperada:
# {
#   "control": {
#     "tpm2": {
#       "pcr": [0,1,3,4,11],
#       "policy": "allow"
#     }
#   },
#   "initramfs": {
#     "clevis": "/usr/lib/clevis/initramfs/ctrl",
#     "tpm2": "/dev/tpm0",
#     "keyfile": "/etc/clevis/keyfile"
#   }
# }

# Verificar el estado del TPM
sudo tpm2_pcrread 0
sudo tpm2_pcrextend 0 --pcr-value 1234567890abcdef

# Verificar que la partición LUKS está cifrada correctamente
lsblk -f

# Verificar que el initramfs incluye los módulos TPM
lsinitcpio /boot/initramfs-linux.img   # Arch Linux
lsinitrd /boot/initrd                  # openSUSE
```

---

## Comandos de Referencia

### Gestión de Clevis

| Comando | Descripción |
|---------|-------------|
| `clevis show` | Mostrar configuración actual |
| `clevis --set-initramfs-configuration-file <file>` | Configurar initramfs |
| `clevis luks create` | Crear volumen LUKS con Clevis |
| `clevis luks open` | Abrir volumen LUKS con TPM2 |
| `clevis luks close` | Cerrar volumen LUKS |

### Gestión de TPM2

```bash
# Listar PCR disponibles
tpm2_pcread 0-15

# Leer valor de un PCR específico
sudo tpm2_pcrread 0

# Extender un PCR con una medida (hash del sistema)
sudo tpm2_pcrextend 0 --pcr-value "$(sha256sum /etc/fstab | cut -d' ' -f1)"

# Resetear TPM (¡CUIDADO! Borra todas las medidas almacenadas!)
tpm2_clear
```

### Gestión de LUKS con Clevis

```bash
# Crear volumen LUKS con múltiples métodos de autenticación
clevis luks create \
  --device /dev/nvme0n1p3 \
  --tpm2-pcr [0,1,3,4,11] \
  --keyfile /etc/clevis/keyfile \
  --label "root"

# Abrir con TPM2
clevis luks open \
  --device /dev/nvme0n1p3 \
  --tpm2-pcr [0,1,3,4,11] \
  --keyfile /etc/clevis/keyfile

# Abrir con passphrase (fallback)
sudo cryptsetup open /dev/nvme0n1p3 root_crypt --type luks2

# Cerrar volumen LUKS
clevis luks close /dev/nvme0n1p3
```

---

## Solución de Problemas

### Problema: TPM no detectado en el initramfs

**Síntoma:** Error "No such device" al intentar abrir LUKS.

**Solución:**
```bash
# Verificar que los módulos están cargados
lsmod | grep tpm

# Si no aparecen, editar /etc/mkinitcpio.conf (Arch) o /etc/dracut.conf.d/clevis.conf
# Asegurar que 'tpm_tis' y 'tpm_tis_bios' estén en MODULES o install_items

# Regenerar el initramfs
sudo mkinitcpio -P   # Arch Linux
sudo dracut -f       # Fedora/RHEL/openSUSE
```

### Problema: TPM2 no responde

**Síntoma:** Error "TPM not available" al abrir LUKS.

**Solución:**
```bash
# Verificar estado del TPM
ls -l /dev/tpm*
sudo tpm2_getcap -c NVIndex -t nvram -i 0x40000000

# Si el chip no responde, puede estar bloqueado o dañado.
# Opción: usar solo keyfile como método de autenticación.
```

### Problema: Fallback a keyfile no funciona

**Síntoma:** No se puede abrir LUKS ni con TPM2 ni con keyfile.

**Solución:**
```bash
# Verificar que el keyfile existe y tiene permisos correctos
ls -l /etc/clevis/keyfile
chmod 600 /etc/clevis/keyfile

# Intentar abrir manualmente con passphrase (si se configuró)
sudo cryptsetup open /dev/nvme0n1p3 root_crypt --type luks2

# Si no hay passphrase, usar el keyfile directamente:
sudo cryptsetup open /dev/nvme0n1p3 root_crypt \
  --key-file /etc/clevis/keyfile
```

### Problema: PCR incorrectos

**Síntoma:** Clevis rechaza la apertura aunque el sistema no haya cambiado.

**Solución:**
```bash
# Verificar los valores de PCR actuales
sudo tpm2_pcrread 0-15

# Comparar con los valores esperados en la configuración de Clevis
# Ajustar los PCRs en /etc/clevis/config.json si es necesario
```

---

## Consideraciones de Seguridad

### ¿Por qué usar TPM2 + LUKS?

| Escenario | TPM2 solo | Keyfile solo | TPM2 + Keyfile (recomendado) |
|-----------|-----------|---------------|-------------------------------|
| Hardware TPM comprometido | ❌ Vulnerable | ✅ Seguro | ✅ Seguro |
| Keyfile perdido/robo | ✅ Seguro | ❌ Vulnerable | ✅ Seguro |
| TPM no presente en hardware | ❌ No funciona | ✅ Funciona | ✅ Funciona (fallback) |
| Ataque de cold boot | ⚠️ Riesgo | ✅ Seguro | ✅ Seguro |

### Mejores prácticas:

1. **Use PCRs 0, 1, 3, 4, 11** — Estos son los más estables y menos propensos a cambios accidentales.
2. **Configure un keyfile como fallback** — Para recuperación en caso de fallo del TPM.
3. **No use passphrase + TPM2 simultáneamente** — Esto debilita la seguridad al permitir ataques offline contra el passphrase.
4. **Proteja el keyfile con permisos restrictivos** (`chmod 600`).
5. **Mantenga una copia de seguridad del keyfile en un medio seguro.**

---

## Referencias

- [Documentación oficial de Clevis](https://github.com/clevis/clevis)
- [Clevis LUKS documentation](https://github.com/clevis/clevis/blob/master/doc/luks.rst)
- [TPM2.0 specification](https://trustedcomputinggroup.org/resource/tpm-2-0-specification/)
- [LUKS2 manual page](https://man7.org/linux/man-pages/man8/cryptsetup.8.html)

---

*Última actualización: 2024*