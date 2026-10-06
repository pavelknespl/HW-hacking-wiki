# hwhack.info

A practical, vendor-neutral wiki for hardware hacking, firmware security, and embedded reverse engineering. Every article is a plain Markdown file under [`docs/`](docs/), organized into categories and served with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/).

## Scope

The wiki documents the repeatable workflow behind most embedded device assessments: reconnaissance, finding a debug or memory interface, extracting firmware or data, and analyzing it. It covers hardware protocols (UART, JTAG, SWD, I2C, SPI, eMMC), the software toolchain (`screen`, `minicom`, `tio`, `flashrom`, `binwalk`, `dd`, OpenOCD, sigrok, UBI/NAND tools), attack techniques (firmware extraction, flash dumping, bootloader access and bypass, fault injection, side-channel analysis), reverse engineering with Ghidra, and an introduction to RF/SDR.

All content is written for authorized security research and education.

## Running locally

```bash
python3 -m pip install -r requirements.txt
mkdocs serve
```

Then open <http://127.0.0.1:8000>.

## Building a static site

```bash
mkdocs build          # output in site/
```

## Deployment

Pushing to `main` triggers `.github/workflows/deploy.yml`, which builds the site and publishes it to the `gh-pages` branch via `mkdocs gh-deploy`.

## Contributing

Each page is one Markdown file in a category folder under `docs/`. Add a page by creating the file and adding it to the `nav:` section of `mkdocs.yml`. Keep prose clear and concrete, prefer working commands and real pin names, and cite datasheets where it matters.

## License

Content is licensed under CC BY-SA 4.0.
