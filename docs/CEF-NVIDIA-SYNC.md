# CEF с исправленной синхронизацией NVIDIA

Тестовый архив этого проекта использует **stock CEF 156.0.14**. Он содержит native-handle
поддержку NVIDIA из [CEF #4238](https://github.com/chromiumembedded/cef/pull/4238), но остаётся
подвержен [CEF #4313](https://github.com/chromiumembedded/cef/issues/4313): Chromium может
передать DMA-BUF до завершения записи GPU. Возможны мерцание и частично нарисованные кадры.
`glFinish()` в obs-browser завершает копирование **потребителя**, но не создаёт отсутствующую
зависимость от записи **производителя** в другом процессе. CPU fallback не обнаруживает каждый
такой повреждённый кадр.

Для исправления нужен собственный CEF с патчем [#4314](https://github.com/chromiumembedded/cef/pull/4314).
Он ожидает завершения GPU-записи перед передачей native-handle кадра. Исправление также принято
в Chromium как [`3cd7aa260699`](https://chromium.googlesource.com/chromium/src/+/3cd7aa26069901a37f73477e381d01842a8072ce)
для ветки 158; это не исправляет готовые бинарники ветки 156.

В репозитории сохранён [точный патч](../patches/viz_capture_native_handle_4313.patch) из CEF PR
на revision `7b7922813eb877afd648e0daeca2e10fdc71d2c8`:

```text
SHA256: 4972339464d6338436ac2d97d90e8fc612085a8fcd9461e718897887c5c1edf8
CEF:    4fa4ab516dacf6c49fc7e6e70d2b7ebce29fa35f
Chrome: 156.0.8078.48
```

Проверено применение всех трёх изменений через `git apply --check -p0` к исходникам именно
Chromium `156.0.8078.48`. **Полная сборка этого CEF и работа патча на NVIDIA здесь не проверены.**

## Сборка зависимости

Это отдельная длительная сборка Chromium, а не пересборка всего OBS. Используйте Ubuntu 24.04
x86_64 по [официальному CEF quick start](https://chromiumembedded.github.io/cef/master_build_quick_start#linux-setup).
Зарезервируйте 150–200 GB свободного места с запасом и несколько часов; объём и время зависят
от машины. Желательно 32 GB RAM или больше. Ниже команды для нового каталога сборки, запускать
из клона obs-browser. Привилегии нужны только для установки системных зависимостей Ubuntu.

```bash
export CEF_SYNC_PATCH="$PWD/patches/viz_capture_native_handle_4313.patch"
export CEF_WORK="$HOME/cef-nvidia-build"
mkdir -p "$CEF_WORK/automate" "$CEF_WORK/chromium_git"
echo "4972339464d6338436ac2d97d90e8fc612085a8fcd9461e718897887c5c1edf8  $CEF_SYNC_PATCH" |
  sha256sum -c -

sudo apt-get update
sudo apt-get install -y curl git file lsb-release procps python3 python3-pip
curl -fsSL \
  'https://chromium.googlesource.com/chromium/src/+/refs/tags/156.0.8078.48/build/install-build-deps.py?format=TEXT' |
  base64 -d > "$CEF_WORK/install-build-deps.py"
sudo python3 "$CEF_WORK/install-build-deps.py" --no-arm --no-chromeos-fonts --no-nacl

git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git "$CEF_WORK/depot_tools"
export PATH="$CEF_WORK/depot_tools:$PATH"
curl -fsSL \
  https://raw.githubusercontent.com/chromiumembedded/cef/4fa4ab516dacf6c49fc7e6e70d2b7ebce29fa35f/tools/automate/automate-git.py \
  -o "$CEF_WORK/automate/automate-git.py"

python3 "$CEF_WORK/automate/automate-git.py" \
  --download-dir="$CEF_WORK/chromium_git" --depot-tools-dir="$CEF_WORK/depot_tools" \
  --branch=8078 --checkout=4fa4ab516dacf6c49fc7e6e70d2b7ebce29fa35f \
  --x64-build --no-build --no-distrib
```

Зарегистрируйте патч в обеих копиях CEF: исходной для automate и рабочей внутри Chromium.
Это идемпотентное добавление в `patch.cfg`; штатный CEF patcher исполнит его перед GN/компиляцией.

```bash
python3 - <<'PY'
import os
from pathlib import Path

root = Path(os.environ["CEF_WORK"])
patch = Path(os.environ["CEF_SYNC_PATCH"]).read_bytes()
name = "viz_capture_native_handle_4313"
for cef in (root / "chromium_git/cef", root / "chromium_git/chromium/src/cef"):
    config = cef / "patch/patch.cfg"
    scope = {}
    exec(compile(config.read_bytes(), str(config), "exec"), scope)
    (cef / f"patch/patches/{name}.patch").write_bytes(patch)
    if not any(item["name"] == name for item in scope["patches"]):
        with config.open("a") as output:
            output.write(f"\npatches.append({{'name': {name!r}}})\n")
PY

export GN_DEFINES='use_sysroot=true is_component_build=false symbol_level=0 blink_symbol_level=0 v8_symbol_level=0 use_thin_lto=false is_cfi=false use_remoteexec=false'
python3 "$CEF_WORK/automate/automate-git.py" \
  --download-dir="$CEF_WORK/chromium_git" --depot-tools-dir="$CEF_WORK/depot_tools" \
  --branch=8078 --checkout=4fa4ab516dacf6c49fc7e6e70d2b7ebce29fa35f \
  --x64-build --no-update --force-build --no-debug-build \
  --build-target=cefsimple --minimal-distrib-only
```

`--no-update` сохраняет локальный патч. `cefsimple` подходит для sysroot и не тянет все CEF
тесты; Linux `chrome_sandbox` automate добавляет автоматически. Готовый minimal distribution
будет в `"$CEF_WORK/chromium_git/chromium/src/cef/binary_distrib/cef_binary_..._linux64_minimal"`.
Перед использованием проверьте, что рабочий файл
`chromium_git/chromium/src/components/viz/service/frame_sinks/video_capture/frame_sink_video_capturer_impl.cc`
содержит `/*populates_mappable_shared_image=*/true` в native-handle blit request. Сохраните лог
сборки, revision CEF и SHA256 патча вместе с бинарником.

## Сборка плагина с собственным CEF

Перенесите minimal distribution на Arch/CachyOS и соберите плагин против установленного OBS 33
и Qt. Подставьте абсолютный путь к **собственной** зависимости:

```bash
cmake -S . -B build-sync -G Ninja \
  -DCEF_ROOT_DIR=/absolute/path/to/cef_binary_156.0.14_linux64_minimal \
  -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr \
  -DOBS_BROWSER_BUILD_TESTS=OFF -DOBS_BROWSER_ENABLE_DEVTOOLS=OFF
cmake --build build-sync -j "$(nproc)"
bash scripts/package-test-build.sh build-sync dist-sync
```

Точное имя каталога содержит полный CEF/Chromium version string; пример выше — только путь
для замены. Не запускайте `scripts/fetch-cef.sh` для этого варианта: он скачает stock CEF без
патча. Добавьте копию патча и его SHA256 к передаваемому архиву, чтобы его нельзя было спутать
со stock-кандидатом. Затем повторите проверки кадров под нагрузкой, resize, hide/show, звука
и завершения OBS на целевой NVIDIA. Наличие патча само по себе не подтверждает эти проверки.
