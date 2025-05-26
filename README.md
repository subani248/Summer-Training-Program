Project Report: Student Expense Management System

Project Content

This project comprises a Flask-based backend application designed for managing student expenses, primarily within a mess or shared living arrangement. The system facilitates the following core functionalities through a RESTful API and basic web page rendering:

Student Management: Registration of new students into the system.
Attendance Tracking: Recording and updating student attendance on a monthly basis.
Expense Calculation: Registration of total monthly expenses, followed by automated calculation of individual student shares based on their attendance.
User Authentication: Login mechanisms for both students (via ID and phone) and administrators (via ID and password, secured with bcrypt). JWT (JSON Web Tokens) are used for secure authentication and authorization.
Expense Visibility: Students can view their historical monthly expense records.
Basic Web Interface: Simple HTML pages are rendered for basic user interactions like registration and login.

Project Code
The project code provided is a Flask application that serves as the backend for a student expense management system. It handles student registration, attendance tracking, monthly expense calculation and recording, and provides login functionalities for both students and administrators. It also includes basic HTML templating for web pages.


from flask import Flask, request, jsonify, render_template, redirect, url_for
from flask_mysqldb import MySQL
from flask_cors import CORS
from datetime import timedelta
from flask_bcrypt import Bcrypt
from flask_jwt_extended import JWTManager, create_access_token, jwt_required, get_jwt_identity # Added jwt_required, get_jwt_identity

app = Flask(__name__)
CORS(app)

# Configure MySQL
app.config['MYSQL_HOST'] = 'localhost'
app.config['MYSQL_USER'] = 'root'
app.config['MYSQL_PASSWORD'] = 'root'
app.config['MYSQL_DB'] = 'student_expense_db1'
app.config['JWT_SECRET_KEY'] = 'messbillcalculation'
app.config['JWT_ACCESS_TOKEN_EXPIRES'] = timedelta(minutes=10)

mysql = MySQL(app)
bcrypt = Bcrypt(app)
jwt = JWTManager(app)

# 1. Register Student
@app.route('/registerstudent', methods=["POST"])
def register_student():
    data = request.json
    st_id = data.get('st_id')
    name = data.get('name')
    abranch = data.get('abranch')
    phone = data.get('phone')

    if not all([st_id, name, abranch, phone]):
        return jsonify({"message": "Missing required fields"}), 400

    cur = mysql.connection.cursor()
    try:
        cur.execute("INSERT INTO student (st_id, name, abranch, phone) VALUES (%s, %s, %s, %s)",
                    (st_id, name, abranch, phone))
        mysql.connection.commit()
    except Exception as e:
        mysql.connection.rollback() # Ensure rollback on error
        return jsonify({"message": f"Error: {str(e)}"}), 400
    finally:
        cur.close()

    return jsonify({"message": "Student registered successfully"}), 201

# 2. Record Attendance
@app.route('/attendance', methods=['POST'])
def add_attendance():
    data = request.json
    student_id = data.get('student_id')
    month = data.get('month')
    days_present = data.get('days_present')

    if not all([student_id, month, days_present is not None]): # Check for explicit None
        return jsonify({"message": "Missing required fields"}), 400

    cur = mysql.connection.cursor()
    try:
        cur.execute("""
            INSERT INTO attendance (student_id, month_name, days_present)
            VALUES (%s, %s, %s)
            ON DUPLICATE KEY UPDATE days_present = %s
        """, (student_id, month, days_present, days_present))
        mysql.connection.commit()
    except Exception as e:
        mysql.connection.rollback()
        return jsonify({"message": f"Error: {str(e)}"}), 400
    finally:
        cur.close()

    return jsonify({"message": "Attendance recorded successfully"})

# 3. Register Monthly Expense
@app.route('/billregister', methods=["POST"])
# @jwt_required() # Assuming only admin can register bills
def bill_register():
    data = request.json
    month = data.get('month')
    # Convert total_month_day and total_expense to float to ensure numerical operations
    try:
        total_month_day = float(data.get('total_month_day'))
        total_expense = float(data.get('total_expense'))
    except (ValueError, TypeError): # Handle both value and type errors for missing/bad data
        return jsonify({"message": "Invalid or missing number format for total_month_day or total_expense"}), 400

    if not month:
        return jsonify({"message": "Missing month"}), 400

    cur = mysql.connection.cursor()

    try:
        # Check if expense for the month already exists
        cur.execute("SELECT expense_id FROM expense WHERE expense_month = %s", (month,))
        existing_expense = cur.fetchone()
        if existing_expense:
            return jsonify({"message": "Monthly expense for this month already registered. Use an update endpoint if needed."}), 409 # Conflict

        cur.execute("INSERT INTO expense (expense_month, total_expense) VALUES (%s, %s)", (month, total_expense))
        expense_id = cur.lastrowid

        cur.execute("""
            SELECT s.st_id, COALESCE(a.days_present, 0)
            FROM student s
            LEFT JOIN attendance a ON s.st_id = a.student_id AND a.month_name = %s
        """, (month,))
        student_days = cur.fetchall()

        total_days_present_across_students = sum([row[1] for row in student_days])

        if total_days_present_across_students == 0:
            mysql.connection.rollback() # Rollback expense insertion
            return jsonify({"message": "No attendance recorded for any student in this month"}), 400

        for student_id, days_present in student_days:
            individual_share = (total_expense / total_month_day) * float(days_present) if days_present > 0 else 0

            # Check if student_expense for this month and student already exists
            cur.execute("SELECT student_expense_id FROM student_expense WHERE student_id = %s AND month_name = %s", (student_id, month))
            existing_student_expense = cur.fetchone()

            if existing_student_expense:
                # Update if exists
                cur.execute("""
                    UPDATE student_expense
                    SET student_month_expense = %s
                    WHERE student_id = %s AND month_name = %s
                """, (individual_share, student_id, month))
            else:
                # Insert if not exists
                cur.execute("""
                    INSERT INTO student_expense (student_id, expense_id, student_month_expense, month_name)
                    VALUES (%s, %s, %s, %s)
                """, (student_id, expense_id, individual_share, month))

        mysql.connection.commit()
        return jsonify({"message": "Monthly bill and student expenses calculated and saved"}), 200

    except Exception as e:
        mysql.connection.rollback()
        return jsonify({"message": f"Error: {str(e)}"}), 500
    finally:
        cur.close()

# 4. Student Login
@app.route('/studentlogin', methods=["POST"])
def student_login():
    data = request.json
    st_id = data.get('st_id')
    phone = data.get('phone')

    if not st_id or not phone:
        return jsonify({"message": "Missing student ID or phone"}), 400

    cur = mysql.connection.cursor()
    try:
        cur.execute("SELECT st_id, name FROM student WHERE st_id=%s AND phone=%s", (st_id, phone))
        student = cur.fetchone()

        if not student:
            return jsonify({"message": "Invalid login credentials"}), 401

        student_id, name = student
        access_token = create_access_token(identity=student_id) # Generate JWT for student

        return jsonify({
            "message": "Login successful",
            "student_id": student_id,
            "name": name,
            "access_token": access_token
        }), 200
    except Exception as e:
        return jsonify({"message": f"Error: {str(e)}"}), 500
    finally:
        cur.close()

# 5. View Student Expenses
@app.route('/studentexpense/<string:student_id>', methods=["GET"]) # Changed to string to match st_id type if it's string
@jwt_required() # Protect this endpoint
def get_student_expense(student_id):
    current_user_id = get_jwt_identity()
    if current_user_id != student_id:
        return jsonify({"message": "Unauthorized access to another student's data"}), 403

    cur = mysql.connection.cursor()
    try:
        cur.execute("SELECT name FROM student WHERE st_id=%s", (student_id,))
        student = cur.fetchone()

        if not student:
            return jsonify({"message": "Student not found"}), 404

        cur.execute("""
            SELECT month_name, student_month_expense
            FROM student_expense
            WHERE student_id=%s
            ORDER BY month_name
        """, (student_id,))
        expenses = cur.fetchall()

        history = [{"month": row[0], "amount": float(row[1])} for row in expenses]

        return jsonify({
            "student_id": student_id,
            "name": student[0],
            "expense_history": history
        }), 200
    except Exception as e:
        return jsonify({"message": f"Error: {str(e)}"}), 500
    finally:
        cur.close()

# 6. Admin login
@app.route("/adminlogin", methods=["POST"])
def admin_login_api():
    data = request.get_json()
    admin_id = data.get("admin_id")
    password = data.get("password")

    if not admin_id or not password:
        return jsonify({"error": "Missing credentials"}), 400

    cur = mysql.connection.cursor()
    try:
        cur.execute("SELECT admin_id, password FROM admin WHERE admin_id = %s", (admin_id,))
        admin = cur.fetchone()

        if not admin:
            return jsonify({"error": "Admin not found"}), 404

        db_admin_id, password_hash = admin

        if bcrypt.check_password_hash(password_hash, password):
            access_token = create_access_token(identity=db_admin_id)
            return jsonify({
                "message": "Admin login successful",
                "access_token": access_token
            }), 200
        else:
            return jsonify({"error": "Invalid password"}), 401
    except Exception as e:
        return jsonify({"message": f"Error: {str(e)}"}), 500
    finally:
        cur.close()

# Basic Web Pages (Render HTML)
@app.route('/')
def home():
    return render_template('index.html')

@app.route('/adminregisterpage')
def admin_register_page():
    return render_template('adminregister.html')

@app.route('/adminloginpage') # Renamed to avoid conflict with API endpoint
def admin_login_page():
    return render_template('admin_login.html')

@app.route('/studentloginpage') # Renamed to avoid conflict with API endpoint
def student_login_page():
    return render_template('student_login.html')

@app.route('/registerstudentpage')
def register_student_page():
    return render_template('register_student.html')

@app.route('/billregisterpage')
def bill_register_page():
    return render_template('bill_register.html')

# Run the app
if __name__ == "__main__":
    app.run(debug=True)


Key Technologies


Flask: A micro web framework for Python, used for building the RESTful API endpoints and serving web pages.
Flask-MySQLdb: Provides MySQL connectivity for Flask applications, enabling interaction with the database.
Flask-CORS: A Flask extension for handling Cross-Origin Resource Sharing (CORS), allowing requests from different origins (e.g., a frontend application running on a different port).
Flask-Bcrypt: An extension that provides bcrypt hashing utilities for Flask, used for securely hashing and verifying passwords (specifically for admin login).
Flask-JWT-Extended: Provides JWT (JSON Web Token) support for Flask. Used for creating and verifying access tokens for authentication and authorization (implemented for admin and student logins, and protecting the get_student_expense endpoint).
MySQL: A relational database management system (RDBMS) used to store all application data (student information, attendance, expenses, admin details).
HTML/CSS/JavaScript (Frontend - Implied): Although not explicitly provided in the Python code, the render_template functions indicate the use of these standard web technologies for the frontend user interface.


Description


This project implements a Student Expense Management System designed to track and manage monthly mess or other collective expenses for students. The system provides functionalities for:

Student Registration: Allows new students to be registered into the system with their ID, name, branch, and phone number.
Attendance Recording: Enables the recording or updating of the number of days a student is present in a given month, which is crucial for calculating individual shares of expenses.
Monthly Expense Registration and Calculation: An administrator can register the total monthly expense and the total number of days in that month. The system then automatically calculates each student's individual share of the total expense based on their recorded attendance for that month.
Student Login: Students can log in using their ID and phone number to access their personal expense details.
View Student Expenses: Students can view their monthly expense history, showing the amount they owe for each month. This endpoint is protected by JWT for authentication.
Admin Login: An administrator can log in using their credentials. Upon successful login, a JWT access token is generated for authenticated access to protected endpoints (e.g., billregister).
Basic Web Pages: The application also serves simple HTML pages for user interaction, allowing students and administrators to register, log in, and perform other actions through a web interface.
The system uses a MySQL database to persist all data. Security is enhanced through bcrypt for password hashing and JWT for token-based authentication.

Output


The project produces API responses in JSON format and serves HTML pages for the user interface.

Examples of JSON Outputs:
![Screenshot 2025-05-26 201355](https://github.com/user-attachments/assets/bd104d23-5be9-4361-8499-1e8c88d93d97)
![Screenshot 2025-05-26 200905](https://github.com/user-attachments/assets/0d7c5ed5-aa19-4b3a-9493-20c90f5f91f3)
![Screenshot 2025-05-26 201515](https://github.com/user-attachments/assets/40df54e6-5ecf-4a51-8c4f-34cf373c1d59)
![Screenshot 2025-05-26 202158](https://github.com/user-attachments/assets/5036f95b-33f6-47af-9451-606a12db5cac)
![Screenshot 2025-05-26 202011](https://github.com/user-attachments/assets/fbbbfa26-a335-4243-a72a-0813f756e8a5)
![Screenshot 2025-05-26 201739](https://github.com/user-attachments/assets/b65546b6-6626-46cb-9981-74597927622c)


Further Research / Future Enhancements



Robust Error Handling and Logging: Implement more detailed error logging to help debug issues in production environments. Improve client-side error feedback.
Input Validation: Implement more thorough input validation for all API endpoints to prevent common vulnerabilities like SQL injection and ensure data integrity (e.g., validating phone number format, ensuring days_present is within a valid range).
Admin Management: Add functionalities for administrators to view, update, or delete student information, attendance records, and past monthly expenses.
Password Reset Functionality: For both students (if they had passwords) and admins.
Data Visualization: Integrate a frontend library (e.g., Chart.js, D3.js) to visualize student expense history or overall monthly expenses, providing a better user experience.
Reporting Features: Generate monthly reports summarizing total expenses, student contributions, and outstanding balances.
Payment Tracking: Add a module to track payments made by students and their outstanding balances.
User Roles and Permissions (Granular): While JWT is used, further refine roles and permissions (e.g., different types of admins with different access levels).
Frontend Framework Integration: Develop a more dynamic and responsive user interface using a modern JavaScript frontend framework (e.g., React, Vue, Angular) to consume the Flask API, providing a richer user experience than simple HTML templates.
Deployment: Prepare the application for production deployment (e.g., using Gunicorn/Waitress with Nginx/Apache, Dockerizing the application).
Security Enhancements:
HTTPS: Enforce HTTPS for all communication in production.
Rate Limiting: Implement rate limiting on API endpoints to prevent brute-force attacks.
CSRF Protection: If using form submissions extensively, add CSRF protection.
Database Security: Use dedicated database users with minimal necessary privileges.
Notifications: Implement email or SMS notifications for students regarding their monthly bills or outstanding payments.
