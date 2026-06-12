# Architectural Patterns & Review Guidelines

> **TODO:** This rule document should be converted into an active custom skill (`@architectural-review`) in the future to automate architectural compliance audits on local branches.

This document documents the professional architectural patterns identified in the Qwiklabs codebase, particularly around the Dashboard, Caching, and Learning Activity statistics systems. These patterns should be followed when implementing similar features or reviewed against during branch audits.

## 1. Caching & Time-Series Data

### Pattern: Cache-Aside with Sub-Range Query Optimization
When fetching time-series data (e.g., daily stats for a date range), avoid fetching the entire range from the database if parts of it are already cached.
*   **Implementation**:
    1. Check cache for the requested range.
    2. Identify if there are missing sub-ranges (gaps) at the start or end of the requested range.
    3. Query the database *only* for the missing sub-ranges.
    4. Write the newly fetched sub-range data back to the cache (e.g., as a session cache with a short TTL if it's a temporary query).
    5. Merge (stitch) the cached data and fresh data together in memory, sorting by date.
*   **Reference**: `DashboardDataService#fetch_by_day_data_with_cache` in [dashboard_data_service.rb](file:///usr/local/google/home/hesantiago/development/qwiklab-website/app/services/dashboard_data_service.rb).

---

## 2. UI Performance & Reactivity

### Pattern: Asynchronous Job-per-Metric (Sidekiq)
Do not calculate heavy statistics inline during page load. Split different metrics into individual background jobs.
*   **Implementation**:
    *   Controller action renders a shell page with cached data (or skeletons) and sets a unique `stream_id` (using ActionCable/Turbo).
    *   Frontend triggers AJAX requests to dispatch jobs for missing metrics.
    *   Each metric has its own job class (e.g., `Dashboard::ActiveLearnersJobNew`, `Dashboard::CompletionLabsJob`).
*   **Reference**: `DashboardInsightable#dispatch_job` in [dashboard_insightable.rb](file:///usr/local/google/home/hesantiago/development/qwiklab-website/app/controllers/concerns/dashboard_insightable.rb).

### Pattern: Reactive UI Updates via Turbo Streams
Instead of polling the server for job completion, use WebSockets to push updates.
*   **Implementation**:
    *   Background jobs perform calculations.
    *   Upon completion, the job uses `Turbo::StreamsChannel.broadcast_replace_to` to push rendered HTML partials (like charts or stats counters) directly to the user's view.
*   **Reference**: `Dashboard::ActiveLearnersJobNew#perform` in [active_learners_job_new.rb](file:///usr/local/google/home/hesantiago/development/qwiklab-website/app/jobs/dashboard/active_learners_job_new.rb).

---

## 3. Data Modeling & Queries

### Pattern: Polymorphic Contextual Aggregation
To track user activities across multiple organizational scopes (e.g., Organization, Team, Program, ProgramGroup) without duplicating activity records or writing complex polymorphic joins in every query.
*   **Implementation**:
    *   Keep a single `LearningActivity` table.
    *   Use a join model `LearningActivityContext` with a polymorphic `context` association.
    *   When an activity is created, create context records for all scopes the user currently belongs to.
    *   Aggregate stats by joining `LearningActivity` to `LearningActivityContext` and filtering on the context type and ID.
*   **Reference**: `LearningActivityContext` in [data_model.md](file:///usr/local/google/home/hesantiago/development/qwiklab-website/.agents/rules/data_model.md).

### Pattern: Type-Safe Cache Serialization
Ensure type safety when serializing data to external stores like Redis.
*   **Implementation**:
    *   Define cache schemas using Sorbet `T::Struct`.
    *   Implement robust deserialization (e.g., `from_redis_hash`) to safely convert Redis string-keyed hashes back into typed structs, handling nested types and type casting explicitly.
*   **Reference**: `DashboardCacheService::DayData` in [dashboard_cache_service.rb](file:///usr/local/google/home/hesantiago/development/qwiklab-website/app/services/dashboard_cache_service.rb).

---

## 4. Security & SQL Safety

### Pattern: Safe SQL Construction via Arel & SafeActiveRecord
For complex dynamic queries, avoid string interpolation. Use Arel or the internal `SafeActiveRecord` safety wrappers.
*   **Implementation**:
    *   Use Arel tables and nodes (`Arel::Table`, `Arel::Nodes::UnionAll`, etc.) to build queries programmatically.
    *   If raw SQL fragments are necessary (e.g., in subqueries), wrap them in `SafeActiveRecord::TrustedSymbol` or sanitize them explicitly using `ActiveRecord::Base.sanitize_sql_array`.
*   **Reference**: `DashboardDataService.priority_localized_title_subquery` and `all_time_most_popular_learning_plans` in [dashboard_data_service.rb](file:///usr/local/google/home/hesantiago/development/qwiklab-website/app/services/dashboard_data_service.rb).
