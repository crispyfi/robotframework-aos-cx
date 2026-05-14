# robotframework-aos-cx

System acceptance tests for HPE Aruba Networking AOS-CX switches, driven by Robot Framework against each device's local REST API.

## Setup

```sh
uv sync
cp .env.example .env                  # fill in CX_USERNAME and CX_PASSWORD
cp site.yaml.example site.yaml        # replace PLACEHOLDERs with real values
```

## Run

```sh
uv run python verify.py -a            # every device in site.yaml
uv run python verify.py -d <hostname> # one device
uv run python verify.py -p access     # every device of a given persona
uv run python verify.py -h            # full flag list
```

Results land in `output/<hostname>/<YYYYMMDD-HHMMSS>/` — open `log.html` for the run report.
