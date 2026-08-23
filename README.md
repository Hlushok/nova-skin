# Nova Skin mirror

Автоматичне byte-for-byte дзеркало плагіна Nova Skin для Lampa.

## Підключення

https://hlushok.github.io/nova-skin/nova_skin.js

## Походження

- Авторське джерело: https://github.com/amikdn/amikdn.github.io
- Оригінальний файл: https://github.com/amikdn/amikdn.github.io/blob/main/nova_skin.js
- Це GitHub-форк і дзеркало, а не переписаний плагін.

У перевіреному upstream-репозиторії не знайдено файла ліцензії. Цей форк не
додає власної ліцензії та не заявляє авторство на upstream-код.

## Автоматичне оновлення

Workflow перевіряє upstream щогодини на 17-й хвилині UTC та підтримує ручний
запуск. Він завантажує лише nova_skin.js з зафіксованого commit SHA, перевіряє
доставку і JavaScript-синтаксис, але не виконує та не аудитить поведінку коду.

GitHub може вимкнути cron у неактивному публічному репозиторії. У такому разі
workflow треба знову ввімкнути в Actions.

## Межі

Інші upstream-файли не переносяться. Lampac bridge зберігається окремо в
Hlushok/lampa-plugin. Автоматичне встановлення у Lampa або розгортання на VPS
не входить до цього репозиторію.
