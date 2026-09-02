# ☁️ aws-ai-resume-screener: AWS AI Resume Screener & Talent Acquisition Pipeline

An intelligent, serverless recruitment pipeline built on **Amazon Web Services (AWS)** that automatically processes resumes, extracts meaningful candidate information using **Amazon Textract and Amazon Comprehend**, matches candidates against job descriptions, ranks candidates using an explainable scoring model, and provides recruiters with an authenticated dashboard for candidate management.

The system is designed to reduce manual resume screening effort while maintaining an auditable and reliable recruitment workflow.

---

## 🚀 Project Overview

Recruiters often need to manually review large numbers of resumes for a single job opening. This process can be time-consuming, inconsistent, and difficult to scale.

This project addresses the problem by building an **AI-powered resume screening pipeline** using AWS serverless services.

The system:

- Accepts a Job Description (JD) and multiple resumes (PDF / DOCX)
- Stores documents securely in Amazon S3
- Extracts resume text using Amazon Textract
- Extracts entities using Amazon Comprehend
- Uses a custom Amazon Comprehend entity recognizer to identify skills
- Extracts candidate email and experience information
- Stores structured candidate information in DynamoDB
- Matches candidates against job requirements
- Calculates an explainable candidate score (50% Skills / 30% Title / 20% Experience)
- Handles concurrent processing using Amazon SQS
- Handles failed processing through an SQS Dead Letter Queue (DLQ)
- Provides authenticated recruiter APIs using Amazon Cognito and API Gateway
- Sends interview emails using Amazon SES
- Provides CSV export of shortlisted candidates

---

## 🎯 Objectives

1. Automate resume ingestion and parsing.
2. Extract meaningful information from unstructured resumes.
3. Identify candidate skills using NLP.
4. Normalize extracted skills for reliable matching.
5. Calculate an explainable candidate-job match score.
6. Rank candidates automatically.
7. Provide recruiters with a centralized dashboard.
8. Handle concurrent resume uploads reliably.
9. Provide failure handling using SQS and DLQ.
10. Secure recruiter-facing APIs using Cognito authentication.
11. Automate interview communication through SES.
12. Maintain an auditable recruitment workflow.

---

## 🏗️ System Architecture

```text
                         ┌──────────────────────┐
                         │      RECRUITER       │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │ Recruiter Dashboard  │
                         │   S3 / CloudFront    │
                         └──────────┬───────────┘
                                    │
                              Cognito Login
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │     API Gateway      │
                         │  Cognito Authorizer  │
                         └──────────┬───────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              │                     │                     │
              ▼                     ▼                     ▼
       Upload / Jobs          Candidate APIs       Shortlist / Reject
              │                     │                     │
              ▼                     ▼                     ▼
        Amazon S3             DynamoDB              Lambda + SES
              │
              ▼
       Amazon SQS Queue
              │
              ▼
       Parsing Lambda
              │
       ┌──────┴────────┐
       │               │
       ▼               ▼
   PDF Resume      DOCX Resume
       │               │
       │          DOCX → PDF
       │               │
       └───────┬───────┘
               ▼
        Amazon Textract
               │
               ▼
        Extracted Text
               │
        ┌──────┴─────────────┐
        │                    │
        ▼                    ▼
 Amazon Comprehend    Custom Comprehend
 Standard Entities       SKILL Recognizer
        │                    │
        └─────────┬──────────┘
                  ▼
          Structured Candidate
                  │
                  ▼
             DynamoDB
                  │
                  ▼
          Scoring Lambda
                  │
       ┌──────────┼──────────┐
       │          │          │
       ▼          ▼          ▼
    Skills      Title     Experience
     50%         30%          20%
       │          │          │
       └──────────┼──────────┘
                  ▼
             Final Score
                  │
                  ▼
        Ranked Candidates
                  │
                  ▼
         Recruiter Dashboard
```

### ☁️ AWS Services Used

| AWS Service | Purpose |
| :--- | :--- |
| **Amazon S3** | Stores resumes, job descriptions, converted documents, and outputs |
| **AWS Lambda** | Serverless processing functions |
| **Amazon Textract** | Extracts raw text from PDF resumes |
| **Amazon Comprehend** | Extracts entities such as `PERSON`, `ORGANIZATION`, `DATE`, and `TITLE` |
| **Amazon Comprehend Custom Entity Recognition** | Identifies resume-specific `SKILL` entities |
| **Amazon DynamoDB** | Stores candidate, job, and processing information |
| **Amazon SQS** | Queues resume processing jobs |
| **Amazon SQS DLQ** | Handles repeatedly failed processing jobs |
| **Amazon Cognito** | Recruiter authentication |
| **Amazon API Gateway** | Secure backend API layer |
| **Amazon SES** | Sends interview invitation emails |
| **Amazon CloudWatch** | Logging and monitoring |

---

## 📂 Project Structure

```text
aws-ai-resume-screener/
│
├── backend/
│   │
│   ├── parsing/
│   │   └── parse_resume.py
│   │
│   ├── scoring/
│   │   └── score_candidate.py
│   │
│   ├── api/
│   │   ├── candidate_api.py
│   │   ├── job_api.py
│   │   ├── shortlist_api.py
│   │   └── csv_export.py
│   │
│   └── shared/
│       └── constants.py
│
├── frontend/
│   ├── src/
│   └── public/
│
├── training/
│   ├── skill_training_docs/
│   └── annotations/
│
├── test-data/
│   ├── resumes/
│   └── jobs/
│
├── README.md
├── .gitignore
└── requirements.txt
```

---

## 🔄 End-to-End Workflow

### 1. Resume Upload
The recruiter uploads one or more resumes (PDF / DOCX) through the recruitment dashboard into Amazon S3.

```text
resumes/
└── job-001/
    ├── candidate-001.pdf
    ├── candidate-002.pdf
    └── candidate-003.docx
```

### 2. DOCX Normalization
Amazon Textract's `DetectDocumentText` API does not directly process DOCX files. DOCX files are converted to PDF before sending to Textract. Temporary converted documents are stored under `converted/`.

### 3. Resume Text Extraction
Amazon Textract extracts text blocks from PDF files and compiles `LINE` blocks into full resume text representation.

### 4. NLP Entity Extraction
Amazon Comprehend extracts standard entities: `PERSON`, `ORGANIZATION`, `DATE`, and `TITLE`.

### 5. Custom SKILL Entity Recognition
A custom Amazon Comprehend entity recognizer extracts `SKILL` entities (e.g., Python, AWS Lambda, Docker, React, SQL, etc.) trained on domain-specific resume data.

### 6. Skill Normalization
Extracted raw skills are normalized (e.g., `"Amazon Web Services"` → `"aws"`).

### 7. Email Extraction
Candidate email addresses are extracted using deterministic regex parsing.

### 8. Experience Extraction
Total non-overlapping professional experience in years is calculated using `DATE` entities and employment context.

### 9. Candidate Data Model
Candidate metadata, skills, entities, experience, and parse status are stored in Amazon DynamoDB.

### 10. Scoring & Ranking
Candidates are scored against Job Descriptions (JDs) using a weighted model:
- **Skill Match (50%)**: `(matched_skills / total_required_skills) * 100`
- **Title Alignment (30%)**: Exact match = 100, Related = 75, None = 0
- **Experience (20%)**: `min(candidate_years / required_years, 1) * 100`

Recommended shortlist threshold: **FinalScore >= 70%** (configurable).

### 11. Asynchronous Queue & Dead Letter Queue (DLQ)
Amazon SQS buffers concurrent resume uploads, and an SQS DLQ captures poison/failed messages into a `FailedJobs` DynamoDB table for auditability.

---

## 👥 Team Responsibilities

| Member | Responsibility | Focus Files / Modules |
| :--- | :--- | :--- |
| **Member 1 (Current Focus)** | **AI Resume Parsing** | `backend/parsing/parse_resume.py`<br>`training/skill_training_docs/`<br>`training/annotations/`<br>`test-data/resumes/` |
| **Member 2** | **Matching, Scoring & Queue Reliability** | `backend/scoring/score_candidate.py`<br>SQS, DLQ, `FailedJobs` table |
| **Member 3** | **Backend APIs, Security & Email** | `backend/api/`, Cognito, API Gateway, SES |
| **Member 4** | **Recruiter Dashboard & Integration** | `frontend/`, UI Components & API integration |

---

## 🧪 Testing & Reliability

- **Unit & Integration Tests**: Test PDF/DOCX ingestion, skill normalization, entity confidence filtering, and scoring math.
- **DLQ Test**: Upload invalid files to ensure failed messages route to DLQ and write audit records into `FailedJobs`.
- **Security Audit**: Enforce least-privilege IAM roles, Cognito API authorization, and ensure PII / full resume text is excluded from CloudWatch logs.

---

## 🧹 Cost Considerations & Cleanup

During development:
- Use small test documents and minimal sample resumes.
- Limit CloudWatch log retention.
- Delete temporary S3 objects in `converted/` and `parsed/`.
- Tear down billable Comprehend custom endpoints when not actively training or testing.

---

## 📜 License

This project is developed for educational and demonstration purposes.
