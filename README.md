# ORM Validation and Properties

## Learning Goals

- Evolve attributes to properties.
- Adapt ORM methods to validate object state with properties

---

## Key Vocab

- **Object-Relational Mapping (ORM)**: a programming technique that provides a
  mapping between an object-oriented data model and a relational database model.
- **Attribute**: variables that belong to an object.
- **Property**: attributes that are controlled by methods.
- **Decorator**: function that takes another function as an argument and returns
  a new function with added functionality.

---

## Introduction

Let's update our company data model to add some constraints on the `Department`
and `Employee` attributes:

- The `Department` name and location must be non-empty strings.
- The `Employee` name and and job title must be non-empty strings.
- The `Employee` department must reference `Department` object that has been
  persisted to the database.

We have a choice of either performing validation within the database schema
itself, or validating within our Python classes prior to persisting object data
to the database. We'll pick the later approach, validating within the Python
classes. We'll do this by evolving the attributes to properties. The property
setter methods will ensure the attributes are assigned valid values.

---

## Code Along

This lesson is a code-along, so fork and clone the repo.

**NOTE: Remember to run `pipenv install` to install the dependencies and
`pipenv shell` to enter your virtual environment before running your code.**

```bash
pipenv install
pipenv shell
```

## Evolve `Department` attributes to properties

We'll start by evolving the `Department` attributes `name` and `location` to be
properties, as shown in the code below. The setter methods will check for
non-empty string values prior to updating the object state:

```py
from config import CURSOR, CONN

class Department:

    def __init__(self, name, location, id=None):
        self.id = id
        self.name = name
        self.location = location

    def __repr__(self):
        return f"<Department {self.id}: {self.name}, {self.location}>"

    @property
    def name(self):
        return self._name

    @name.setter
    def name(self, name):
        if isinstance(name, str) and len(name) > 0:
            self._name = name
        else:
            raise ValueError(
                "Name cannot be empty and must be a string"
            )

    @property
    def location(self):
        return self._location

    @location.setter
    def location(self, location):
        if isinstance(location, str) and len(location) > 0:
            self._location = location
        else:
            raise ValueError(
                "Location cannot be empty and must be a string"
            )

    # Existing ORM methods ....

```

We'll also update the `Employee` class to evolve the `name`, `job_title` and
`department` attributes to properties. Note the `department` setter method
checks to ensure we are assigning a valid department by checking the foreign key
reference in the database:

```py
from config import CURSOR, CONN
from department import Department


class Employee:

    def __init__(self, name, job_title, department, id=None):
        self.id = id
        self.name = name
        self.job_title = job_title
        self.department = department

    def __repr__(self):
        return (
            f"<Employee {self.id}: {self.name}, {self.job_title}, "
            + f"Department: {self.department.name} >"
        )

    @property
    def name(self):
        return self._name

    @name.setter
    def name(self, name):
        if isinstance(name, str) and len(name) > 0:
            self._name = name
        else:
            raise ValueError(
                "Name cannot be empty and must be a string"
            )

    @property
    def job_title(self):
        return self._job_title

    @job_title.setter
    def job_title(self, job_title):
        if isinstance(job_title, str) and len(job_title) > 0:
            self._job_title = job_title
        else:
            raise ValueError(
                "job_title cannot be empty and must be a string"
            )

    @property
    def department(self):
        return self._department

    @department.setter
    def department(self, department):
        if isinstance(department, Department) and Department.find_by_id(department.id) is not None:
            self._department = department
        else:
            raise ValueError(
                "Department must be class instance and reference existing entity in database")

     # Existing ORM methods ....

```

The `lib/testing` folder has two new test files for testing the properties,
`department_property_test.py` and `employee_property_test.py`. Run the tests to
ensure the property validations work correctly:

```bash
pytest -x
```

You can also use an `ipdb` session to test the properties.

```bash
python lib/debug.py
```

The `debug.py` file seeds the tables with random data. We can attempt to assign
invalid values (NOTE: your data will look different):

```py
ipdb> Department.get_all()
[<Department 1: Payroll, Building A, 5th Floor>, <Department 2: Human Resources, Building C, East Wing>]
ipdb> payroll = Department.find_by_name("Payroll")
ipdb> payroll
<Department 1: Payroll, Building A, 5th Floor>
ipdb> payroll.location = 7
*** ValueError: Location cannot be empty and must be a string
ipdb>
```

Let's try to set an invalid department id for an employee:

```bash
ipdb> employee = Employee.find_by_id(1)
ipdb> employee
<Employee 1: Courtney Riley, Manager, Department: Payroll >
ipdb> employee.department = 1000
*** ValueError: Department must be class instance and reference existing entity in database
ipdb> employee.department = Department("HR", "building b") #not persisted in db
*** ValueError: Department must be class instance and reference existing entity in database
```

## Conclusion

Properties control the values we assign to Python object attributes. We can use
property setter methods to ensure we assign valid values prior to persisting
objects to the database.

## Solution Code

```py
from config import CURSOR, CONN


class Department:

    def __init__(self, name, location, id=None):
        self.id = id
        self.name = name
        self.location = location

    def __repr__(self):
        return f"<Department {self.id}: {self.name}, {self.location}>"

    @property
    def name(self):
        return self._name

    @name.setter
    def name(self, name):
        if isinstance(name, str) and len(name) > 0:
            self._name = name
        else:
            raise ValueError(
                "Name cannot be empty and must be a string"
            )

    @property
    def location(self):
        return self._location

    @location.setter
    def location(self, location):
        if isinstance(location, str) and len(location) > 0:
            self._location = location
        else:
            raise ValueError(
                "Location cannot be empty and must be a string"
            )

    @classmethod
    def create_table(cls):
        """ Create a new table to persist the attributes of Department class instances """
        sql = """
            CREATE TABLE IF NOT EXISTS departments (
            id INTEGER PRIMARY KEY,
            name TEXT,
            location TEXT)
        """
        CURSOR.execute(sql)
        CONN.commit()

    @classmethod
    def drop_table(cls):
        """ Drop the table that persists Department class instances """
        sql = """
            DROP TABLE IF EXISTS departments;
        """
        CURSOR.execute(sql)
        CONN.commit()

    def save(self):
        """ Insert a new row with the name and location values of the current Department object.
        Update object id attribute using the primary key value of new row"""
        sql = """
            INSERT INTO departments (name, location)
            VALUES (?, ?)
        """

        CURSOR.execute(sql, (self.name, self.location))
        CONN.commit()

        self.id = CURSOR.lastrowid

    @classmethod
    def create(cls, name, location):
        """ Initialize a new Department object and save the object to the database """
        department = Department(name, location)
        department.save()
        return department

    def update(self):
        """Update the table row corresponding to the current Department object."""
        sql = """
            UPDATE departments
            SET name = ?, location = ?
            WHERE id = ?
        """
        CURSOR.execute(sql, (self.name, self.location, self.id))
        CONN.commit()

    def delete(self):
        """Delete the table row corresponding to the current Department class instance"""
        sql = """
            DELETE FROM departments
            WHERE id = ?
        """

        CURSOR.execute(sql, (self.id,))
        CONN.commit()

    @classmethod
    def new_from_db(cls, row):
        """Return a new Department object using the values from the table row."""
        department = cls(row[1], row[2])
        department.id = row[0]
        return department

    @classmethod
    def get_all(cls):
        """Return a list containing a new Department object for each row in the table"""
        sql = """
            SELECT *
            FROM departments
        """

        rows = CURSOR.execute(sql).fetchall()

        cls.all = [cls.new_from_db(row) for row in rows]
        return cls.all

    @classmethod
    def find_by_id(cls, id):
        """Return a new Department object corresponding to the table row matching the specified primary key"""
        sql = """
            SELECT *
            FROM departments
            WHERE id = ?
        """

        row = CURSOR.execute(sql, (id,)).fetchone()
        return cls.new_from_db(row) if row else None

    @classmethod
    def find_by_name(cls, name):
        """Return a new Department object corresponding to first table row matching specified name"""
        sql = """
            SELECT *
            FROM departments
            WHERE name is ?
        """

        row = CURSOR.execute(sql, (name,)).fetchone()
        return cls.new_from_db(row) if row else None

    def employees(self):
        from employee import Employee
        sql = """
            SELECT * FROM employees
            WHERE department_id = ?
        """
        CURSOR.execute(sql, (self.id,),)

        rows = CURSOR.fetchall()
        return [
            Employee(row[1], row[2], self, row[0]) for row in rows
        ]

```

```py
from config import CURSOR, CONN
from department import Department


class Employee:

    def __init__(self, name, job_title, department, id=None):
        self.id = id
        self.name = name
        self.job_title = job_title
        self.department = department

    def __repr__(self):
        return (
            f"<Employee {self.id}: {self.name}, {self.job_title}, "
            + f"Department: {self.department.name} >"
        )

    @property
    def name(self):
        return self._name

    @name.setter
    def name(self, name):
        if isinstance(name, str) and len(name) > 0:
            self._name = name
        else:
            raise ValueError(
                "Name cannot be empty and must be a string"
            )

    @property
    def job_title(self):
        return self._job_title

    @job_title.setter
    def job_title(self, job_title):
        if isinstance(job_title, str) and len(job_title) > 0:
            self._job_title = job_title
        else:
            raise ValueError(
                "job_title cannot be empty and must be a string"
            )

    @property
    def department(self):
        return self._department

    @department.setter
    def department(self, department):
        if isinstance(department, Department) and Department.find_by_id(department.id) is not None:
            self._department = department
        else:
            raise ValueError(
                "Department must be class instance and reference existing entity in database")

    @classmethod
    def create_table(cls):
        sql = """
            CREATE TABLE IF NOT EXISTS employees (
            id INTEGER PRIMARY KEY,
            name TEXT,
            job_title TEXT,
            department_id INTEGER,
            FOREIGN KEY (department_id) REFERENCES departments(id))
        """
        CURSOR.execute(sql)
        CONN.commit()

    @classmethod
    def drop_table(cls):
        """ Drop the table that persists Employee class instances """
        sql = """
            DROP TABLE IF EXISTS employees;
        """
        CURSOR.execute(sql)
        CONN.commit()

    def save(self):
        sql = """
            INSERT INTO employees (name, job_title, department_id)
            VALUES (?, ?, ?)
        """

        CURSOR.execute(sql, (self.name, self.job_title, self.department.id))
        CONN.commit()

        self.id = CURSOR.lastrowid

    def update(self):
        sql = """
            UPDATE employees
            SET name = ?, job_title = ?, department_id = ?
            WHERE id = ?
        """
        CURSOR.execute(sql, (self.name, self.job_title,
                             self.department.id, self.id))
        CONN.commit()

    @classmethod
    def create(cls, name, job_title, department):
        """ Initialize a new Employee object and save the object to the database """
        employee = Employee(name, job_title, department)
        employee.save()
        return employee

    def delete(self):
        sql = """
            DELETE FROM employees
            WHERE id = ?
        """

        CURSOR.execute(sql, (self.id,))
        CONN.commit()

    @classmethod
    def new_from_db(cls, row):
        """Initialize a new Employee object using the values from the table row."""
        department = Department.find_by_id(row[3])
        employee = cls(row[1], row[2], department)
        employee.id = row[0]
        return employee

    @classmethod
    def get_all(cls):
        """Return a list containing one Employee object per table row"""
        sql = """
            SELECT *
            FROM employees
        """

        rows = CURSOR.execute(sql).fetchall()

        cls.all = [cls.new_from_db(row) for row in rows]
        return cls.all

    @classmethod
    def find_by_id(cls, id):
        """Return Employee object corresponding to the table row matching the specified primary key"""
        sql = """
            SELECT *
            FROM employees
            WHERE id = ?
        """

        row = CURSOR.execute(sql, (id,)).fetchone()
        return cls.new_from_db(row) if row else None

    @classmethod
    def find_by_name(cls, name):
        """Return Employee object corresponding to first table row matching specified name"""
        sql = """
            SELECT *
            FROM employees
            WHERE name is ?
        """

        row = CURSOR.execute(sql, (name,)).fetchone()
        return cls.new_from_db(row) if row else None

```
