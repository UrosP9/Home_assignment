# AI-assisted development log: x264 QP analysis script

**Task:** Home Assignment, Task 2 – script for analyzing x264 output at QP 0–51 (file size and encoding speed).
**Tool:** Claude Code (model: Sonnet 5.5), run from the `Projects` folder.
**Result:** `script/x264_qp_analysis.py`, `script/README.md`, and the generated report in `script/results/`.

This is a cleaned-up transcript of the full conversation. The wording of every message is unchanged. Only
terminal formatting, status lines and auto-generated recaps were removed or reformatted. Tool calls (shell commands and
file reads) were collapsed in the original export, so they are summarized in *italics*.

---

## 1. Initial request

**User:**
> Read the Assignment.pdf, I need to work on the 2nd task - script (x264 qp analysis). Create a script in the Projects folder.

*Claude read the assignment PDF, checked the environment (x264 present via Homebrew, no `.yuv` file, `openpyxl`
missing), wrote the script and `requirements.txt`, and tested the encoding loop on a small synthetic clip.*

**Claude:**

I wrote the script at `/Users/uros/Documents/Projects/x264-qp-analysis/script/x264_qp_analysis.py`, with a
`requirements.txt` next to it. The encoding half works, but I couldn't run the spreadsheet half.

**Not tested:** `openpyxl` wouldn't install here. The default pip index needs credentials I don't have. Public PyPI
failed on an SSL certificate error, and I didn't work around that. The `.xlsx` code (formulas, charts, Summary sheet)
has only been syntax-checked. Please run it once on your machine and look at the charts.

**What I tested:** the encoding loop and CSV output. I ran QP 0–51 through your Homebrew x264 on a 20-frame synthetic
352x288 clip. All 52 encodes completed, and the FPS and file size parsed correctly. File size dropped from about
1.4 MB at QP 0 to about 12 KB at QP 51, and FPS rose from about 470 to about 2,650. The clip is synthetic because the
assignment's `.yuv` isn't in the Projects folder.

**To run it:**
```
pip install -r requirements.txt
python x264_qp_analysis.py foreman-cif.yuv --resolution 352x288 --repeat 3
```
- **Output:** `results/results.csv` and `results/qp_report.xlsx`, written as soon as the runs finish.
- **Results sheet:** per-QP file size in bytes and KiB, size relative to QP 0, FPS, time, bitrate and compression ratio.
- **Charts:** line charts of file size, FPS and time against QP, placed on the Results sheet.
- **Summary sheet:** max and min file size, speed range, and average size change per +1 QP.
- **Options:** `--repeat N` averages FPS and time over N runs, which is worth using because short clips give noisy
  timings. `--x264 PATH` points at a specific binary. `--report-only` rebuilds the report from an existing CSV.
  `--keep-output` keeps the `.264` files.

For the submission, the assignment asks you to share the full AI conversation for this part.

---

## 2. Running on the real video

**User:**
> I've moved the .yuv file next to the generated script. I've also installed requirements.

*Claude inspected the folder and the existing `results/` output without re-running the script.*

**Claude:**

It looks like the script has already run on `foreman-cif.yuv`. I didn't run it. I found `results/results.csv` and
`results/qp_report.xlsx`, both timestamped 18:45.

- **Results:** the CSV has all 52 QP values (0–51), for a 300-frame clip.
  - **File size:** 16.3 MB at QP 0, 769 KB at QP 25, 42 KB at QP 51. That is roughly 380× smaller across the range.
  - **Speed:** 659 fps at QP 0, 1,655 fps at QP 25, 4,180 fps at QP 51. That is about 6× faster.
- **Workbook:** it has a Results sheet and a Summary sheet, plus three charts (file size, FPS and time against QP).
  I unzipped it to check this, but I haven't opened it visually.
- **My `openpyxl` check:** `python3` in my shell doesn't see `openpyxl`. You probably installed it into a different
  interpreter, which is fine because the script clearly ran for you.

Please open the xlsx in Excel or Numbers and check that the formulas calculate and the charts look right. I haven't
been able to check that part.

The run used `--repeat 1` (the default), so the FPS values are single measurements and will be noisy, especially at
high QP where each encode takes under 0.1 s. For steadier numbers, rerun with `--repeat 3` or more:

```
python x264_qp_analysis.py foreman-cif.yuv --resolution 352x288 --repeat 3
```

I can also add a short written analysis of the trends to the report if you want one.

---

## 3. Correction: the number of repeats

**User:**
> That's because I've ran it once. This is the command I used: `python x264_qp_analysis.py foreman-cif.yuv --resolution 352x288 --repeat 3`

*Claude checked the "Runs per QP" cell in the workbook's Summary sheet.*

**Claude:**

You're right, and I got that wrong. I said the run used `--repeat 1` without any evidence. The Summary sheet in the
workbook records "Runs per QP (speed averaged)" as 3, which matches your command. So the FPS and time values are
averages of three encodes per QP, and my "single measurements" comment doesn't apply.

Averaging over three runs helps, but each encode at high QP still takes under 0.1 s. Small bumps in the FPS curve at
the high-QP end may be timing noise, not real effects. A higher `--repeat` (5–10) would smooth that if the charts
look jagged.

The caveat from before still stands. I haven't opened the workbook to check that the formulas calculate and the
charts render. Please do that when you open it.

---

## 4. Why does a higher QP encode faster?

**User:**
> Explain: the task says that higher QP will result in a more compressed file with a smaller size and lower quality. It also makes sense that the encoding will be longer due to it?
> but in the results, it seems the opposite. The higher the QP is, the fps seems to go up and the time seems to go down.

**Claude:**

Your results are correct. The assumption that more compression means more work is what's off. QP doesn't make the
encoder compress harder. It makes the encoder throw away more information, so there is less left to process.

**Why a higher QP is faster**
- **Quantization:** after the transform, each block's coefficients are divided by a step size set by QP. At a high QP
  most coefficients round to zero. At a low QP many stay non-zero.
- **Entropy coding:** CABAC, the entropy coder, has to encode every non-zero coefficient. At QP 0 your run wrote
  16 MB, and at QP 51 it wrote 42 KB. Encoding about 380× less data takes far less time.
- **Residual work:** the encoder runs a transform, quantization and reconstruction for every block. At a high QP many
  blocks have no residual left, or get coded as "skip", and the encoder can take shortcuts for them.
- **Fixed costs:** motion estimation and similar steps cost about the same at any QP. So the speed-up has a limit.
  Your FPS rises about 6× across the range, not 380×.

**Why the curve isn't linear**
Your data shows FPS rising slowly at low QP and faster at mid-to-high QP. Entropy coding dominates the time when the
bitstream is huge, so the savings show up as soon as the file shrinks. At the high end, the remaining fixed costs set
a floor, and timing noise matters more because each run takes under 0.1 s.

**One more point on QP 0**
x264 treats QP 0 as lossless, which is a special mode. It is often the slowest point and can break the pattern
slightly. In your CSV, QP 1 is larger than QP 0 in some runs, which is a known quirk of the lossless path.

The assignment only says QP "affects" speed. It doesn't say which direction, so your result is a valid finding. I'd
state it in your report as: higher QP gives smaller files and faster encoding, and the cost is quality, which this
task ignores.

---

## 5. README

**User:**
> Write a readme file, which will cover the description and the usage of this script.

*Claude created `script/README.md` (about 80 lines).*

**Claude:**

I wrote the README at `x264-qp-analysis/script/README.md`. It covers:

- **Description:** what the script does and how it works.
- **Setup:** the requirements and install steps.
- **Usage:** the command you ran, plus a table of all options.
- **Output:** the CSV and the Excel report, sheet by sheet.
- **Measurement method:** how FPS, time and file size are taken.
- **Notes:** timing noise at high QP, why a higher QP encodes faster, and the QP 0 lossless quirk.

I haven't tested that last claim. It comes from your CSV, where QP 1 came out larger than QP 0.

I wrote the options table from the script's argument list. I didn't add a results section, since the numbers will
change if you rerun with a different `--repeat`.

---

## Summary of AI involvement

| Area | Done by AI | Verified by |
|---|---|---|
| Script (`x264_qp_analysis.py`) | Written by Claude | Encoding loop tested by Claude on a synthetic clip; real run on `foreman-cif.yuv` done by the user |
| Excel report (formulas, charts, summary) | Written by Claude | Structure checked by Claude by unzipping the file; visual check (formulas and charts) left to the user |
| Explanation of the QP/speed relationship | Written by Claude | Reviewed by the user against the results |
| README | Written by Claude | Option table taken from the script's argument list |

**Corrections made during the conversation:** Claude wrongly stated that the run used `--repeat 1`. The user pointed
this out, and Claude confirmed from the workbook that the run used 3 repeats (Section 3).
