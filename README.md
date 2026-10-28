# course-catalog-db

PostgreSQL schema for the course catalog (Lab 6, “Advanced Web Technologies”).

## Constraint violation log

Commands were run with `\set VERBOSITY verbose` on PostgreSQL 17.11.

| # | What we violate | SQLSTATE | Constraint | API response |
|---|-----------------|----------|------------|--------------|
| 1 | NULL in courses.title | 23502 | NOT NULL has no name — the column `title` (COLUMN NAME) | 422; if Pydantic let an empty value through, it's a bug in the code (500) |
| 2 | courses.credits = 0 | 23514 | `courses_credits_check` | 422 |
| 3 | courses.likes = -1 | 23514 | `courses_likes_check` | 500: only our code changes the counter (`likes + 1`), so a negative value means a bug |
| 4 | a one-letter student name | 23514 | `students_name_check` | 422 |
| 5 | a duplicate student email | 23505 | `students_email_key` | 409 Conflict |
| 6 | a duplicate course id | 23505 | `courses_pkey` | 409 Conflict |
| 7 | a repeated enrollment in a course | 23505 | `enrollments_course_id_student_id_key` | 409 Conflict |
| 8 | enrolling in a nonexistent course | 23503 | `enrollments_course_id_fkey` | 404: course not found |
| 9 | enrolling a nonexistent student | 23503 | `enrollments_student_id_fkey` | 404: student not found |
| 10 | deleting a course that has enrollments | 23503 (PostgreSQL 17; 23001 on 18) | `enrollments_course_id_fkey` | 409 Conflict: a course with enrollments can't be deleted |
| 11 | deleting a student | — no error | `ON DELETE CASCADE` on `enrollments_student_id_fkey` | the enrollments were deleted by cascade: 6 → 3, 6 after ROLLBACK |

Extra check: inserting `students.id = 100` by hand gives `428C9`
(`cannot insert a non-DEFAULT value into column "id"`, the column is
`GENERATED ALWAYS`) — not a user error, so the API would answer 500.

After all the experiments the tables still have 6, 3 and 6 rows. In the reply
to command 4 the row was `(4, A, a@example.com)`: the number 4 was used up
before the check failed, so ids can have gaps.

**Why check in the database if Pydantic already checks the name?** The API is
not the only one who writes to the database: there are psql sessions, seed and
migration scripts, admin tools and maybe other services later. Pydantic only
protects requests that go through our API, while a constraint in the database
protects the data from every source, and it's the last line of defense if the
code has a bug.
