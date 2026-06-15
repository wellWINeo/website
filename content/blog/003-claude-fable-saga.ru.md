+++
title = "Claude Fable Saga или как мир становится все более разделенным"
date = 2026-06-15
description = "Разбираю историю с отключением Claude Fable 5 и Mythos 5 — и пытаюсь понять, что это было: геополитика или удар по Anthropic."
+++

# Предыстория

> Можно смело скипать, если вы в контексте истории

- **7 апреля** — Anthropic [анонсировала](https://www.anthropic.com/project/glasswing) Claude Mythos и программу Project Glasswing: доступ к модели только у ~50 отобранных партнёров (Amazon, Apple, Microsoft и ИБ-компании), потому что Mythos умеет сам находить уязвимости и писать эксплойты.
- **7 апреля, тот же день** — группа в Discord [получила доступ](https://www.techradar.com/pro/security/mythos-accessed-by-unauthorized-users-as-anthropic-says-were-investigating-cracks-may-be-showing-in-project-glasswing-as-unknown-users-access-model-via-third-parties) к Mythos Preview, угадав URL по соглашениям об именовании Anthropic.
- **21 апреля** — [второй инцидент](https://thenextweb.com/news/anthropic-mythos-unauthorized-access-vendor-breach): другая группа пробралась через среду стороннего вендора.
- **9 июня** — [вышел Claude Fable 5](https://www.anthropic.com/news/claude-fable-5-mythos-5): публичная обрезанная версия Mythos с guardrails. Опасные запросы автоматически перенаправляются в Opus 4.8
- **10 июня** — [выяснилось](https://fortune.com/2026/06/10/anthropic-accu-claude-fable-5-limits-capabilities-ai-researchers-developers/), что Fable 5 ещё и тихо саботировал разработку конкурирующих моделей — без предупреждений, просто работал хуже. Anthropic [признала ошибку](https://www.engadget.com/2192004/anthropic-walks-back-policy-sabotaging-research/) и пообещала сделать все ограничения явными.
- **12 июня** — исследователи Amazon нашли джейлбрейк, CEO Энди Джасси [доложил в Белый дом](https://techcrunch.com/2026/06/13/amazon-ceo-reportedly-raised-anthropic-model-concerns-before-government-crackdown/). Министр торговли Лутник выпустил [директиву об экспортном контроле](https://www.tomshardware.com/tech-industry/artificial-intelligence/us-export-control-order-forces-anthropic-to-disable-claude-fable-5-and-mythos-5-worldwide): отрубить доступ всем иностранным гражданам, включая сотрудников самой Anthropic. Поскольку compliant-вариантов не осталось — [обе модели отключили](https://9to5mac.com/2026/06/12/anthropic-pulls-claude-mythos-5-and-claude-fable-5-following-us-government-directive/) глобально. Anthropic [не согласна](https://www.tomshardware.com/tech-industry/artificial-intelligence/trump-adviser-david-sacks-says-anthropic-refused-to-fix-fable-5-jailbreak-before-us-export-controls), но подчинилась.

# С какой стороны можно взглянуть на это?

## Разделение мира: ограничение доступа к frontier-моделям

Изначально, когда еще вышел ChatGPT 3.5, меня удивляло, что у любого
человека в мире есть свободный доступ к этой технологии. Технологии,
которая определяет все наше десятилетие, если не век.

Но на тот момент LLM-модели представляли из себя "продвинутые болталки",
ну да, может генерировать текст, выдавать саммари, генерировать отдельные
куски кода, быстро что-то подсказывать, но в целом, это все было не столь
интересным.

Действительно что-то интересное начало появляться за последние год-два,
теперь LLM - не просто болталка, а сердце агентов, анализировать
ситуацию, контекст, собирать и систематизировать данные, принимать
решения и использовать инструменты для достижения целей.

И тут становится интересной новость, что военные США использовали
[Claude Opus 4.6 для планирования военных операций](https://www.scientificamerican.com/article/anthropics-safety-first-ai-collides-with-the-pentagon-as-claude-expands-into/). Это уже, по
сути, превращает LLM в технологию двойного назначения.

И у любого человека в мире есть, в целом, равный доступ к этой
технологии (не будем учитывать лимиты/токены) - у высшего военного
руководства, у инженера big-tech компании или даже у самого простого
человека из какой-то глухой страны, если он смог найти 20$ и оплатить
Claude Pro?

Это меня долгое время поражало и было понимание, что продлится это 
недолго. ИИ становится главным оружием 21-го века, причиной новой
холодной войны и доступ к frontier-моделям будут предоставляться 
только союзникам.

Эту версию косвенно подтверждает [материал Semafor](https://www.semafor.com/article/06/13/2026/white-house-move-to-limit-anthropic-linked-to-concerns-about-chinese-access-to-mythos):
реальным триггером для директивы стала не столько сама уязвимость в Fable 5,
сколько опасение, что китайские лаборатории могут добраться до Mythos и
дистиллировать его. Опасения небеспочвенны — ещё в феврале [Anthropic обвинила
DeepSeek, Moonshot и MiniMax](https://techcrunch.com/2026/02/23/anthropic-accuses-chinese-ai-labs-of-mining-claude-as-us-debates-ai-chip-exports/)
в том, что те качали данные через 24 000 фейковых аккаунтов. DeepSeek уже
показал, что, имея в разы меньше ресурсов, можно добиться весьма сопоставимого
качества именно таким путём — а Mythos, напомним, умеет писать эксплойты.


## Умышленные палки в колеса Anthropic

С другой стороны, у Anthropic и правительства США уже много 
месяцев очень натянутые отношения. Пентагон [внёс Anthropic в список угроз национальной безопасности](https://www.cnbc.com/2026/04/08/anthropic-pentagon-court-ruling-supply-chain-risk.html)
27 февраля 2026 года — исторически такой статус получали только иностранные
компании вроде Huawei. Anthropic пошла в суд, апелляционный суд отказал во
временной блокировке, суды продолжаются.

Сейчас оба ведущих игрока, [OpenAI и Anthropic, готовятся к IPO](https://finance.yahoo.com/markets/article/spacex-openai-and-anthropic-here-are-the-most-anticipated-ipos-in-2026-114439441.html)
до конца года — оба подали конфиденциальные S-1 в июне.
Выпуск Claude Fable 5 — мощный положительный драйвер для оценки Anthropic.

В это же время, спустя всего двое суток, правительство США, которое
уже судится с Anthropic, выпускает директиву об ограничении
распространения их флагманской модели?

И триггером стал именно Amazon — крупнейший инвестор Anthropic, её облачный
провайдер. Это Энди Джасси лично позвонил в Белый дом и сообщил о джейлбрейке.
Компания, которая вложила в Anthropic $4 млрд, фактически запустила shutdown
её флагманского продукта. Совпадение?

Отдельная ирония — сам Пентагон. Он внёс Anthropic в список угроз, судится с
ней месяцами, а [потом всё равно использовал Claude для координации ударов по
Ирану](https://www.washingtonpost.com/technology/2026/03/04/anthropic-ai-iran-campaign/).
То есть модель достаточно хороша, чтобы на неё опираться в реальных военных
операциях — но недостаточно безопасна, чтобы к ней имел доступ условный
разработчик из Берлина.
