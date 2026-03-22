# RaftKV

A linearizable distributed key-value store built on a from-scratch Raft consensus implementation in Go.

Implements the Raft protocol as described in [Ongaro & Ousterhout (2014)](https://raft.github.io/raft.pdf). Correctness is verified through linearizability checking via [Porcupine](https://github.com/anishathalye/porcupine) under fault injection (network partitions, crashes, message loss).

> Work in progress.

## References

- Ongaro, Ousterhout — [In Search of an Understandable Consensus Algorithm (2014)](https://raft.github.io/raft.pdf)
- Ongaro — [Consensus: Bridging Theory and Practice, Stanford PhD dissertation (2014)](https://web.stanford.edu/~ouster/cgi-bin/papers/OngaroPhD.pdf)
- Athalye — [Porcupine: A fast linearizability checker in Go](https://github.com/anishathalye/porcupine)

## License

MIT