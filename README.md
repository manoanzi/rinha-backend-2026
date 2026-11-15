# rinha-backend-2026

My entry for [Rinha de Backend 2026](https://github.com/zanfranceschi/rinha-de-backend-2026): **real-time fraud scoring with vector search**, in Rust, inside **1 CPU and 350 MB** for the whole stack.

🇧🇷 [Resumo em português](#-resumo-em-português)

- **`main` branch:** source code
- **`submission` branch:** only what the official test needs (`docker-compose.yml` at the root)

## The challenge

- `POST /fraud-score` receives a transaction and returns `{"approved", "fraud_score"}`; `GET /ready` is the health check. Both on port 9999.
- A load balancer plus at least 2 API instances, with **1 CPU and 350 MB in total**
- Public `linux-amd64` images, `bridge` networking

## Architecture

```
            :9999
client ──► lb (0.16 CPU · 30 MB)
             │  accepts TCP and hands off the socket (SCM_RIGHTS over Unix sockets)
             ├──► api1 (0.42 CPU · 160 MB)
             └──► api2 (0.42 CPU · 160 MB)
                    epoll reactor → parse → vectorize (14 dims) → k-NN (k=5) → pre-built response
```

| Piece | Decision | Why |
|---|---|---|
| **Load balancer** | Handwritten in Rust (`crates/lb`). It passes the accepted file descriptor to an API instance instead of proxying bytes | After the handoff, the LB is out of the request path: no copy and no extra hop |
| **API I/O** | `epoll` reactor, kernel and socket busy-poll, `SCHED_FIFO`, `mlockall` | Predictable latency on a fraction of a CPU |
| **HTTP/JSON** | Minimal parser with static field finders and responses pre-built as byte arrays | Every request skips allocation and serialization |
| **Vectorization** | 14 normalized features (amount, installments, time, distance, MCC risk, 24h counts…) | Follows the challenge's normalization rules |
| **Search** | Offline index (`crates/build-index`) with a partitioned KD-tree/IVF, `i16` quantization and AVX2 (`_mm256_madd_epi16`) scans with early termination | Search compute went **from 85 µs to 22 µs** per query |
| **Images** | Static `musl` binaries on `FROM scratch` | Tiny images, nothing besides the binary in memory |
| **Quality** | `crates/eval`: an accuracy and latency harness against the reference dataset | Each optimization is checked against the exact result |

## Running locally

```bash
make data    # download the challenge's reference files
make index   # build resources/index.bin (~5s, deterministic)
make up      # docker compose up
make test    # cargo test
```

## Stack

Rust · epoll · SCM_RIGHTS · AVX2 SIMD · musl · Docker (`scratch`)

---

## 🇧🇷 Resumo em português

Submissão para a Rinha de Backend 2026: detecção de fraude por busca vetorial, em Rust, com **1 CPU e 350 MB** para todo o sistema.

- **Load balancer próprio:** aceita a conexão TCP e repassa o socket para a API via `SCM_RIGHTS`, saindo do caminho da requisição.
- **API:** reactor `epoll` com busy-poll, `SCHED_FIFO` e `mlockall`; parser HTTP/JSON mínimo e respostas pré-montadas.
- **Busca:** índice construído offline (KD-tree particionada/IVF), quantização `i16` e varredura AVX2 com early termination. O cálculo caiu de **85 µs para 22 µs** por consulta.
- **Imagens:** binários estáticos `musl` em `FROM scratch`.
- **Validação:** harness de acurácia e latência em `crates/eval`.
