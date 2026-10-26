# XNU: условное завершение TLBI и physmap из таблиц

## Итог
Добавлены два отдельных opt-in сценария поверх прежнего TTBR consensus experiment:
1. `--tlbi-completion-model`: реальный код XNU проходит `TLBI VMALLE1IS; DSB ISH; ISB`. Без дополнительных aliases затем останавливается при обращении к реально созданному XNU physmap VA.
2. `--owned-physmap-model`: после той же условной последовательности модель обнаруживает aliases **из настоящих таблиц XNU**, разрешая только уже выделенную физическую RAM. Код проходит дальше и останавливается на защищённой попытке изменения активного PTE.

Это software experiments при явно НЕдоказанной архитектурной допустимости предыдущего switch/BBM. Ни аппаратное завершение TLBI, ни рабочий XNU на Redmi не заявляются.

## TLBI не no-op
Отдельная state machine различает issue, completion DSB, local context ISB и применение нового программного поколения. Старые возможные трансляции удерживаются до финальной ISB. ISB без completion DSB, DSB ISHST вместо completion, неупорядоченные stores, изменение контекста, неизвестные registers, другой TLBI или несколько CPU отклоняются. Поддержана ровно одна последовательность VMALLE1IS; повторные/другие maintenance остаются стопами.

Обычные guest stores и моделируемые DC ZVA учитываются как записи. Для issue необходим предшествующий подходящий DSB после записей. При pending maintenance разрешена только поддержанная последовательность барьеров. Это ограничение модели, не утверждение, что архитектура запрещает любые промежуточные инструкции.

Архитектурная основа: [Arm AArch64 TLB maintenance and barriers](https://developer.arm.com/documentation/ddi0487/mc/-Part-K-Appendixes/-Appendix-K11-Barrier-Litmus-Tests/-K11-5-Cache-and-TLB-maintenance-instructions-and-barriers/-K11-5-3-TLB-maintenance-instructions-and-barriers), [Arm barriers](https://support.arm.com/documentation/100941/0101/Barriers).

## Physmap не вычисляется догадкой
Первый базовый fault: VA `0xfffffff0260b3000`. Независимый walk показал PA `0x422b3000` внутри уже выделенного 32-MiB arena. Поэтому исправление не добавляет выдуманную RAM, а открывает XNU-created alias того же shared backing.

Обход обоих текущих деревьев ограничен table/page budgets. Для каждого разрешённого VA повторяется независимый software walk, проверяются owned PA, AF, права, Normal-WB MAIR=0xff и shareability. Чужие физические выходы, неподдержанные attributes и конфликты с original VA не отображаются. Все old/new table-pages остаются RO во всех aliases. Новые VA не могут перекрыть неизвестный host mapping. DC ZVA и обычные STR видят один shared backing.

Для локального точного kernel из CI 40287702717: обнаружены 8116 разрешённых aliases, из них 3978 новых виртуальных; **0 новых байтов физического backing**, 0 чужих leaf-выходов. Защищена 61 страница таблиц. Это не карта всей 2-GB памяти телефона — модель по-прежнему ограничена 32-MiB arena.

## Следующий конкретный блокер
XNU дошёл до `STR x8, [x27], #8` внутри arm_vm_init, предлагающего запись по PA `0x4227ca40` через physmap. Unicorn RO-защита отклонила store; все table hashes сохранились.

Read-only аудит заблокированного предложения:
- Старый descriptor: `0x0` — invalid.
- Предложенный: `0x18000000422c9003` — table descriptor уровня 2.
- Охватываемый диапазон начинается с VA `0xfffffff029000000`, span 2 MiB.
- Новый child-table PA `0x422c9000` находится в owned arena; сохранён его хэш.
- **Запись НЕ применена; subtree, publish ordering, TLBI requirements/BBM для предложения не объявлены проверенными.**

Полный сценарий: 1 088 162 instruction-hook events, 5006 моделируемых DC ZVA и 59 AT с проверенными возвратами XNU. Возврат из arm_vm_init НЕ достигнут. Цифры относятся к указанному локальному бинарнику; CI проверяет семантические критерии отдельно.

## Тесты и воспроизведение
Добавлены отдельные unit-тесты state machine, применения нового поколения, discovered physmap aliases и read-only PTE audit. Есть реальные исполнения fetch/STR: coherence, RO/PXN, table protection, запрет foreign PA, host collisions и отсутствие раннего commit.

```sh
python tests/test_tlbi_completion.py
python tests/test_post_tlbi_aliases.py
python tests/test_owned_physmap.py
python tests/test_pte_write_audit.py
python tests/emulate_image_handoff.py --build build/handoff-cpu-model --kernel build/macos-dandelion/xnu.obj/kernel.development.dandelion --virtual-bootstrap --cache-profile tests/fixtures/dandelion-cpu0-cache-profile.json --cache-profile-only --enter-arm-vm-model --ttbr-consensus-model --tlbi-completion-model --owned-physmap-model --out build/owned-physmap-results.json
python tests/validate_tlbi_physmap.py build/owned-physmap-results.json --profile tests/fixtures/dandelion-cpu0-cache-profile.json --expect-owned-physmap
```

Без новых флагов прежние защитные границы сохранены. Не моделируются full hardware TLB/walk-cache/speculation/multi-hit/BBM correctness, multicore, IRQ, ATF, watchdog и устройства. **Нового образа для прошивки нет. Телефон трогать и повторять probe не требуется.** Сырые expdb/Boot/Recovery/DTB не публикуются.
