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

For years, I have encouraged my research students to organise their work so that they can reproduce the final paper from scratch.

I want them to reach the point where one command builds the paper from raw experimental data and manuscript source, then produces the final PDF with its analysis, numerical results, tables and figures.

This is good research practice. It also solves a more immediate problem: papers change.

## The cost of a small correction

Suppose you notice that the axis label on a graph is wrong.

The code change may take only a few seconds. But if someone exported the graph and pasted it into a Word document, you must also:

1. rerun the plotting code;
2. find the newly generated image;
3. replace the old image in the manuscript;
4. check its size and position; and
5. make sure you have replaced it in the latest version of the paper.

That is the simple case.

Now suppose you discover a small bug in the analysis code. Correcting it changes a reported result from, say, 98.4% to 97.9%. The change may not affect the paper’s conclusions, but the old number could appear in several places:

* a results table;
* one or more graphs;
* the body of the results section;
* a figure or table caption;
* the abstract; and
* perhaps even the conclusion.

Every manually copied value becomes another opportunity for the paper to contradict itself.

The correction is easy. Finding every place where the result has spread through the paper is not.

## Treat the paper as the output of the research

A paper pipeline treats the manuscript as another generated research artefact.

Derived material should flow from its source:

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

You can use the same generated value in the abstract, results section and conclusion. When the experiment changes, the value changes everywhere the next time you build the paper.

The same idea applies to tables and figures. Scripts generate them; authors should not edit or copy them into place by hand. Readers should be able to understand generated figures without colour alone, and the source should contain the alternative text.

## Why Markdown?

Microsoft Word provides fields, references and linked content, but it does not suit this style of computational paper production.

LaTeX suits it better, but LaTeX can add friction. Most people quickly learn commands such as `\section` and `\subsection`, but constructing a particular type of table or adjusting an unfamiliar layout can take more effort.

I therefore tend to write papers in Markdown and use Pandoc to produce LaTeX and PDF.

Markdown keeps the manuscript easy to read and edit:

```markdown
# Results

The best-performing model achieved an accuracy of
\BestAccuracy{}.

\input{build/tables/results.tex}

![Bar chart comparing validation accuracy for each model; Model B is highest.](build/figures/results.svg)
```

The alternative text is part of the source, not an afterthought added to the final PDF. The same applies to table captions and figure captions: they should describe the result, not just name the file.

LaTeX remains available where its precision is useful, without forcing the whole manuscript into LaTeX source. Modern AI coding tools are also quite good at helping with the occasional obscure LaTeX requirement.

## Why GNU Make?

The pipeline still needs something to describe how its parts depend on one another.

For this I use GNU Make.

Make is old, and it certainly has limitations. Filenames containing spaces, for example, can cause unnecessary complications. In a research repository this is rarely a serious constraint: `experiment-results.csv` generally works just as well as `Experiment Results.csv`.

Make gives you a compact and widely supported way of stating dependencies:

```make
build/values.tex: scripts/generate_metrics.py data/results.csv
	python scripts/generate_metrics.py

build/figures/results.svg: scripts/generate_figures.py data/results.csv
	python scripts/generate_figures.py

build/paper.pdf: paper.md build/values.tex \
                 build/figures/results.svg
	pandoc paper.md --output build/paper.pdf
```

If the results file changes, Make knows which values, figures and paper components to rebuild.

The objective is that this command succeeds from a clean checkout:

```bash
make clean && make all
```

This gives you a much stronger definition of reproducibility than keeping the analysis code somewhere in the repository.

## Formatting numbers properly

Generated numbers also make formatting easier.

The pipeline uses the LaTeX package `siunitx` to format numbers and units consistently. This avoids hand-formatting every percentage, measurement or error value.

Suppose the underlying result is:

```text
98.9828393201
```

That does not mean the paper should display every decimal place. If the uncertainty is about plus or minus one percentage point, reporting `98.9828393201%` suggests a level of precision that the experiment does not support.

The generated value should still preserve the full number:

```latex
\newcommand{\BestAccuracy}{\num{98.9828393201}}
```

Then `siunitx` can decide how to print it. You can set the rounding rule once for the whole document:

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

You can supply uncertainty as part of the number, and `siunitx` can switch between compact and separated forms without changing the underlying value:

```latex
\num{98.9828393201 +- 0.73}
\sisetup{uncertainty-mode=separate}
\num{98.9828393201 +- 0.73}
```

Automating formatting does not remove the researcher’s responsibility to decide what precision is scientifically defensible. It makes that decision easier to apply consistently across prose, tables and figures.

## Turning the approach into an agent skill

I have explained this approach to research students many times, but few have adopted it fully.

The reasons are practical. A student must learn the research method, the repository structure, Make, Markdown, Pandoc, LaTeX conventions and the distinction between source files and generated files. Each part may be simple; the combined workflow can still feel like too much.

I captured the approach as an installable agent skill called **paper-pipeline**.

The skill tells an AI coding agent how to organise a reproducible paper repository and how to modify one. Its rules include:

* do not hard-code numerical results in prose;
* generate reported values from scripts;
* generate tables and figures rather than editing them manually;
* keep figure alt text, captions and table descriptions in the source;
* inspect the Makefile before making changes;
* edit source data, configuration or generation scripts rather than generated files;
* run the relevant Make target after a change; and
* verify that the complete paper rebuilds.

The repository is available at:

```text
https://github.com/jbrusey/paper-pipeline
```

In an agent environment that supports installing skills from a URL, install it with:

```text
install https://github.com/jbrusey/paper-pipeline
```

The repository also contains instructions for installing the skill into Agent Skills-compatible tools such as Codex and Claude Code. The central file is:

```text
skills/paper-pipeline/SKILL.md
```

## Some practical limits

A pipeline does not make research frictionless. It removes one class of manual copying errors, but a few details still need deliberate handling.

### Environments and dependencies

Make tracks whether files are out of date. It does not know which Python version you used, which system libraries Pandoc needed or which LaTeX packages your laptop had installed.

For a short-lived student project, a `requirements.txt`, `uv.lock`, `environment.yml` or `pixi.lock` file may be enough. For work that must build years later or on another institution’s machine, a container such as Docker can capture more of the operating environment.

Aim for a build command the next person can run without remembering your laptop setup:

```bash
make all
```

### Long-running computations

The command `make clean && make all` should work, but you do not need to regenerate every intermediate file before every draft.

If training a model or running a simulation takes days, treat its output as a source artefact for the paper pipeline. Save a documented summary file, such as a CSV of experimental results, and have the paper pipeline turn that summary into values, tables and figures.

That keeps the manuscript build fast while preserving provenance. Heavy computation can live in its own pipeline; the paper pipeline can start from the checked and documented outputs of that computation.

### Collaborating with Word or Overleaf users

Co-authors may still want Track Changes in Word, comments in Google Docs or edits in Overleaf. A Markdown and Make workflow does not remove that.

One workable pattern is to keep the authoritative manuscript in the pipeline, then export review copies when needed:

```bash
pandoc paper.md --output build/paper.docx
```

The lead author can fold comments and suggested edits back into the source. For LaTeX-heavy collaborations, Overleaf can work as the editing front end while scripts continue to generate the values, tables and figures that the paper inputs.

## What changed with the skill

The skill did more than let an AI agent reproduce my preferred workflow.

It explained the workflow to a student better than my previous explanations had.

The first student I gave it to used it to construct the complete pipeline and return a working paper quickly. The agent generated values, tables, figures and captions correctly, and the student could rebuild the paper through the pipeline.

That points to another use for agent skills.

We often treat skills as a way to make an AI tool repeat a task reliably. They can also encode expert working practices and put them in front of people while they work.

A written guide explains what someone ought to do. An agent skill can help them do it.

Research supervisors spend a lot of time teaching craft: how to organise experiments, preserve provenance, structure repositories, validate results and prepare papers.

Encoding some of that craft into skills does not replace supervision. It gives students a concrete and executable version of the advice.

## Reproducibility as everyday convenience

People often present reproducibility as an obligation to future readers, reviewers or researchers.

That matters, but it can make reproducibility sound like extra work done for somebody else.

A paper pipeline gives you a more immediate benefit. Reproducibility reduces the cost of changing your own paper. It makes correcting mistakes safer, rerunning experiments easier and responding to reviewers less tedious. It reduces the chance that the abstract contains an obsolete result or that a graph no longer agrees with its accompanying table.

The repository becomes an executable account of how you produced the paper.

That helps the scientific community. It also helps the person trying to finish the paper.

## References

The Turing Way has useful practical guidance on reproducible research:

* [Reproducible Research](https://book.the-turing-way.org/reproducible-research/reproducible-research/)
* [Make for reproducible research](https://book.the-turing-way.org/reproducible-research/make/)
