# AgentPassport

A Know-Your-Agent (KYA) trust layer for autonomous agents running on Google AX.

Platforms such as Google AX make it easy to run thousands of agents in isolated sandboxes, but they do not answer the question every service on the receiving end will ask: which agent is this, who is responsible for it, what is it allowed to do, and should I still trust it?

AgentPassport gives every AX task a verifiable passport, signs its outbound requests, lets third-party services check them, and watches agent behaviour at runtime so that a hijacked agent can be stopped within seconds.

## Status

Proposal and system design, before implementation. This is a graduation project at Hacettepe University, Department of Computer Engineering. The repository currently holds the design document (Draft v0.2, September 2026).

## Design document

- [Design document (PDF)](docs/AgentPassport_Design_Document.pdf)
- LaTeX source: [`docs/main.tex`](docs/main.tex) and [`docs/sections/`](docs/sections/)
- Figures: [`docs/figures/`](docs/figures/)

To build the PDF, run XeLaTeX from `docs/`. The fonts Bitstream Charter, Liberation Sans and DejaVu Sans Mono must be installed.

```sh
cd docs
latexmk -xelatex main.tex
```

## License

[Apache License 2.0](LICENSE)
