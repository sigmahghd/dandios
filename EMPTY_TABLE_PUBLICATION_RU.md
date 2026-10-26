# XNU: возврат из arm_vm_init в условной программной модели

## Новый результат
В отдельном opt-in сценарии `--empty-table-publication-model` настоящий код XNU **вернулся из arm_vm_init**. Проверяется не только достижение адреса: зафиксированы RET внутри `_arm_vm_init`, восстановленный entry-SP, target равный сохранённому LR и исполнение следующей инструкции в caller. Это результат Unicorn/software model, НЕ аппаратная загрузка на Redmi.

До возврата модель выполнила 10 узко разрешённых переходов `0 -> empty table` в активных L2 entries. После возврата исполнение продолжилось и остановилось при попытке добавить внешнюю framebuffer page. Запись этой page НЕ применена.

Для точного локального kernel из CI 40287702717: RET-hook count 1 094 089, caller hook 1 094 090, конечный hook count 1 101 482. Новые таблицы увеличили защищённый inventory с 61 до 71 страницы; aliases и физический backing не расширялись.

## Это не снятие RO со всех PTE
Исходная guest-инструкция поддержана только в форме `STR Xt, [Xn], #8` с разными source/base registers. До её исполнения:
- Parent обязан быть singly referenced TTBR1 L2 entry с **точным нулём**, а его page hash — совпадать с разрешённым состоянием.
- Descriptor допускает только поддержанные table bits и owned aligned child PA.
- Child обязан целиком состоять из 4096 нулевых байтов, не быть активной таблицей, кодом, стеком, live critical memory или чужой RAM.
- **Все aliases child получают реальные RO-права Unicorn до записи parent pointer.** Активные parents RO не теряют.
- Только прошедший проверки 8-byte store моделируется через host-write; исходная STR пропускается, а post-index writeback выполняется и проверяется ровно один раз.

После каждой операции сверяется вся parent-page с ожидаемым результатом ровно этой записи, сохраняются before/after SHA256 и хэш пустого child. Новая child-page включается в защищённый inventory. Обычные STR/STP и DC ZVA в неё по-прежнему запрещены.

## Карантин и честный учёт изменений
Добавление пустой таблицы не создаёт ни одного valid leaf mapping. Затронутые 2-MiB виртуальные диапазоны остаются в явном quarantine; AT в них отклоняется, host mappings не добавляются. Будущие populate/update/flush автоматически не разрешаются.

В отчёте **`protected_tables_unchanged=false`**, поскольку 10 parent descriptors действительно изменены в модели. Отдельно `protected_tables_match_authorized_state=true` означает совпадение всех страниц с журналируемым разрешённым состоянием. Это не выдаётся за полную неизменность. Hash chain и новые empty children независимо проверяются валидатором.

Ни ordering для аппаратной публикации, ни BBM/speculative walks/walk caches, ни multicore этим не доказаны. Работа идёт поверх ранее объявленного условного TTBR/TLBI-сценария. Старые сценарии без нового флага сохраняют прежние защитные остановки.

## Следующий фактический блокер — framebuffer mapping
После возврата из arm_vm_init XNU предлагает page descriptor `0x6000007ec50607`:
- Output PA `0x7ec50000`, 4 KiB — вне owned 32-MiB arena.
- Этот PA совпадает с framebuffer constant в `platform/dandelion_fbcon.h`, но это **не разрешение** считать внешнюю память обычной RAM модели.
- AttrIndx=1, фактический MAIR byte=0x44, AF=1, PXN/UXN=1.
- Пустой child entry должен стать `invalid -> page`. Данный переход не входит в разрешённые empty-table publications: **store блокируется RO-защитой**.

0x44 — Normal Inner/Outer Non-cacheable, а не Device-nGnRnE: [Arm MAIR initialization](https://developer.arm.com/documentation/102416/0201/Single-level-table-at-EL3/Initialize-the-MAIR). Отдельная модель framebuffer/reservation/NC attributes пока не добавлена. Пиксели, вывод XNU на настоящем экране и работа iOS не заявляются.

## Воспроизведение
```sh
python tests/test_empty_table_publication.py
python tests/emulate_image_handoff.py --build build/handoff-cpu-model --kernel build/macos-dandelion/xnu.obj/kernel.development.dandelion --virtual-bootstrap --cache-profile tests/fixtures/dandelion-cpu0-cache-profile.json --cache-profile-only --enter-arm-vm-model --ttbr-consensus-model --tlbi-completion-model --owned-physmap-model --empty-table-publication-model --out build/empty-table-publication-results.json
python tests/validate_empty_publication.py build/empty-table-publication-results.json --profile tests/fixtures/dandelion-cpu0-cache-profile.json
```

Тесты включают точный STR/writeback, hash chain, RO aliases, quarantine, запрет populated/executable/foreign children, неподдержанных encodings, контекстных изменений, DC ZVA и неконтролируемых записей. Доступны полный журнал и отчёт заблокированного framebuffer PTE.

**Нового образа для прошивки нет. Не повторять probe и не менять Boot/LK/vbmeta. Телефон сейчас трогать не требуется.**
