```dataviewjs
// === настройки ===
const source = "periodic/Assignments";
const startOnMonday = true;

// === сбор данных ===
const pages = dv.pages(`"${source}"`);
const tasks = pages.file.tasks.where(t => t.due);

// индекс задач по датам YYYY-MM-DD
const byDate = {};
for (const t of tasks) {
  if (!t.due) continue;
  const key = moment(t.due).format("YYYY-MM-DD");
  (byDate[key] ??= []).push(t);
}

// границы календаря
const today = moment();
let start = today.clone().startOf("month").startOf("week");
let end   = today.clone().endOf("month").endOf("week");
if (startOnMonday) {
  while (start.isoWeekday() !== 1) start.subtract(1, "day");
  while (end.isoWeekday() !== 7)   end.add(1, "day");
}

// контейнер
const root = dv.el("div", "", { cls: "dv-cal-grid" });

// заголовки дней
const weekNames = startOnMonday
  ? ["Пн","Вт","Ср","Чт","Пт","Сб","Вс"]
  : ["Вс","Пн","Вт","Ср","Чт","Пт","Сб"];
const head = root.createEl("div", { cls: "dv-cal-head" });
weekNames.forEach(n => head.createEl("div", { text: n, cls: "dv-cal-head-cell" }));

// ячейки календаря
for (let d = start.clone(); d.isSameOrBefore(end); d.add(1, "day")) {
  const cell = root.createEl("div", { cls: "dv-cal-cell" });
  cell.createEl("div", { text: d.format("DD"), cls: "dv-cal-date" });
  const key = d.format("YYYY-MM-DD");
  (byDate[key] ?? []).slice(0, 4).forEach(t => {
    cell.createEl("div", { text: "• " + t.text, cls: "dv-cal-item" });
  });
}

// ——— стили (без document.currentScript) ———
const STYLE_ID = "dv-cal-style";
if (!document.getElementById(STYLE_ID)) {
  const style = document.createElement("style");
  style.id = STYLE_ID;
  style.textContent = `
.dv-cal-grid{display:grid;grid-template-columns:repeat(7,1fr);gap:6px}
.dv-cal-head{display:grid;grid-template-columns:repeat(7,1fr);gap:6px;margin-bottom:6px}
.dv-cal-head-cell{font-weight:600;opacity:.8;text-align:center}
.dv-cal-cell{border:1px solid var(--background-modifier-border);padding:6px;min-height:90px;border-radius:6px}
.dv-cal-date{font-weight:700;margin-bottom:4px;opacity:.9}
.dv-cal-item{font-size:0.9em;line-height:1.2em;overflow:hidden;text-overflow:ellipsis;white-space:nowrap}
`;
  document.head.appendChild(style);
}

```
