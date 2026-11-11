---
name: velixo-variance-analyzer
description: >
  Variance analyzer on Acumatica, Sage Intacct or Business Central: why a number moved or missed.
  Load for a month-end or quarter-end flux review, budget or forecast versus actual, or a department,
  project or cost overrun. Decomposes each material variance, drives it to an evidenced root cause and
  classifies it. Produces findings; velixo-commentary-writer writes the commentary.
license: "Copyright © 2026 Velixo Solutions Inc. All rights reserved. Proprietary: for use only by a licensed Velixo user, and not to be redistributed, distilled or modified for any other purpose. By installing or using this plugin, you agree to be bound by the agreements at https://velixo.com/agreements. See the LICENSE file shipped with this plugin."
user-invocable: false
disable-model-invocation: false
metadata:
  author: Velixo
  display_name: "Variance Analyzer"
---


# Variance analyzer

Pulls the ledger, budget and transaction data with validated queries, decomposes each material
variance, drives it to an evidenced root cause and quantifies its forward effect. It produces
findings, not prose.

## The connection and its ERP

**Choosing the connection.** Call `list_connections` before any query, and count the connections
whose `status` is `ok`.

- **One usable:** use it, and name it in the basis line.
- **The request or the calling command names one** ("on gab-dev", "use the demo connection"): use
  that one, if it is usable; if it is not, relay its status and stop.
- **Several usable and none named:** ask which one, **before any `execute_query`**, in the same
  message as any other question. Never send a call without `connectionName` to see which one the
  server picks: it refuses rather than guesses, and a probe that happens to resolve runs the
  analysis against a tenant nobody chose. The one exception is evidence in the conversation that the
  user is working inside an ERP instance (the host says so, or the user does): then the server
  resolves the connection from that instance, and a call without `connectionName` is how to let it.
- **The choice, as choices the user can tap.** A tappable picker holds four options. With four
  usable connections or fewer, offer them all. With more, offer the three most likely and
  **Other**, and list every remaining name in the message itself, so a user who picks Other can
  answer with one of them. The most likely are, in order: a name the conversation has used before,
  a name containing a word from the request (a company, an environment such as `prod` or `demo`),
  then the order `list_connections` returned them in. Never pick one silently because it is first.
**Ask once.** When the connection is a question, ask in the same message for whatever the request
leaves open among the period, the baseline (prior period, prior year or budget) and, when the review
selects among many lines rather than named accounts, the materiality threshold (an amount, a
percentage, both or either). The ERP's instructions then use those answers and do not ask again.

**Then read the connection's `type` from `list_connections`** and open this skill's instructions for
that ERP, before any query: the file `instructions.md` in the ERP's folder under `references/`.

| `type` | Folder |
|---|---|
| `acumatica`, `cegid_xrp_flex`, `myob_acumatica`, `jamis_prime`, `haufe_x360` | `acumatica` |
| `sage_intacct` | `sage-intacct` |
| `business_central` | `business-central` |

- **Read that file in full and follow it from its first step**, carrying over what the user already
  told you. It is this skill's method for that ERP: its queries are written for that ERP's schema and
  fail or return nothing on another one. Every reference file it or its method names is in the same
  ERP folder; open each when the step that needs it is reached.
- **Open it with Read, by its full path** under this skill's base directory, which heads what the
  Skill tool returned. Never search the file system for it.
- **A type in no row, or an ERP this skill has no folder for:** say that no Velixo skill does this for
  that ERP yet, and stop. Never run another ERP's queries on it.
This skill has instructions for: Acumatica, Sage Intacct and Business Central.
