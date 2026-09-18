# Документация администратора

Репозиторий документации в подходе Docs-as-Code: исходники в Markdown, публикация через статический генератор, изменения проходят через ветку и pull request.

## Структура

```
docs/
└── admin-guide/
    └── nginx-https-reverse-proxy.md   Настройка HTTPS на Nginx с reverse proxy
```

## Локальная проверка

- Линтер Markdown: `markdownlint docs/**/*.md`
- Проверка орфографии: `yaspeller docs/`
- Сборка и проверка битых ссылок: `npm run build`
