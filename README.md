## Application Description

**Application Name:** Student Registration and Grading Management System

This application is developed using Oracle PL/SQL to manage student information, course registrations, and academic grades.

The project demonstrates the use of records, collections, implicit cursors, explicit cursors, bulk processing, stored procedures, functions, and exception handling.

## Repository Structure


│
├── README.md
├── .gitignore
│
├── 00_schema/
│   ├── 01_create_tables.sql
SET SERVEROUTPUT ON;

BEGIN EXECUTE IMMEDIATE 'DROP TABLE enrollments CASCADE CONSTRAINTS'; EXCEPTION WHEN OTHERS THEN NULL; END;
/
BEGIN EXECUTE IMMEDIATE 'DROP TABLE courses CASCADE CONSTRAINTS'; EXCEPTION WHEN OTHERS THEN NULL; END;
/
BEGIN EXECUTE IMMEDIATE 'DROP TABLE students CASCADE CONSTRAINTS'; EXCEPTION WHEN OTHERS THEN NULL; END;
/
BEGIN EXECUTE IMMEDIATE 'DROP TABLE departments CASCADE CONSTRAINTS'; EXCEPTION WHEN OTHERS THEN NULL; END;
/

CREATE TABLE departments (
  dept_id    NUMBER       PRIMARY KEY,
  dept_name  VARCHAR2(100) NOT NULL UNIQUE
);

CREATE TABLE students (
  student_id   NUMBER        PRIMARY KEY,
  full_name    VARCHAR2(100) NOT NULL,
  email        VARCHAR2(120) NOT NULL UNIQUE,
  dept_id      NUMBER        NOT NULL,
  CONSTRAINT fk_student_dept FOREIGN KEY (dept_id) REFERENCES departments (dept_id)
);

CREATE TABLE courses (
  course_id    NUMBER       PRIMARY KEY,
  course_code  VARCHAR2(10) NOT NULL UNIQUE,
  title        VARCHAR2(120) NOT NULL,
  credits      NUMBER(2)    NOT NULL CHECK (credits BETWEEN 1 AND 6),
  capacity     NUMBER(4)    NOT NULL CHECK (capacity > 0),
  dept_id      NUMBER       NOT NULL,
  CONSTRAINT fk_course_dept FOREIGN KEY (dept_id) REFERENCES departments (dept_id)
);

CREATE TABLE enrollments (
  enrollment_id NUMBER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  student_id    NUMBER      NOT NULL,
  course_id     NUMBER      NOT NULL,
  grade         VARCHAR2(2),
  CONSTRAINT fk_enr_student FOREIGN KEY (student_id) REFERENCES students (student_id) ON DELETE CASCADE,
  CONSTRAINT fk_enr_course  FOREIGN KEY (course_id) REFERENCES courses (course_id) ON DELETE CASCADE,
  CONSTRAINT uq_enrollment  UNIQUE (student_id, course_id),
  CONSTRAINT ck_grade CHECK (grade IN ('A','B+','B','C','D','F') OR grade IS NULL)
);
COMMIT;
│   └── 02_insert_sample_data.sql
INSERT INTO departments VALUES (1, 'Computer Science');
INSERT INTO departments VALUES (2, 'Business Administration');

INSERT INTO students VALUES (101, 'Alice Uwase',   'alice@auca.ac.rw',  1);
INSERT INTO students VALUES (102, 'Bob Mugisha',   'bob@auca.ac.rw',    1);
INSERT INTO students VALUES (103, 'Cara Niyonsaba','cara@auca.ac.rw',   1);
INSERT INTO students VALUES (104, 'David Habimana','david@auca.ac.rw',  2);

INSERT INTO courses VALUES (201, 'INSY8311', 'Database Development', 3, 30, 1);
INSERT INTO courses VALUES (202, 'INSY8201', 'Data Structures',      4, 25, 1);
INSERT INTO courses VALUES (203, 'BUSI8101', 'Principles of Mgmt',   3, 40, 2);

INSERT INTO enrollments (student_id, course_id, grade) VALUES (101, 201, 'A');
INSERT INTO enrollments (student_id, course_id, grade) VALUES (101, 202, 'B+');
INSERT INTO enrollments (student_id, course_id, grade) VALUES (102, 201, 'B');
INSERT INTO enrollments (student_id, course_id, grade) VALUES (104, 203, 'A');
COMMIT;
│
├── 01_procedures/
│   └── procedure_name.sql
CREATE OR REPLACE PROCEDURE register_student (
  p_student_id IN  NUMBER,
  p_full_name  IN  VARCHAR2,
  p_email      IN  VARCHAR2,
  p_dept_id    IN  NUMBER,
  p_out_msg    OUT VARCHAR2
) AS
  TYPE emp_rec_type IS RECORD (
    student_id students.student_id%TYPE,
    full_name  students.full_name%TYPE,
    email      students.email%TYPE
  );
  v_student emp_rec_type;

  e_duplicate_student EXCEPTION;
  v_count NUMBER;
BEGIN
  SELECT COUNT(*) INTO v_count FROM students WHERE student_id = p_student_id;
  IF v_count > 0 THEN
    RAISE e_duplicate_student;
  END IF;

  v_student.student_id := p_student_id;
  v_student.full_name  := p_full_name;
  v_student.email      := p_email;

  INSERT INTO students (student_id, full_name, email, dept_id)
  VALUES (v_student.student_id, v_student.full_name, v_student.email, p_dept_id);

  IF SQL%ROWCOUNT = 1 THEN
    p_out_msg := 'Student ' || v_student.full_name || ' registered successfully.';
  END IF;
  COMMIT;

EXCEPTION
  WHEN e_duplicate_student THEN
    ROLLBACK;
    p_out_msg := 'Error: Student ID ' || p_student_id || ' already exists.';
  WHEN OTHERS THEN
    ROLLBACK;
    p_out_msg := 'Unexpected error: ' || SQLERRM;
END;
/
CREATE OR REPLACE PROCEDURE bulk_enroll_students (
  p_student_ids IN  SYS.ODCINUMBERLIST,
  p_course_id   IN  NUMBER,
  p_out_msg     OUT VARCHAR2
) AS
  TYPE t_id_tab IS TABLE OF NUMBER;
  v_student_ids t_id_tab;
  v_errors NUMBER;
BEGIN
  SELECT column_value BULK COLLECT INTO v_student_ids FROM TABLE(p_student_ids);

  BEGIN
    FORALL i IN 1 .. v_student_ids.COUNT SAVE EXCEPTIONS
      INSERT INTO enrollments (student_id, course_id)
      VALUES (v_student_ids(i), p_course_id);
  EXCEPTION
    WHEN OTHERS THEN
      v_errors := SQL%BULK_EXCEPTIONS.COUNT;
      p_out_msg := 'Bulk enrollment partially failed. Errors: ' || v_errors;
      FOR j IN 1 .. v_errors LOOP
        DBMS_OUTPUT.PUT_LINE('Error at index ' || SQL%BULK_EXCEPTIONS(j).ERROR_INDEX || 
                             ' Code: ' || SQL%BULK_EXCEPTIONS(j).ERROR_CODE);
      END LOOP;
      RETURN;
  END;

  p_out_msg := 'Bulk enrollment complete: ' || SQL%ROWCOUNT || ' students enrolled.';
  COMMIT;
END;
/
CREATE OR REPLACE PROCEDURE assign_grade (
  p_student_id IN  NUMBER,
  p_course_id  IN  NUMBER,
  p_grade      IN  VARCHAR2,
  p_out_msg    OUT VARCHAR2
) AS
  e_no_enrollment EXCEPTION;
BEGIN
  IF p_grade NOT IN ('A','B+','B','C','D','F') THEN
    RAISE_APPLICATION_ERROR(-20010, 'Invalid grade: ' || p_grade);
  END IF;

  UPDATE enrollments SET grade = p_grade
   WHERE student_id = p_student_id AND course_id = p_course_id;

  IF SQL%NOTFOUND THEN
    RAISE e_no_enrollment;
  ELSIF SQL%ROWCOUNT = 1 THEN
    p_out_msg := 'Grade ' || p_grade || ' assigned.';
  END IF;
  COMMIT;

EXCEPTION
  WHEN e_no_enrollment THEN
    ROLLBACK;
    p_out_msg := 'Error: No enrollment found for student ' || p_student_id;
  WHEN OTHERS THEN
    ROLLBACK;
    p_out_msg := 'Error: ' || SQLERRM;
END;
/
CREATE OR REPLACE PROCEDURE list_department_students (
  p_dept_id IN  NUMBER,
  p_out_msg OUT VARCHAR2
) AS
  CURSOR c_students IS
    SELECT student_id, full_name FROM students WHERE dept_id = p_dept_id;
  v_rec c_students%ROWTYPE;
  v_buf VARCHAR2(4000);
BEGIN
  v_buf := 'Students in dept ' || p_dept_id || ':' || CHR(10);
  
  OPEN c_students;
  IF c_students%ISOPEN THEN
    DBMS_OUTPUT.PUT_LINE('Cursor opened successfully.');
  END IF;

  LOOP
    FETCH c_students INTO v_rec;
    EXIT WHEN c_students%NOTFOUND;
    v_buf := v_buf || v_rec.student_id || ' - ' || v_rec.full_name || CHR(10);
  END LOOP;
  
  CLOSE c_students;
  IF NOT c_students%ISOPEN THEN
    DBMS_OUTPUT.PUT_LINE('Cursor is successfully closed.');
  END IF;
  
  p_out_msg := v_buf;
END;
/
│
├── 02_functions/
│   └── function_name.sql
CREATE OR REPLACE FUNCTION get_student_gpa (
  p_student_id IN NUMBER
) RETURN NUMBER AS
  v_total_points  NUMBER := 0;
  v_total_credits NUMBER := 0;
  v_points        NUMBER;
BEGIN
  
  FOR r IN (
    SELECT e.grade, c.credits
      FROM enrollments e JOIN courses c ON c.course_id = e.course_id
     WHERE e.student_id = p_student_id AND e.grade IS NOT NULL
  ) LOOP
    v_points := CASE r.grade 
                  WHEN 'A' THEN 4.0 WHEN 'B+' THEN 3.5 WHEN 'B' THEN 3.0 
                  WHEN 'C' THEN 2.0 WHEN 'D' THEN 1.0 ELSE 0.0 END;
    v_total_points  := v_total_points + (v_points * r.credits);
    v_total_credits := v_total_credits + r.credits;
  END LOOP;

  IF v_total_credits = 0 THEN RETURN NULL; END IF;
  RETURN ROUND(v_total_points / v_total_credits, 2);
END;
/
CREATE OR REPLACE FUNCTION get_all_student_emails 
RETURN SYS_REFCURSOR IS
  emp_cursor SYS_REFCURSOR;
BEGIN
  OPEN emp_cursor FOR 
    SELECT student_id, full_name, email FROM students;
  RETURN emp_cursor;
END;
/
│
├── 03_tests/
│   ├── test_success_cases.sql
SET SERVEROUTPUT ON;
DECLARE
  v_msg VARCHAR2(4000);
  v_cursor SYS_REFCURSOR;
  v_id students.student_id%TYPE;
  v_name students.full_name%TYPE;
  v_email students.email%TYPE;
BEGIN
  DBMS_OUTPUT.PUT_LINE('=== TEST: Register Student ===');
  register_student(105, 'Eve Keza', 'eve@auca.ac.rw', 1, v_msg);
  DBMS_OUTPUT.PUT_LINE(v_msg);

  DBMS_OUTPUT.PUT_LINE('=== TEST: Enroll Course ===');
  enroll_course(105, 201, v_msg);
  DBMS_OUTPUT.PUT_LINE(v_msg);

  DBMS_OUTPUT.PUT_LINE('=== TEST: Assign Grade ===');
  assign_grade(105, 201, 'A', v_msg);
  DBMS_OUTPUT.PUT_LINE(v_msg);

  DBMS_OUTPUT.PUT_LINE('=== TEST: Bulk Enroll ===');
  bulk_enroll_students(SYS.ODCINUMBERLIST(102, 103), 202, v_msg);
  DBMS_OUTPUT.PUT_LINE(v_msg);

  DBMS_OUTPUT.PUT_LINE('=== TEST: List Department ===');
  list_department_students(1, v_msg);
  DBMS_OUTPUT.PUT_LINE(v_msg);

  DBMS_OUTPUT.PUT_LINE('=== TEST: SYS_REFCURSOR Function ===');
  v_cursor := get_all_student_emails;
  LOOP
    FETCH v_cursor INTO v_id, v_name, v_email;
    EXIT WHEN v_cursor%NOTFOUND;
    DBMS_OUTPUT.PUT_LINE(v_id || ' : ' || v_name || ' - ' || v_email);
  END LOOP;
  CLOSE v_cursor;
END;
/
│   └── test_error_cases.sql
SET SERVEROUTPUT ON;
DECLARE
  v_msg VARCHAR2(4000);
BEGIN
  DBMS_OUTPUT.PUT_LINE('=== ERROR TEST: Duplicate Student ===');
  register_student(101, 'Duplicate Alice', 'dup@auca.ac.rw', 1, v_msg);
  DBMS_OUTPUT.PUT_LINE(v_msg);

  DBMS_OUTPUT.PUT_LINE('=== ERROR TEST: Invalid Grade ===');
  BEGIN
    assign_grade(101, 201, 'Z', v_msg);
  EXCEPTION WHEN OTHERS THEN
    DBMS_OUTPUT.PUT_LINE('Caught: ' || SQLERRM);
  END;

  DBMS_OUTPUT.PUT_LINE('=== ERROR TEST: Bulk Enroll with Duplicates (SAVE EXCEPTIONS) ===');
  bulk_enroll_students(SYS.ODCINUMBERLIST(101, 101, 999), 201, v_msg);
  DBMS_OUTPUT.PUT_LINE(v_msg);
END;
/
│
├── screenshots/
<img width="1525" height="1695" alt="testt1" src="https://github.com/user-attachments/assets/c3ae3a7d-4f94-464b-b368-56ecc0d4d03c" />
<img width="1740" height="1785" alt="errors" src="https://github.com/user-attachments/assets/b2d8156f-1984-4412-b9a3-0d5ee3bc35c4" />



│
└── docs/
    └── IMPLEMENTATION_NOTES.md
```

## How to Run

1. Open Oracle SQL Developer and establish a database connection.
2. Execute `00_schema/01_create_tables.sql` to create the database tables.
3. Execute `00_schema/02_insert_sample_data.sql` to insert sample records.
4. Execute the stored procedure files inside `01_procedures/`.
5. Execute the stored function files inside `02_functions/`.
6. Run `03_tests/test_success_cases.sql`.
7. Run `03_tests/test_error_cases.sql`.
8. Check the execution results and screenshots in the `screenshots/` folder.
9. Review `docs/IMPLEMENTATION_NOTES.md` for implementation and performance explanations.
