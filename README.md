# pr-review-bot

Автоматическое агентное ревью pull request'ов с помощью Claude Code
([`anthropics/claude-code-action`](https://github.com/anthropics/claude-code-action)):
агент сам ходит по коду репозитория, читает связанные файлы и оставляет
inline-комментарии и итоговый комментарий. Запросы к модели идут через
[OpenRouter](https://openrouter.ai).

Ревьюятся **только PR из форков, которые не являются черновиками**. Когда черновик
переводят в Ready for review, ревью запускается автоматически.

## Как это устроено

```
project/.github/workflows/ai-review.yml        (pull_request_target, короткий caller)
        │  uses: nikita715/pr-review-bot/.github/workflows/claude-review.yml@main
        ▼
pr-review-bot/.github/workflows/claude-review.yml   (reusable)
        ├─ checkout base-ветки в корень
        ├─ checkout кода PR в pr-head/ (только чтение)
        └─ claude-code-action: inline-комментарии + итоговый комментарий
```

Промпт и настройки живут здесь и действуют для всех подключённых проектов.

## Подключение проекта

1. Добавьте секрет `OPENROUTER_API_KEY` в репозиторий проекта
   (Settings → Secrets and variables → Actions). Ключ создаётся на
   https://openrouter.ai/keys — задайте для него лимит расходов.
2. По желанию добавьте переменную `REVIEW_MODEL` (Settings → Secrets and
   variables → Actions → Variables) — ID модели из https://openrouter.ai/models.
   Если её нет, используется модель по умолчанию в action.
3. Если `pr-review-bot` приватный: в его Settings → Actions → General → Access
   разрешите доступ из других своих репозиториев. Иначе сделайте его публичным.
4. Скопируйте `examples/ai-review.yml` в `.github/workflows/ai-review.yml` проекта.
5. По желанию добавьте в корень проекта `CLAUDE.md` с правилами ревью
   (шаблон — `examples/CLAUDE.md`).

Устанавливать Claude GitHub App не нужно: комментарии публикуются через
`GITHUB_TOKEN` от имени `github-actions[bot]`.

## Настройки

Задаются в caller через `with:`.

| Параметр | По умолчанию | Описание |
|---|---|---|
| `model` | модель action | ID модели в формате OpenRouter |
| `max_turns` | `30` | Максимум шагов агента — ограничивает стоимость |
| `extra_instructions` | — | Дополнительный текст в конец промпта |

Если успешная сессия вышла за `max_turns` (финальный ответ считается лишним ходом),
action падает, но шаг `Check review result` проверяет лог сессии и засчитывает
такое ревью как успешное с предупреждением. Настоящие ошибки по-прежнему валят job.

## OpenRouter

- Лучше всего работают модели Claude: Claude Code рассчитан на них, с моделями
  других провайдеров агентный режим может работать некорректно.
- В настройках OpenRouter поставьте Anthropic первым в приоритете провайдеров —
  совместимость гарантируется только с ним.
- Код PR проходит через OpenRouter. По умолчанию OpenRouter не логирует промпты,
  не включайте prompt logging для этого ключа.

## Безопасность

PR из форков пишут посторонние люди, а `pull_request_target` даёт workflow доступ
к секретам репозитория. Поэтому:

- код PR **никогда не исполняется** — никаких install/build/test;
- код PR кладётся в `pr-head/` (`allow-unsafe-pr-checkout: true` — допустимо,
  потому что он только читается); из него удаляются все симлинки (они могут
  указывать за пределы workspace), а также `CLAUDE.md`, `CLAUDE.local.md` и
  `.claude/`, чтобы автор PR не мог подложить агенту «правила проекта»;
- агенту разрешены только `Read`, `Grep`, `Glob`, `gh pr view/diff`,
  inline-комментарии и `Write` в единственный файл `.ai-review-summary.md`;
  `Edit`, `gh pr comment` и `git push` запрещены. Итоговый комментарий из этого
  файла публикует отдельный шаг workflow;
- чтение вне workspace запрещено (`blockReadsOutsideWorkingDirectories` плюс
  deny-правила на `/proc`, `/sys` и `$RUNNER_TEMP`), чтобы prompt injection не
  могла прочитать ключи из окружения и опубликовать их;
- в промпте указано, что содержимое PR — недоверенные данные;
- `GITHUB_TOKEN` с минимальными правами (`contents: read`, `pull-requests: write`),
  без PAT.

Остаточный риск — prompt injection в коде или описании PR. Худший реалистичный
сценарий — бот оставит в PR бессмысленный или вредный комментарий. Для ключа
`OPENROUTER_API_KEY` обязательно выставьте лимит расходов в настройках OpenRouter.

**Не добавляйте** в этот workflow шаги, которые запускают код из PR.
