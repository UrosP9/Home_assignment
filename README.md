# x264 QP analysis

A script that encodes one uncompressed `.yuv` video with x264 for every QP value from 0 to 51
and reports how QP affects **file size** and **encoding speed** (FPS / time). Video quality is
deliberately ignored.

When all encodes finish, the script writes the results to CSV and builds an Excel report with
formulas, a summary sheet and charts.

## Requirements

- Python 3.8+
- [x264](https://www.videolan.org/developers/x264.html) available on `PATH`
  (macOS: `brew install x264`) or passed with `--x264`
- Python package `openpyxl`:

```
pip install -r requirements.txt
```

## Usage

```
python x264_qp_analysis.py <input.yuv> --resolution WxH [options] [-- <extra x264 args>]
```

Example (the command used for the assignment):

```
python x264_qp_analysis.py foreman-cif.yuv --resolution 352x288 --repeat 3
```

For each QP the script runs the equivalent of:

```
x264 --qp <QP> --input-res 352x288 --no-progress -o qp<QP>.264 foreman-cif.yuv
```

### Options

| Option | Default | Description |

| `input` | required | Uncompressed `.yuv` file |
| `--resolution WxH` | required | Input resolution, e.g. `352x288` |
| `--x264 PATH` | `x264` from `PATH` | Path to the x264 executable |
| `--qp-min`, `--qp-max` | `0`, `51` | QP range to test |
| `--repeat N` | `1` | Encodes per QP. FPS and time are averaged over N runs; file size is taken from the last run (it is deterministic) |
| `--outdir DIR` | `results` | Where the report files are written |
| `--keep-output` | off | Keep the encoded `.264` files in `DIR/encoded/` (deleted by default) |
| `--report-only` | off | Skip encoding and rebuild the report from an existing `DIR/results.csv` |
| `-- <args>` | none | Extra arguments passed to x264, e.g. `-- --threads 1` |

## Output

Files are written to `--outdir` (default `results/`):

- `results.csv` – raw results, one row per QP: `qp, file_size_bytes, fps, time_sec, bitrate_kbps`
- `qp_report.xlsx` – the report:
  - **Results** sheet: per-QP file size (bytes and KiB), size relative to QP 0 (%), FPS, time,
    bitrate and compression ratio (input size / output size), plus three line charts:
    file size vs QP, FPS vs QP, time vs QP.
  - **Summary** sheet: input, resolution, runs per QP, largest/smallest file, slowest/fastest FPS,
    size reduction and speed-up from the lowest to the highest QP, and the average size change per +1 QP.
    Values are Excel formulas over the Results sheet.

## How the measurements are taken

- **FPS and bitrate** are parsed from the summary line x264 prints on stderr
  (`encoded N frames, X fps, Y kb/s`).
- **Time** is wall-clock time of the whole x264 process, measured by the script.
- **File size** is the size of the `.264` output on disk.

## Notes

- Encodes at high QP are very short (well under 0.1 s for CIF), so timing is noisy. Use a higher
  `--repeat` (e.g. 5–10) for smoother speed curves, and avoid running other heavy work meanwhile.
- Higher QP gives smaller files **and** faster encoding: coarser quantization leaves fewer
  non-zero coefficients, so there is less data to entropy-code.
- QP 0 is x264's lossless mode, so it can behave differently from QP 1 (e.g. QP 1 may produce a larger file).
- The Excel formulas are calculated when the file is opened in Excel/Numbers/LibreOffice; viewers
  that don't recalculate may show empty cells for formula columns.
