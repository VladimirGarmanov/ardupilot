# ArduPilot: обзор репозитория для старта

Этот файл написан как карта по текущему репозиторию ArduPilot. Фокус: Copter,
Plane, VTOL/QuadPlane и MAVLink. Если твой опыт ближе к Arduino и
микроконтроллерам, то полезная аналогия такая: здесь тоже есть `setup`/`loop`,
драйверы, параметры и обмен по UART, но все разнесено по большому C++ проекту,
планировщику задач и HAL-слою под разные платы.

## На чем написано

Основной полетный код написан на C/C++.

- C++: автопилот, режимы полета, контроллеры, MAVLink, HAL, датчики.
- C: часть низкоуровневых библиотек, сторонние модули, сгенерированный MAVLink.
- Python: сборочные/тестовые/симуляционные инструменты, `sim_vehicle.py`,
  генераторы параметров, автотесты.
- Lua: пользовательские скрипты ArduPilot Scripting.
- XML: описания MAVLink-сообщений и часть конфигураций.
- Shell/PowerShell: установка окружения, CI, вспомогательные скрипты.

По локальному дереву видно тысячи файлов `.h`, `.cpp`, `.c`, `.py`, `.xml`.
Главная сборочная система - `waf`, файл запуска лежит в корне: `./waf`.

## Главная структура репозитория

- `ArduCopter/` - приложение для мультикоптеров и вертолетов.
- `ArduPlane/` - приложение для самолетов, включая QuadPlane/VTOL.
- `Rover/`, `ArduSub/`, `Blimp/`, `AntennaTracker/` - другие типы техники.
- `libraries/` - общие библиотеки, из которых собраны все аппараты.
- `libraries/AP_HAL*` - аппаратная абстракция: UART, I2C, SPI, GPIO, таймеры,
  storage, scheduler для разных платформ.
- `libraries/AP_HAL_ChibiOS/` - слой для STM32/ChibiOS плат типа Pixhawk/Cube.
- `libraries/AP_HAL_Linux/` - слой для Linux-плат.
- `libraries/AP_HAL_SITL/` и `libraries/SITL/` - симулятор.
- `libraries/GCS_MAVLink/` - общий код MAVLink/GCS.
- `modules/mavlink/` - MAVLink как submodule: XML-диалекты, генераторы,
  `pymavlink`, C-заголовки.
- `Tools/autotest/` - SITL, автотесты, стандартные `.parm` для симуляции.
- `Tools/environment_install/` - скрипты установки зависимостей под macOS,
  Ubuntu, Windows и другие системы.
- `libraries/AP_HAL_ChibiOS/hwdef/` - описания плат: пины, UART, SPI, I2C,
  IMU, baro, flash, bootloader и т.п.
- `BUILD.md` - основная локальная инструкция по сборке.

## Как запускается полетный код

В Arduino ты обычно видишь `setup()` и `loop()`. В ArduPilot вход более общий:

- `ArduCopter/Copter.cpp` содержит таблицу задач Copter и в конце
  `AP_HAL_MAIN_CALLBACKS(&copter)`.
- `ArduPlane/Plane.cpp` создает глобальный объект `Plane plane`.
- `ArduPlane/ArduPlane.cpp` содержит таблицу задач Plane и в конце
  `AP_HAL_MAIN_CALLBACKS(&plane)`.
- `libraries/AP_HAL/AP_HAL_Main.h` раскрывает макросы, которые создают обычный
  `main(...)`.
- `libraries/AP_Vehicle/` содержит базовую логику общего жизненного цикла
  аппарата.

То есть программа стартует через HAL, создает объект аппарата, и дальше
планировщик (`AP_Scheduler`) вызывает задачи с заданной частотой.

Пример из Copter: быстрые задачи обновляют IMU, rate controller, моторы, AHRS,
инерциальную навигацию, режим полета; более медленные задачи читают RC, GPS,
батарею, MAVLink, логирование и т.д.

Пример из Plane: быстрые задачи обновляют AHRS, режим управления, стабилизацию
и сервы; периодические задачи читают радио/GPS/airspeed, считают навигацию,
обрабатывают MAVLink, failsafe и логирование.

## Copter: где что лежит

Главные файлы:

- `ArduCopter/Copter.cpp` - таблица задач планировщика.
- `ArduCopter/Copter.h` - главный класс Copter и его поля.
- `ArduCopter/system.cpp` - инициализация системы.
- `ArduCopter/mode.h` - базовые классы режимов.
- `ArduCopter/mode_*.cpp` - конкретные режимы: `stabilize`, `althold`,
  `loiter`, `auto`, `guided`, `rtl`, `land`, `acro` и т.д.
- `ArduCopter/Attitude.cpp` - логика управления ориентацией.
- `ArduCopter/motors.cpp`, `motor_test.cpp` - моторы и тест моторов.
- `ArduCopter/navigation.cpp` - навигационная логика.
- `ArduCopter/failsafe.cpp`, `crash_check.cpp`, `ekf_check.cpp` - защиты.
- `ArduCopter/Parameters.cpp`, `Parameters.h` - параметры Copter.
- `ArduCopter/GCS_Mavlink.cpp`, `GCS_Copter.cpp` - Copter-специфичная часть
  MAVLink/GCS.

Общие библиотеки, важные для Copter:

- `libraries/AC_AttitudeControl/` - attitude/rate control для коптера.
- `libraries/AC_WPNav/` - навигация по точкам для коптера.
- `libraries/AP_Motors/` - микширование моторов, frame class/type, PWM/DShot.
- `libraries/AP_InertialNav/`, `AP_AHRS/`, `AP_NavEKF2/`, `AP_NavEKF3/` -
  оценка положения/ориентации.
- `libraries/AP_GPS/`, `AP_Compass/`, `AP_Baro/`, `AP_InertialSensor/` -
  датчики.

Типичный поток управления Copter:

1. HAL читает датчики и RC.
2. EKF/AHRS оценивает положение, скорость и ориентацию.
3. Активный `Mode*` решает, какие целевые углы/скорости/позиции нужны.
4. `AC_AttitudeControl` превращает целевые углы в rate/throttle.
5. `AP_Motors` микширует команды на моторы.
6. HAL выводит PWM/DShot на железо.

## Plane: где что лежит

Главные файлы:

- `ArduPlane/Plane.cpp` - объект `Plane`.
- `ArduPlane/Plane.h` - главный класс Plane и его поля.
- `ArduPlane/ArduPlane.cpp` - таблица задач планировщика.
- `ArduPlane/mode.h`, `mode_*.cpp` - режимы самолета: `manual`, `stabilize`,
  `fbwa`, `fbwb`, `cruise`, `auto`, `guided`, `rtl`, `loiter`, `takeoff` и др.
- `ArduPlane/Attitude.cpp` - управление креном/тангажом/рысканьем.
- `ArduPlane/navigation.cpp`, `commands_logic.cpp` - миссии и навигация.
- `ArduPlane/servos.cpp` - вывод на сервы/рули/газ.
- `ArduPlane/altitude.cpp`, `takeoff.cpp`, `is_flying.cpp` - высота, взлет,
  определение полета.
- `ArduPlane/Parameters.cpp`, `Parameters.h` - параметры Plane.
- `ArduPlane/GCS_Mavlink.cpp`, `GCS_Plane.cpp` - Plane-специфичный MAVLink/GCS.

Общие библиотеки, важные для Plane:

- `libraries/APM_Control/` - PID/контроллеры fixed-wing.
- `libraries/AP_L1_Control/` - L1-навигация по маршруту.
- `libraries/AP_TECS/` - управление энергией самолета: скорость/высота/газ.
- `libraries/AP_Airspeed/` - датчик воздушной скорости.
- `libraries/SRV_Channel/` - назначение функций на серво-выходы.

Типичный поток управления Plane:

1. AHRS/EKF оценивает положение и ориентацию.
2. Режим полета задает цели: курс, высота, скорость, roll/pitch.
3. L1 считает боковую навигацию к waypoint.
4. TECS считает высоту/скорость/газ.
5. Fixed-wing контроллеры считают рули.
6. `servos.cpp` отправляет команды на `SRV_Channel`.

## VTOL / QuadPlane

VTOL в ArduPilot для самолетов находится внутри `ArduPlane`, не в
`ArduCopter`. Это важно: QuadPlane - это Plane с дополнительной мультикоптерной
частью.

Главные файлы:

- `ArduPlane/quadplane.cpp`, `quadplane.h` - ядро QuadPlane/VTOL.
- `ArduPlane/transition.h` - состояния перехода между VTOL и fixed-wing.
- `ArduPlane/tiltrotor.cpp`, `tiltrotor.h` - tiltrotor.
- `ArduPlane/tailsitter.cpp`, `tailsitter.h` - tailsitter.
- `ArduPlane/VTOL_Assist.cpp`, `VTOL_Assist.h` - помощь VTOL-моторами в
  самолетном режиме.
- `ArduPlane/mode_qstabilize.cpp`, `mode_qhover.cpp`, `mode_qloiter.cpp`,
  `mode_qland.cpp`, `mode_qrtl.cpp`, `mode_qacro.cpp`, `mode_qautotune.cpp` -
  Q-режимы.
- `Tools/autotest/default_params/quadplane*.parm`,
  `plane-tailsitter.parm` - готовые параметры для SITL/тестов.

Ключевые параметры QuadPlane объявлены в `QuadPlane::var_info` внутри
`ArduPlane/quadplane.cpp`. У них обычно префикс `Q_`:

- `Q_ENABLE` - включает QuadPlane.
- `Q_FRAME_CLASS`, `Q_FRAME_TYPE` - тип мультикоптерной рамы для VTOL-моторов.
- `Q_ANGLE_MAX` - максимальный наклон в VTOL-режимах.
- `Q_TRANSITION_MS` - время перехода после достижения минимальной скорости.
- `Q_ASSIST_SPEED` - скорость, ниже которой VTOL-моторы помогают самолету.
- `Q_RTL_ALT`, `Q_RTL_MODE` - логика возврата для VTOL.
- `Q_GUIDED_MODE` - использование VTOL в Guided.
- `Q_ESC_CAL` - калибровка ESC VTOL-моторов.
- `Q_PILOT_SPD_UP`, `Q_PILOT_SPD_DN`, `Q_PILOT_ACCEL_Z` - вертикальная скорость
  и ускорение от пилота.

По смыслу QuadPlane работает так:

1. В fixed-wing режимах самолет летит как обычный Plane через TECS/L1/сервы.
2. В Q-режимах он использует мультикоптерную часть: `AC_AttitudeControl`,
   `AC_WPNav`, `AP_Motors`.
3. При переходе вперед VTOL-моторы держат аппарат, пока самолет набирает
   достаточную скорость, затем управление переходит к крылу.
4. При переходе назад fixed-wing часть снижает скорость/заходит на посадку,
   затем VTOL-моторы берут висение/вертикальную посадку.

## MAVLink: где сделан протокол обмена

В репозитории есть две разные части MAVLink:

- `modules/mavlink/` - upstream MAVLink: XML-описания сообщений, генераторы,
  `pymavlink`, диалекты `common.xml`, `ardupilotmega.xml` и др.
- `libraries/GCS_MAVLink/` - интеграция MAVLink в ArduPilot: прием команд,
  отправка телеметрии, параметры, миссии, маршрутизация, FTP, signing.

Основные файлы ArduPilot MAVLink:

- `libraries/GCS_MAVLink/GCS_Common.cpp` - общий прием/отправка сообщений для
  всех аппаратов.
- `libraries/GCS_MAVLink/GCS_MAVLink.cpp`, `.h` - канал MAVLink, UART, parse,
  stream rates.
- `libraries/GCS_MAVLink/GCS_Param.cpp` - чтение/запись параметров по MAVLink.
- `libraries/GCS_MAVLink/MissionItemProtocol*.cpp` - загрузка/скачивание миссий,
  rally, fence.
- `libraries/GCS_MAVLink/MAVLink_routing.cpp` - маршрутизация сообщений между
  каналами.
- `ArduCopter/GCS_Mavlink.cpp` и `ArduPlane/GCS_Mavlink.cpp` - сообщения и
  команды, специфичные для конкретного аппарата.

Как это работает:

1. `AP_SerialManager` по параметрам `SERIALx_PROTOCOL`, `SERIALx_BAUD`,
   `SERIALx_OPTIONS` находит UART/порт MAVLink.
2. `GCS_MAVLINK::init()` открывает порт, задает baudrate и режим MAVLink1/2.
3. В scheduler регулярно вызываются `GCS::update_receive` и `GCS::update_send`.
4. На входе байты парсятся в `mavlink_message_t`.
5. `GCS_Common.cpp` обрабатывает общие сообщения: heartbeat, параметры,
   миссии, command_long/int, timesync, logs, FTP и т.п.
6. Если сообщение зависит от типа аппарата, обработка уходит в Copter/Plane
   классы.
7. На выходе ArduPilot отправляет `HEARTBEAT`, `ATTITUDE`,
   `GLOBAL_POSITION_INT`, `VFR_HUD`, `SYS_STATUS`, `BATTERY_STATUS`,
   `RC_CHANNELS`, `SERVO_OUTPUT_RAW`, `PARAM_VALUE` и другие сообщения.

Параметры MAVLink/телеметрии обычно настраиваются не в коде, а через параметры:

- `SYSID_THISMAV` - system id аппарата.
- `SYSID_MYGCS` - id наземной станции.
- `SERIALx_PROTOCOL` - что висит на UART: MAVLink, GPS, RC и т.д.
- `SERIALx_BAUD` - скорость UART.
- `SRx_*` - частоты потоков телеметрии для MAVLink-канала.
- `MAV_OPTIONS`, signing/options - настройки поведения MAVLink, если включены в
  конкретной сборке.

## Где искать параметры

Параметры в ArduPilot - это не `#define`, а runtime-настройки, которые хранятся
во flash/EEPROM/файловом storage и доступны через Mission Planner, MAVProxy,
QGroundControl, MAVLink `PARAM_*`.

Основные места:

- `ArduCopter/Parameters.cpp`, `.h` - параметры Copter.
- `ArduPlane/Parameters.cpp`, `.h` - параметры Plane.
- `ArduPlane/quadplane.cpp` - параметры QuadPlane с префиксом `Q_`.
- `libraries/*/*_params.cpp` или `var_info[]` внутри библиотек - параметры
  библиотек.
- `libraries/AP_Param/` - система регистрации, хранения и доступа к параметрам.
- `Tools/autotest/default_params/*.parm` - готовые наборы параметров для SITL.
- `Tools/Frame_params/*.param` - параметры для конкретных рам/готовых моделей.

В коде параметры описываются макросами вроде:

- `GSCALAR(...)`, `ASCALAR(...)` - скалярный параметр в основном списке.
- `AP_GROUPINFO(...)` - параметр внутри группы.
- `AP_SUBGROUPPTR(...)`, `AP_SUBGROUPVARPTR(...)` - подключение параметров
  вложенной библиотеки.

Комментарии `@Param`, `@DisplayName`, `@Description`, `@Range`, `@Units`,
`@Values`, `@Bitmask` используются генераторами документации параметров.

## Сборка

Сборка делается из корня репозитория через `./waf`.

Проверить платы:

```sh
./waf list_boards
```

SITL-сборка Copter:

```sh
./waf configure --board sitl
./waf copter
```

SITL-сборка Plane:

```sh
./waf configure --board sitl
./waf plane
```

Сборка под CubeBlack/Pixhawk-подобную плату:

```sh
./waf configure --board CubeBlack
./waf copter
```

Или Plane:

```sh
./waf configure --board CubeBlack
./waf plane
```

После сборки бинарники лежат в:

```text
build/<board>/bin/
```

Примеры:

- `build/sitl/bin/arducopter`
- `build/sitl/bin/arduplane`
- `build/CubeBlack/bin/arducopter`
- `build/CubeBlack/bin/arduplane`

Не запускай `waf` через `sudo`: это ломает права и окружение.

## Установка зависимостей

Скрипты окружения лежат здесь:

- macOS: `Tools/environment_install/install-prereqs-mac.sh`
- Ubuntu: `Tools/environment_install/install-prereqs-ubuntu.sh`
- Windows: `Tools/environment_install/install-prereqs-windows.ps1`

Для macOS скрипт ставит Homebrew-зависимости, Python-пакеты, MAVProxy,
`pymavlink`, `pexpect`, `empy==3.3.4`, `dronecan`, опционально STM32
toolchain. В текущем локальном окружении `Tools/autotest/sim_vehicle.py --help`
упал с `ModuleNotFoundError: No module named 'pexpect'`, значит Python-зависимости
для SITL установлены не полностью.

Минимально для SITL нужны Python-пакеты из скрипта окружения, особенно:

```text
future lxml pymavlink MAVProxy pexpect geocoder empy==3.3.4 dronecan
```

## Загрузка на железо

Для Pixhawk/Cube/STM32-плат типовой путь такой:

1. Узнать имя платы через `./waf list_boards`.
2. Сконфигурировать сборку:

```sh
./waf configure --board CubeBlack
```

3. Собрать нужный аппарат:

```sh
./waf copter
```

4. Подключить полетный контроллер USB.
5. Залить:

```sh
./waf --targets bin/arducopter --upload
```

Для Plane:

```sh
./waf plane
./waf --targets bin/arduplane --upload
```

Для Linux-плат типа Navio2 используется `rsync`-загрузка:

```sh
./waf configure --board navio2 --rsync-dest root@192.168.1.2:/
./waf --targets bin/arducopter --upload
```

Альтернативный практичный путь: собрать `.apj`/firmware и загрузить его через
Mission Planner/QGroundControl, если плата и bootloader это поддерживают.

## SITL: запуск кастомной сборки в симуляции

SITL - это та же логика ArduPilot, но HAL не STM32, а симулятор. Очень удобно
для проверки кода до реального железа.

Обычный запуск Copter:

```sh
Tools/autotest/sim_vehicle.py -v ArduCopter
```

Обычный запуск Plane:

```sh
Tools/autotest/sim_vehicle.py -v ArduPlane
```

QuadPlane/VTOL:

```sh
Tools/autotest/sim_vehicle.py -v ArduPlane -f quadplane
```

Tailsitter:

```sh
Tools/autotest/sim_vehicle.py -v ArduPlane -f tailsitter
```

Если хочешь явно пересобрать перед запуском:

```sh
Tools/autotest/sim_vehicle.py -v ArduCopter --rebuild
Tools/autotest/sim_vehicle.py -v ArduPlane --rebuild
Tools/autotest/sim_vehicle.py -v ArduPlane -f quadplane --rebuild
```

После старта обычно открывается MAVProxy. К аппарату можно подключать Mission
Planner/QGroundControl по UDP. Частые адреса/порты SITL:

- MAVProxy master: TCP `127.0.0.1:5760`
- UDP out для GCS часто `127.0.0.1:14550`

Если запускаешь в виртуальной машине:

1. Собери и запускай SITL внутри VM.
2. Открой/пробрось UDP-порт `14550` или TCP `5760` наружу.
3. В GCS на хосте подключись к IP виртуалки и нужному порту.
4. Для VirtualBox/VMware проще всего использовать bridged network или явно
   настроить port forwarding.
5. Если GCS тоже внутри VM, ничего пробрасывать не нужно: подключайся к
   `127.0.0.1`.

Для запуска кастомной сборки важно понимать: `sim_vehicle.py --rebuild`
пересобирает текущий код из рабочей директории. То есть меняешь код,
запускаешь `--rebuild`, получаешь SITL именно из твоей ветки.

## Что читать первым

Если цель - разобраться как разработчик, хороший порядок такой:

1. `BUILD.md` - сборка и upload.
2. `ArduCopter/Copter.cpp` и `ArduPlane/ArduPlane.cpp` - scheduler.
3. `ArduCopter/mode.h`, затем один простой режим, например
   `mode_stabilize.cpp`.
4. `ArduPlane/mode_fbwa.cpp`, `mode_auto.cpp`, `Attitude.cpp`.
5. `ArduPlane/quadplane.cpp` и `mode_q*.cpp` для VTOL.
6. `libraries/GCS_MAVLink/GCS_Common.cpp` и `GCS_Param.cpp` для MAVLink.
7. `libraries/AP_Param/` для параметров.
8. `libraries/AP_HAL/AP_HAL_Main.h` и нужный HAL: `AP_HAL_ChibiOS` или
   `AP_HAL_SITL`.

## Ментальная модель для Arduino-разработчика

- `setup()` примерно соответствует инициализации в vehicle/HAL/system-коде.
- `loop()` заменен на `AP_Scheduler`, где каждая задача имеет частоту и
  бюджет времени.
- `Serial` заменен на `AP_HAL::UARTDriver`, а выбор назначения порта идет
  через `AP_SerialManager` и параметры `SERIALx_*`.
- `digitalWrite/PWM` заменены на HAL RCOutput и `SRV_Channel`/`AP_Motors`.
- Датчики не читаются напрямую из режима полета: обычно данные проходят через
  драйвер, AHRS/EKF и общие библиотеки.
- Настройки не хардкодятся в одном `.ino`: они объявляются в `AP_Param` и
  меняются с наземной станции.
- Реальное время критично: нельзя просто добавить долгий `delay()` или тяжелый
  блок в быстрый цикл.

## Где вносить изменения

- Новый режим Copter: смотреть `ArduCopter/mode.h` и существующие
  `mode_*.cpp`.
- Новый режим Plane: смотреть `ArduPlane/mode.h` и `mode_*.cpp`.
- Изменить VTOL-логику: сначала `ArduPlane/quadplane.cpp`, затем Q-режимы и
  transition/tailsitter/tiltrotor по типу аппарата.
- Добавить параметр: соответствующий `Parameters.cpp` или `var_info[]`
  библиотеки.
- Добавить MAVLink-команду без изменения протокола: обработка в
  `GCS_Common.cpp` или vehicle-specific `GCS_Mavlink.cpp`.
- Добавить новое MAVLink-сообщение: менять XML в `modules/mavlink`, генерировать
  заголовки и добавлять обработку/отправку в `libraries/GCS_MAVLink`.
- Добавить новую плату: `libraries/AP_HAL_ChibiOS/hwdef/<BoardName>/hwdef.dat`.

## Быстрые команды

```sh
# список плат
./waf list_boards

# Copter SITL
./waf configure --board sitl
./waf copter

# Plane SITL
./waf configure --board sitl
./waf plane

# запуск Copter SITL с пересборкой
Tools/autotest/sim_vehicle.py -v ArduCopter --rebuild

# запуск Plane SITL
Tools/autotest/sim_vehicle.py -v ArduPlane --rebuild

# запуск QuadPlane/VTOL SITL
Tools/autotest/sim_vehicle.py -v ArduPlane -f quadplane --rebuild

# сборка под CubeBlack и upload Copter
./waf configure --board CubeBlack
./waf copter
./waf --targets bin/arducopter --upload

# сборка под CubeBlack и upload Plane
./waf configure --board CubeBlack
./waf plane
./waf --targets bin/arduplane --upload
```

