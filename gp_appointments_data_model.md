# GP Appointments Data Model – `test-gp-appointments`

## 1. Purpose & high-level design

The `test-gp-appointments` database contains **curated GP appointments data** from NHS England, structured for analytics and for use as a knowledge base in LangChain / NLP agents.

It has two main tables:

- `practice` – appointment counts at **GP practice level**.
- `pcn_subicb` – appointment counts aggregated at **Sub-ICB location level** (with region & ICB context).

Both tables:

- Represent **monthly snapshots** of appointment activity.
- Are **partitioned** (typically by `year` and `month`) for efficient querying in Athena/S3.
- Use `count_of_appointments` as the main metric for how many appointments occurred under each combination of grouping fields.

This dataset is considered **valid against the official NHS GP appointments publication** and can be used for reporting on access, waiting times, mode of consultation, and workload by geography.

For **NLP / LangChain**, you can describe this database as:

> A star-like structure with two wide fact tables (`practice` and `pcn_subicb`) that already include their dimension attributes (practice, PCN, Sub-ICB, ICB, region, HCP type, appointment mode, national category, waiting-time band, status and month). Most questions about “how many GP appointments…?” can be answered by grouping or filtering these tables and summing `count_of_appointments`.

---

## 2. Table: `practice` – GP practice–level appointments

### Business description

The `practice` table contains **counts of appointments recorded at GP practices**, broken down by:

- GP practice
- Primary Care Network (PCN)
- Sub-ICB location
- HCP type (e.g. GP)
- Appointment mode (face-to-face, telephone, etc.)
- National category (e.g. planned clinics)
- Waiting-time band (days between booking and appointment)
- Appointment status (attended, unknown, etc.)
- Month (year + month and appointment_month_start_date)

Each row represents **the number of appointments for one combination of these attributes in a given month**.

### Technical info

- **Table name:** `test-gp-appointments.practice`
- **Grain:** 1 row = 1 combination of:
  - appointment month
  - GP practice
  - PCN
  - Sub-ICB
  - HCP type
  - appointment mode
  - national category
  - waiting-time band
  - appointment status
- **Primary measure:** `count_of_appointments` (integer).
- **Partitioning:** The table is partitioned in S3/Athena (intended by `year` and `month`). Adjust documentation if actual partition keys differ.

### Columns

From the sample:

- `appointment_month_start_date` – Start date of the appointment month, e.g. `01MAY2024` (string in source; often parsed to a date). All appointments in that row fall within this calendar month.
- `gp_code` – GP practice code (ODS code, e.g. `L85020`). Can be joined to `nhs_ods_silver.gp_practice_dim.orgid`.
- `gp_name` – Practice name (e.g. `BECKINGTON FAMILY PRACTICE`).
- `supplier` – Clinical system supplier used to record the appointment (e.g. `EMIS`, `TPP`, `VISION`).
- `pcn_code` – Primary Care Network code (e.g. `U21951`).
- `pcn_name` – PCN name (e.g. `MENDIP PCN`).
- `sub_icb_location_code` – Sub-ICB location code (e.g. `11X`).
- `sub_icb_location_name` – Sub-ICB location name (e.g. `NHS Somerset ICB - 11X`).
- `hcp_type` – Type of healthcare professional providing the appointment. Example: `GP`, but can include other staff groups (nurse, other clinical staff, etc.).
- `appt_mode` – Mode of appointment, e.g. `Face-to-Face`, `Telephone`, `Video/online` (depending on official data categories).
- `national_category` – National appointment category grouping, e.g. `Planned Clinics`, `Same Day`, `Walk-in`, etc.
- `time_between_book_and_appt` – Banded waiting time between booking and appointment, e.g. `1 Day`, `2 to 7 Days`, `22  to 28 Days`. This is a categorical string, not a numeric value.
- `count_of_appointments` – Number of appointments matching this combination of attributes in that month.
- `appt_status` – Appointment status, e.g. `Attended`, `Unknown`, `Did Not Attend`, `Cancelled` (depending on official categories).
- `year` – Numeric year component of the appointment month (e.g. `2024`).
- `month` – Numeric month component (1–12) corresponding to the appointment month (e.g. `5` for May).

### Relationships & joins

- **To ODS / organisation model:**
  - `gp_code` ↔ `nhs_ods_silver.gp_practice_dim.orgid` (to bring in addresses, LA/ICB, region, etc.).
  - `sub_icb_location_code` aligns with Sub-ICB codes used in NHS geographies.
- **Between appointment tables:**
  - `sub_icb_location_code`, `year`, `month`, `hcp_type`, `appt_mode`, `time_between_book_and_appt`, `appt_status` can be matched to aggregated values in `pcn_subicb` for cross-checks or multi-level reporting.

### Example semantic usage

- Monthly time series for a single practice:
  - Filter on `gp_code = 'L85020'`, group by `year`, `month`, sum `count_of_appointments`.
- Mode split for a PCN:
  - Filter `pcn_code = 'U21951'`, group by `appt_mode` and `appointment_month_start_date`.
- Waiting-time profile for an ICB via practice-level data:
  - Filter `sub_icb_location_code = '11X'`, group by `time_between_book_and_appt` and `appointment_month_start_date`.

### NLP / LangChain hints

When a natural-language question mentions:

- “GP practice”, “surgery”, “this practice” → use `practice` table and filter on `gp_code`/`gp_name` (possibly joined via `gp_practice_dim` for exact matching).
- “Primary Care Network / PCN” → filter on `pcn_code`/`pcn_name` in `practice` table.
- “Sub-ICB” or “Somerset ICB” → filter on `sub_icb_location_code` or `sub_icb_location_name` in `practice` table.
- “Appointments” (count) → always aggregate `SUM(count_of_appointments)`.
- “Face-to-face”, “telephone”, “online” → map to `appt_mode` values.
- “Seen within X days / booked X days in advance” → map to `time_between_book_and_appt` bands.
- “Attended appointments”, “DNAs”, “cancelled” → filter on `appt_status`.

For “latest data”, select the max (`year`, `month`) combination available.

---

## 3. Table: `pcn_subicb` – Sub-ICB-level appointments

### Business description

The `pcn_subicb` table contains **appointments aggregated at Sub-ICB location level** and above, without practice or PCN breakdown. It is useful for **high-level ICB or regional reporting** and cross-checking totals against official published figures.

Each row represents **the number of appointments for one combination of**:

- Sub-ICB location
- ICB
- Region
- Appointment month
- HCP type
- Appointment mode
- Waiting-time band
- Appointment status

### Technical info

- **Table name:** `test-gp-appointments.pcn_subicb`
- **Grain:** 1 row = 1 combination of:
  - Sub-ICB location (`sub_icb_location_code`)
  - Appointment month (`appointment_month` / `year` / `month`)
  - ICB & region context
  - HCP type
  - Appointment mode
  - Waiting-time band
  - Appointment status
- **Primary measure:** `count_of_appointments`.
- **Partitioning:** The table is partitioned (intended by `year`, `month`).

### Columns

From your sample:

- `sub_icb_location_code` – Sub-ICB location code (e.g. `00L`).
- `sub_icb_location_ons_code` – ONS geography code for the Sub-ICB location (e.g. `E38000130`).
- `sub_icb_location_name` – Sub-ICB location name (e.g. `NHS North East and North Cumbria ICB - 00L`).
- `icb_ons_code` – ICB ONS code (e.g. `E54000050`).
- `icb_name` – ICB name (e.g. `NHS North East and North Cumbria Integrated Care Board`).
- `region_ons_code` – Region ONS code (e.g. `E40000012`).
- `region_name` – Region name (e.g. `North East and Yorkshire`).  
- `appointment_month` – Start date of appointment month, e.g. `01MAR2025`.
- `appt_status` – Appointment status (e.g. `Attended`).  
- `hcp_type` – Type of healthcare professional (e.g. `GP`).  
- `appt_mode` – Mode of appointment (e.g. `Face-to-Face`).  
- `time_between_book_and_appt` – Banded waiting time (e.g. `1 Day`, `15  to 21 Days`).  
- `count_of_appointments` – Number of appointments for this combination.  
- `year` – Year (numeric).  
- `month` – Month (numeric).  

### Relationships & joins

- Can be joined to other geography/ODS reference datasets via:
  - `sub_icb_location_ons_code` or `sub_icb_location_code` → ODS / geography lookup tables.
  - `icb_ons_code` / `icb_name` → ICB-level geographies.
  - `region_ons_code` / `region_name` → region geographies.
- Can be conceptually related to practice-level data by aggregating the `practice` table to Sub-ICB level and comparing sums.

### Example semantic usage

- “Total attended GP appointments in North East and Yorkshire region in March 2025”:
  - Filter `region_name = 'North East and Yorkshire'`, `year = 2025`, `month = 3`, `appt_status = 'Attended'`, then sum `count_of_appointments`.
- “Face-to-face vs telephone appointments by Sub-ICB in 2025-03”:
  - Filter `year = 2025`, `month = 3`, group by `sub_icb_location_name` and `appt_mode`.
- “Proportion of appointments within 2 days”:
  - Filter waiting-time bands such as `1 Day`, `Same Day`, `2 Days` (depending on how categories are defined) and divide by total across all bands for the same grouping.

### NLP / LangChain hints

Use the `pcn_subicb` table when:

- The query is explicitly about **ICB or region-level** totals:
  - “in the North East and Yorkshire region”
  - “by Sub-ICB location”
- No practice-level breakdown is required.

Map common phrases:

- “ICB” → `icb_name` / `icb_ons_code`.
- “Region” → `region_name` / `region_ons_code`.
- “Sub-ICB” → `sub_icb_location_name` / `sub_icb_location_code`.
- “Appointments” → `SUM(count_of_appointments)`.

---

## 4. How to describe this dataset to an LLM (LangChain context)

You can provide the following condensed description as a context document for an agent that needs to write SQL over the GP appointments data:

> The `test-gp-appointments` database contains curated GP appointments data. There are two main tables:
>
> - `practice`: monthly appointment counts at GP practice level. Each row is a combination of appointment month, GP practice (`gp_code`, `gp_name`), PCN (`pcn_code`, `pcn_name`), Sub-ICB (`sub_icb_location_code`, `sub_icb_location_name`), healthcare professional type (`hcp_type`), appointment mode (`appt_mode`), national category, waiting-time band (`time_between_book_and_appt`), and appointment status (`appt_status`). The main measure is `count_of_appointments`. The table also includes `year` and `month` columns for partitioning and filtering.
> - `pcn_subicb`: monthly appointment counts aggregated at Sub-ICB location level, with ICB and region fields. It includes similar breakdowns by `appt_status`, `hcp_type`, `appt_mode`, `time_between_book_and_appt`, plus `sub_icb_location_code`, `sub_icb_location_name`, `icb_ons_code`, `icb_name`, `region_ons_code`, and `region_name`.
>
> For almost any question of the form “How many GP appointments…?” you should sum `count_of_appointments` grouped by the relevant dimensions. Use `practice` when the question mentions specific practices or PCNs, and `pcn_subicb` when the question is only about Sub-ICB, ICB, or region-level totals.
>
> Time filters can use:
>
> - `year` and `month` for numeric filters.
> - `appointment_month_start_date` or `appointment_month` for more literal date values (they represent the first day of the month).
>
> When a question mentions waiting times in days (“within 2 days”, “more than 28 days”), map these to the appropriate `time_between_book_and_appt` categories. When it mentions “attended appointments”, “DNAs” or “cancelled” appointments, use the `appt_status` column to filter. When it mentions “face-to-face”, “telephone”, or “online”, use `appt_mode`.
>
> For geography, link GP practices to organisational reference data via `gp_code` = `nhs_ods_silver.gp_practice_dim.orgid` when you need LA or ICB context, or directly use the Sub-ICB / ICB / region fields in the `pcn_subicb` table when working at those levels.
