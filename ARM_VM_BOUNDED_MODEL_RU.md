# XNU: ограниченный проход внутрь arm_vm_init

## Результат
В отдельном opt-in software CPU/MMU-сценарии XNU теперь исполняет начало **тела** arm_vm_init, готовит новые таблицы и доходит до `_set_mmu_ttb_alternate`. Остановка происходит **перед записью TTBR1_EL1**. Новый корень не активирован; завершение arm_vm_init, аппаратная MMU и загрузка XNU на Redmi этим не заявляются.

На локальном точном kernel из CI 40287702717: 3540 моделируемых DC ZVA и 58 AT S1E1R; во всех 58 вызовах настоящий код XNU вернул ожидаемый PA. Аудит новой TTBR1-кандидатуры: 56 страниц таблиц, 2495 leaf-дескрипторов; следующие инструкция/caller LR/стек/boot_args сохраняют PA и необходимые права. Все leaf-выходы этой кандидатуры входят в ограниченную модель RAM. Эти числа относятся только к указанному бинарнику/сценарию, а не к работающей прошивке и не к аппаратному времени.

## Исправлена настоящая ошибка защиты модели
Проверено на Unicorn 2.1.4: `emu_stop()` из pre-write hook **не отменяет** соответствующую запись. Прежнего hook-запрета было бы недостаточно для продолжения VM-init. Прежние сценарии до VM-init активные PTE не переписывали.

Теперь активные table-pages защищены реальными RO-правами эмулятора во **всех** физических/виртуальных aliases. STR/STP в них отклоняются механизмом защиты памяти. В конце прогона хэши активных table-pages сверяются с исходными. Это дополнительная защита модели, а не утверждение, что сами аппаратные дескрипторы XNU имеют RO.

## DC ZVA не является no-op
Эффект определяется измеренным DCZID=0x4: 64-байтный выровненный блок. До host-write проверены writable alias, фактические права Unicorn, owned RAM, отсутствие active table-page, HCR TDZ/TGE и DZP. MMIO/чужая RAM/RO/активные PTE отклоняются. Все aliases делят backing; translation blocks Unicorn по каждому alias инвалидируются, чтобы не исполнять старый код после обнуления. Это модель записи RAM, не аппаратная cache-coherency.

## AT/PAR без выдуманного NS
AT S1E1R идёт по действительным таблицам, созданным XNU, и проверяет выход в известный owned alias. Поддержаны только нужные этому сценарию Normal-WB attributes; неизвестные fault status, modes/attributes не заполняются нулями. Ожидается ISB перед чтением PAR.

Успешный PAR формируется из PA/MAIR/shareability и RES1. **NS в Non-secure state оставлен явно UNKNOWN**, а не выдан за измеренный 0/1. Перед MRS PAR проверяются следующие четыре реальные инструкции XNU: restore DAIF, TBNZ только известного F, BFXIL с заменой всех low-12 bits, AND address mask. BFXIL уничтожает неизвестный NS до использования адреса. Unit-тест исполняет настоящий BFXIL/AND с обоими NS=0/1 и получает одинаковый PA. При другом потребителе модель останавливается. На RET каждой функции сравнивается реально вычисленный XNU результат с software walk.

Архитектурная основа: [Arm PAR_EL1 reference](https://support.arm.com/documentation/100048/0002/system-control/aarch64-register-descriptions/physical-address-register--el1) и [Arm AT S1E1R](https://support.arm.com/documentation/ddi0601/2025-09/AArch64-Instructions/AT-S1E1R--Address-Translate-Stage-1-EL1-Read). Старое описание PAR консервативно используется для неизвестного NS; поддержки расширений новых CPU не заявляется.

## Аудит кандидатуры не равен её активации
Аудитор read-only: проходит дерево с ограничением количества страниц/циклов, проверяет расположение table-pages в arena, выравнивания и критические PA/права. Чужие leaf-выходы, если появятся, учитываются отдельно и **не отображаются** в host RAM. В текущем принятом сценарии их нет. Ни TTBR, ни host mappings аудитор не меняет.

Следующий отдельный барьер — корректный switch и TLB/barrier semantics, включая старые cached translations и новые права. Автоматически считать ISB полным TLBI или просто подменить TTBR в словаре нельзя. Поэтому текущий защитный стоп сохранён.

## Воспроизведение
Нужны восстановленные слои исходников workflow, собранный CPU MODEL, точный kernel и pinned emulator dependencies.

```sh
python tests/test_virtual_bootstrap_memory.py
python tests/test_model_dc_zva.py
python tests/test_model_at_par.py
python tests/test_audit_ttbr_candidate.py
python tests/emulate_image_handoff.py --build build/handoff-cpu-model --kernel build/macos-dandelion/xnu.obj/kernel.development.dandelion --virtual-bootstrap --cache-profile tests/fixtures/dandelion-cpu0-cache-profile.json --cache-profile-only --enter-arm-vm-model --out build/arm-vm-bounded-results.json
python tests/validate_arm_vm_probe.py build/arm-vm-bounded-results.json --profile tests/fixtures/dandelion-cpu0-cache-profile.json
```

Обычный measured-cache сценарий без нового флага по-прежнему останавливается до arm_vm_init. Синтетический счётчик/ADT/seed/один CPU, отсутствие IRQ/ATF/watchdog/devices и bounded RAM остаются ограничениями. Сырой expdb, Boot/Recovery/DTB не публикуются. **Нового образа для прошивки телефона нет. Повторять аппаратный регистровый probe не требуется.**
