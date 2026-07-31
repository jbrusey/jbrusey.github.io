---
layout: single
author_profile: true
read_time: true
comments: true
share: true
related: true
title: "From Results to Paper: Building a Reproducible Paper Pipeline"
date: 2026-07-31 00:00:00 +0000
categories: ["research"]
tags: ["reproducibility", "markdown", "latex", "make", "agent-skills"]
permalink: "/2026/07/31/paper-pipeline-skill/"
---

For years, I have encouraged my research students to organise their work so that the final paper can be reproduced from scratch.

The goal is to reach the point where a single command can build---or rebuild---the entire paper: starting from raw experimental data and the manuscript source, and producing a final PDF complete with the analysis, numerical results, tables and figures.

This sounds like good research practice---and it is---but it also solves a much more immediate problem: papers change.

## The hidden cost of a small correction

Imagine that you notice that the axis label on a graph is wrong.

The code change may take only a few seconds. But if the graph was previously exported and pasted into a Word document, you must also:

1. rerun the plotting code;
2. find the newly generated image;
3. replace the old image in the manuscript;
4. check its size and position; and
5. make sure you have replaced it in the latest version of the paper.

That is still a relatively simple case.

Now imagine that you discover a small bug in the analysis code. Correcting it changes a reported result from, say, 98.4% to 97.9%. The change may not affect the paper’s conclusions, but the old number could appear in several places:

* a results table;
* one or more graphs;
* the body of the results section;
* a figure or table caption;
* the abstract; and
* perhaps even the conclusion.

Every manually copied value becomes another opportunity for the paper to contradict itself.

The difficulty is not usually the correction itself. It is finding and updating every place where the result has propagated.

## Treat the paper as the output of the research

A paper pipeline treats the manuscript as another generated research artefact.

The underlying principle is that derived material should flow from its source:

```text
data and configuration
        ↓
analysis scripts
        ↓
results, values, tables and figures
        ↓
manuscript
        ↓
final PDF
```

Rather than typing a result such as “98%” directly into the manuscript, a script generates it from the experimental results and makes it available to the paper as a variable.

In LaTeX, for example, the script might generate:

```latex
\newcommand{\BestAccuracy}{\qty{98.9828393201}{\percent}}
```

The manuscript then contains:

```latex
The best-performing model achieved an accuracy of \BestAccuracy{}.
```

The same generated value can be used in the abstract, results section and conclusion. When the experiment changes, the value changes everywhere the next time the paper is built.

The same idea applies to tables and figures. They are outputs of scripts, not objects that are manually edited or copied into place. Generated figures should also be readable without colour alone and have source-controlled alternative text.

## Why Markdown?

Microsoft Word provides some mechanisms for fields, references and linked content, but it is not particularly well suited to this style of computational paper production.

LaTeX is much better suited to it, but LaTeX can also introduce unnecessary friction. Most people quickly learn commands such as `\section` and `\subsection`, but constructing a particular type of table or adjusting an unfamiliar layout can require rather more effort.

I therefore tend to write papers in Markdown and use Pandoc to produce LaTeX and PDF.

Markdown keeps the manuscript relatively easy to read and edit:

```markdown
# Results

The best-performing model achieved an accuracy of
\BestAccuracy{}.

\input{build/tables/results.tex}

![Bar chart comparing validation accuracy for each model; Model B is highest.](build/figures/results.svg)
```

The alternative text is part of the source, not an afterthought added to the final PDF. The same applies to table captions and figure captions: they should describe the result, not just name the file.

LaTeX remains available where its precision is useful, without requiring the entire manuscript to be written as LaTeX source. And, of course, modern AI coding tools are now quite good at helping with the occasional obscure LaTeX requirement.

## Why GNU Make?

The pipeline still needs something to describe how its parts depend on one another.

For this I use GNU Make.

Make is old, and it certainly has limitations. Filenames containing spaces, for example, can cause unnecessary complications. In a research repository this is rarely a serious constraint: `experiment-results.csv` generally works just as well as `Experiment Results.csv`.

What Make offers is a compact and widely supported way of stating dependencies:

```make
build/values.tex: scripts/generate_metrics.py data/results.csv
	python scripts/generate_metrics.py

build/figures/results.svg: scripts/generate_figures.py data/results.csv
	python scripts/generate_figures.py

build/paper.pdf: paper.md build/values.tex \
                 build/figures/results.svg
	pandoc paper.md --output build/paper.pdf
```

If the results file changes, Make knows which values, figures and paper components need to be rebuilt.

The objective is that this command succeeds from a clean checkout:

```bash
make clean && make all
```

That is a much stronger definition of reproducibility than merely having the analysis code somewhere in the repository.

## Formatting numbers properly

Once numbers are generated automatically, further improvements become possible.

The pipeline uses the LaTeX package `siunitx` to format numbers and units consistently. This avoids manually deciding how every percentage, measurement or error value should be displayed.

Suppose the underlying result is:

```text
98.9828393201
```

That does not mean the paper should display every decimal place. If the uncertainty is approximately plus or minus one percentage point, reporting `98.9828393201%` suggests a level of precision that the experiment does not support.

The generated value should still preserve the full number:

```latex
\newcommand{\BestAccuracy}{\num{98.9828393201}}
```

Then `siunitx` can decide how to print it. The rounding rule can be set once for the whole document:

```latex
\sisetup{round-mode=places, round-precision=1}
```

or locally for one value:

```latex
\num[round-mode=figures, round-precision=3]{98.9828393201}\,\si{\percent}
```

The same package also supports aligned numerical columns in tables, so a generated table can contain full-precision values while LaTeX controls their presentation:

```latex
\begin{tabular}{l S[table-format=2.1]}
\toprule
Model & {Accuracy / \si{\percent}} \\
\midrule
Baseline & 97.438192 \\
Improved & 98.9828393201 \\
\bottomrule
\end{tabular}
```

Uncertainty can be supplied as part of the number, and `siunitx` can switch between compact and separated forms without changing the underlying value:

```latex
\num{98.9828393201 +- 0.73}
\sisetup{uncertainty-mode=separate}
\num{98.9828393201 +- 0.73}
```

Automating formatting does not remove the researcher’s responsibility to decide what precision is scientifically defensible. It does, however, make it much easier to apply that decision consistently across prose, tables and figures.

## Turning the approach into an agent skill

Although I have explained this approach to research students many times, relatively few have adopted it fully.

There are understandable reasons. A student must simultaneously understand the research method, the repository structure, Make, Markdown, Pandoc, LaTeX conventions and the distinction between source files and generated files. Even when each part is fairly simple, the combined workflow can appear daunting.

I therefore captured the approach as an installable agent skill called **paper-pipeline**.

The skill tells an AI coding agent how a reproducible paper repository should be organised and, importantly, how it should behave when modifying one. Its rules include:

* do not hard-code numerical results in prose;
* generate reported values from scripts;
* generate tables and figures rather than editing them manually;
* keep figure alt text, captions and table descriptions in the source;
* inspect the Makefile before making changes;
* edit source data, configuration or generation scripts rather than generated files;
* run the relevant Make target after a change; and
* verify that the complete paper can be rebuilt.

The repository is available at:

```text
https://github.com/jbrusey/paper-pipeline
```

In an agent environment that supports installing skills from a URL, installation can be as simple as:

```text
install https://github.com/jbrusey/paper-pipeline
```

The repository also contains instructions for installing the skill into Agent Skills-compatible tools such as Codex and Claude Code. The central file is:

```text
skills/paper-pipeline/SKILL.md
```

## Some practical limits

A pipeline does not make the whole research process frictionless. It removes one class of manual copying errors, but a few details still need deliberate handling.

### Environments and dependencies

Make tracks whether files are out of date. It does not know which Python version you used, which system libraries Pandoc needed or which LaTeX packages happened to be installed on your laptop.

For a short-lived student project, a `requirements.txt`, `uv.lock`, `environment.yml` or `pixi.lock` file may be enough. For work that must build years later or on another institution’s machine, a container such as Docker can capture more of the operating environment.

Aim for a build command the next person can run without remembering your laptop setup:

```bash
make all
```

### Long-running computations

The command `make clean && make all` should be possible, but that does not mean every intermediate file must be regenerated before every draft.

If training a model or running a simulation takes days, treat its output as a source artefact for the paper pipeline. Save a documented summary file, such as a CSV of experimental results, and have the paper pipeline turn that summary into values, tables and figures.

That keeps the manuscript build fast while preserving provenance. Heavy computation can live in its own pipeline; the paper pipeline can start from the checked and documented outputs of that computation.

### Collaborating with Word or Overleaf users

Co-authors may still want Track Changes in Word, comments in Google Docs or edits in Overleaf. A Markdown and Make workflow does not remove that social fact.

One workable pattern is to keep the authoritative manuscript in the pipeline, then export review copies when needed:

```bash
pandoc paper.md --output build/paper.docx
```

The lead author can fold comments and suggested edits back into the source. For LaTeX-heavy collaborations, Overleaf can work as the editing front end while scripts continue to generate the values, tables and figures that the paper inputs.

## The surprising result

The most interesting result was not that the skill allowed an AI agent to reproduce my preferred workflow.

It was that the skill communicated the workflow to a student much more effectively than my previous explanations had.

The first student to whom I gave the skill was immediately able to use it to construct the complete pipeline and return a working paper remarkably quickly. Values, tables, figures and captions were generated correctly, and the paper could be rebuilt through the pipeline.

This suggests a broader role for agent skills.

We often think of skills as a way of making an AI tool repeat a task reliably. But they can also encode expert working practices and make those practices available to other people at the point where the work is being done.

A written guide explains what someone ought to do. An agent skill can help them actually do it.

For research supervision, that distinction may be important. Considerable supervisory time is spent explaining not only the intellectual content of research, but also the practical craft surrounding it: how to organise experiments, preserve provenance, structure repositories, validate results and prepare papers.

Encoding some of that craft into skills does not replace supervision. It gives students immediate access to a concrete and executable version of the advice.

## Reproducibility as everyday convenience

Reproducibility is often presented mainly as an obligation to future readers, reviewers or researchers.

That is important, but it can make reproducibility seem like additional work undertaken for somebody else.

A paper pipeline shows the more immediate benefit. Reproducibility reduces the cost of changing your own paper. It makes correcting mistakes safer, rerunning experiments easier and responding to reviewers less tedious. It reduces the chance that the abstract contains an obsolete result or that a graph no longer agrees with its accompanying table.

The repository becomes not merely an archive of what was done, but an executable account of how the paper was produced.

That is useful to the scientific community. It is also extremely useful to the person trying to finish the paper.

## References

The Turing Way has useful practical guidance on reproducible research:

* [Reproducible Research](https://book.the-turing-way.org/reproducible-research/reproducible-research/)
* [Make for reproducible research](https://book.the-turing-way.org/reproducible-research/make/)
