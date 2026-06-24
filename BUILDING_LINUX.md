# Building EddyPro GUI on Linux

Tested on Ubuntu 22.04 / 24.04 with GCC 11 and Qt 5.

## 1. Install dependencies

```bash
sudo apt install \
    qtbase5-dev \
    qtchooser \
    qt5-qmake \
    qtbase5-dev-tools \
    qttools5-dev-tools \
    libqt5svg5-dev \
    libquazip5-dev
```

## 2. Extract bundled fonts

The Open Sans fonts are bundled as a zip that must be extracted before building:

```bash
cd fonts/
unzip open-sans.zip
```

## 3. Compile the translation file

The Qt translation binary (`.qm`) must be generated from the source (`.ts`):

```bash
/usr/lib/qt5/bin/lrelease tra/eddypro_en.ts
```

## 4. Configure source code patches

The project was originally set up to use a bundled QuaZip build targeting CentOS.
Building against the system QuaZip on a modern Linux distribution requires three
small changes.

### 4a. `includes.pri` — add system QuaZip include path

Add the following line after the existing `INCLUDEPATH` entry:

```
linux: INCLUDEPATH += /usr/include/quazip5
```

### 4b. `libs.pri` — use system QuaZip library

Replace both Linux blocks (debug and release) with:

```
linux {
    # quazip (system install)
    LIBS += -lquazip5
}
```

This removes the reference to the non-existent CentOS pre-built QuaZip and the
`QMAKE_PRE_LINK` script that copied it.

### 4c. Missing `QPainterPath` includes

GCC 11 is stricter about transitive includes. Add `#include <QPainterPath>` to:

- `src/angles_view.cpp`
- `src/windfilter_view.cpp`

In both files, insert it after the existing `#include <QPainter>` line.

### 4d. Deprecated `QString::SkipEmptyParts`

Replace all occurrences of the deprecated `QString::SkipEmptyParts` with
`Qt::SkipEmptyParts` in:

- `src/globalsettings.cpp`
- `src/ancillaryfiletest.cpp`

### 4e. Crash when opening a project file (`updateFourthGasSettings`)

The application crashes with a segfault when opening an existing `.eddypro` file.
The root cause is a static initialisation order fiasco: `QStringLiteral` macros
and `Defs::SUBTWO`/`Defs::SUBTHREE` constants in `src/basicsettingspage.cpp` are
used inside `BasicSettingsPage::updateFourthGasSettings()`, which is triggered
early during startup before those static objects are initialised.

In `src/basicsettingspage.cpp`, replace the string constant block near the top of
`updateFourthGasSettings()` with runtime-constructed strings:

```cpp
// Use runtime string construction to avoid static initialisation order issues
const QChar subTwo(0x2082);
const QChar subThree(0x2083);
const QString N2OStr = QString(QLatin1Char('N')) + subTwo + QLatin1Char('O');
const QString COStr = QString::fromLatin1("CO");
const QString SO2Str = QString::fromLatin1("SO") + subTwo;
const QString O3Str = QString(QLatin1Char('O')) + subThree;
const QString NH3Str = QString::fromLatin1("NH") + subThree;
const QString NOStr = QString::fromLatin1("NO");
const QString NO2Str = QString::fromLatin1("NO") + subTwo;
```
These changes are implemented in the current version.

## 5. Build

```bash
mkdir -p build && cd build
/usr/lib/qt5/bin/qmake -r ../eddypro_lin.pro
make -j$(nproc)
```

The binary is produced at `build/release/eddypro`.

## 6. Integrate eddypro engine
Compile the eddypro engine from [Li-Cor Environmental Eddypro-engine repository](https://github.com/LI-COR-Environmental/eddypro-engine)
Copy the resulting engine binaries (e.g. from ~/bin/eddypro/bin/linux/) to a ./bin folder underneath eddypro-gui:
`cp ~/bin/eddypro/bin/linux/eddypro_rp  ~/bin/eddypro-gui/bin/`
`cp ~/bin/eddypro/bin/linux/eddypro_fcc ~/bin/eddypro-gui/bin/`

## 6. Run

```bash
./build/release/eddypro
```

