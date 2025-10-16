# Duolingo Data Warehouse Implementation Plan

I would like to say in advance that I don't have a ready-made solution and visual presentation of results, however I would like to outline my execution plan.

## Approach Selection

When approaching the task of building a data warehouse for Duolingo company for analysts, we can take two paths:

1) **Bottom-up**: from analyzing all business entities, the business as a whole, and building corresponding data models to implementing data marts
2) **Top-down**: from defining a list of metrics to modeling business entities

Within the framework of this task, given the tight timeframes, the second option is more suitable.

However, without thinking through business processes, we won't be able to generate truly realistic test data.

## Business Logic Philosophy

I believe that business logic: score calculation, group creation, league creation, etc., happens on the backend and is an operational matter. We should receive already calculated data in the analytical warehouse. So I simulate operational calculations only for adequate test data generation.

## Key Business Process

The key business process for us is **moving users between groups**, determining the composition of these groups, etc.

I proceed from the following:
- User league changes happen **once a week** based on their performance results for the last week
- Users compete in a group with other users of the same league. The maximum size of such a group is **fixed and equals 30**
- The only criterion determining whether a user remains in this group after a week is their **position on the leaderboard** in their group
- **5 are promoted** to the league above, **10 to the league below**, the rest remain in the current group
- Every week **new users register** in the application, new groups are created according to the number of new users

## Architecture Understanding

A little bit of attention needs to be paid to arrive at a common understanding of the warehouse architecture and data flows.

I proceed from the fact that:
- Data on user interaction with the application comes to ClickHouse in the form of **clickstream** (event records). The application transmits event details in `json_payload`
- Due to the necessity of maintaining **data consistency and their sensitivity** (GDPR), all data except clickstream from the application comes to ClickHouse **from PostgreSQL**
- Given that data comes from PostgreSQL via **CDC**, we will create separately **staging table** (for CDC) and **SCD2 table** for storing history and analytical queries
- In order to extract insights needed by analysts from the clickstream, there will be a **separate clickstream parsing process**

## Entity Analysis

The list of entities after business analysis looks as follows:

### **Anticipated Reference Tables:**
- Users
- League types (gold, silver, etc.)
- League groups (those where users are placed)
- Lessons
- Event types
- Calendar

### **Anticipated Fact Tables:**
- Application clickstream events
- User league group assignments
- User activity based on clickstream parsing results

For each entity except clickstream, we create **two tables** (one staging for CDC processing, another SCD2 for storing validity periods of entity parameters and analytics).

## Data Generation Cycle

Over **4 weeks**, new users come to us, and existing ones take lessons and move between groups. This means we will perform **generation in a cycle**.

### Generation Order

The generation order based on dependencies between entities is as follows:
- **Leagues** - Once
- **Event types** - Once
- **Lessons** - Once
- **Users** - Initial and weekly
- **League groups** - Weekly
- **User league assignments** - Weekly

## Detailed Implementation Process

If we go into a bit more detail, the generation and population order is as follows:

1. **Generate dataframe** of main reference tables (league types, lessons, event types, calendar) and **load to staging**
2. **Run SQL script** to populate corresponding SCD2 tables based on staging data
3. **Generate dataframe** for league groups (based on number of new users)
4. **Assign users to league groups** (while updating `actual_participants` attribute) based on previous results
5. **Generate clickstream** only for users who are assigned to league groups
6. **Parse clickstream** and extract lesson completion information, **populate user_activity table**
7. **Close validity periods** for groups (with setting `actual_participants` for the previous period)
8. **Close validity periods** for user group assignments (with setting `final_rank`, `total_xp_earned`, `outcome` based on their weekly activity)

## Architecture Diagram

```mermaid
erDiagram
    %% STAGING TABLES (CDC)
    ligue_tiers_staging {
        String op "INSERT/UPDATE/DELETE"
        UInt8 tier_id PK
        String tier_name
        UInt8 promotion_slots
        UInt8 relegation_slots
        DateTime cdc_timestamp
        Bool processed
        DateTime inserted_at
    }

    event_types_staging {
        String op "INSERT/UPDATE/DELETE"
        UInt16 event_type_id PK
        String event_name
        String event_category
        String description
        DateTime cdc_timestamp
        Bool processed
        DateTime inserted_at
    }

    lessons_staging {
        String op "INSERT/UPDATE/DELETE"
        UInt32 lesson_id PK
        String lesson_name
        String language
        String difficulty
        String category
        UInt16 xp_reward
        UInt8 estimated_minutes
        DateTime cdc_timestamp
        Bool processed
        DateTime inserted_at
    }

    users_staging {
        String op "INSERT/UPDATE/DELETE"
        UInt64 user_id PK
        String username
        String email
        FixedString country_code
        String region
        String gender
        Date birth_date
        String subscription_tier
        Date signup_date
        UInt8 current_tier
        Bool is_deleted
        DateTime cdc_timestamp
        Bool processed
        DateTime inserted_at
    }

    ligue_instances_staging {
        String op "INSERT/UPDATE/DELETE"
        UInt64 instance_id PK
        UInt8 tier_id FK
        Date week_start_date
        Date week_end_date
        UInt16 max_participants
        UInt16 actual_participants
        DateTime cdc_timestamp
        Bool processed
        DateTime inserted_at
    }

    ligue_assignment_staging {
        String op "INSERT/UPDATE/DELETE"
        UInt64 user_id FK
        UInt64 instance_id FK
        UInt8 tier_id FK
        Date week_start_date
        Date week_end_date
        UInt16 starting_rank "nullable"
        UInt16 final_rank "nullable"
        UInt32 total_xp_earned
        String outcome "nullable"
        DateTime cdc_timestamp
        Bool processed
        DateTime inserted_at
    }

    %% SCD2 DIMENSION TABLES
    ligue_tiers {
        UInt8 tier_id PK
        String tier_name
        UInt8 promotion_slots
        UInt8 relegation_slots
        DateTime valid_from
        DateTime valid_to
        Bool is_current
        UInt32 version
        Bool is_active
        String source_system
        DateTime inserted_at
        DateTime updated_at
    }

    event_types {
        UInt16 event_type_id PK
        String event_name
        String event_category
        String description
        DateTime valid_from
        DateTime valid_to
        Bool is_current
        UInt32 version
        Bool is_active
        String source_system
        DateTime inserted_at
        DateTime updated_at
    }

    lessons {
        UInt32 lesson_id PK
        String lesson_name
        String language
        String difficulty
        String category
        UInt16 xp_reward
        UInt8 estimated_minutes
        DateTime valid_from
        DateTime valid_to
        Bool is_current
        UInt32 version
        Bool is_active
        String source_system
        DateTime inserted_at
        DateTime updated_at
    }

    users {
        UInt64 user_id PK
        String username
        String email
        FixedString country_code
        String region
        String gender
        Date birth_date
        String subscription_tier
        Date signup_date
        UInt8 current_tier FK
        Bool is_deleted
        DateTime valid_from
        DateTime valid_to
        Bool is_current
        UInt32 version
        Bool is_active
        String source_system
        DateTime inserted_at
        DateTime updated_at
    }

    ligue_instances {
        UInt64 instance_id PK
        UInt8 tier_id FK
        Date week_start_date FK
        Date week_end_date
        UInt16 max_participants
        UInt16 actual_participants
        DateTime valid_from
        DateTime valid_to
        Bool is_current
        UInt32 version
        Bool is_active
        String source_system
        DateTime inserted_at
        DateTime updated_at
    }

    %% CALENDAR DIMENSION
    dim_weeks {
        Date week_start_date PK
        Date week_end_date
        UInt8 week_number
        UInt8 week_of_year
        UInt8 month
        UInt16 year
        UInt8 quarter
        Bool is_current_week
        Bool is_month_start_week
        Bool is_month_end_week
        Bool is_quarter_start_week
        Bool is_quarter_end_week
        DateTime created_at
    }

    %% FACT TABLES
    event_stream {
        String event_id PK
        UInt64 user_id FK
        UInt16 event_type_id FK
        DateTime64 event_timestamp
        Date event_date
        String payload
        String session_id
        String platform
        String app_version
        String device_id
        DateTime inserted_at
    }

    user_activity {
        UInt64 activity_id PK
        UInt64 user_id FK
        UInt32 lesson_id FK
        DateTime activity_timestamp
        Date activity_date
        Date week_start_date FK
        String status
        UInt16 xp_earned
        UInt8 score
        UInt8 mistakes_made
        UInt16 time_spent_seconds
        String session_id
        String platform
        DateTime inserted_at
    }

    ligue_assignment {
        UInt64 user_id FK
        UInt64 instance_id FK
        UInt8 tier_id FK
        Date week_start_date FK
        Date week_end_date
        UInt16 starting_rank "nullable"
        UInt16 final_rank "nullable"
        UInt32 total_xp_earned
        String outcome "nullable"
        DateTime valid_from
        DateTime valid_to
        Bool is_current
        UInt32 version
        Bool is_active
        String source_system
        DateTime inserted_at
        DateTime updated_at
    }

    %% RELATIONSHIPS
    %% Staging to SCD2 (CDC flow)
    ligue_tiers_staging ||--o{ ligue_tiers : "SCD2 processing"
    event_types_staging ||--o{ event_types : "SCD2 processing"
    lessons_staging ||--o{ lessons : "SCD2 processing"
    users_staging ||--o{ users : "SCD2 processing"
    ligue_instances_staging ||--o{ ligue_instances : "SCD2 processing"
    ligue_assignment_staging ||--o{ ligue_assignment : "SCD2 processing"

    %% Business relationships
    users ||--o{ ligue_assignment : "user participates"
    ligue_instances ||--o{ ligue_assignment : "instance contains"
    ligue_tiers ||--o{ ligue_instances : "tier defines"
    ligue_tiers ||--o{ users : "current_tier"
    dim_weeks ||--o{ ligue_instances : "week period"
    dim_weeks ||--o{ ligue_assignment : "week period"

    %% Event processing flow
    users ||--o{ event_stream : "user generates"
    event_types ||--o{ event_stream : "event classification"
    event_stream ||--o{ user_activity : "parsed from"
    lessons ||--o{ user_activity : "lesson completion"
    users ||--o{ user_activity : "user activity"
    dim_weeks ||--o{ user_activity : "week aggregation"
```

### **Diagram Legend:**
- **Staging Tables**: CDC operations (op, cdc_timestamp, processed)
- **SCD2 Tables**: Historicity (valid_from/valid_to, is_current, version)
- **Fact Tables**: Business events (append-only or SCD2)
- **Calendar Dimension**: Time periods

## Implementation Flow Diagram

```mermaid
graph TD
    %% Initialization (one-time)
    A[📅 dim_weeks] --> B[🏆 ligue_tiers_staging]
    B --> C[🏆 ligue_tiers SCD2]
    C --> D[⚡ event_types_staging]
    D --> E[⚡ event_types SCD2]
    E --> F[📚 lessons_staging]
    F --> G[📚 lessons SCD2]

    %% Weekly cycle - beginning of week
    G --> H[👥 users_staging]
    H --> I[👥 users SCD2]
    I --> J[🏟️ ligue_instances_staging]
    J --> K[🏟️ ligue_instances SCD2]
    K --> L[📋 ligue_assignment_staging INSERT]
    L --> M[📋 ligue_assignment SCD2]

    %% Event generation and activity
    M --> N[🎯 event_stream generation]
    N --> O[📊 user_activity parsing]

    %% End of week - UPDATE operations
    O --> P[🔄 ligue_instances UPDATE]
    P --> Q[📋 ligue_assignment_staging UPDATE]
    Q --> R[📋 ligue_assignment SCD2 finalization]

    %% Loop to next week
    R --> S{More weeks?}
    S -->|Yes| H
    S -->|No| T[✅ Complete]

    %% Styles
    classDef staging fill:#ffeaa7,stroke:#fdcb6e,stroke-width:2px
    classDef scd2 fill:#81ecec,stroke:#00cec9,stroke-width:2px
    classDef process fill:#fd79a8,stroke:#e84393,stroke-width:2px
    classDef decision fill:#a29bfe,stroke:#6c5ce7,stroke-width:2px

    class B,D,F,H,J,L,Q staging
    class C,E,G,I,K,M,R scd2
    class N,O,P process
    class S decision
```

### **Flow Legend:**
- **🟡 Staging**: CDC data loading to staging tables
- **🔵 SCD2**: Historical data processing with versioning
- **🔴 Process**: Business logic execution (events, parsing, updates)
- **🟣 Decision**: Flow control for weekly cycles
