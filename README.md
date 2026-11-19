# pr-review-bot

Автоматическое ревью pull request'ов двумя ревьюерами параллельно — для сравнения:

- **Claude Code** ([`anthropics/claude-code-action`](https://github.com/anthropics/claude-code-action)) —
  агентное ревью: сам ходит по коду репозитория, читает связанные файлы;
- **pr-agent** ([`qodo-ai/pr-agent`](https://github.com/qodo-ai/pr-agent)) — собирает
  diff с контекстом и делает фиксированный набор запросов к модели.

Оба работают через [OpenRouter](https://openrouter.ai) на одной и той же модели.

Ревьюятся **только PR из форков, которые не являются черновиками**. Когда черновик
переводят в Ready for review, ревью запускается автоматически.

## Как это устроено

```
project/.github/workflows/ai-review.yml        (pull_request_target, короткий caller)
        │
        ├─ job claude ──► pr-review-bot/.github/workflows/claude-review.yml
        │                   ├─ checkout base-ветки в корень
        │                   ├─ checkout кода PR в pr-head/ (только чтение)
        │                   └─ claude-code-action: inline-комментарии + итоговый комментарий
        │
        └─ job pr-agent ─► pr-review-bot/.github/workflows/pr-agent-review.yml
                            └─ pr-agent: /review + /improve (код получает через GitHub API)
```

Промпт и настройки живут здесь и действуют для всех подключённых проектов.

## Подключение проекта

1. Добавьте секрет `OPENROUTER_API_KEY` в репозиторий проекта
   (Settings → Secrets and variables → Actions). Ключ создаётся на
   https://openrouter.ai/keys — задайте для него лимит расходов.
2. Добавьте переменную `REVIEW_MODEL` (Settings → Secrets and variables → Actions →
   Variables) — ID модели Claude из https://openrouter.ai/models, без префикса
   `openrouter/`. Её используют оба ревьюера.
3. Если `pr-review-bot` приватный: в его Settings → Actions → General → Access
   разрешите доступ из других своих репозиториев. Иначе сделайте его публичным.
4. Скопируйте `examples/ai-review.yml` в `.github/workflows/ai-review.yml` проекта.
   Если нужен только один ревьюер, удалите второй job.
5. По желанию добавьте в корень проекта `CLAUDE.md` с правилами ревью
   (шаблон — `examples/CLAUDE.md`). Его читает только Claude Code.

Устанавливать GitHub App не нужно: оба ревьюера публикуют комментарии через
`GITHUB_TOKEN` от имени `github-actions[bot]`. Отличить их можно по оформлению:
у pr-agent заголовки «PR Reviewer Guide» и «PR Code Suggestions».

## Настройки

Задаются в caller через `with:`.

**claude-review.yml**

| Параметр | По умолчанию | Описание |
|---|---|---|
| `model` | модель action | ID модели в формате OpenRouter |
| `max_turns` | `30` | Максимум шагов агента — ограничивает стоимость |
| `extra_instructions` | — | Дополнительный текст в конец промпта |

**pr-agent-review.yml**

| Параметр | По умолчанию | Описание |
|---|---|---|
| `model` | — (обязателен) | ID модели в формате OpenRouter, без префикса `openrouter/` |
| `max_input_tokens` | `128000` | Лимит входных токенов. Нужен, если pr-agent не знает модель; не больше контекста модели. Больший diff обрезается |

pr-agent запускает автоматически `/review` и `/improve`. `/describe` выключен, чтобы
не переписывать описание PR постороннего автора. Команды в комментариях
(`issue_comment`) не подключены — иначе их мог бы запускать кто угодно.

## Как сравнивать

1. Обоим ревьюерам задана одна модель — разница показывает подход
   (агент против заранее собранного контекста), а не силу модели.
2. Откройте из форка несколько тестовых PR:
   - баг, заметный только из кода за пределами diff (изменился контракт функции,
     вызывающий код не обновлён);
   - локальный баг прямо в diff;
   - уязвимость (например, SQL-конкатенация с пользовательским вводом);
   - чистый PR без проблем — для оценки ложных срабатываний.
3. Сравните: найденные реальные проблемы, шумные и ложные замечания, стоимость
   (в OpenRouter; заведите два ключа, если нужна раздельная статистика), время работы.

## OpenRouter

- Используйте модели Claude: Claude Code рассчитан на них, с моделями других
  провайдеров агентный режим может работать некорректно.
- В настройках OpenRouter поставьте Anthropic первым в приоритете провайдеров —
  совместимость гарантируется только с ним.
- Код PR проходит через OpenRouter. По умолчанию OpenRouter не логирует промпты,
  не включайте prompt logging для этого ключа.

## Безопасность

PR из форков пишут посторонние люди, а `pull_request_target` даёт workflow доступ
к секретам репозитория. Поэтому:

- код PR **никогда не исполняется** — никаких install/build/test;
- Claude Code:
  - код PR кладётся в `pr-head/` (`allow-unsafe-pr-checkout: true` — допустимо,
    потому что он только читается); `CLAUDE.md`, `CLAUDE.local.md` и `.claude/`
    из него удаляются, чтобы автор PR не мог подложить агенту «правила проекта»;
  - агенту разрешены только `Read`, `Grep`, `Glob`, `gh pr view/diff/comment` и
    inline-комментарии; `Write`, `Edit` и `git push` запрещены;
  - чтение вне workspace запрещено (`blockReadsOutsideWorkingDirectories` плюс
    deny-правила на `/proc`, `/sys` и `$RUNNER_TEMP`, `gh pr comment --body-file`),
    чтобы prompt injection не могла прочитать ключи из окружения и опубликовать их;
  - в промпте указано, что содержимое PR — недоверенные данные;
- pr-agent: checkout не делается вовсе, код читается через GitHub API;
- `GITHUB_TOKEN` с минимальными правами (`contents: read`, `issues: write`,
  `pull-requests: write`), без PAT.

Остаточный риск — prompt injection в коде или описании PR. Худший реалистичный
сценарий — бот оставит в PR бессмысленный или вредный комментарий. Для ключа
`OPENROUTER_API_KEY` обязательно выставьте лимит расходов в настройках OpenRouter.

**Не добавляйте** в эти workflow шаги, которые запускают код из PR.
