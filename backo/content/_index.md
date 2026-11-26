+++
title = "Backo"
description = "ORM + RESTFul API + Storage abstraction + Custom code + RBAC/RLAC"

[extra.devlab.home]
layout = "wide"

[[extra.devlab.home.blocks]]
type = "hero"
eyebrow = "Backo 0.4.0+"
description = "Opensource backoffice low code in python"
image = "/images/backo-quickstart-800x400.gif"
actions = [
  { label = "Build your first backend", path = "@/documentation/_index.md" },
  { label = "Documentation", path = "@/documentation/_index.md" },
]

[[extra.devlab.home.blocks]]
type = "learn"
eyebrow = "Features"
title = "This is low code"
description = "More describe than code your backend in python. Just the business code."
items = [
  { kicker = "ORM", title = "Object-Relational Mapping", description = "Store Datas in collections with links between them.", path = "@/features/orm.md" },
  { kicker = "RESTFul API", title = "application programming interface (API)", description = "Build the API following the design principles of the REST.", path = "@/features/restfull.md" },
  { kicker = "Storage abstraction", title = "Use major opensource databases", description = "Store datas in a lot of kind of databases.", path = "@/features/database/_index.md" },
  { kicker = "RBAC - RLAC", title = "Precise right management", description = "Role-Based Access Control (RBAC) and Row-Level Access Control (RLAC) embedded.", path = "@/features/_index.md" },
]


[[extra.devlab.home.blocks]]
type = "workflow"
eyebrow = "Quickstart"
title = "From scratch to your backend"
items = [
  { title = "Install backo", description = "Install the python package." },
  { title = "backo-init", description = "Launch backo-init script and answers few questions. It will create all .py files." },
  { title = "backoffice.py", description = "Launch backoffice.py. You have your first running backend !" },
  { title = "Modify", description = "Edit and modify .py files for adaptation." },
]


[[extra.devlab.home.blocks]]
type = "content"

+++

## Quick installation



{% <devlab.tabs sync="local-shell" label="Development server command"> %}
{% <devlab.tab name="mongoDB" value="mongo" selected={true}> %}
```bash
pip install backo[mongo]
```
{% </devlab.tab> %}
{% <devlab.tab name="postgres" value="postgres"> %}
```bash
pip install backo[postgres]
```
{% </devlab.tab> %}
{% </devlab.tabs> %}



