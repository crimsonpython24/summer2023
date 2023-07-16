# Chapter 3

## M > V and C

- models are classes that provide an OOP way of dealing with databases
- model modules can be imported from the command line in such cases
  - django interactive shell
  - test scripts
  - async tasks such as those through Celery

## Class Diagrams

Drawing a class diagram helps establish an early form of the database structure and offers chances for revision
**Entity-relationship model (ER-model)**: a one-to-many (`1-n`) relationship where the `n` side is where the foreign key is declared

## Normalized Models

- Helps efficiently store data: once fully normalized, there will not be any _redundancies_
  - Each model should only contain data that is logically related to it
  - I.e., there will not be multiple fields that contain data representing the same purpose
    > Pattern: design models to be fully normalized, then selectively denormalize for performance reasons

### First Normal Form

A table must have:

- No cell with multiple values
- A primary key (most commonly UID) defined as a single column or a set of columns (aka composite key)
- I.e., if there is a cell (by design or not) that has multiple values, split each value into an individual row

### Second Normal Form

A table must have:

- Everything in the first normal form (1NF)
- All non-primary key columns must be dependent on the entire primary key
- E.g., if a column is not dependent on a composite PK, then extract it into a separate table

### Third Normal Form

A table must have:

- Everything in the second normal form (2NF)
- All non-primary key columns must be independent of each other
- E.g., if a column is dependent on other non-primary key columns, extract it into a separate table

### Example

**Original table:**

| Name      | Origin      | Power               | First Used At                                              |
| --------- | ----------- | ------------------- | ---------------------------------------------------------- |
| Blitz     | Alien       | Freeze; Flight      | +40, -73, USA, 2014/07/03; +40, -73, USA, 2013/03/12       |
| Hexa      | Scientist   | Telekinesis; Flight | +35, +139, Japan, 2010/02/17; +35, +139, Japan, 2010/02/19 |
| Traveller | Billionaire | Time travel         | +43, +1, France, 2010/11/10                                |

**After 1NF:**

| Name      | Origin      | Power       | Latitude | Longitude | Country | Time       |
| --------- | ----------- | ----------- | -------- | --------- | ------- | ---------- |
| Blitz     | Alien       | Freeze      | +40      | -73       | USA     | 2014/07/03 |
| Blitz     | Alien       | Flight      | +40      | -73       | USA     | 2013/03/12 |
| Hexa      | Scientist   | Telekinesis | +35      | +139      | Japan   | 2010/02/17 |
| Hexa      | Scientist   | Flight      | +35      | +139      | Japan   | 2010/02/19 |
| Traveller | Billionaire | Time travel | +43      | +1        | France  | 2010/11/10 |

**After 2NF:**

Main table:

| Name      | Power       | Latitude | Longitude | Country | Time       |
| --------- | ----------- | -------- | --------- | ------- | ---------- |
| Blitz     | Freeze      | +40      | -73       | USA     | 2014/07/03 |
| Blitz     | Flight      | +40      | -73       | USA     | 2013/03/12 |
| Hexa      | Telekinesis | +35      | +139      | Japan   | 2010/02/17 |
| Hexa      | Flight      | +35      | +139      | Japan   | 2010/02/19 |
| Traveller | Time travel | +43      | +1        | France  | 2010/11/10 |

Origin table:

| Name      | Origin      |
| --------- | ----------- |
| Blitz     | Alien       |
| Hexa      | Scientist   |
| Traveller | Billionaire |

> Origin is dependent on `name` only, but not the composite PK `name` and `power`, therefore it is extracted into a separate table with a single PK `name`

**After 3NF:**
Main table:

| User ID | Power       | Location ID | Time       |
| ------- | ----------- | ----------- | ---------- |
| 1       | Freeze      | 1           | 2014/07/03 |
| 1       | Flight      | 1           | 2013/03/12 |
| 2       | Telekinesis | 2           | 2010/02/17 |
| 2       | Flight      | 2           | 2010/02/19 |
| 3       | Time travel | 3           | 2010/11/10 |

Origin table:

| User ID | Name      | Origin      |
| ------- | --------- | ----------- |
| 1       | Blitz     | Alien       |
| 2       | Hexa      | Scientist   |
| 3       | Traveller | Billionaire |

Location Table:

| Location ID | Latitude | Longitude | Country |
| ----------- | -------- | --------- | ------- |
| 1           | +40      | -73       | USA     |
| 2           | +35      | +139      | Japan   |
| 3           | +43      | +1        | France  |

> The `latitude` and `longitude` columns are dependent on the `country` column, which is NOT a PK; therefore, the data is extracted into a separate table
> The User ID is also replaced for convenience in this step, but it does not directly have to do with 3NF (Django will assign each user with a PK by default)
