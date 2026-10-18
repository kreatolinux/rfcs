# Kreato Linux RFCs

Design documents for major feature changes across the Kreato Linux projects
(Jumpstart, kpkg, Kongue, and the rest). An RFC describes what is changing and
why, precisely enough that an implementation can be checked against it.

RFCs live in `text/`, numbered in the order they were opened:
`text/NNNN-short-name.md`.

| RFC | Title | Project |
| --- | --- | --- |
| [0001](text/0001-jumpstart-cores.md) | Jumpstart cores | Jumpstart |

## Process

1. Copy `0000-template.md` to `text/NNNN-short-name.md` (next free number).
2. Open a pull request. Discussion happens there.
3. Set `Status` in the header as it moves: `Draft`, `Accepted`, `Implemented`,
   `Superseded`, `Withdrawn`.
4. After acceptance, keep the RFC current with the implementation: bump the
   revision and record what changed. An implementation that disagrees with its
   accepted RFC is a bug in one of the two.

Small changes (bug fixes, refactors, new packages) do not need an RFC.

## License

The RFCs are licensed under [Creative Commons Attribution-ShareAlike 4.0 International](LICENSE)
(CC BY-SA 4.0). Code samples inside them may be used under the same terms.
