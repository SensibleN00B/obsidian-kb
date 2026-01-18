---
tags:
  - scrum
  - agile
  - project-management
aliases:
  - Scrum Framework
  - Скрам
created: 2025-01-10
topic: Agile Methodologies
---

# <img src="https://img.icons8.com/?id=43096&format=png&size=48" width="48" height="48" alt="Scrum" style="vertical-align: middle;"/> Scrum

> [!SUMMARY] TL;DR
> Scrum - Agile framework для ітеративної розробки. Короткі sprints (1-4 тижні), 3 ролі (PO, SM, Developers), 5 events, 3 artifacts. Базується на емпіризмі: transparency → inspection → adaptation.

## 1. Основи

### Три стовпи
- **Transparency** - процес видимий для всіх
- **Inspection** - регулярна перевірка прогресу
- **Adaptation** - швидке коригування

### П'ять цінностей
Commitment, Focus, Openness, Respect, Courage

---

## 2. Ролі

### Product Owner
- Максимізує цінність продукту
- Управляє Product Backlog
- Приймає рішення про features
- Одна людина з повноваженнями

### Scrum Master
- Servant leader (НЕ PM)
- Усуває blockers
- Facilitate events
- Coaching команди в Scrum

### Developers
- Створюють Increment
- Self-managing (3-9 людей)
- Cross-functional
- Відповідальні як команда

---

## 3. Events

### Sprint (1-4 тижні)
Контейнер для всіх events. Фіксована тривалість.

### Sprint Planning (max 8h)
**WHY** → Sprint Goal  
**WHAT** → вибір items  
**HOW** → plan delivery

### Daily Scrum (15 хв)
Sync команди:
1. Що зробив вчора?
2. Що сьогодні?
3. Є blockers?

### Sprint Review (max 4h)
Demo Increment + feedback stakeholders

### Sprint Retrospective (max 3h)
Обговорення покращень:
- Start doing
- Stop doing
- Continue doing

---

## 4. Artifacts

### Product Backlog
- Впорядкований список всього що потрібно
- Ownership: Product Owner
- Commitment: **Product Goal**

### Sprint Backlog
- Sprint Goal + вибрані items + plan
- Ownership: Developers
- Commitment: **Sprint Goal**

### Increment
- Working software (usable)
- Відповідає Definition of Done
- Commitment: **Definition of Done**

**Приклад DoD:**
```
✅ Code review done
✅ Tests pass (>80% coverage)
✅ Deployed to staging
✅ No critical bugs
✅ Documentation updated
```

---

## 5. Практика

### Story Points Estimation
Relative estimate (Fibonacci: 1,2,3,5,8,13,21)

**Planning Poker:**
1. Read story
2. Discuss
3. Everyone picks card (secretly)
4. Reveal simultaneously
5. Discuss extremes
6. Re-vote до consensus

### Velocity
Average story points per Sprint (для planning)

### Burndown Chart
Залишок роботи vs час Sprint'у

---

## 6. Типові проблеми

> [!WARNING] Анті-патерни

**"Daily 45 хв"**
- Fix: таймер 15 хв, детальні дискусії after

**"Sprint Goal = list tickets"**
- Fix: describe business value, не tasks

**"No Sprint Retrospective changes"**
- Fix: concrete action items з assignee

**"PO never available"**
- Fix: PO має бути accessible для команди

**"Scope changes mid-Sprint"**
- Fix: protect Sprint Goal, зміни тільки між Sprints

---

## 7. Порівняння

| Scrum | Kanban | Waterfall |
|-------|--------|-----------|
| Fixed sprints | Continuous | Phases |
| 3 roles | Flexible | PM roles |
| Sprint boundaries | Anytime | Upfront |
| Story points | Cycle time | Hours |

---

**Пов'язані теми:**
- [[SDLC]]
- [[Agile Methodologies]]
- [[Kanban]]
- [[User Stories]]

**Ресурси:**
- [Scrum Guide](https://scrumguides.org/) - офіційна документація
- [Scrum.org](https://www.scrum.org/) - certification
