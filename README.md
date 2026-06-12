# DeskReservation

Joan API test artifacts and technical notes for the Guest Visit Automation / Desk Reservation project.

## Contents

- `Joan api technical summary v1.1.md` - technical summary and API test conclusions.
- `test.ipynb` - Python notebook used for Joan API testing.
- `desks_all.json` - paginated desk export from Joan.
- `desks_dump.json` and `assets_dump.json` - raw first-page API dumps for desks and assets.
- `external_desk_pool.csv` - approved external desk pool seed list.
- `sixth_floor_desks.csv` - desks returned by Joan with `floor.name = 6th floor`.

## Not Included

- `.env` is intentionally excluded because it contains Joan API credentials.
- `reservations_dump.json` is intentionally excluded because it contains employee reservation data and email addresses.
