# XNU: условный эксперимент TTBR1 + TTBR0

## Что сделано
Отдельный opt-in сценарий software CPU-модели прошёл реальные setter-инструкции XNU для TTBR1_EL1 и TTBR0_EL1 и остановился **перед `TLBI VMALLE1IS`** внутри arm_vm_init. Это продолжение исследования, не аппаратное подтверждение корректного переключения MMU.

В локальном сценарии с точным kernel из CI 40287702717: TTBR1 `0x41377000 -> 0x4227b000`, TTBR0 `0x41376000 -> 0x4227a000`; 1 075 735 instruction-hook events, включая остановленную TLBI. Первый новый корень имеет 56 страниц таблиц, второй — одну пустую root-page. После обоих шагов модель защищает 61 страницу старых и новых таблиц; их хэши неизменны.

## Старые трансляции не выброшены
ISB не принимается за TLBI. Для каждой уже отображённой страницы сравниваются software walks старого и нового корня. Доступ остаётся разрешён только при совпадении PA и учитываемых атрибутов, кроме прав доступа; права берутся как пересечение прежних, старого walk и нового walk. Неизвестная/отсутствующая/отличающаяся трансляция блокируется, а не исправляется догадкой. Новые VA/PA mappings не создаются.

После TTBR1 остаются 10 106 usable aliases, блокируются 5254. После TTBR0 остаются 2426; дополнительно блокируются 7680 identity aliases. Пересечение первого шага сохраняется при втором. Эти цифры относятся к конкретному программному сценарию, не ко всей физической памяти Redmi.

Реально исполненные перед setter DSB SY для TTBR1 и DSB ISH для TTBR0 проверены по trace. После каждого MSR допускается только немедленная ISB SY, проверяемая по машинной кодировке; Capstone печатает эту ISB без operand. TCR/SCTLR остаются прежними. При следующей TLBI модель останавливается **до её эффекта**: ни инвалидация, ни её завершение не подделываются.

## Защита
Все старые и новые table-pages получают реальные RO-права Unicorn во всех оставшихся aliases. Несогласованные aliases получают NONE; адреса не переназначаются. Проверки выполняются до host-write DC ZVA. Root ASID/CnP, A1/AS, WXN, stage2/TGE, неполное состояние и NSTable inheritance не поддерживаются и отклоняются. Тесты исполняют настоящие STR/fetch, проверяя отказ без записи.

## Важное ограничение
Это **условная модель согласия двух software walks**, а НЕ полный архитектурный TLB-симулятор и НЕ доказательство допустимости переключения на Cortex-A53. Она не моделирует speculative table walks, walk caches, multicore, overlapping/multiple TLB hits и все требования break-before-make. Совпадение PA/атрибутов в этом эксперименте не заменяет эти проверки. В отчёте явно установлены `architecture_valid_switch_proven=false` и `unmodeled_walk_cache_speculation_and_tlb_conflicts=true`.

Архитектурные источники: [Arm barriers](https://support.arm.com/documentation/100941/0101/Barriers), [Arm TTBR1_EL1](https://support.arm.com/documentation/102670/0301/AArch64-registers/AArch64-register-descriptions/AArch64-Generic-System-control-register-description/TTBR1-EL1--Translation-Table-Base-Register-1--EL1-). Поэтому перенос выводов модели на телефон без аппаратной проверки запрещён.

## Воспроизведение
```sh
python tests/test_ttbr_consensus.py
python tests/test_audit_ttbr_candidate.py
python tests/emulate_image_handoff.py --build build/handoff-cpu-model --kernel build/macos-dandelion/xnu.obj/kernel.development.dandelion --virtual-bootstrap --cache-profile tests/fixtures/dandelion-cpu0-cache-profile.json --cache-profile-only --enter-arm-vm-model --ttbr-consensus-model --out build/ttbr-consensus-results.json
python tests/validate_ttbr_consensus.py build/ttbr-consensus-results.json --profile tests/fixtures/dandelion-cpu0-cache-profile.json
```

Без нового флага прежний arm_vm-сценарий по-прежнему останавливается перед TTBR1. Синтетические seed/ADT/счётчик, один CPU, отсутствие IRQ/ATF/watchdog/devices остаются ограничениями. **Нового образа для прошивки телефона нет; никаких новых действий на телефоне не требуется.**
