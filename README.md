# Python_Flask
from flask import Flask, render_template, request, redirect
import mysql.connector

app = Flask(__name__)

# Database connection
conn = mysql.connector.connect(
    host="localhost",
    user="root",
    password="*********",
    database="inventory_db"
)
cursor = conn.cursor()

cursor.execute("""
CREATE TABLE IF NOT EXISTS Product (
    product_id VARCHAR(10) PRIMARY KEY,
    product_name VARCHAR(100)
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS Location (
    location_id VARCHAR(10) PRIMARY KEY,
    location_name VARCHAR(100)
)
""")

cursor.execute("""
CREATE TABLE IF NOT EXISTS ProductMovement (
    movement_id INT AUTO_INCREMENT PRIMARY KEY,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    from_location VARCHAR(10),
    to_location VARCHAR(10),
    product_id VARCHAR(10),
    qty INT,
    FOREIGN KEY (product_id) REFERENCES Product(product_id),
    FOREIGN KEY (from_location) REFERENCES Location(location_id),
    FOREIGN KEY (to_location) REFERENCES Location(location_id)
)
""")

conn.commit()

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/add_product', methods=['GET', 'POST'])
def add_product():
    if request.method == 'POST':
        product_id = request.form['product_id']
        product_name = request.form['product_name']
        cursor.execute("INSERT INTO Product (product_id, product_name) VALUES (%s, %s)", (product_id, product_name))
        conn.commit()
        return redirect('/')
    return render_template('add_product.html')

@app.route('/add_location', methods=['GET', 'POST'])
def add_location():
    if request.method == 'POST':
        location_id = request.form['location_id']
        location_name = request.form['location_name']
        cursor.execute("INSERT INTO Location (location_id, location_name) VALUES (%s, %s)", (location_id, location_name))
        conn.commit()
        return redirect('/')
    return render_template('add_location.html')

@app.route('/move_product', methods=['GET', 'POST'])
def move_product():
    if request.method == 'POST':
        from_location = request.form['from_location']
        to_location = request.form['to_location']
        product_id = request.form['product_id']
        qty = request.form['qty']
        cursor.execute("INSERT INTO ProductMovement (from_location, to_location, product_id, qty) VALUES (%s, %s, %s, %s)",
                       (from_location, to_location, product_id, qty))
        conn.commit()
        return redirect('/')
    return render_template('move_product.html')

@app.route('/inventory_report')
def inventory_report():
    cursor.execute("""
    SELECT p.product_name, l.location_name, 
           SUM(CASE WHEN pm.to_location = l.location_id THEN pm.qty ELSE 0 END) -
           SUM(CASE WHEN pm.from_location = l.location_id THEN pm.qty ELSE 0 END) AS balance_qty
    FROM ProductMovement pm
    JOIN Product p ON pm.product_id = p.product_id
    JOIN Location l ON l.location_id = pm.to_location OR l.location_id = pm.from_location
    GROUP BY p.product_name, l.location_name
    """)
    report_data = cursor.fetchall()
    return render_template('report.html', report_data=report_data)

if __name__ == '__main__':
    app.run(debug=True)
