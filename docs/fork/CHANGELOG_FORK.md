# Changelog форка (macOS-порт)

Все значимые изменения в форке `Nodal.AcadMcp` документируются здесь.
Формат основан на [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
версии следуют [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

Основной апстрим-changelog проекта ведётся в `CHANGELOG.md`; этот файл
описывает только изменения, внесённые в рамках форка и портирования на macOS.

## [Unreleased]

### Added
- База документации форка:
  - `docs/fork/ADR/0001-macos-porting-strategy.md` — ADR со стратегией
    портирования на macOS под AutoCAD 2027 при сохранении обратной
    совместимости с Windows.
  - `docs/fork/TODO.md` — пошаговый план портирования.
  - `docs/fork/CHANGELOG_FORK.md` — этот файл.
- `docs/PROJECT_GUIDE.md` — гид по репозиторию для нового разработчика:
  суть проекта и стек, карта репозитория, жизненный цикл запроса, устройство
  инструментов и навыков, сборка/запуск/отладка и точки входа для новичков.

### Documentation
- Правила `.kilocode/rules/fork-docs.md`, обязывающие в будущих задачах
  обновлять `TODO.md` и `CHANGELOG_FORK.md` и создавать новые ADR при
  архитектурных изменениях.