# XNU: cache-init с измеренными регистрами dandelion

Аппаратный регистровый probe из CI 40287702717 успешно оставил полный CPU0-снимок в предоставленном пользователем expdb. Сырой дамп повторно декодирован с точным манифестом: профиль полностью совпал, build-tag `0xdc1e1897024cc6af`, один успешный кадр, CSSELR восстановлен. Это свидетельство выполнения **регистрового probe**, не загрузки XNU/iOS. Источник — снимок пользователя, не криптографическая аттестация.

## Измерено
- Cortex-A53 MIDR 0x410fd034, CNTFRQ 13 MHz.
- CLIDR 0x0a200023, CTR 0x84448004, DCZID 0x4.
- L1D 32 КиБ / 4 ways / 128 sets, L1I 32 КиБ / 2 ways / 256 sets.
- L2 unified 512 КиБ / 16 ways / 512 sets; все строки 64 байта.
- Дополнительно сохранены DFR/PFR/ISAR/MMFR/REVIDR. В tests/fixtures размещены только выбранные архитектурные регистры и хэши происхождения. **Сырой expdb, Boot/Recovery/DTB и персональные данные не опубликованы.**

## Что изменилось в модели
Реальные CLIDR/CCSIDR сняли прежний стоп в `machine_read_clidr`. Затем обнаружен отсутствующий `DAIF` в модели. Он не заполнен произвольным нулём: состояние теперь выводится из проверенного SPSR_EL2=0x3c5 при моделируемом ERET, с корректными DAIFSet/DAIFClr. Снятие IRQ/FIQ-масок останавливает модель: доставки IRQ/FIQ нет. Для этих правил добавлены отдельные отрицательные unit-тесты.

XNU в программной модели прошёл `do_cacheid`, `machine_do_debugid` и дошёл **до входа в arm_vm_init**. Проверена не только трасса: из памяти XNU прочитан 48-байтный cache_info_t, установленный c_valid и вычисленные самим ядром размеры кэшей/line-size/associativity сравнены с измеренными CCSIDR.

Локальный прогон на точном kernel из CI 40287702717: 733131 срабатывание instruction-hook (включая остановку перед первой инструкцией arm_vm_init); целевой стоп `before_arm_vm_init`, без memory fault/panic. Число инструкций относится только к этому бинарнику и сценарию, не является аппаратным временем.

## Границы результата
- Это **software CPU/MMU alias model**, не аппаратное включение MMU и не аппаратный ERET в XNU.
- Счётчик по-прежнему явно синтетический; IRQ, GIC, ATF и watchdog как аппаратные устройства не моделируются.
- ADT/топология/seed и владение RAM в модели синтетические; один CPU/кластер. Поддержка SMP не заявляется.
- Тело arm_vm_init **ещё не исполняется**. Смена таблиц/TCR/TTBR требует отдельной когерентной модели; существующие защиты от записей таблиц и выхода за RAM не сняты.
- USB, storage, display/touch, scheduler и iOS userspace не реализованы этим изменением.
- Нового образа для прошивки телефона в этом этапе нет. **Повторять регистровый probe и прошивать CPU-MODEL/ядро не нужно.**

## Воспроизведение
После восстановления слоёв исходников workflow и сборки CPU-модели:

```sh
python tests/test_pstate_masks.py
python tests/test_cache_state.py
python tests/emulate_image_handoff.py --build build/handoff-cpu-model --kernel build/macos-dandelion/xnu.obj/kernel.development.dandelion --virtual-bootstrap --cache-profile tests/fixtures/dandelion-cpu0-cache-profile.json --cache-profile-only --out build/measured-cache-bootstrap-results.json
python tests/validate_measured_bootstrap.py build/measured-cache-bootstrap-results.json --profile tests/fixtures/dandelion-cpu0-cache-profile.json
```

Для emulator нужны SHA-pinned зависимости из ci/image-test-requirements.txt. Обычные сценарии без профиля остаются отдельными регрессионными проверками. Неизвестные регистры по-прежнему вызывают остановку, DC ZVA по-прежнему не пропускается как no-op.
