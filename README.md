# GP Appointments – AWS Ingestion

This repository contains code to ingest **Appointments in General Practice** data into an AWS data lake.

The “Appointments in General Practice” publication describes activity and usage of GP appointments in England, including appointment volumes, mode (face-to-face, telephone, online), and patient types.  [oai_citation:18‡NHS England Digital](https://digital.nhs.uk/data-and-information/publications/statistical/appointments-in-general-practice?utm_source=chatgpt.com)

---

## Source dataset

- **Name:** Appointments in General Practice  
- **Owner:** NHS England  
- **Publication series:** Statistical publication “Appointments in General Practice”  [oai_citation:19‡NHS England Digital](https://digital.nhs.uk/data-and-information/publications/statistical/appointments-in-general-practice?utm_source=chatgpt.com)  
- **Licence:** Typically Open Government Licence – confirm on the publication page.

---

## Geographic coverage & granularity

- **Coverage:** England  
- **Granularity:**  
  - National  
  - Region  
  - ICB / former CCG  
  - GP Practice (for some tables)

---

## Time coverage & refresh

- Monthly snapshots of GP appointments, with historical data back to 2018/2019 (depending on series).  [oai_citation:20‡NHS England Digital](https://digital.nhs.uk/data-and-information/publications/statistical/appointments-in-general-practice?utm_source=chatgpt.com)  
- Data is typically published each month with a 1–2 month lag.

---

## This project’s data model

- **Bronze layer**  
  - Raw CSVs from each monthly “Appointments in General Practice” release.  
  - Stored with `year`, `month`, and `file_type` partitions.  

- **Silver layer**  
  - Normalised fact table, e.g. `fact_gp_appointments`, with fields like:
    - `appointment_date` / `reporting_period`  
    - `org_code` (practice, ICB, or higher-level code)  
    - `appointment_mode` (face-to-face, telephone, online, etc.)  
    - `patient_type`, `age_band` (if available)  
    - `count_appointments`

---

## Repository contents

- `Untitled.ipynb` – initial Bronze ingestion notebook (to be renamed to something like `bronze_gp_appointments.ipynb`).  
- Future Glue scripts/notebooks for Silver transformations can be added as `silver_gp_appointments.py` or similar.

---

## How to use

1. Download the CSV files from the “Appointments in General Practice” publication pages.  [oai_citation:21‡NHS England Digital](https://digital.nhs.uk/data-and-information/publications/statistical/appointments-in-general-practice?utm_source=chatgpt.com)  
2. Run the Bronze notebook to upload them to S3.  
3. Run the Silver transformation to build a tidy fact table for dashboarding (e.g. Tableau, Power BI, Athena).

---

## Attribution

Please acknowledge NHS England and the “Appointments in General Practice” publication in any outputs based on this data.  [oai_citation:22‡NHS England Digital](https://digital.nhs.uk/data-and-information/publications/statistical/appointments-in-general-practice?utm_source=chatgpt.com)
