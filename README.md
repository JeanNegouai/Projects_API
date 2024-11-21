from bottle import Bottle, run, request, response, HTTPError
import sqlite3
import json

# Create SQLite database
def create_database():
    conn = sqlite3.connect('projects.db') # Connects to the SQLite database 'project.db'. If it doesn't exist, it will be created.
    c = conn.cursor() # Creates a cursor object that allows to execute SQL queries and fetch data from a database.
    c.execute('''                             
        CREATE TABLE IF NOT EXISTS projects (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            project_name TEXT NOT NULL, 
            grade TEXT,
            start_date TEXT,
            complete BOOLEAN
        )
    ''')
    conn.commit() # saving changes to the database
    conn.close() # closes the connection to the database 

create_database() # Runs code to create database and table 

# Create the Bottle application
app = Bottle() # creates a new instance of the Bottle application, will define routes and handle HTTP request 

# Database helper function
def get_db_connection(): # establishes a connection to the SQLite database
    conn = sqlite3.connect('projects.db')  # Ensure this is the correct database name 'projects.db'
    conn.row_factory = sqlite3.Row # Configures the connection so that rows returned by queries are accessible by column name 
    return conn # Returns the connection object so it can be used in the routes to interact with the database.

# SHOW ALL PROJECTS (GET /projects)
@app.route('/projects', method='GET') # This decorator registers a route that listens for GET requests at /projects.
def get_projects(): # Function that handles the GET request 
    conn = get_db_connection() # Establishes a connection to the SQLite database 
    projects = conn.execute('SELECT * FROM projects').fetchall() # SQL query to select all rows from the projects table. Results are stored in project.
    conn.close() # Closes the database connection 
    projects_list = [{'id': project['id'], 'project_name': project['project_name'], 'grade': project['grade'], 'start_date': project['start_date'], 'complete': project['complete']} for project in projects]
    # Convert the results into a list of dictionaries
    return json.dumps(projects_list)  # Return as JSON

# SHOW A SINGLE PROJECT (GET /projects/<id>)
@app.route('/projects/<id>', method='GET') # This route listens for GET requests at /projects/<id>. <id> is a dynamic URL parameter representing the project ID. 
def get_project(id): # functions that handles this GET request 
    conn = get_db_connection() # Establishes a connection with the SQLite database
    project = conn.execute('SELECT * FROM projects WHERE id = ?', (id,)).fetchone() # SQL a query to fetch the project with the specified id, results is a single row. 
    conn.close() # Closes the database connection 

    if project is None:
        raise HTTPError(404, "Project not found") # If no project with that ID is found, raise a 404 error with HTTPError.

    return json.dumps({
        'id': project['id'],
        'project_name': project['project_name'],
        'grade': project['grade'],
        'start_date': project['start_date'],
        'complete': project['complete']
    }) # If the project is found, return its details as a JSON string 

# CREATE A NEW PROJECT (POST /projects)
@app.route('/projects', method='POST') # Route that establishes a POST request at /projects, which are used to create a new project.
def add_project(): # function that handles the POST request 
    project_name = request.json.get('project_name')
    grade = request.json.get('grade')
    start_date = request.json.get('start_date')
    complete = request.json.get('complete', False)
    # retrieves the protect_name, grade, start_date and complete field from the upcoming JSON request

    if not project_name or not grade or not start_date:
        raise HTTPError(400, "Missing required fields")
        # if the project_name, grade or start_date are not retried, raise the error 

    conn = get_db_connection() # Establishes a connection with the SQLite database
    conn.execute('''
        INSERT INTO projects (project_name, grade, start_date, complete)
        VALUES (?, ?, ?, ?)
    ''', (project_name, grade, start_date, complete))
    # Insert new projects into the database 
    conn.commit() # saving changes to the database 
    conn.close() # closes the connection to the database 

    response.status = 201  # Created status
    return json.dumps({'message': 'Project added successfully'}) # return a success message in JSON format 

# UPDATE A PROJECT (PUT /projects/<id>)
@app.route('/projects/<id>', method='PUT') # Route that establishes a PUT requests at /projects/<id>, which are used to update an existing project 
def edit_project(id): # parameter (id) represents the unique identifier of the project that needs to be updated when passed via URL
    # request.json.get(): retrieves the JSON data sent in the PUT request's body. 
    project_name = request.json.get('project_name') # new name of the project 
    grade = request.json.get('grade') # new grade for the project 
    start_date = request.json.get('start_date') # new start_date for the project 
    complete = request.json.get('complete', False)

    if not project_name or not grade or not start_date:
        raise HTTPError(400, "Missing required fields") # if the project_name, grade or start_date are not retried, raise the error 

    conn = get_db_connection()
    conn.execute('''
        UPDATE projects
        SET project_name = ?, grade = ?, start_date = ?, complete = ?
        WHERE id = ?
    ''', (project_name, grade, start_date, complete, id))
    # SQL UPDATE query will modify the existing record in the projects table
    # The ? placeholders will be replaced by the new values from the request.json
    conn.commit()
    conn.close()

    return json.dumps({'message': 'Project updated successfully'}) # converts the Python object into a JSON-formatted string
                                                                    # returns message: project... 

# DELETE A PROJECT (DELETE /projects/<id>)
@app.route('/projects/<id>', method='DELETE') # Route that establishes a DELETE request at /projects/<id>, which is used to delete an existing project
                                            # DELETE request to /projects/3 would attempt to delete the project with ID 3.
def delete_project(id): # parameter (id) represents the unique identifier of the project that needs to be updated when passed via URL
    conn = get_db_connection() # Establishes a connection with the SQLite database
    conn.execute('DELETE FROM projects WHERE id = ?', (id,)) # SQL query is executed to delete the project that matches with the id
    conn.commit()
    conn.close()

    return json.dumps({'message': 'Project deleted successfully'}) # converts the Python object into a JSON-formatted string
                                                                    # returns message: project... 

# Run the application
run(app, host='localhost', port=8080, debug=True) # Start the bottle application, local host at port 8080, 
                                                    # debug=True option ensures detailed error message if something goes wrong 
