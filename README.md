# 📄 Проект: Стенд Gentoo (musl + OpenRC)

## 1. Целевая архитектура и стек

Концепция системы строится вокруг максимальной легковесности, безопасности и отказа от монолитных компонентов (glibc, systemd, традиционные загрузчики) с полным использованием возможностей процессора Intel i7-1260P (Alder Lake) и 24 ГБ оперативной памяти.
 * Base OS: Gentoo Linux, профиль default/linux/amd64/23.0/musl/llvm.
 * Init & Service Manager: OpenRC + elogind (для управления сессиями Wayland).
 * Compiler Toolchain: LLVM/Clang (из категории llvm-core/), LLD, compiler-rt, libcxx.
 * Boot Chain: Direct EFI Stub \rightarrow UKI (Unified Kernel Image) \rightarrow Dracut.
 * Security & Crypto: Secure Boot (sbctl), LUKS2, автоматическая расшифровка через TPM 2.0 (фреймворк Clevis).
 * GUI Space: Wayland, композитор Niri.
 * Compatibility Layer: Distrobox (контейнеры с glibc для проприетарного софта).
 
## 2. Базовая конфигурация (make.conf)

При настройке тулчейна важно придерживаться строгого синтаксиса key="value" без использования export.

`/etc/portage/make.conf`

```bash
Оптимизации под Alder Lake
COMMON_FLAGS="-O3 -march=alderlake -pipe"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}"
FFLAGS="${COMMON_FLAGS}"

# Нативный LLVM тулчейн
CC="clang"
CXX="clang++"
AR="llvm-ar"
NM="llvm-nm"
RANLIB="llvm-ranlib"
LDFLAGS="${LDFLAGS} -fuse-ld=lld -rtlib=compiler-rt -unwindlib=libunwind -Wl,-O2 -Wl,--as-needed"

# Глобальные USE-флаги для минимизации зависимостей
USE="-systemd -X wayland musl llvm clang lto pgo elogind"
```

## 3. Ядро, Инициализация и Криптография

Сборка ядра осуществляется компилятором LLVM с включенным ThinLTO. Вся логика загрузки и расшифровки переносится в dracut.
 1. Привязка LUKS к TPM 2.0:

   Осуществляется через clevis (пакеты app-admin/clevis, sys-dracut/dracut-clevis).
  
   clevis luks bind -d /dev/nvme0n1pX tpm2 '{"pcr_ids":"7"}'
   
   
 2. Сборка UKI через Dracut:
 
   В /etc/dracut.conf.d/uki.conf задаются параметры для объединения ядра, initramfs и параметров загрузки (включая UUID зашифрованного корня) в единый linuxx64.efi. Обязательно подключаются модули clevis, tpm2-tss и crypt.
 
## 4. Настройка загрузки (Direct EFI Stub)

После успешной сборки UKI-образа, он подписывается собственными ключами и регистрируется напрямую в UEFI материнской платы.

 1. Подпись образа: sbctl sign -s /boot/EFI/Gentoo/kernel-musl.efi
 2. Регистрация записи в NVRAM:
  
```bash
efibootmgr --create --disk /dev/nvme0n1 --part 1 --label "Gentoo Musl (Direct)" --loader "/EFI/Gentoo/kernel-musl.efi"
```   
   
## 5. Пользовательское окружение и Wayland

Так как systemd-logind отсутствует, его функции по управлению доступом к устройствам ввода и GPU перехватывает elogind.
 * `Niri`: При переносе конфигурационного файла композитора стоит быть внимательным с синтаксисом (в частности, перепроверить блок с параметром allow-direct-scanout, чтобы избежать ошибок парсинга).
 * `Dotfiles`: Для синхронизации пользовательских конфигураций используется Chezmoi, но с осторожностью при накатывании профилей от основной glibc-системы.
 
## 6. Автоматизация (Снапшоты и Хуки)
Управление Btrfs-снапшотами (Snapper) перед операциями emerge будет осуществляться через пользовательские bash-хуки в /etc/portage/bashrc.
 * *Важно:* При написании bash-скриптов для OpenRC-окружения необходимо тщательно проверять условные конструкции на наличие всех необходимых логических операторов (&&, ||), так как ошибки синтаксиса могут прервать процесс сборки пакетов.
