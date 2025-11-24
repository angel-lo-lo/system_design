# System Design: Cloud File Storage Service (e.g., Google Drive, Dropbox)

## 1. Requirements

### Functional Requirements
1.  **Upload File**: Users can upload files from their devices.
2.  **Download File**: Users can download files.
3.  **File Sync**: Files should be synchronized across multiple devices automatically.
4.  **File Sharing**: Users can share files with others via a link or permissions.
5.  **File Revisions**: The system should support version history (automatic versioning).

### Non-Functional Requirements
1.  **Reliability & Durability**: Data loss is unacceptable (99.999999999% durability).
2.  **Availability**: The service should be highly available.
3.  **Scalability**: Must handle petabytes/exabytes of data and millions of users.
4.  **Security**: Data should be encrypted at rest and in transit.
5.  **Performance**: Large file uploads/downloads should be optimized (chunking).

## 2. Capacity Estimation

### Traffic & Storage
*   **Total Users**: 50 Million
*   **Daily Active Users (DAU)**: 10 Million
*   **Storage per User**: 10 GB
*   **Total Storage**: $50M * 10 \text{ GB} = 500 \text{ PB}$ (Petabytes)
*   **QPS**: Assume 1 file upload/download per active user per day.
    *   $10M / 86400 \approx 115$ requests/sec (Average).
    *   Peak QPS: $115 * 2 = 230$ requests/sec.
    *   *Note*: QPS is low, but bandwidth and storage are the bottlenecks.

## 3. High-Level Architecture

The system is split into two main parts: **Block Storage** (for actual file data) and **Metadata Storage** (for file info).
<img width="1311" height="658" alt="Screenshot 2025-11-24 at 10 38 14 AM" src="https://github.com/user-attachments/assets/34c6e9ee-9c02-4b45-9696-4095d22f4d76" />


## 4. Detailed Design

### A. File Chunking & Deduplication
*   **Strategy**: Split large files into smaller chunks (e.g., 4MB).
*   **Benefit 1 (Resumable Uploads)**: If upload fails, retry only the failed chunk.
*   **Benefit 2 (Deduplication)**: Calculate hash (SHA-256) of each chunk. If a chunk already exists in the system (uploaded by any user), don't upload it again. Just link the metadata. This saves massive storage space.

### B. Metadata Database
*   **Schema**:
    *   `users`: User info.
    *   `files`: `file_id`, `user_id`, `file_name`, `parent_folder_id`, `is_folder`.
    *   `versions`: `version_id`, `file_id`, `timestamp`.
    *   `blocks`: `version_id`, `block_order`, `block_hash` (points to S3 object).
*   **Choice**: Needs ACID for metadata consistency (e.g., moving a folder). **Relational DB** (PostgreSQL/MySQL) sharded by `user_id` is a common choice.

### C. Sync Conflict Resolution
*   **Scenario**: Two users edit the same file offline and come online.
*   **Strategy**: "Last Write Wins" is dangerous. Better to create a **Conflict File** (e.g., `Report_conflicted_copy.txt`) and let the user resolve it manually.
*   **Versioning**: Keep previous versions so users can restore if a sync overwrites data.

## 5. API Design

1.  **Upload File (Resumable)**
    *   `POST /files/upload-session` -> Returns `upload_id`
    *   `PUT /files/upload-session/{upload_id}?offset=0` -> Upload binary data
    *   `POST /files/commit` -> Finish upload, link blocks to file metadata.

2.  **Download File**
    *   `GET /files/{file_id}` -> Returns metadata & download URL.

3.  **List Files**
    *   `GET /files?folder_id=xyz`

