# Ronin 48 — Open-Source Expert System Models

A suite of open-source AI models fine-tuned on Llama 3.3 70B Instruct for professionals in law enforcement, emergency services, fire service, and the legal system. Each model is purpose-built for a specific domain with real-world operational data.

All models are free, open-source (Apache 2.0), and built to be run locally — on your hardware, without sending case data to a third party.

---

## The Suite

| Model | Full Name | Role | Users |
|-------|-----------|------|-------|
| [SELMA](https://codeberg.org/Ronin48/SELMA) | Statute Enumeration and Legal Mapping Assistant | Statute identification and charge analysis | Patrol officers, detectives, special agents |
| [ABBY](https://codeberg.org/Ronin48/ABBY) | Artifact, Ballistic, and Binary Yield | Forensic methodology, evidence standards, chain of custody | Forensic examiners, digital investigators |
| [BONES](https://codeberg.org/Ronin48/BONES) | Biomedical On-scene Navigator for Emergency Services | EMS clinical decision support | EMRs, EMTs, AEMTs, Paramedics |
| [BRUNO](https://codeberg.org/Ronin48/BRUNO) | Building Rescue and Unified Navigation Operations | Fire service tactical decision support | Firefighters, Company Officers, Incident Commanders |
| [ATTICUS](https://codeberg.org/Ronin48/ATTICUS) | — | Defense strategy and constitutional analysis | Public defenders, defense attorneys |

---

## Models

### SELMA — Statute Enumeration and Legal Mapping Assistant

[![HuggingFace](https://img.shields.io/badge/HuggingFace-Ronin48LLC%2Fselma--lora--adapter-yellow)](https://huggingface.co/Ronin48LLC/selma-lora-adapter)
[![Repo](https://img.shields.io/badge/Codeberg-Ronin48%2FSELMA-blue)](https://codeberg.org/Ronin48/SELMA)

Given a factual description of a crime or incident, SELMA identifies applicable federal and state statutes, elements of each offense, sentencing ranges, and the legal reasoning connecting the facts to the charges.

**Built for:** Patrol officers writing reports, detectives building cases, special agents preparing affidavits.

**Coverage:** Federal criminal code, all 50 state statutes, case law, sentencing guidelines.

---

### ABBY — Artifact, Ballistic, and Binary Yield

[![HuggingFace](https://img.shields.io/badge/HuggingFace-Ronin48LLC%2Fabby--lora--adapter-yellow)](https://huggingface.co/Ronin48LLC/abby-lora-adapter)
[![Repo](https://img.shields.io/badge/Codeberg-Ronin48%2FABBY-blue)](https://codeberg.org/Ronin48/ABBY)

Named after Abby Sciuto from NCIS. Given an evidence description or examination request, ABBY identifies the correct forensic methodology, applicable legal standards for admissibility, chain-of-custody requirements, and relevant statutes governing digital and physical evidence.

**Built for:** Forensic examiners, digital investigators, crime scene analysts.

**Coverage:** Digital forensics (FTK, Cellebrite, EnCase), physical/trace evidence, biological evidence, Daubert/Frye standards, Fourth Amendment digital case law (*Riley*, *Carpenter*, *Jones*).

---

### BONES — Biomedical On-scene Navigator for Emergency Services

[![HuggingFace](https://img.shields.io/badge/HuggingFace-Ronin48LLC%2Fbones--lora--adapter-yellow)](https://huggingface.co/Ronin48LLC/bones-lora-adapter)
[![Repo](https://img.shields.io/badge/Codeberg-Ronin48%2FBONES-blue)](https://codeberg.org/Ronin48/BONES)

Named after Dr. Leonard "Bones" McCoy. Given a patient presentation or dispatch scenario, BONES provides protocol guidance, drug references, triage support, and clinical decision support across the full EMS scope of practice chain.

**Built for:** EMRs, EMT-Basics, AEMTs, and Paramedics in the field and in training.

**Coverage:** AHA ACLS/PALS/BLS, NAEMSP/NASEMSO protocols, START/SALT triage, pediatric dosing (Broselow), toxicology, trauma, OB/Peds emergencies, PCR documentation.

---

### BRUNO — Building Rescue and Unified Navigation Operations

[![HuggingFace](https://img.shields.io/badge/HuggingFace-Ronin48LLC%2Fbruno--lora--adapter-yellow)](https://huggingface.co/Ronin48LLC/bruno-lora-adapter)
[![Repo](https://img.shields.io/badge/Codeberg-Ronin48%2FBRUNO-blue)](https://codeberg.org/Ronin48/BRUNO)

Named after Chief Alan Brunacini of the Phoenix Fire Department — the father of modern Incident Command. Given a fireground scenario, BRUNO provides tactical guidance, ICS structure, hazmat references, and rescue decision support.

**Built for:** Firefighters, Company Officers, and Incident Commanders.

**Coverage:** NFPA codes, IFSTA curriculum, ICS/NIMS, hazmat (DOT/ERG), wildland/WUI, structural firefighting, vehicle extrication, Mayday/RIT operations.

---

### ATTICUS — Coming Soon

[![Repo](https://img.shields.io/badge/Codeberg-Ronin48%2FATTICUS-blue)](https://codeberg.org/Ronin48/ATTICUS)

Named after Atticus Finch. ATTICUS provides defense strategy analysis, constitutional violation identification, and evidentiary weakness assessment.

**Built for:** Public defenders and defense attorneys.

**Status:** Training data in development.

---

## How These Models Work Together

```
Incident occurs
     │
     ├── BONES   — EMS responds, clinical decision support
     ├── BRUNO   — Fire responds, tactical decision support
     │
     ├── ABBY    — Evidence collected, forensic methodology
     ├── SELMA   — Charges identified, statute mapping
     │
     └── ATTICUS — Defense analysis, constitutional review
```

Every model in the suite reflects the same obligation: to the facts, to the standard, and to getting it right.

---

## Accessing the Models

All adapters are LoRA fine-tunes on `meta-llama/Llama-3.3-70B-Instruct`.

| Model | LoRA Adapter | Status |
|-------|-------------|--------|
| SELMA | [Ronin48LLC/selma-lora-adapter](https://huggingface.co/Ronin48LLC/selma-lora-adapter) | Available |
| ABBY | [Ronin48LLC/abby-lora-adapter](https://huggingface.co/Ronin48LLC/abby-lora-adapter) | Available |
| BONES | [Ronin48LLC/bones-lora-adapter](https://huggingface.co/Ronin48LLC/bones-lora-adapter) | Training |
| BRUNO | [Ronin48LLC/bruno-lora-adapter](https://huggingface.co/Ronin48LLC/bruno-lora-adapter) | Training |
| ATTICUS | [Ronin48LLC/atticus-lora-adapter](https://huggingface.co/Ronin48LLC/atticus-lora-adapter) | Pending |

---

## License

**Apache License 2.0** — Copyright 2026 Ronin 48, LLC.

Base model weights are subject to the [Meta Llama 3.1 Community License](https://llama.meta.com/llama3/license/). All fine-tuned adapter weights and original contributions remain Apache 2.0.
