# Learning Platform Database Schema

## Overview

This document describes the complete database schema for the learning platform. All 24 tables are also consolidated in `00_schema.json`.

The schema covers the full learner journey: from assessment and skill gap analysis, through personalized learning path generation, course and module delivery via the 6E framework, artifact submission and evaluation, and XP-based progression.

---

## Table Index

| # | Table | Description |
|---|-------|-------------|
| 01 | `learner_profiles` | Core learner/student records |
| 02 | `learning_tracks` | Track recommendations per learner from assessment |
| 03 | `learning_paths` | Personalized learning path per learner per track |
| 04 | `courses` | Course catalog — one capability + one level transition per course |
| 05 | `modules` | Modules within a course, delivered via the 6E framework |
| 06 | `modules_content` | Maps modules to their 6E stages (stage containers) |
| 07 | `learner_capabilities` | Per-learner capability scores, gap analysis, and role readiness |
| 08 | `learning_path_courses` | Ordered course assignments within a learning path |
| 09 | `learner_module_progress` | Module-level progress header per learner per module |
| 10 | `learner_6e_stage_progress` | Per-learner per-stage progress within a module |
| 11 | `module_artifacts` | Practice and final artifacts defined per module |
| 12 | `artifact_templates` | Downloadable templates attached to artifacts |
| 13 | `artifact_submissions` | Learner submission attempts for module artifacts |
| 14 | `skill_gap` | Skill gap summary generated from assessment output |
| 15 | `profile_snapshot` | Learner profile snapshot from assessment (key patterns, aptitudes) |
| 16 | `learning_track_evidence` | Evidence statements supporting each track recommendation |
| 17 | `domains` | Master domain taxonomy (e.g. Backend Engineering, Frontend Engineering) |
| 18 | `capabilities_master` | Master capability reference from Role L1 Gap Model |
| 19 | `artifact_evaluation_flows` | AI → Staff → Industry evaluation pipeline per submission |
| 20 | `artifact_questions` | Individual questions/tasks within a module artifact |
| 21 | `roles` | Master list of industry roles |
| 22 | `6e_content` | Individual content pieces within a 6E stage |
| 23 | `artifact_submission_files` | Learner-uploaded files per question per submission |
| 24 | `xp_events` | Append-only XP event log per learner |

---

## Table Details

### 01 · learner_profiles

Core learner record. Capability scores and skill recommendations live in `learner_capabilities`.

**Key fields:** `id`, `student_id` (FK → students), `grade_level`, `skill_level`, `is_active`

**grade_level values:** `middle_school`, `high_school`, `higher_secondary`, `ug`, `pg`, `professional`

**skill_level values:** `beginner`, `foundation`, `intermediate`, `advanced`, `expert`

---

### 02 · learning_tracks

Track recommendations generated per learner from an assessment. Each row is one recommended track with fit score and justification.

**Key fields:** `id`, `learner_id`, `assessment_id`, `fit` (High/Medium/Low), `track`, `match_score`, `topics`, `duration`, `why_it_fits`

**Relationships:** → `learner_profiles`, → `learning_track_evidence` (one-to-one evidence record per track)

---

### 03 · learning_paths

Personalized learning path for a learner on a chosen track. Targets one industry role. Tracks overall path status and role readiness percentage.

**Key fields:** `id`, `learner_id`, `learning_track_id`, `role_id`, `role_readiness_percentage`, `level` (1–5), `status`, `is_active`, `version_no`, `is_latest`

**Unique constraint:** one path per learner per track.

---

### 04 · courses

Course catalog. Each course covers exactly one capability and advances the learner one level (`from_level → to_level`). The recommendation engine uses `capability_id + from_level` to match a course to a learner gap.

**Key fields:** `course_id`, `course_code`, `capability_id`, `from_level`, `to_level`, `title`, `difficulty_level`, `job_pathway_role`, `domain_id`

**Constraint:** `to_level = from_level + 1` always. Unique on `(capability_id, from_level)`.

---

### 05 · modules

Modules within a course. Each module maps to one engineering workflow stage and is delivered via the 6E framework. Contains rich metadata: problem statement, pressure points, learning content, tools, and knowledge mapping.

**Key fields:** `id`, `course_id`, `module_no`, `title`, `engineering_workflow_stage`, `learning_content` (JSON), `support` (JSON), `knowledge` (JSON), `tools` (JSON)

---

### 06 · modules_content

Maps a module to its 6E stages. One row per module per stage. Acts as the stage container — actual content assets live in `6e_content`.

**6E stages (in order):** `engage` → `explore` → `explain` → `express` → `empower` → `evolve`

**Key fields:** `id`, `module_id`, `stage_name`, `stage_order`, `is_active`

**Unique constraint:** one row per `(module_id, stage_name)`.

---

### 07 · learner_capabilities

Single source of truth for the gap engine. One row per learner + assessment + learning path + role + capability. Stores current level, required level, gap, priority band, and course recommendation context. `skill_gap_capabilities` has been merged into this table.

**Key fields:** `id`, `learner_id`, `assessment_id`, `learning_path_id`, `role_id`, `capability_id`, `current_level`, `required_level`, `gap`, `has_gap`, `gap_score`, `priority_band` (A/B), `why_needed`, `how_to_build`

**Level scale:** L0 = no exposure, L1 = awareness, L2 = working knowledge, L3 = applied skill, L4 = mastery.

---

### 08 · learning_path_courses

Ordered course assignments within a learning path. One row per course per path. Courses unlock sequentially — the first course per capability gap is immediately available; subsequent courses unlock only after the previous artifact is approved.

**Key fields:** `id`, `learning_path_id`, `course_id`, `capability_id`, `from_level`, `to_level`, `sequence_no`, `capability_sequence_no`, `is_locked`, `unlocks_after_submission_id`, `status`, `completion_percentage`

---

### 09 · learner_module_progress

Module-level progress header per learner per module. Tracks overall module status, current 6E stage, stages completed count, artifact submission state, and lock state.

**Key fields:** `id`, `learner_id`, `learning_path_id`, `course_id`, `module_id`, `module_status`, `current_stage`, `stages_completed`, `completion_percentage`, `artifact_submitted`, `artifact_approval_status`

**Progression trigger:** when `artifact_approval_status` is set to `approved`, the next module unlocks.

---

### 10 · learner_6e_stage_progress

One row per learner per 6E stage per module. A row is inserted as `in_progress` when the learner first enters a stage, and updated to `completed` when they finish. Absence of a row means the stage has not been reached. The `evolve` stage gates artifact submission.

**Key fields:** `id`, `learner_module_progress_id`, `learner_id`, `module_id`, `stage_name`, `stage_order`, `status`, `started_at`, `completed_at`, `time_spent_seconds`

**Unique constraint:** one row per `(learner_id, module_id, stage_name)`.

---

### 11 · module_artifacts

Defines practice and final artifacts inside each module. Includes scoring rules, resubmission policy, and max attempts. Question-level content lives in `artifact_questions`.

**Key fields:** `id`, `module_id`, `artifact_type` (practice/final), `total_score`, `passing_score`, `allow_resubmission`, `require_review_unlock`, `max_attempts`

---

### 12 · artifact_templates

Downloadable templates or external links attached to artifacts. Can be scoped to a specific question or to the whole artifact.

**Key fields:** `id`, `artifact_id`, `question_id` (nullable), `file_name`, `file_url`, `file_type`, `version`, `is_downloadable`

---

### 13 · artifact_submissions

Learner submission records for module artifacts. One row per attempt. Tracks attempt number, version label, submission mode, status, and links to the evaluation flow.

**Key fields:** `id`, `artifact_id`, `learner_id`, `learner_module_progress_id`, `attempt_no`, `version_label`, `is_latest`, `previous_submission_id`, `submission_mode` (fast/normal), `status`, `evaluation_id`

**Status values:** `draft`, `submitted`, `under_review`, `evaluated`, `returned_for_rework`, `approved`, `rejected`

---

### 14 · skill_gap

Per-learner skill gap summary generated from assessment output. Stores priority skills (A/B bands), recommended learning tracks, and current strengths.

**Key fields:** `id`, `learner_id`, `assessment_id`, `attempt_id`, `priorities` (JSON), `learning_tracks` (JSON), `current_strengths` (JSON), `recommended_track`

**Unique constraint:** one record per `(learner_id, assessment_id)`.

---

### 15 · profile_snapshot

Per-learner profile snapshot from assessment output. Captures key narrative patterns (strength, enjoyment, work style, motivation) and top aptitude strengths with percentiles.

**Key fields:** `id`, `learner_id`, `assessment_id`, `attempt_id`, `key_patterns` (JSON), `aptitude_strengths` (JSON)

**Unique constraint:** one record per `(learner_id, assessment_id)`.

---

### 16 · learning_track_evidence

Evidence statements supporting each learning track recommendation. Split from `learning_tracks` for normalization. One row per track per learner per assessment.

**Key fields:** `id`, `learning_track_id`, `learner_id`, `assessment_id`, `evidence_values`, `evidence_aptitude`, `evidence_interest`, `evidence_personality`, `evidence_employability`, `evidence_adaptive_aptitude`

---

### 17 · domains

Master domain taxonomy derived from the Role L1 model. Used to classify courses and capabilities.

**Key fields:** `id`, `code`, `name`, `industry`, `is_active`

**Examples:** Backend Engineering, Frontend Engineering

---

### 18 · capabilities_master

Master capability reference table. Each row is one unique capability sourced from the client's Role L1 Gap Model. `capability_id` is not unique alone — the same ID groups multiple named capabilities under one library domain.

**Key fields:** `id`, `capability_id`, `capability_library`, `capability`, `role_id`, `domain_id`, `industry_outcome`, `capability_gap`, `skill_gap_learning_outcomes`, `gap_review_status`

**Unique constraint:** `(capability_id, capability)`.

**Note:** `domain_id` is intentionally denormalized from `roles.domain_id` for fast filtering without joins.

---

### 19 · artifact_evaluation_flows

Tracks the full AI → Staff → Industry evaluation pipeline for a submission. One row per stage per submission. When the final stage reaches `approved`, the progression engine fires: capability level is incremented, next module/course unlocks, and XP is awarded.

**Key fields:** `id`, `submission_id`, `stage` (ai/staff/industry), `stage_order`, `status`, `evaluated_by`, `score`, `feedback`, `improvements`, `decision`, `overall_status`, `is_current_stage`, `progression_triggered`

**Decision values:** `pass`, `fail`, `return`, `escalate`, `skip`

**Progression trigger steps:** update `learner_capabilities.current_level` → update skill gap → update module progress (`artifact_approval_status = approved`, `module_status = completed`) → if all modules complete, set `learning_path_courses.status = completed` and unlock next course by setting `learning_path_courses.is_locked = false` → award XP → mark `progression_triggered = true`.

---

### 20 · artifact_questions

Individual questions or tasks within a module artifact. One artifact can have multiple questions, each with its own title, description, instructions, and submission requirement.

**Key fields:** `id`, `artifact_id`, `order`, `title`, `description`, `instructions`, `submission_required`, `is_active`

**Unique constraint:** `(artifact_id, order)`.

---

### 21 · roles

Master list of industry roles. Each role maps to capabilities learners in that role are expected to develop.

**Key fields:** `id`, `domain_id`, `name`, `description`, `is_active`

**Examples:** Backend Engineer, Full-Stack Engineer, DevOps Engineer

---

### 22 · 6e_content

Individual content pieces within a 6E stage of a module. One row per content item. Multiple items can exist per stage.

**Key fields:** `id`, `modules_content_id`, `content_type` (pdf/doc/video/image/slide/link/audio/text), `title`, `description`, `url`, `sort_order`, `duration_seconds`, `mime_type`, `file_size_bytes`, `version`, `status` (draft/published/archived), `metadata_json`

---

### 23 · artifact_submission_files

Stores learner-uploaded solution files for each question within a submission attempt. One row per file per question.

**Key fields:** `id`, `submission_id`, `question_id`, `file_name`, `file_url`, `file_type`, `file_size_bytes`

**Allowed file types:** `doc`, `ppt`, `excel`, `pdf`, `mp4`, `mp3`, `jpeg`, `png`, `sheet`, `xls`, `docs`, `text`, `zip`

---

### 24 · xp_events

Append-only log of every XP-earning action for a learner. Total XP is derived by summing `xp_awarded` for a learner. Written by the progression engine.

**Key fields:** `id`, `learner_id`, `source_type`, `source_id`, `xp_awarded`, `capability_id` (nullable), `earned_at`

**source_type values:** `stage_completed` (10 XP), `artifact_approved` (50 XP), `course_completed` (100 XP)

**Idempotency:** before inserting, check `WHERE source_id = :id AND source_type = :type` to prevent double XP on retry.

---

## Key Relationships Diagram

```
learner_profiles
  ├── learning_tracks ──── learning_track_evidence
  ├── learning_paths
  │     ├── learner_capabilities (gap engine source of truth)
  │     └── learning_path_courses
  │           └── courses ──── modules
  │                               ├── modules_content ──── 6e_content
  │                               ├── module_artifacts
  │                               │     ├── artifact_questions
  │                               │     └── artifact_templates
  │                               └── learner_module_progress
  │                                     ├── learner_6e_stage_progress
  │                                     └── artifact_submissions
  │                                           ├── artifact_submission_files
  │                                           └── artifact_evaluation_flows
  ├── skill_gap
  ├── profile_snapshot
  └── xp_events

domains ──── roles ──── capabilities_master
```

---

## Progression Flow

1. Learner completes all 6 stages of a module (tracked in `learner_6e_stage_progress`)
2. Learner submits the final artifact (`artifact_submissions`)
3. Evaluation pipeline runs: AI → Staff → Industry (`artifact_evaluation_flows`)
4. On `approved`:
   - `learner_capabilities.current_level` incremented
   - `learner_module_progress.artifact_approval_status` → `approved`, `module_status` → `completed`
   - If all modules in the course are complete → `learning_path_courses.status` → `completed`
   - Next course in the capability gap sequence unlocked: `learning_path_courses.is_locked = false` on the next `learning_path_courses` row
   - XP awarded (`xp_events` insert)
   - `progression_triggered = true` set on evaluation flow row
