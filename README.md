# AUA Operating Systems — Homework 4

Multi-file C compilation experiment using `gcc`, `nm`, `objdump`, and `readelf`.

## Build

```bash
gcc -c main.c
gcc -c math_utils.c
gcc main.o math_utils.o -o square_prog
./square_prog
```

Expected output:

```text
7 squared is 49
```

See [report.md](report.md) for the analysis. Complete terminal outputs collected on the course Ubuntu server are stored in [`outputs/`](outputs/).
