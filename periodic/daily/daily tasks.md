## Задачи на сегодня

```dataview
task
from "Путь/к/твоей/заметке/с/задачами"
where contains(text, "scheduled:: " + dateformat(date(today), "yyyy-MM-dd"))
