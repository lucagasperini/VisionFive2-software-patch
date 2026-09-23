# Istruzioni per la patch

## Configurazione iniziale
Il nostro sistema target è debian, vogliamo cross compilare i pacchetti per riscv64, dunque abbiamo bisogno dei tool di sistema adatti:
```
sudo apt install build-essential dh-sequence-pkgkde-symbolshelper devscripts debhelper fakeroot dpkg-dev lintian quilt dpkg-dev
sudo dpkg --add-architecture riscv64
sudo apt update
sudo apt install \
    gcc-riscv64-linux-gnu \
    g++-riscv64-linux-gnu \
    libc6-dev-riscv64-cross \
    binutils-riscv64-linux-gnu
```
## Applicazione
### Linux kernel patch
```
git clone https://github.com/starfive-tech/VisionFive2 -b JH7110_VisionFive2_6.6.y_devel
cd VisionFive2
git submodule update --init --recursive

export VF2_VERSION="JH7110_VF2_6.6_v6.0.0"

cd buildroot && git checkout $VF2_VERSION && cd ../u-boot && git checkout $VF2_VERSION && cd ../linux && git checkout $VF2_VERSION && cd ../opensbi && git checkout $VF2_VERSION && cd ../soft_3rdpart && git checkout $VF2_VERSION && cd ..

patch --dry-run -p1 < ../patch/linux-kernel-vf2.patch
patch -p1 < ../patch/linux-kernel-vf2.patch
```

### Mesa patch

```
wget https://archive.mesa3d.org//older-versions/22.x/mesa-22.1.7.tar.xz
tar -xf mesa-22.1.7.tar.xz -C .
cd mesa-22.1.7

patch --dry-run -p1 < ../patch/mesa-22.1.7-vf2.patch
patch -p1 < ../patch/mesa-22.1.7-vf2.patch
```

> [!NOTE]
> Per ottenere i file debian per pacchettizzare prendere da https://salsa.debian.org/xorg-team/lib/mesa/-/tree/debian-bookworm?ref_type=heads usato questo commit come base:
>```
> $ git log -1
> commit ebbc6e2147352d20950761c9b209ab66a8bc1677 (HEAD)  
> Author: Timo Aaltonen <tjaalton@debian.org>  
> Date:   Thu Aug 4 09:52:50 2022 +0300  
>   version bump
>```
### Qt Wayland

```
apt source qt6-wayland=6.10.2
cd qt6-wayland-6.10.2/
patch --dry-run -p1 < ../patch/qt6-wayland-6.10.2-vf2.patch
patch -p1 < ../patch/qt6-wayland-6.10.2-vf2.patch
```

### KWin wayland

```
apt source kwin=6.7.4
cd kwin-6.7.4/
patch --dry-run -p1 < ../patch/kwin-6.7.4-vf2.patch
patch -p1 < ../patch/kwin-6.7.4-vf2.patch
```

### Qt6 WebEngine

```
apt source qt6-webengine=6.10.2+dfsg
cd qt6-webengine-6.10.2+dfsg/
patch --dry-run -p1 < ../patch/qt6-webengine=6.10.2+dfsg-vf2.patch
patch -p1 < ../patch/qt6-webengine=6.10.2+dfsg-vf2.patch
```

> [!WARNING]
> Per compilare Qt6 WebEngine è necessario buildare tutto ffmpeg, chromium e v8, la compilazione richiederà tanto tempo!
> Inoltre ho fatto dei trucchi nella patch per permettere la cross compilazione da x86_64 a riscv64.
> Successivamente riporto parte di questi trucchi, anche se sarebbe da integrare meglio la cross compilazione.

Si pretende il gn distribuito con il pacchetto, lo buildo all'inizio su `/tmp`
```
cmake -S src/gn \
    -B /tmp/qtwebengine-gn \
    -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_INSTALL_PREFIX=/tmp/qtwebengine-gn-install
    
cmake --build /tmp/qtwebengine-gn --parallel
```

Sono richieste delle librerie sia come architettura nativa che come architettura di cross compilazione. I pacchetti debian non permettono di installare nel sistema entrambe le versioni del pacchetto, quindi le scarico su `/tmp` (le riscv64 in questo caso sono installate)

```
mkdir -p /tmp/nss-host/root 
cd /tmp/nss-host 
apt-get download libnss3-dev:amd64 libnss3:amd64 libnspr4-dev:amd64 libnspr4:amd64
for deb in *.deb; do 
	dpkg-deb -x "$deb" root/ 
done
```

## Installazione
Installare le dipendenze del pacchetto di riferimento e cross compilarlo.
```
apt build-dep --host-architecture riscv64 .
CC=riscv64-linux-gnu-gcc \
CXX=riscv64-linux-gnu-g++ \
DEB_BUILD_OPTIONS=nocheck \
debuild -us -uc -b --host-arch riscv64
```
> [!TIP]
> In base alla situazione, si possono valutare altri parametri per `debuild`:
> - `-nc` se hai già iniziato la compilazione e c'è stato un errore, puoi riprendere da dove eri arrivato perchè non pulisce la build
> - `-d` se ci sono dei problemi con delle dipendenze puoi provare a skippare la parte delle dipendenze debian
