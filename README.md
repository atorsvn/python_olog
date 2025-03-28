# Python Olog Module with SQLite Integration

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python Version](https://img.shields.io/badge/python-3.7+-blue.svg)](https://www.python.org/)

## Description

This repository provides a Python module for representing and interacting with **Ologs (Ontology Logs)**, a concept introduced by David Spivak and Robert Kent. Ologs use category theory principles to create structured models of knowledge domains.

This implementation focuses on:
* Representing the core structure of an Olog (Objects as types/boxes, Morphisms as relationships/arrows).
* Integrating with **SQLite** for persisting instances (data) associated with Olog Objects, particularly useful when the conceptual target category is `Set`.
* Automatic (basic) database schema generation based on the Olog structure.
* Methods for adding and querying instance data via the database.
* Visualization of the Olog structure using **Graphviz**.
* Designed primarily for use within **Jupyter Notebooks**.

## Features

* Object-oriented representation of Olog components (`Olog`, `OlogObject`, `OlogMorphism`).
* Optional validation for Olog object naming conventions ("a"/"an").
* Association of an Olog with an SQLite database file.
* Automatic generation of SQLite tables for Olog Objects.
    * Includes `id` (primary key) and `value` (text) columns.
* Representation of functional morphisms (`is_functional_fk=True`) as foreign key columns in corresponding tables.
* Automatic generation of `FOREIGN KEY` constraints.
* Pythonic API for adding instances (`add_instance`) and querying data (`get_instances`, `get_instance_by_id`, `get_related_instance`).
* Context manager support for safe database connection handling.
* Generation of DOT language for visualization via Graphviz (`to_dot`, `view`).

## Target Environment

* **Python 3.7+**
* **Jupyter Notebook / Jupyter Lab** (for visualization rendering and typical usage)

## Dependencies

1.  **Python Libraries:**
    * `graphviz`: The Python interface for Graphviz. Install using pip:
        ```bash
        pip install graphviz
        ```
2.  **System Tools:**
    * **Graphviz**: The core visualization software. You **must** install this separately for the `graphviz` Python library to function. Installation instructions vary by OS: [https://graphviz.org/download/](https://graphviz.org/download/)

## Installation

1.  **Clone the repository:**
    ```bash
    git clone https://github.com/atorsvn/python_olog
    cd https://github.com/atorsvn/python_olog
    ```
2.  **Install dependencies:**
    ```bash
    pip install graphviz
    # Ensure Graphviz system tools are installed (see Dependencies section)
    ```
3.  **Use in Jupyter:** The primary way to use this module is by importing the classes (or running the defining cells) within a Jupyter Notebook (`.ipynb`).

## Usage

*(Assuming the classes `Olog`, `OlogObject`, `OlogMorphism` are defined as in the provided code, either in a `.py` file imported into your notebook, or directly in notebook cells)*

```python
# Example within a Jupyter Notebook cell

# Import necessary classes (if defined in a separate .py file)
# from olog import Olog, OlogObject, OlogMorphism # Adjust import as needed

# --- 1. Define Olog Structure ---
db_file = "my_knowledge_base.db"
my_olog = Olog(
    name="Example Domain",
    db_path=db_file, # Associate with a database file
    description="A simple example Olog with DB integration"
)

# Add objects (which will become tables)
my_olog.add_object("a user")
my_olog.add_object("an email address")
my_olog.add_object("a profile")

# Add morphisms (relationships)
# Mark 'has_primary_email' and 'has_profile' as functional foreign keys
my_olog.add_morphism("a user", "an email address", "has_primary_email", is_functional_fk=True)
my_olog.add_morphism("a user", "a profile", "has_profile", is_functional_fk=True)
# Non-functional relationship (won't create a default FK column)
my_olog.add_morphism("a user", "an email address", "has_secondary_email")

print("--- Olog Definition ---")
print(my_olog)

# --- 2. Visualize Structure ---
# Requires Graphviz system tools and Python library
viz = my_olog.view()
# Display the visualization object in Jupyter
# viz

# --- 3. Create Database Schema ---
print("\n--- Creating DB Schema ---")
# Use context manager for safe connection handling
try:
    with my_olog as olog_db:
        # drop_existing=True clears tables first - use with caution!
        olog_db.create_tables_from_olog(drop_existing=True)
except Exception as e:
    print(f"Schema creation failed: {e}")

# --- 4. Add Instances (Data) ---
print("\n--- Adding Instances ---")
try:
    with my_olog as olog_db:
        # Add instances that will be referenced first
        email1_id = olog_db.add_instance("an email address", "alice@example.com")
        email2_id = olog_db.add_instance("an email address", "bob@example.com")
        profile1_id = olog_db.add_instance("a profile", "Alice's Profile Data...")
        profile2_id = olog_db.add_instance("a profile", "Bob's Simple Profile")

        # Add 'user' instances, linking via FK morphism labels
        user1_id = olog_db.add_instance("a user", "Alice",
                                       has_primary_email=email1_id,
                                       has_profile=profile1_id)
        user2_id = olog_db.add_instance("a user", "Bob",
                                       has_primary_email=email2_id,
                                       has_profile=profile2_id) # Link Bob to Profile 2

        print(f"Added User Alice (ID: {user1_id}), Bob (ID: {user2_id})")
        print(f"Added Email IDs: {email1_id}, {email2_id}")
        print(f"Added Profile IDs: {profile1_id}, {profile2_id}")

except Exception as e:
    print(f"Adding instances failed: {e}")

# --- 5. Query Instances ---
print("\n--- Querying Instances ---")
try:
    with my_olog as olog_db:
        # Get all users
        users = olog_db.get_instances("a user")
        print("Users:")
        for user in users:
            print(f"  ID: {user['id']}, Name: {user['value']}, Email FK: {user['has_primary_email_id']}, Profile FK: {user['has_profile_id']}")

        # Get Alice (assuming ID is user1_id) and her primary email
        if 'user1_id' in locals():
            alice = olog_db.get_instance_by_id("a user", user1_id)
            if alice:
                 print(f"\nAlice's Data: {dict(alice)}")
                 # Follow the FK relationship using the morphism label
                 alice_email = olog_db.get_related_instance(
                     source_obj_label="a user",
                     source_instance_id=user1_id,
                     morphism_label="has_primary_email"
                 )
                 if alice_email:
                     print(f"  Alice's Primary Email: {alice_email['value']} (ID: {alice_email['id']})")

except Exception as e:
    print(f"Querying failed: {e}")

# The 'with' block automatically closes the DB connection
print("\n--- Usage Example Complete ---")
