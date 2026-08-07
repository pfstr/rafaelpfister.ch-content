---
title: "Paperless-ngx: self-hosted document management"
blatt: "paperless-ngx"
description: "An overview of the open source DMS: the consume pipeline from scan to archive, OCR and full-text search, organization through tags, correspondents, and document types, storage architecture, and operation in containers."
fakten:
  - { label: "Project", wert: "Paperless-ngx (open source, community fork)", href: "https://github.com/paperless-ngx/paperless-ngx" }
  - { label: "Purpose", wert: "Digitizing, tagging, archiving, and retrieving documents" }
  - { label: "Core pipeline", wert: "Consume folder, OCR, classification, archive" }
  - { label: "OCR", wert: "OCRmyPDF/Tesseract, searchable PDF/A archives" }
  - { label: "Organization system", wert: "Tags, correspondents, document types, storage paths" }
  - { label: "Operation", wert: "Docker Compose with database, broker, and volumes" }
werbung: ["newsletter"]
ctaThemen: ["paperless-ngx"]
---

# Paperless-ngx: self-hosted document management

Paperless-ngx is the most widely used self-hosted document management system in the private and small-business space: paper is scanned, put through automatic text recognition, tagged, and ends up in a searchable archive. The project is the actively maintained community successor to the earlier Paperless variants.

## The consume pipeline

The basic principle is an assembly line: documents enter the system through the **consume folder**, by mail retrieval, or by upload. There they pass through **OCR** (text recognition via OCRmyPDF and Tesseract, configurable for multiple languages), are stored as a searchable **PDF/A archive copy**, and are classified automatically. A learning mechanism proposes correspondent, document type, and tags based on the recognized text, and it becomes more accurate with every manual correction. The original is preserved untouched, while the archive version serves display and search.

## Organization without a folder tree

Instead of nested folders, Paperless-ngx organizes by metadata: **correspondents** (who), **document types** (what), **tags** (arbitrary facets), and date fields. Full-text search combines these dimensions with the OCR text; saved views represent recurring queries. Physical storage in the file system is handled by configurable **storage paths**, which turn metadata into a comprehensible structure, so that the archive remains readable outside the application as well.

## Architecture and operation

The reference installation runs as a **Docker Compose stack**: web application, database (PostgreSQL or SQLite/MariaDB), Redis as the task broker, and the volumes for originals, archive, and index. As with any containerized service, the data lives in the volumes, and the separation of originals, archive, and database determines the backup strategy; the built-in export additionally produces a portable full backup. Because archives grow over the years while local storage is finite, storage architecture is part of the picture, for example moving the document holdings to larger or remote storage targets.

## Assessment

The strength of the system lies in the combination of automation and data sovereignty: once set up, the assembly line processes incoming paper almost unattended, and the entire collection stays in the owner's hands as open files plus metadata, with no subscription and no cloud dependency.
