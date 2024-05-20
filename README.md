### Создание Vagrant-образа виртуальной машины, работающей под управлением Debian Linux 12 - Bookworm.
### Обновление ядра операционной системы.
В нашем примере используется гипервизор Qemu-KVM, библиотека Libvirt. В качестве хостовой системы - OpenSuse Leap 15.5.
Для работы Vagrant с Libvirt установлен пакет vagrant-libvirt:
```
Сведения — пакет vagrant-libvirt:
---------------------------------
Репозиторий            : Основной репозиторий
Имя                    : vagrant-libvirt
Версия                 : 0.10.2-bp155.1.19
Архитектура            : x86_64
Поставщик              : openSUSE
Размер после установки : 658,3 KiB
Установлено            : Да
Состояние              : актуален
Пакет с исходным кодом : vagrant-libvirt-0.10.2-bp155.1.19.src
Адрес источника        : https://github.com/vagrant-libvirt/vagrant-libvirt
Заключение             : Провайдер Vagrant для libvirt
Описание               : 

    This is a Vagrant plugin that adds a Libvirt provider to Vagrant, allowing
    Vagrant to control and provision machines via the Libvirt toolkit.
```
Образ операционной системы создём заранее, для этого установим [Debian Linux из официального образа netinst](https://www.debian.org/distrib/netinst)
При установке заведём нового пользователя с именем vagrant. 
Если при установке операционной системы не был создан пользователь vagrant, то создадим его с помощью команды:
```
useradd -m -s /bin/bash -G sudo vagrant
```
 Где:
```
-m - создать домашний каталог;
-s /bin/bash - задать командную оболочку;
-G sudo - включить пользователя в группу sudo, что позволит ему выполнять команды с sudo;
```
После установки и запуска операционной системы, залогинимся под учетной записью пользователя vagrant 
(предварительно, во время установки ОС отметим пункт SSH-сервер) и загрузим и зарегистрируем 
стандартный ключ ssh vagrant для доступа без пароля: 
```
vagrant@debian12:~$ wget https://raw.githubusercontent.com/hashicorp/vagrant/master/keys/vagrant.pub
vagrant@debian12:~$ ssh-copy-id -f -i vagrant.pub vagrant@localhost
```
Ранее мы создали каталог в файловой системе, который будет использоваться как хранилище образов виртуальных машин: 
![Хранилище образов виртуальных машин](20240422-01.png)
В этом хранилище уже находится образ диска новой виртуальной машины. Подготовим [сценарий](create-box.sh) конвертации образов виртуальных машин qcow2 в контейнеры vagrant. 
Оригинал сценария доступен по [ссылке](https://raw.githubusercontent.com/vagrant-libvirt/vagrant-libvirt/master/tools/create_box.sh)
оригинальное описание применения сценария доступно по [ссылке](https://github.com/vagrant-libvirt/vagrant-libvirt)

Ниже текст сценария:
```
#!/usr/bin/env bash
set -xe
error() {
    local msg="${1}"
    echo "==> ERROR: ${msg}"
    exit 1
}

usage() {
    echo "Usage: ${0} IMAGE [BOX] [Vagrantfile.add]"
    echo
    echo "Package a qcow2 image into a vagrant-libvirt reusable box"
    echo ""
    echo "If packaging from a Vagrant machine ensure 'config.ssh.insert_key = false' was "
    echo "set in the original Vagrantfile to avoid removal of the default ssh key, "
    echo "otherwise vagrant will not be able to connect to machines created from this box"
}

# Print the image's backing file
backing(){
    local img=${1}
    qemu-img info "$img" | grep 'backing file:' | cut -d ':' -f2
}

# Rebase the image
rebase(){
    local img=${1}
    qemu-img rebase -p -b "" "$img"
    [[ "$?" -ne 0 ]] && error "Error during rebase"
}

# Is absolute path
isabspath(){
    local path=${1}
    [[ "$path" =~ ^/.* ]]
}

if [ -z "$1" ] || [ "$1" = "-h" ] || [ "$1" = "--help" ]; then
    usage
    exit 1
fi

IMG=$(readlink -e "$1")
[[ "$?" -ne 0 ]] && error "'$1': No such image"

IMG_DIR=$(dirname "$IMG")
IMG_BASENAME=$(basename "$IMG")

BOX=${2:-}
# If no box name is supplied infer one from image name
if [[ -z "$BOX" ]]; then
    BOX_NAME=${IMG_BASENAME%.*}
    BOX=$BOX_NAME.box
else
    BOX_NAME=$(basename "${BOX%.*}")
fi

[[ -f "$BOX" ]] && error "'$BOX': Already exists"

CWD=$(pwd)
TMP_DIR="$CWD/_tmp_package"
TMP_IMG="$TMP_DIR/box.img"

mkdir -p "$TMP_DIR"

[[ ! -r "$IMG" ]] && error "'$IMG': Permission denied"

if [ -n "$3" ] && [ -r "$3" ]; then
  VAGRANTFILE_ADD="$(cat $3)"
fi

# We move / copy (when the image has master) the image to the tempdir
# ensure that it's moved back / removed again
if [[ -n $(backing "$IMG") ]]; then
    echo "==> Image has backing image, copying image and rebasing ..."
    trap "rm -rf $TMP_DIR" EXIT
    cp "$IMG" "$TMP_IMG"
    rebase "$TMP_IMG"
else
    if fuser -s "$IMG" &> /dev/null ; then
        error "Image '$IMG_BASENAME' is used by another process"
    fi

    # move the image to get a speed-up and use less space on disk
    trap 'mv "$TMP_IMG" "$IMG"; rm -rf "$TMP_DIR"' EXIT
    mv "$IMG" "$TMP_IMG"
fi

cd "$TMP_DIR"

#Using the awk int function here to truncate the virtual image size to an
#integer since the fog-libvirt library does not seem to properly handle
#floating point.
IMG_SIZE=$(qemu-img info --output=json "$TMP_IMG" | awk '/virtual-size/{s=int($2)/(1024^3); print (s == int(s)) ? s : int(s)+1 }')

echo "{$IMG_SIZE}"

cat > metadata.json <<EOF
{
    "provider": "libvirt",
    "format": "qcow2",
    "virtual_size": ${IMG_SIZE}
}
EOF

cat > Vagrantfile <<EOF
Vagrant.configure("2") do |config|

  config.nfs.verify_installed

  config.vm.provider :libvirt do |libvirt|

    libvirt.driver = "kvm"
    libvirt.host = ""
    libvirt.connect_via_ssh = false
    libvirt.storage_pool_name = "images"

  end

${VAGRANTFILE_ADD:-}
end
EOF

echo "==> Creating box, tarring and gzipping"

if type pigz >/dev/null 2>/dev/null; then
  GZ="pigz"
else
  GZ="gzip"
fi
tar cv -S --totals ./metadata.json ./Vagrantfile ./box.img | $GZ -c > "$BOX"

# if box is in tmpdir move it to CWD before removing tmpdir
if ! isabspath "$BOX"; then
    mv "$BOX" "$CWD"
fi

echo "==> ${BOX} created"
echo "==> You can now add the box:"
echo "==>   'vagrant box add ${BOX} --name ${BOX_NAME}'"

```

Необходимо сохранить сценарий в файле именем create-box.sh в рабочем каталоге и сделать исполняемым:
```
chmod u+x create-box.sh
```
Каталог для хранения образов вируальных машин по умолчанию в libvirt - /var/lib/libvirt/images/. 
Как правило в каталоге var, смонтированном обычно в корневой раздел, недостаточно места для разворачивания крупных образов.
Поэтому ранее мы создали новое хранилище - images:
![хранилище - images](20240422-01.png)
Изменим в скрипте create-box.sh строку:
```
libvirt.storage_pool_name = "default"
```
на 
```
libvirt.storage_pool_name = "images"
```
Выполним конвертацию образа виртуальной машины в контейнер vagrant с помощью сценария: 
```
./create-box.sh /home/max/libvirt/images/debian12.qcow2 /home/max/vagrant/images/debian12 vagrantfile.info
```
Где содержимое файла vagrantfile.info: 
```
config.vm.guest = "debian";
config.nfs.verify_installed = false;
config.vm.synced_folder ".", "/vagrant", disabled: true
```
 Здесь:
```
config.vm.guest = «debian» - тип виртуальной машины;
config.nfs.verify_installed = false; - запрет проверки наличия клиента монтирования NFS (включить, если монтирование не используется);
config.vm.synced_folder «.», «/vagrant», disabled: true - если не требуется копирование текущего каталога в каталог /vagrant виртуальной машины, то запретить его
```
Доступные типы виртуальных машин можно найти в каталоге расширений для Vagrant. Чтобы узнать, где пакет vagrant-libvirt хранит расширения, выполним команду:
```
localhost:~ # rpm -ql vagrant | egrep 'plugins/guests$'
/usr/share/vagrant/gems/gems/vagrant-2.2.18/plugins/guests
```
Проверим содержимое каталога:
```
max@localhost:~/vagrant/vg3> ls -l /usr/share/vagrant/gems/gems/vagrant-2.2.18/plugins/guests
итого 160
drwxr-xr-x 3 root root 4096 апр 16 12:11 alpine
drwxr-xr-x 3 root root 4096 апр 16 12:11 alt
drwxr-xr-x 3 root root 4096 апр 16 12:11 amazon
drwxr-xr-x 3 root root 4096 апр 16 12:11 arch
drwxr-xr-x 3 root root 4096 мая 13  2020 astra
drwxr-xr-x 3 root root 4096 апр 16 12:11 atomic
drwxr-xr-x 3 root root 4096 апр 16 12:11 bsd
drwxr-xr-x 3 root root 4096 апр 16 12:11 centos
drwxr-xr-x 3 root root 4096 апр 16 12:11 coreos
drwxr-xr-x 3 root root 4096 апр 16 12:11 darwin
drwxr-xr-x 3 root root 4096 апр 16 12:11 debian
drwxr-xr-x 2 root root 4096 апр 16 12:11 dragonflybsd
drwxr-xr-x 2 root root 4096 апр 16 12:11 elementary
drwxr-xr-x 3 root root 4096 апр 16 12:11 esxi
drwxr-xr-x 3 root root 4096 апр 16 12:11 fedora
drwxr-xr-x 3 root root 4096 апр 16 12:11 freebsd
drwxr-xr-x 3 root root 4096 апр 16 12:11 funtoo
drwxr-xr-x 3 root root 4096 апр 16 12:11 gentoo
drwxr-xr-x 3 root root 4096 апр 16 12:11 haiku
drwxr-xr-x 2 root root 4096 апр 16 12:11 kali
drwxr-xr-x 3 root root 4096 апр 16 12:11 linux
drwxr-xr-x 2 root root 4096 апр 16 12:11 mint
drwxr-xr-x 3 root root 4096 апр 16 12:11 netbsd
drwxr-xr-x 3 root root 4096 апр 16 12:11 nixos
drwxr-xr-x 3 root root 4096 апр 16 12:11 omnios
drwxr-xr-x 3 root root 4096 апр 16 12:11 openbsd
drwxr-xr-x 3 root root 4096 апр 16 12:11 openwrt
drwxr-xr-x 3 root root 4096 апр 16 12:11 photon
drwxr-xr-x 3 root root 4096 апр 16 12:11 pld
drwxr-xr-x 3 root root 4096 апр 16 12:11 redhat
drwxr-xr-x 3 root root 4096 апр 16 12:11 rocky
drwxr-xr-x 3 root root 4096 апр 16 12:11 slackware
drwxr-xr-x 3 root root 4096 апр 16 12:11 smartos
drwxr-xr-x 3 root root 4096 апр 16 12:11 solaris
drwxr-xr-x 3 root root 4096 апр 16 12:11 solaris11
drwxr-xr-x 3 root root 4096 апр 16 12:11 suse
drwxr-xr-x 3 root root 4096 апр 16 12:11 tinycore
drwxr-xr-x 2 root root 4096 апр 16 12:11 trisquel
drwxr-xr-x 2 root root 4096 апр 16 12:11 ubuntu
drwxr-xr-x 4 root root 4096 апр 16 12:11 windows
max@localhost:~/vagrant/vg3>
```
Создадим новый файл Vagrantfile, указав имя файла с образом vagrant: 
```
max@localhost:~/vagrant/vg3> vagrant init /home/max/vagrant/images/debian12
```
После этого добавим оперативной памяти и виртуальных процессоров для наших будущих образов (см. Vagrantfile).
```
  config.vm.provider "libvirt" do |lv|
    lv.memory = "2048"
    lv.cpus = "2"
    lv.title = "Debian12"
    lv.description = "Виртуальная машина на базе дистрибутива Debian Linux"
  end
```

Теперь можно запустить новый образ. 
```
max@localhost:~/vagrant/vg3> vagrant up
```
При первом запуске будет создан базовый образ машины, который необходим для дальнейших разворачиваний. 
Последующие запуски vagrant-образов будут использовать этот созданный базовый образ, в связи с чем их запуск будет происходить гораздо быстрее.

Установка новой версии ядра с использованием backports.

Версия ядра после установки ОС:
![Версия ядра после установки ОС](20240422-02.png)

После входа в консоль новой машины, поднимем привилегии и пропишем новый репозиторий:
```
root@debian12:~# apt edit-sources
```
Добавим строку:
```
deb http://deb.debian.org/debian bookworm-backports main contrib non-free
```
Ищем пакеты ядра:
```
root@debian12:~# apt search linux-image
```
Устанавливаем самое последнее ядро из репозитария bpo:
```
root@debian12:~# apt install linux-image-6.7.12+bpo-amd64-unsigned
```
Перезагружаемся:
```
root@debian12:~# reboot
```
Версия ядра после обновления:
![Версия ядра после обновления](20240422-03.png)
