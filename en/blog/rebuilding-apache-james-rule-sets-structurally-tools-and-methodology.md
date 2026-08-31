---
title: "Rebuilding Apache James Rule Sets in a Structured Way: Tools and Method"
navTitle: "Rebuild rule set"
description: "Mailet rule sets that have grown over the years contain dead paths that no one recognizes anymore. Learn how to analyze the rule set as a graph, reliably find unreachable code, and design the migration so that a single Mailet keeps the rollback path open."
date: "2026-08-11"
kategorie: "Totemomail"
timeToRead: "16 min read"
themen:
  - totemomail
  - e-mail-verschluesselung
  - smtp-mailflow
hauptthema: "totemomail"
produkte:
  - "totemomail"
  - "apache-james"
protokolle:
  - "smtp"
  - "troubleshooting"
  - "verschluesselung"
related:
  - totemomail-m365
  - totemomail-licensed-user-limit-ldap-cleanup
  - mailflow-fehlersuche-kontrollierte-experimente
slug: "rebuilding-apache-james-rule-sets-structurally-tools-and-methodology"
translationId: "article-b9c98459a0ff6352"
draft: false
translationOf: apache-james-ruleset-strukturiert-neu-aufbauen
translationSourceHash: ebcf5bf98f1f74aa7784c74c558da4db240e69f02de722a0251dd832d1224403
translationModel: gpt-5.6-terra
translatedAt: 2026-08-31T10:01:00.989Z
translationReview: automatic
url: https://rafaelpfister.ch/en/blog/rebuilding-apache-james-rule-sets-structurally-tools-and-methodology
---

# Rebuilding Apache James Rule Sets in a Structured Way: Tools and Method

Mail gateways based on Apache James, including Totemomail, control their entire message flow through an XML rule set. After several years of operation, this rule set develops a property that hardly anyone notices: a substantial part of it is never executed. Rules were added, switches were placed before them, branches led nowhere, and because nothing broke, everything remained in place.

The problem is not disk space. It is that no one can say anymore which rule actually applies. Anyone planning a change reads a file containing hundreds of Mailets and does not know which of them are relevant at all. That is precisely what can be answered mechanically.

This article describes the method and the tools for doing so: analyzing the rule set as a directed graph, reliably finding unreachable code, and designing the migration so that a single Mailet keeps the rollback path open.

## The model in four sentences

A rule set consists of **processors**, meaning named chains. Each chain contains **Mailets** that do something, and each Mailet has a **matcher** that determines whether it applies to the current message. A Mailet of the `ToProcessor` class passes the message to another chain.

The entry point is usually called `root`. Everything else branches out from there.

```xml
<processor name="root">
   <mailet class="ToProcessor" match="RecipientIs?Recipient(s)=journal@example.com">
      <processor>dropJournal</processor>
   </mailet>
   <mailet class="ToProcessor" match="HostIsLocal">
      <processor>incoming</processor>
   </mailet>
   <mailet class="ToProcessor" match="All">
      <processor>outgoing</processor>
   </mailet>
</processor>
```

This makes the structure a directed graph: processors are nodes, `ToProcessor` targets are edges. And once you see it that way, the question of dead code becomes a standard task: reachability analysis.

## Two types of dead code

Before measuring, you need to know what you are looking for. There are two forms, and the second is the more treacherous one.

**Unreachable processors.** Entire chains that no `ToProcessor` points to anymore. They are present in the file but are never entered. This is the obvious case.

**Dead remainder within a chain.** A `ToProcessor` with `match="All"` matches **every** message and passes it on. Everything further down in the same chain is never reached. The same applies to Mailets with `passThrough=false`: they consume the message and handle further processing themselves; subsequent Mailets do not see it.

This second form cannot be found by a simple text search, because the lines look completely normal. You need the order within the chain.

## Tool 1: Extracting the graph

The starting point is an analysis that extracts processors and their targets. The following script uses only the standard library and runs on any Python installation:

```python
import re
import xml.dom.minidom

PFAD = "regelwerk.xml"

daten = open(PFAD, "rb").read().decode("utf-8").replace("\r\n", "\n")

# Extract processors and their blocks
starts = [(m.start(), m.group(1))
          for m in re.finditer(r'<processor name="([^"]+)">', daten)]

bloecke = {}
for i, (pos, name) in enumerate(starts):
    ende = starts[i + 1][0] if i + 1 < len(starts) else daten.find("</spoolmanager>")
    bloecke[name] = daten[pos:ende]

def ziele(block):
    """Alle ToProcessor-Ziele einer Kette."""
    return re.findall(r"<processor>\s*([^<>]+?)\s*</processor>", block)

for name, block in bloecke.items():
    print(f"{name} -> {', '.join(ziele(block)) or '(kein Ziel)'}")
```

Note the difference between the **definition tag** `<processor name="...">` and the **target tag** `<processor>name</processor>` within a `ToProcessor` Mailet. They have the same name but mean different things. Anyone who confuses them gets meaningless results. This is also the source of the error described below.

## Tool 2: Reachability from the entry point

With the graph, the analysis is a breadth-first search starting at `root`. Everything not visited is dead:

```python
erreichbar = set()
stapel = ["root"]

while stapel:
    knoten = stapel.pop()
    if knoten in erreichbar:
        continue
    erreichbar.add(knoten)
    for ziel in ziele(bloecke.get(knoten, "")):
        if ziel not in erreichbar:
            stapel.append(ziel)

def anzahl_mailets(block):
    return len(re.findall(r"<mailet ", block))

tot = [n for n in bloecke if n not in erreichbar]

print(f"Prozessoren gesamt: {len(bloecke)}")
print(f"Erreichbar:         {len(erreichbar)}")
print(f"Tot:                {len(tot)}")

for name in tot:
    print(f"  - {name} ({anzahl_mailets(bloecke[name])} Mailets)")
```

Typical output from a rule set that has grown over time:

```text
Prozessoren gesamt: 38
Erreichbar:         18
Tot:                20
  - addExtSender (7 Mailets)
  - decrypt (6 Mailets)
  - externalDelivery (14 Mailets)
  - outgoingProcessExceptionTriggers (12 Mailets)
  ...
```

Twenty of 38 processors, with more than 160 Mailets combined, that are never executed. This is not an outlier but the normal case in an environment that has undergone several rebuilds.

## Tool 3: Finding the dead remainder within chains

Now for the second form. Go through every reachable chain Mailet by Mailet and mark everything after the first unconditional exit:

```python
def toter_rest(block):
    """Index des ersten Mailets, ab dem nichts mehr erreicht wird."""
    mailets = re.findall(r"<mailet\b.*?(?:/>|</mailet>)", block, re.S)
    for i, m in enumerate(mailets):
        ist_all = 'match="All"' in m
        ist_weiche = 'class="ToProcessor"' in m
        konsumiert = "<passThrough>false</passThrough>" in m
        if ist_all and (ist_weiche or konsumiert):
            return i + 1, len(mailets)
    return None, len(mailets)

for name in sorted(erreichbar):
    ab, gesamt = toter_rest(bloecke[name])
    if ab is not None and ab < gesamt:
        print(f"{name}: Mailets {ab + 1} bis {gesamt} werden nie erreicht")
```

This finding is more valuable than the processor list because it sits in the middle of active chains. Anyone adding a rule and placing it below a `ToProcessor match="All"` has written a rule that will never apply and will subsequently wonder why it has no effect.

## Tool 4: Structural validation

Well-formed XML alone is not enough. These four checks catch the errors that a parser accepts but the gateway does not:

```python
dom = xml.dom.minidom.parseString(daten.replace("\n", "\r\n").encode("utf-8"))

namen = [p.getAttribute("name")
         for p in dom.getElementsByTagName("processor")
         if p.getAttribute("name")]

# 1. Duplicate processor names
doppelt = {n for n in namen if namen.count(n) > 1}

# 2. Every Mailet must be a direct child of a processor and have class + match
fehler = []
for m in dom.getElementsByTagName("mailet"):
    if m.parentNode.tagName != "processor":
        fehler.append(("falscher Elternknoten", m.getAttribute("class")))
    if not m.getAttribute("class") or not m.getAttribute("match"):
        fehler.append(("class oder match fehlt", m.parentNode.getAttribute("name")))

# 3. Every ToProcessor target must exist
zielnamen = set()
for m in dom.getElementsByTagName("mailet"):
    if m.getAttribute("class") == "ToProcessor":
        for el in m.getElementsByTagName("processor"):
            text = "".join(c.data for c in el.childNodes if c.nodeType == c.TEXT_NODE)
            zielnamen.add(text.strip())

print("Doppelte Namen:      ", doppelt or "keine")
print("Strukturfehler:      ", fehler or "keine")
print("Ziele ohne Definition:", sorted(zielnamen - set(namen)) or "keine")
```

A `ToProcessor` pointing to a processor that does not exist is the classic error after renaming. The XML remains well formed, but the gateway fails only at runtime, usually with an unhelpful message.

## Context: this is compiler construction

What you are doing here has a name and theory behind it. A rule set is a **control-flow graph**, the same model compilers have used to analyze programs for decades. This is worth knowing because it gives you established algorithms and, more importantly, clear statements about their limits.

| Question in the rule set | Model | Method |
|---|---|---|
| Which processors are dead? | Reachability from the entry node | Breadth-first or depth-first search, complexity `O(V+E)` |
| Which rules in a chain are dead? | Nodes after an unconditional jump | The same search on a finer-grained graph |
| Where can a mail loop occur? | **Cycle in the graph** | Strongly connected components |
| Where must a rule be placed so it is guaranteed to apply? | **Dominator** of the entry node | Dominator tree |

The last two rows are the most valuable in practice. A mail loop is not a mysterious operational phenomenon, but a cycle in the routing graph; the hop counter at runtime is merely the emergency brake, while you can find the loop structurally beforehand. And if you want to place a rule that **every** message must pass through, such as a filter for non-routable sender domains, you ask for a dominator. That is not a matter of taste, but something that can be calculated.

### Finding cycles before they become mail loops

Breadth-first search answers the question of dead code. For loops, you need depth-first search, because a **back edge** indicates a cycle. The method is the classic three-color marking:

```python
def zyklen_finden(bloecke, ziele):
    WEISS, GRAU, SCHWARZ = 0, 1, 2
    farbe = {n: WEISS for n in bloecke}
    pfad, gefunden = [], []

    def besuche(knoten):
        farbe[knoten] = GRAU
        pfad.append(knoten)
        for ziel in ziele(bloecke.get(knoten, "")):
            if ziel not in farbe:
                continue
            if farbe[ziel] == GRAU:                 # Rueckwaertskante = Zyklus
                gefunden.append(pfad[pfad.index(ziel):] + [ziel])
            elif farbe[ziel] == WEISS:
                besuche(ziel)
        farbe[knoten] = SCHWARZ
        pfad.pop()

    for knoten in bloecke:
        if farbe[knoten] == WEISS:
            besuche(knoten)
    return gefunden

for zyklus in zyklen_finden(bloecke, ziele):
    print(" -> ".join(zyklus))
```

```text
outgoing -> processOutgoing -> outgoing
```

Such a finding is not proof of a loop, because the edges are guarded and may never be taken together. But it is the complete list of places where one **can** occur, and that is exactly what you want to know before a migration. The hop counter at runtime is merely the emergency brake; here, you see the construction.

The **limit** of the method is just as important. The edges are guarded by matchers, and those depend on message content. Exact reachability is therefore undecidable in general; the analysis provides an over-approximation. This results in an asymmetrical evidentiary value that you need to understand:

- **“Unreachable” is reliable.** If no path leads there, no message can arrive there. You may delete this code.
- **“Reachable” only means “not structurally excluded.”** The graph does not tell you whether any real message will ever meet the conditions.

The analysis therefore does not replace testing; it reduces the test space. In practice, this is still an enormous benefit: 38 processors become 18 that you need to examine at all.

You explicitly do not need machine learning methods such as Graph Neural Networks or node embeddings here. They are worthwhile for large graphs with unknown structure and statistical patterns. A rule set has a few dozen nodes, a fully known structure, and deterministic semantics. Exact algorithms are not only cheaper here; they provide proofs rather than probabilities.

## Pitfalls of automated editing

When you modify a rule set with a script, there are three errors that occur reliably. I have made all three myself.

**The greedy pattern across processor boundaries.** Anyone trying to remove a processor with a regular expression will naturally reach for:

```python
muster = re.compile(r'   <processor name="%s">.*?</processor>\n' % name, re.S)
```

That is wrong. Within the chain, every `ToProcessor` Mailet contains a `<processor>ziel</processor>`, and the non-greedy `.*?` stops exactly there. The result: half the processor is removed, a fragment of `</mailet>` and `</processor>` remains, and the XML is broken. Instead, anchor on the indentation of the closing tag and check the tag balance against:

```python
muster = re.compile(r'   <processor name="%s">.*?\n   </processor>\n' % name, re.S)

treffer = muster.search(daten)
assert treffer, "Prozessor nicht gefunden"
block = treffer.group(0)
assert block.count("<mailet ") == block.count("</mailet>"), "Tag-Bilanz stimmt nicht"
```

**Line endings.** The configuration typically uses CRLF. Read it in Python with `rb`, normalize it to `\n` for editing, and convert it back to CRLF at the end. Anyone who forgets this produces a file with mixed line endings that may be silently rejected depending on the product.

**Special characters.** Keep the file in pure ASCII and write umlauts as character references (`&#228;` for ä). This saves you any discussion of encodings between the editor, script, and the gateway’s web interface.

After every change, check at least for well-formedness, unchanged line endings, and an unchanged processor count. Three lines of validation save a rollback.

## The migration method: parallel tree with one switch

Now for the actual rebuild. The obvious approach of converting the existing rule set step by step is the worst one: you cannot cleanly go back, and you can no longer read the old state.

The parallel tree has proven effective instead:

**Step 1: Build the new tree alongside it.** Create the new processors with a name suffix, such as `rootV2`, `incomingV2`, `outgoingV2`. The old tree remains completely intact and unchanged.

**Step 2: A single switch.** Place exactly one Mailet at the start of the existing entry point:

```xml
<processor name="root">
   <mailet class="ToProcessor" match="All">
      <processor>rootV2</processor>
   </mailet>
   <!-- alles Bisherige bleibt hier stehen, ist aber nun unerreichbar -->
</processor>
```

This routes all traffic through the new tree. The old one is unreachable but remains fully present. **The rollback consists of removing these three lines**, and that is understandable in every situation, even for someone who did not perform the migration.

**Step 3: Reachability as acceptance criterion.** Run the analysis from Tool 2 and check three points: the new entry point is referenced exactly once, all new processors are reachable, and the old tree is completely unreachable. This is an objective acceptance criterion rather than a visual inspection.

**Step 4: Clean up only after it has proven itself.** Once the new tree has been validated in operation, remove the old one and drop the suffixes. Only then do you lose the rollback path in the file, and until then, you have not needed it.

For intermediate steps that you want to observe but not activate yet, pure observation Mailets are suitable: they log but do not change routing. This lets you gather the data missing for the decision without risk.

## Build visibility in at the same time

When rebuilding, it is worth considering two things that will make a difference later in operation.

**Never discard directly in the main chain.** A Mailet that discards a message leaves only the note that it was deleted in the message history, without a reason. Instead, branch to a separately named processor, such as `dropNonRoutable`. The name alone appears in the history and already says what is happening.

**Not every log entry ends up in the message history.** Many products have two mechanisms: one for the server log and one for the history that support can also see. Only the latter is visible in the history. Anyone setting only the former has logged the event, but the trace still says only “message deleted.” Write history entries in plain language and name the rule: “intentionally discarded by the rule for non-routable sender domains, no delivery failure” saves a great deal of follow-up inquiry in operation.

## The cluster is part of the task

One point that is regularly underestimated: if the gateway runs on multiple nodes, the configuration must be stored **identically on all nodes and persistently across restarts**. If it is active on only one node, behavior depends on which node processes the message, and your tests measure chance.

The case where a change is running but was not persisted is especially unpleasant. The node then works correctly until it restarts and subsequently reverts to the old state. Therefore, after every deployment, check both: the same state on all nodes, and that the state survives a restart.

## Summary

Treat the rule set as a graph, not as a text file. A breadth-first search from the entry point separates live code from dead code in just a few lines, and analysis within the chains additionally finds rules that are present but never reached after an unconditional exit.

For the rebuild itself, the parallel tree with a single switch is the method with the best ratio of effort to security. And reachability analysis provides the corresponding acceptance criterion at the same time.

## Sources

1.  [Apache James: Mailet container configuration](https://james.apache.org/server/config-mailetcontainer.html): Structure of the spool manager, processors, Mailets, and matchers, as well as processing order.

2.  [Apache James: Provided mailets](https://james.apache.org/server/dev-provided-mailets.html): Reference for the included Mailets, including ToProcessor and the parameters for forwarding and consumption.

3.  [Apache James: Provided matchers](https://james.apache.org/server/dev-provided-matchers.html): Reference for matchers, including All, HostIsLocal, and recipient-based variants.

4.  [Apache James: Mailet API](https://james.apache.org/server/dev-mailet-api.html): Contract between Mailet and container, the basis for understanding consumption and forwarding.
