---
statuses:
  - {id: todo,        name: To Do,       category: open}
  - {id: in-progress, name: In Progress, category: active}
  - {id: done,        name: Done,        category: closed}
transitions:
  - {from: todo,        to: [in-progress]}
  - {from: in-progress, to: [todo, done]}
---
