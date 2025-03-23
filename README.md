## **Использованные ресурсы**
* **[Настройка IOMMU](#enable--verify-iommu)**
* **[Установка пакетов](#install-required-tools)**
* **[Включение сервисов](#enable-required-services)**
* **[Настройка гостевой ОС](#setup-guest-os)**
* **[Подключение PCI устройств](#attaching-pci-devices)**
* **[Libvirt Hooks](#libvirt-hooks)**
* **[Проброс клавиатуры/мыши](#keyboardmouse-passthrough)**
* **[Обнаружение виртуализации видеокарты](#video-card-driver-virtualisation-detection)**
* **[Проброс аудио](#audio-passthrough)**
* **[Патчинг vBIOS](#vbios-patching)**

### **Включение и проверка IOMMU**
***Настройки BIOS*** \
Необходимо включить ***Intel VT-d***, если используете процессор от Intel, или ***AMD-Vi***, если используете процессор от AMD. Если подобных опций нет, то скорее всего ваше железо не поддерживает IOMMU.

Выключение ***Resizable BAR Support*** в настройках BIOS. 
Адаптеры, поддерживающие Resizable BAR, работают нестабильно, если он включен в настройках BIOS/UEFI. Выключение не приведет к сильному падению производительности, так что лучше его выключить, по крайней мере до появления полноценной поддержки ReBar в KVM.

***Установка необходимых параметров ядра для процессора.***

<details>
  <summary><b>GRUB</b></summary>

***Отредактируйте конфигурацию GRUB***
| /etc/default/grub |
| ----- |
| `GRUB_CMDLINE_LINUX_DEFAULT="... intel_iommu=on iommu=pt ..."` |
| Или |
| `GRUB_CMDLINE_LINUX_DEFAULT="... amd_iommu=on iommu=pt ..."` |

***Генерация grub.cfg***
```sh
grub-mkconfig -o /boot/grub/grub.cfg
```
</details>
<details>
  <summary><b>Systemd Boot</b></summary>

***Отредактируйте boot entry***
| /boot/loader/entries/*.conf |
| ----- |
| `options root=UUID=...intel_iommu=on iommu=pt..` |
| Или |
| `options root=UUID=...amd_iommu=on iommu=pt..` |
</details>

Чтобы изменения вступили в силу, необходимо перезагрузить машину.

***Для проверки IOMMU можно использовать следующую команду.***
```sh
dmesg | grep IOMMU
```
Если всё работает, то выводом будет `Intel-IOMMU: enabled` для Intel, или `AMD-Vi: AMD IOMMUv2 loaded and initialized` для AMD.

Чтобы посмотреть группы IOMMU и подключенные устройства, можно запустить следующий скрипт:
```sh
#!/bin/bash
shopt -s nullglob
for g in `find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V`; do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

### **Установка необходимых инструментов**
<details>
  <summary><b>Gentoo Linux</b></summary>

  ```sh
  emerge -av qemu virt-manager libvirt ebtables dnsmasq
  ```
</details>

<details>
  <summary><b>Arch Linux</b></summary>

  ```sh
  pacman -S qemu libvirt edk2-ovmf virt-manager dnsmasq ebtables
  ```
</details>

<details>
  <summary><b>Fedora</b></summary>

  ```sh
  dnf install @virtualization
  ```
</details>

<details>
  <summary><b>Ubuntu</b></summary>

  ```sh
  apt install qemu-kvm qemu-utils libvirt-daemon-system libvirt-clients bridge-utils virt-manager ovmf
  ```
</details>

### **Запуск необходимых сервисов**
<details>
  <summary><b>SystemD</b></summary>

  ```sh
  systemctl enable --now libvirtd
  ```
</details>

<details>
  <summary><b>OpenRC</b></summary>

  ```sh
  rc-update add libvirtd default
  rc-service libvirtd start
  ```
</details>

Иногда нужно запустить сеть вручную.
```sh
virsh net-start default
virsh net-autostart default
```

### **Настройка гостевой ОС**
Чтобы запускать виртуальные машины без прав root-пользователя, следует добавить пользователя в группу ***libvirt***. А также в группы ***input*** и ***kvm*** для проброса устройств ввода.
```sh
usermod -aG kvm,input,libvirt username
```

Скачайте драйвер [virtio](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso). \
Запустите ***virt-manager*** и создайте новую виртуальную машину. На последнем шаге выберите ***Customize before install***. \
В разделе ***Overview*** установите ***Chipset*** в ***Q35***, а ***Firmware*** в ***UEFI***. \
В разделе ***CPUs*** установите ***CPU model*** в ***host-passthrough***, а ***CPU Topology*** в соответствии с вашей системой. \
Для диска ***SATA*** виртуальной машины установите ***Disk Bus*** в ***virtio***. \
В разделе ***NIC*** установите ***Device Model*** в ***virtio***. \
Добавьте оборудование > CDROM: virtio-win.iso. \
Теперь начните установку. Windows не может обнаружить ***virtio disk***, поэтому вам нужно будет ***Load Driver*** и выбрать ***virtio-iso/amd64/win10***, когда появится запрос. \
После успешной установки Windows установите драйверы virtio с CDROM. После этого можно удалить virtio iso.

### **Подключение PCI устройств**
Удалите Channel Spice, Display Spice, Video QXL, Sound ich* и другие ненужные устройства. \
Теперь нажмите на ***Add Hardware***, выберите ***PCI Devices*** и добавьте PCI Host устройства для вашего GPU VGA и HDMI Audio.

### **Libvirt Hooks**
Libvirt hooks автоматизируют процесс выполнения определенных задач при изменении состояния виртуальной машины. \
Подробнее: [PassthroughPost](https://passthroughpo.st/simple-per-vm-libvirt-hooks-with-the-vfio-tools-hook-helper/)

**Примечание**: Закомментируйте строку Unbind/rebind EFI framebuffer в скриптах start и stop, если вы используете видеокарты AMD 6000 серии (https://github.com/QaidVoid/Complete-Single-GPU-Passthrough/issues/9).
Также переместите строку выгрузки AMD kernel module ниже отключения устройств от хоста. Это также может относиться к более старым видеокартам AMD.

<details>
  <summary><b>Создание Libvirt Hook</b></summary>

  ```sh
  mkdir /etc/libvirt/hooks
  touch /etc/libvirt/hooks/qemu
  chmod +x /etc/libvirt/hooks/qemu
  ```
  <table>
  <tr>
  <th>
    /etc/libvirt/hooks/qemu
  </th>
  </tr>

  <tr>
  <td>

  ```sh
  #!/bin/bash

GUEST_NAME="$1"
HOOK_NAME="$2"
STATE_NAME="$3"
MISC="${@:4}"

BASEDIR="$(dirname $0)"

HOOKPATH="$BASEDIR/qemu.d/$GUEST_NAME/$HOOK_NAME/$STATE_NAME"
set -e # Если скрипт завершается с ошибкой, мы также должны завершиться.

if [ -f "$HOOKPATH" ]; then
  eval \""$HOOKPATH"\" "$@"
elif [ -d "$HOOKPATH" ]; then
  while read file; do
    eval \""$file"\" "$@"
  done <<< "$(find -L "$HOOKPATH" -maxdepth 1 -type f -executable -print;)"
fi
  ```

  </td>
  </tr>
  </table>
</details>

<details>
  <summary><b>Создание скрипта запуска</b></summary>
  
  ```sh
  mkdir -p /etc/libvirt/hooks/qemu.d/win10/prepare/begin
  touch /etc/libvirt/hooks/qemu.d/win10/prepare/begin/start.sh
  chmod +x /etc/libvirt/hooks/qemu.d/win10/prepare/begin/start.sh
  ```
**Примечание**: Если вы используете KDE Plasma (Wayland), вам нужно завершить пользовательские сервисы вместе с display-manager (https://github.com/QaidVoid/Complete-Single-GPU-Passthrough/issues/31).

  <table>
  <tr>
  <th>
    /etc/libvirt/hooks/qemu.d/win10/prepare/begin/start.sh
  </th>
  </tr>

  <tr>
  <td>

  ```sh
#!/bin/bash
set -x

# Остановка display manager
systemctl stop display-manager
# systemctl --user -M YOUR_USERNAME@ stop plasma*

# Отключение VTconsoles: может не понадобиться
echo 0 > /sys/class/vtconsole/vtcon0/bind
echo 0 > /sys/class/vtconsole/vtcon1/bind

# Отключение EFI Framebuffer
echo efi-framebuffer.0 > /sys/bus/platform/drivers/efi-framebuffer/unbind

# Выгрузка модулей NVIDIA
modprobe -r nvidia_drm nvidia_modeset nvidia_uvm nvidia

# Выгрузка модуля AMD
# modprobe -r amdgpu

# Отключение GPU устройств от хоста
# Используйте ваш GPU и HDMI Audio PCI host device
virsh nodedev-detach pci_0000_01_00_0
virsh nodedev-detach pci_0000_01_00_1

# Загрузка модуля vfio
modprobe vfio-pci
  ```

  </td>
  </tr>
  </table>
</details>

<details>
  <summary><b>Создание скрипта остановки</b></summary>

  ```sh
  mkdir -p /etc/libvirt/hooks/qemu.d/win10/release/end
  touch /etc/libvirt/hooks/qemu.d/win10/release/end/stop.sh
  chmod +x /etc/libvirt/hooks/qemu.d/win10/release/end/stop.sh
  ```
  <table>
  <tr>
  <th>
    /etc/libvirt/hooks/qemu.d/win10/release/end/stop.sh
  </th>
  </tr>

  <tr>
  <td>

  ```sh
#!/bin/bash
set -x

# Подключение GPU устройств к хосту
# Используйте ваш GPU и HDMI Audio PCI host device
virsh nodedev-reattach pci_0000_01_00_0
virsh nodedev-reattach pci_0000_01_00_1

# Выгрузка модуля vfio
modprobe -r vfio-pci

# Загрузка модуля AMD
#modprobe amdgpu

# Подключение framebuffer к хосту
echo "efi-framebuffer.0" > /sys/bus/platform/drivers/efi-framebuffer/bind

# Загрузка модулей NVIDIA
modprobe nvidia_drm
modprobe nvidia_modeset
modprobe nvidia_uvm
modprobe nvidia

# Подключение VTconsoles: может не понадобиться
echo 1 > /sys/class/vtconsole/vtcon0/bind
echo 1 > /sys/class/vtconsole/vtcon1/bind

# Перезапуск Display Manager
systemctl start display-manager
```

  </td>
  </tr>
  </table>
</details>

### **Проброс клавиатуры/мыши**
Для использования клавиатуры/мыши в виртуальной машине можно либо пробросить USB Host устройство, либо использовать Evdev passthrough.

Использование USB Host Device простое, \
***Add Hardware*** > ***USB Host Device***, добавьте ваши устройства клавиатуры и мыши.

Для Evdev passthrough следуйте этим шагам: \
Измените конфигурацию libvirt вашей виртуальной машины. \
**Примечание**: Сохраняйте только после добавления устройств клавиатуры и мыши, иначе изменения будут потеряны. \
Измените первую строку на:

<table>
<tr>
<th>
virsh edit win10
</th>
</tr>

<tr>
<td>

```xml
<domain type='kvm' xmlns:qemu='http://libvirt.org/schemas/domain/qemu/1.0'>
```

</td>
</tr>
</table>

Найдите вашу клавиатуру и мышь в ***/dev/input/by-id***. Обычно названия устройств заканчиваются на ***event-kbd*** и ***event-mouse** для мыши и клавиатуры соответсвенно*. Добавьте устройства в конфигурацию перед закрытием тега ***`</domain>`***. \
Замените ***MOUSE_NAME*** и ***KEYBOARD_NAME*** на идентификаторы ваших устройств.

<table>
<tr>
<th>
virsh edit win10
</th>
</tr>

<tr>
<td>

```xml
...
  <qemu:commandline>
    <qemu:arg value='-object'/>
    <qemu:arg value='input-linux,id=mouse1,evdev=/dev/input/by-id/MOUSE_NAME'/>
    <qemu:arg value='-object'/>
    <qemu:arg value='input-linux,id=kbd1,evdev=/dev/input/by-id/KEYBOARD_NAME,grab_all=on,repeat=on'/>
  </qemu:commandline>
</domain>
```

</td>
</tr>
</table>

Нужно включить эти устройства в конфигурацию qemu.
<table>
<tr>
<th>
/etc/libvirt/qemu.conf
</th>
</tr>

<tr>
<td>

```sh
...
user = "YOUR_USERNAME"
group = "kvm"
...
cgroup_device_acl = [
    "/dev/input/by-id/KEYBOARD_NAME",
    "/dev/input/by-id/MOUSE_NAME",
    "/dev/null", "/dev/full", "/dev/zero",
    "/dev/random", "/dev/urandom",
    "/dev/ptmx", "/dev/kvm", "/dev/kqemu",
    "/dev/rtc","/dev/hpet", "/dev/sev"
]
...
```

</td>
</tr>
</table>

Также переключитесь с PS/2 устройств на virtio устройства. Добавьте устройства внутри блока ***`<devices>`***.
<table>
<tr>
<th>
virsh edit win10
</th>
</tr>

<tr>
<td>

```xml
...
<devices>
  ...
  <input type='mouse' bus='virtio'/>
  <input type='keyboard' bus='virtio'/>
  ...
</devices>
...
```

</td>
</tr>
</table>

### **Проброс аудио**
Аудио с гостевой ОС можно перенаправить на хост с помощью ***Pipewire*** или ***Pulseaudio***. \
Также можно использовать [Scream](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Passing_VM_audio_to_host_via_Scream).

<details>
    <summary><b>Pipewire</b></summary>

Из [ArchWiki](https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF#Passing_audio_from_virtual_machine_to_host_via_JACK_and_PipeWire)

Нужен Pipewire с поддержкой JACK.

***Примечание***: Можено использовать [Carla](https://kx.studio//Applications:Carla) для определения подходящего ввода/вывода. Замените `1000` на ваш текущий идентификатор пользователя.
<table>
<tr>
<th>
virsh edit win10
</th>
</tr>

<tr>
<td>

```xml
...
  <devices>
    ...
    <audio id="1" type="jack">
      <input clientName="win10" connectPorts="your-input"/>
      <output clientName="win10" connectPorts="your-output"/>
    </audio>
  </devices>
  <qemu:commandline>
    <qemu:env name="PIPEWIRE_RUNTIME_DIR" value="/run/user/1000"/>
    <qemu:env name="PIPEWIRE_LATENCY" value="512/48000"/>
  </qemu:commandline>
</domain>
```

</td>
</tr>
</table>
</details>

<details>
    <summary><b>Pulseaudio</b></summary>

***Примечание***: Замените `1000` на ваш текущий идентификатор пользователя.

<table>
<tr>
<th>
virsh edit win10
</th>
</tr>

<tr>
<td>

```xml
...
  <qemu:commandline>
    ...
    <qemu:arg value="-device"/>
    <qemu:arg value="ich9-intel-hda,bus=pcie.0,addr=0x1b"/>
    <qemu:arg value="-device"/>
    <qemu:arg value="hda-micro,audiodev=hda"/>
    <qemu:arg value="-audiodev"/>
    <qemu:arg value="pa,id=hda,server=/run/user/1000/pulse/native"/>
  </qemu:commandline>
</domain>
```

</td>
</tr>
</table>
</details>

### **Обнаружение виртуализации видеокарты**
Драйверы видеокарт отказываются работать в виртуальной машине, поэтому нужно подделать Hyper-V Vendor ID.
<table>
<tr>
<th>
virsh edit win10
</th>
</tr>

<tr>
<td>

```xml
...
<features>
  ...
  <hyperv>
    ...
    <vendor_id state='on' value='whatever'/>
    ...
  </hyperv>
  ...
</features>
...
```

</td>
</tr>
</table>

Гостевые драйверы NVIDIA также требуют скрытия KVM CPU leaf:
<table>
<tr>
<th>
virsh edit win10
</th>
</tr>

<tr>
<td>

```xml
...
<features>
  ...
  <kvm>
    <hidden state='on'/>
  </kvm>
  ...
</features>
...
```

</td>
</tr>
</table>

### **Патчинг vBIOS**
***ПРИМЕЧАНИЕ: Вы вносите изменения только в дамп ROM файла. Ваше оборудование в безопасности(Отвечаю).*** \
Хотя большинство GPU можно пробросить с использованием стокового vBIOS, некоторым GPU требуется патчинг vBIOS для проброса. \
Для патчинга vBIOS сначала нужно сдампить vBIOS вашей видеокарты. \
Если у вас установлена Windows, вы можете использовать [GPU-Z](https://www.techpowerup.com/gpuz) для дампа vBIOS. \
Для дампа vBIOS на Linux используйте следующую команду (замените PCI id на ваш):
```sh
echo 1 > /sys/bus/pci/devices/0000:01:00.0/rom
cat /sys/bus/pci/devices/0000:01:00.0/rom > path/to/dump/vbios.rom
echo 0 > /sys/bus/pci/devices/0000:01:00.0/rom
```
Если вы не в root shell, используйте команды с sudo:
```sh
echo 1 | sudo tee /sys/bus/pci/devices/0000:01:00.0/rom
sudo cat /sys/bus/pci/devices/0000:01:00.0/rom > path/to/dump/vbios.rom
echo 0 | sudo tee /sys/bus/pci/devices/0000:01:00.0/rom
```
Для патчинга vBIOS используйте Hex Editor (например, [Okteta](https://utils.kde.org/projects/okteta)) и удалите ненужные заголовки. \
Для NVIDIA GPU, используя hex editor, найдите строку “VIDEO” и удалите всё до HEX значения 55. \
Для AMD процесс похожий, но я не уверен в деталях.

Чтобы использовать пропатченный vBIOS, отредактируйте конфигурацию виртуальной машины, чтобы включить пропатченный vBIOS внутри блока ***hostdev*** VGA.

  <table>
  <tr>
  <th>
  virsh edit win10
  </th>
  </tr>

  <tr>
  <td>

  ```xml
  ...
  <hostdev mode='subsystem' type='pci' managed='yes'>
    <source>
      ...
    </source>
    <rom file='/home/me/patched.rom'/>
    ...
  </hostdev>
  ...
  ```

  </td>
  </tr>
  </table>

### **Смотрите также**
> [Single GPU Passthrough Troubleshooting](https://docs.google.com/document/d/17Wh9_5HPqAx8HHk-p2bGlR0E-65TplkG18jvM98I7V8)<br/>
> [Single GPU Passthrough by joeknock90](https://github.com/joeknock90/Single-GPU-Passthrough)<br/>
> [Single GPU Passthrough by YuriAlek](https://gitlab.com/YuriAlek/vfio)<br/>
> [Single GPU Passthrough by wabulu](https://github.com/wabulu/Single-GPU-passthrough-amd-nvidia)<br/>
> [ArchLinux PCI Passthrough](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF)<br/>
> [Gentoo GPU Passthrough](https://wiki.gentoo.org/wiki/GPU_passthrough_with_libvirt_qemu_kvm)<br/>

---
