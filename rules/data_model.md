# Dashboard & Learning Activity Data Model

This document describes the data model and relationships between Organizations, Programs, Program Groups, Teams, Users, and their Learning Activities, primarily focusing on how data is structured for dashboard statistics.

## Core Entities & Contexts

The application tracks learning progress across different organizational scopes (contexts). These contexts are used to aggregate statistics on the dashboards.

*   **Organization**: Represents an enterprise customer.
*   **OrganizationTeam**: A subdivision within an Organization (e.g., "Team Dev", "Team Ops"). Belongs to an Organization.
*   **Program**: Represents a learning program.
    *   *Enterprise Program*: Linked to an Organization via `ProgramMembership`.
    *   *Consumer Program*: Used by `ProgramGroup`s.
*   **ProgramGroup**: A cohort of users within a consumer program. Belongs to a consumer `Program`.
*   **User**: The learner.

## Memberships & Scopes

Users are associated with contexts through membership models, which define their roles (e.g., member, admin) and status (e.g., active, pending, declined, revoked).

*   **OrganizationMembership**: Connects `User` to `Organization`.
*   **OrganizationTeamMembership**: Connects `User` to `OrganizationTeam`.
*   **ProgramGroupMembership**: Connects `User` to `ProgramGroup`.

```mermaid
classDiagram
    class Organization {
        slug
        name
    }
    class OrganizationTeam {
        name
        organization_id
    }
    class Program {
        slug
        name
        type (Enterprise/Consumer)
    }
    class ProgramMembership {
        program_id
        member_id (polymorphic: Organization)
        status (active/expired)
    }
    class ProgramGroup {
        slug
        name
        program_id
    }
    class User {
        email
    }
    class OrganizationMembership {
        user_id
        organization_id
        role (member/admin)
        status (active/pending/declined/revoked/expired)
    }
    class OrganizationTeamMembership {
        user_id
        organization_team_id
        role
        status
    }
    class ProgramGroupMembership {
        user_id
        program_group_id
        role
        status (active/pending/declined/revoked/expired)
    }

    Organization "1" --* "many" OrganizationTeam : has
    Organization "1" --* "many" OrganizationMembership : has
    User "1" --* "many" OrganizationMembership : has
    
    Program "1" --* "many" ProgramMembership : has
    Organization "1" ..|> ProgramMembership : member (via ProgramMembership)
    
    ProgramGroup --|> Program : belongs_to (consumer program)
    ProgramGroup "1" --* "many" ProgramGroupMembership : has
    User "1" --* "many" ProgramGroupMembership : has
    
    OrganizationTeam "1" --* "many" OrganizationTeamMembership : has
    User "1" --* "many" OrganizationTeamMembership : has
```

## Learning Activity Tracking

The core of the statistics engine is `LearningActivity` and its association with contexts via `LearningActivityContext`.

*   **LearningActivity**: Records a single learning event for a `User` (e.g., starting/completing a lab, enrolling/completing a course, completing a certification).
    *   `activity`: Polymorphic association pointing to the actual execution record:
        *   `LabInstance` (for labs)
        *   `CourseEnrollment` (for courses, which wrap a `CourseSession`)
        *   `CertificationEnrollment` (for certifications)
    *   `parent`: Optional association (e.g., points to a `LearningPlan` if the activity was completed as part of that plan).
    *   Metrics: `started_at`, `completed_at`, `is_passing`, `time_spent_seconds`.

*   **LearningActivityContext**: A join model that links a `LearningActivity` to one or more contexts (`Organization`, `Program`, `ProgramGroup`, `OrganizationTeam`).
    *   **Crucial for Dashboards**: When an activity is recorded, `LearningActivityContext` entries are created for all contexts the user belongs to. Dashboards query `LearningActivity` through these context joins to aggregate statistics (e.g., "Labs completed by Team Dev in Org X").

```mermaid
classDiagram
    class LearningActivity {
        user_id
        activity_type (LabInstance/CourseEnrollment/CertificationEnrollment)
        activity_id
        parent_type (LearningPlan)
        parent_id
        started_at
        completed_at
        is_passing
        time_spent_seconds
    }
    class LearningActivityContext {
        learning_activity_id
        context_type (Organization/Program/ProgramGroup/OrganizationTeam)
        context_id
    }
    class LabInstance {
        lab_id
        user_id
        elapsed_seconds
        deleted_at (completed_at)
    }
    class CourseEnrollment {
        course_session_id
        user_id
        completed_at
    }
    class CertificationEnrollment {
        certification_id
        user_id
        started_at
        completed_at
    }

    User "1" --* "many" LearningActivity : performs
    LearningActivity "1" --* "many" LearningActivityContext : has
    
    LearningActivityContext "many" ..> Organization : context (polymorphic)
    LearningActivityContext "many" ..> Program : context (polymorphic)
    LearningActivityContext "many" ..> ProgramGroup : context (polymorphic)
    LearningActivityContext "many" ..> OrganizationTeam : context (polymorphic)

    LearningActivity "1" ..> LabInstance : activity (polymorphic)
    LearningActivity "1" ..> CourseEnrollment : activity (polymorphic)
    LearningActivity "1" ..> CertificationEnrollment : activity (polymorphic)
```

## Auxiliary Models

*   **Catalog & CatalogItem**: Used to map content (Labs, Courses, Certifications) to Organizations via `OrganizationAvailableCatalog` and `CatalogOrganizationFilter`.
*   **BadgeAward & CredlyBadgeAward**: Track standard and external (Credly) badge completions.
*   **Searchjoy::Search**: Tracks searches performed by users within an Organization, optionally scoped to `OrganizationTeam`s. Used for "Search Trends" statistics.
