# DeskReservation

Joan API test artifacts and technical notes for the Guest Visit Automation / Desk Reservation project.

## Contents

- `Joan api technical summary v1.1.md` - technical summary and API test conclusions.
- `assets_dump.json` - raw first-page API dump for Joan parking/assets.
- `external_desk_pool.csv` - approved external desk pool seed list.
- `sixth_floor_desks.csv` - desks returned by Joan with `floor.name = 6th floor`.
- `.gitignore` - keeps local secrets and sensitive reservation dumps out of git.

## Not Included

- `.env` is intentionally excluded because it contains Joan API credentials.
- `reservations_dump.json` is intentionally excluded because it contains employee reservation data and email addresses.
- `test.ipynb`, `desks_all.json`, and `desks_dump.json` remain local for now. Uploading exact large local files is better done with a working local git/gh login; the current GitHub CLI login on this machine is invalid.
